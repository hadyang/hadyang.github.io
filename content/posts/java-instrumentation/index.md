---
title: Java Instrumentation
date: 2016-12-28
categories:
  - Java
type: post
tags:
  - 虚拟机
  - 字节码
---

最近在看公司的工具包的时候，发现有个很好用的功能 -- `TraceId` 和 LogBack 的 [MDC](http://logback.qos.ch/manual/mdc.html) 的功能，可以通过一个 `TraceId` 来跟踪一个请求在不同系统中的执行情况，这可以很好的提高对线上业务排查的效率。但是这项功能的实现比较复杂，利用了一个 `Instrumentation` 的 JVM 底层代理的功能，`Instrumentation`是通过JVMTI与JVM进行交互，`JVMIT`是JVM对外提供的用户接口，`JVMTI`是基于事件驱动的，JVM每执行到一定的逻辑就会调用一些事件的回调接口，这些接口可以供开发者扩展自己的逻辑。

<!--more-->

`Instrumentation` 的最大作用，就是类定义动态改变和操作。在 Java5 及其后续版本当中，开发者可以在一个普通 Java 程序（带有 main 函数的 Java 类）运行时，通过 `–javaagent` 参数指定一个特定的 jar 文件（包含 `Instrumentation` 代理）来启动 `Instrumentation` 的代理程序。

在 Java6 里面，`instrumentation` 包被赋予了更强大的功能：启动后的 instrument、本地代码（native code）instrument，以及动态改变 classpath 等等。这些改变，意味着 Java 具有了更强的动态控制、解释能力，它使得 Java 语言变得更加灵活多变。

> 本案例中的有些类路径需要写全限定名

### PreMain

在 Java5 当中，开发者可以让 Instrumentation 代理在 main 函数运行前执行。这里我们首先编写被代理的程序，在Main函数中调用TransClass.getNumber方法，返回整数 1，在代理程序中我们将返回的 1 变为 2。

#### TransClass

在这里我们通过更换class文件的字节码来实现上面的效果，首先将下面的类编译为.class文件，并且重命名为TransClass2.class，然后将`return 2`改为`return 1`，并保存。

```java
public class TransClass {
    public int getNumber() {
        return 2;
    }
}
```

#### Transformer

Transformer是我们执行字节码替换的类，通过transform方法，将TransClass类的字节码替换为TransClass2.class的字节码。

```java
public class Transformer implements ClassFileTransformer {
    public static final String classNumberReturns2 = "TransClass2.class";

    public static byte[] getBytesFromFile(String fileName) {
        try {
            // precondition
            File file = new File(fileName);
            InputStream is = new FileInputStream(file);
            long length = file.length();
            byte[] bytes = new byte[(int) length];

            // Read in the bytes
            int offset = 0;
            int numRead = 0;
            while (offset < bytes.length
                    && (numRead = is.read(bytes, offset, bytes.length - offset)) >= 0) {
                offset += numRead;
            }

            if (offset < bytes.length) {
                throw new IOException("Could not completely read file "
                        + file.getName());
            }
            is.close();
            return bytes;
        } catch (Exception e) {
            System.out.println("error occurs in _ClassTransformer!"
                    + e.getClass().getName());
            return null;
        }
    }

    public byte[] transform(ClassLoader l, String className, Class<?> c,
                            ProtectionDomain pd, byte[] b) throws IllegalClassFormatException {

        //这里要写全限定名
        if (!className.equals("TransClass")) {
            return null;
        }

        return getBytesFromFile(classNumberReturns2);
    }
}

```

#### TestMainInJar

该类即被代理程序的入口类：

```java
public class TestMainInJar {

    public static void main(String[] args) {
        System.out.println(new TransClass().getNumber());
    }
}
```

#### Premain

该类即代理类：

```java
import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;

public class Premain {
    public static void premain(String agentArgs, Instrumentation inst)
            throws ClassNotFoundException, UnmodifiableClassException {
        System.out.println("premain");
        inst.addTransformer(new Transformer());
    }
}
```

#### MANIFEST.MF

在MANIFEST.MF中添加如下内容：

```
Manifest-Version: 1.0
Main-Class: TestMainInJar
Premain-Class: Premain
```

现在在命令行中运行jar包

```
java -javaagent:TestInstrument.jar -cp TestInstrument.jar TestMainInJar
```

可以看到输出为2，说明TransClass类的字节码替换成功。

### AgentMain

在 Java6 的 Instrumentation 当中，有一个跟 premain “并驾齐驱” 的 agentmain 方法，可以在 main 函数开始运行之后再运行。

为了简单起见，我们举例简化如下：依然用类文件替换的方式，将一个返回 1 的函数替换成返回 2 的函数，Attach API 写在一个线程里面，用睡眠等待的方式，每隔半秒时间检查一次所有的 Java 虚拟机，当发现有新的虚拟机出现的时候，就调用 attach 函数，随后再按照 Attach API 文档里面所说的方式装载 Jar 文件。等到 5 秒钟的时候，attach 程序自动结束。而在 main 函数里面，程序每隔半秒钟输出一次返回值（显示出返回值从 1 变成 2）。

#### AgentMain

```java
public class AgentMain {
    public static void agentmain(String agentArgs, Instrumentation inst)
            throws ClassNotFoundException, UnmodifiableClassException,
            InterruptedException {
        inst.addTransformer(new Transformer(), true);
        inst.retransformClasses(TransClass.class);
        System.out.println("Agent Main Done");
    }
}
```

#### AttachThread

```java
// 一个运行 Attach API 的线程子类
class AttachThread extends Thread {
    private final List<VirtualMachineDescriptor> listBefore;
    private final String jar;

    AttachThread(String attachJar, List<VirtualMachineDescriptor> vms) {
        listBefore = vms;  // 记录程序启动时的 VM 集合
        jar = attachJar;
    }

    public void run() {
        VirtualMachine vm = null;
        List<VirtualMachineDescriptor> listAfter = null;
        try {
            int count = 0;
            while (true) {
                listAfter = VirtualMachine.list();
                for (VirtualMachineDescriptor vmd : listAfter) {
                    if (!listBefore.contains(vmd)) {
                        // 如果 VM 有增加，我们就认为是被监控的 VM 启动了
                        // 这时，我们开始监控这个 VM
                        vm = VirtualMachine.attach(vmd);
                        break;
                    }
                }
                Thread.sleep(500);
                count++;
                if (null != vm || count >= 10) {
                    break;
                }
            }
            vm.loadAgent(jar);
            vm.detach();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    public static void main(String[] args) throws InterruptedException {
        new AttachThread("TestInstrument.jar", VirtualMachine.list()).start();

    }
}
```

#### TestMainInJar

同时我们也需要修改 TestMainInJar 类

```java
public class TestMainInJar {
    public static void main(String[] args) throws InterruptedException {
        int i = 0;
        while (i++ < 5) {
            System.out.println(new TransClass().getNumber());
            Thread.sleep(500);
        }
    }
}
```

#### MANIFEST.MF

MANIFEST.MF修改如下：

```
Manifest-Version: 1.0
Main-Class: TestMainInJar
Premain-Class: Premain
Agent-Class: AgentMain
Can-Retransform-Classes: true
```

运行时，可以首先运行上面这个启动新线程的 main 函数

```shell
java -cp TestInstrument.jar AttachThread
```

> 在运行的时候如果jar不能运行，提示NoClassDefFoundError:com/sun/tools/attach/VirtualMachie，请确保 $JAVA_HOME/lib/tools.jar在你的CLASSPATH中；如果是Mac系统，请将 $JAVA_HOME/lib/tools.jar 复制到 /Library/Java/Extensions 中。

然后，在 5 秒钟内（仅仅简单模拟 JVM 的监控过程）运行如下命令启动测试 Jar 文件 :

```shell
java –jar TestInstrument.jar
```
如果时间掌握得不太差的话，程序首先会在屏幕上打出 1，这是改动前的类的输出，然后会打出一些 2，这个表示 agentmain 已经被 Attach API 成功附着到 JVM 上，代理程序生效了，当然，还可以看到“Agent Main Done”字样的输出。


### 参考文档

[Java SE 6 新特性: Instrumentation 新功能](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/)
