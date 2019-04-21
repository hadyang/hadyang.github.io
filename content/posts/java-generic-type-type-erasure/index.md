---
title: Java泛型--类型擦除
date: 2016-11-15
categories: Java
type: post
tags:
  - 泛型
  - 虚拟机
---

从Java1.5开始，Java引入泛型的概念。Java中的泛型和C语言中的模板类有些不同，主要是由于Java在最初版本并没有支持泛型，1.5之后为了实现泛型并且能与旧代码兼容，Java编译器使用了类型擦除机制。

<!--more-->

### 泛型类型

在进一步介绍类型擦除之前，我们首先来了解一下什么是泛型。**泛型是一个泛化（一般化）的类或者接口，并且拥有参数化类型**。下面的Box类是一个非泛型类：

```Java
public class Box {
    private Object object;

    public void set(Object object) { this.object = object; }
    public Object get() { return object; }
}
```

可以看到，为了支持所有的类型，Box中使用 Object 来存储变量，这样会导致大量的类型转换，比如当我们取出 Box 中的数据时，就需要将 Object 类型转换为我们需要的类型，如果转换错误还会抛出异常。

当我们使用泛型的时候，代码就会更加优雅：

```Java
public class Box<T> {
    // T stands for "Type"
    private T t;

    public void set(T t) { this.t = t; }
    public T get() { return t; }
}

public class Main{
  public static void main(String[] args) {
    Box<String> box = new Box<>();
    System.out.println(box.get());
  }
}
```

在使用泛型后，原来的 Object 都替换为 T，虽然变化不大，但是我们就不需要大量的类型转换。在 Main 函数中我们通过 `Box<String> box = new Box<>();`来创建 Box 实例。

> `Box<String>`中的 String 我们称之为实参，T 我们称之为形参。

### 泛型方法

**泛型方法拥有自己的类型参数，与声明泛型类型相似，但是这个类型参数的作用域仅限于声明该类型参数的方法内，静态和非静态的泛型方法都是允许的，同时也可以定义泛型构造方法**。

```Java
public static <T> void out(T t){
    //do something
}
```

### 类型擦除

Java的类型擦除通过以下方式来实现泛型：

  - 将所有泛型类型和泛型方法的类型参数替换为它们的边界或者 Object，因此，所产生的字节码只包含原始的类、接口和方法。
  - 为保证类型安全，编译器会在必要的时候插入类型转换的代码。
  - 为保证泛型类型的继承结构不被破环，编译器会在必要的时候插入桥接方法。

对于类型擦除的影响，我们可以通过代码进行验证：

```Java
public static void main(String[] args) throws NoSuchMethodException {
    List<Integer> integers = new ArrayList<>();
    List<String> strings = new ArrayList<>();

    System.out.println(integers.getClass() == strings.getClass());
    System.out.println(integers.getClass());
    System.out.println(Arrays.toString(integers.getClass().getTypeParameters()));
    System.out.println(Arrays.toString(strings.getClass().getTypeParameters()));
}
```

在上面的代码中，我们新建了两个泛型的 List ，通过比较它们的 Class 对象来进行验证，下面我们来看看输出结果：

```
true
class java.util.ArrayList
[E]
[E]
```

可以看到，List<Integer> 和 List<String> 的 Class 对象是相同的，都为ArrayList。而且打印的类型参数也同为 `[E]` ，所以JVM不能知道 ArrayList 中存储的类型，即类型擦除确实将泛型的信息擦除掉了。

### 泛型信息真的全部被擦除了吗？

考虑如下代码：

```Java
public class Main {

    public static void main(String[] args) throws NoSuchMethodException {
        Method method = Main.class.getMethod("say", List.class);

        Type[] genericParameterTypes = method.getGenericParameterTypes();

        for(Type genericParameterType : genericParameterTypes){
            if(genericParameterType instanceof ParameterizedType){
                ParameterizedType aType = (ParameterizedType) genericParameterType;
                Type[] parameterArgTypes = aType.getActualTypeArguments();
                for(Type parameterArgType : parameterArgTypes){
                    Class parameterArgClass = (Class) parameterArgType;
                    System.out.println("parameterArgClass = " + parameterArgClass);
                }
            }
        }
    }

    public void say(List<String> list){
            System.out.println(list.size());
    }
}
```

你猜到输出了么？

```
parameterArgClass = class java.lang.String
```

类型信息居然没有被删除，还可以通过反射获取，这是不是很颠覆？但是当你仔细想想就可以明白，这是由于 `say` 方法本身并不是泛型方法。对于 say 方法来说，它的参数类型始终为 `List<String>`。但是其方法签名为`(Ljava/util/List;)V`，即如下方法是不能重载的：

```Java
public void say(List<Integer> list){
  System.out.println(list.size());
}
```

>这个方法是不能重载的

参考文章：[Java Documentation Type Erasure](https://docs.oracle.com/javase/tutorial/java/generics/erasure.html)
