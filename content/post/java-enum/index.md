---
title: Java Enum解析
date: 2016-08-30
categories:
  - Java

tags:
    - 虚拟机
    - 字节码
---

最近在刷题的时候碰到一个Enum相关的题目，突然发现自己对Enum类的了解知之甚少，于是这篇文章就通过Java字节码来深入了解Enum类。

<!--more-->

`Enum` 的全称为 `enumeration`， 是 `JDK 1.5` 中引入的新特性，存放在 `java.lang` 包中。一般用来表示一组相同类型的常量。如性别、日期、月份、颜色等。对这些属性用常量的好处是显而易见的，不仅可以保证单例，且比较时候可以用 `==` 来替换 `equals()` 。

### 字节码

我们通过下面的示例代码来了解Enum：

```Java
enum Status {
    A,

    B,

    C
}
```

通过 javac 命令将`Status`类编译为`.class`文件，然后通过 `javap -p -v Status.class`查看字节码：

```
Classfile /E:/Project/Idea/Offer/src/Status.class
  Last modified 2016-8-30; size 798 bytes
  MD5 checksum 821f46a14a69308237f81a6f4e4503f1
  Compiled from "Status.java"
final class Status extends java.lang.Enum<Status>
  minor version: 0
  major version: 52
  flags: ACC_FINAL, ACC_SUPER, ACC_ENUM

Constant pool:
//此处省略常量池，我们主要看下面的字节码

{
  public static final Status A;
    descriptor: LStatus;
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL, ACC_ENUM

  public static final Status B;
    descriptor: LStatus;
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL, ACC_ENUM

  public static final Status C;
    descriptor: LStatus;
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL, ACC_ENUM

  private static final Status[] $VALUES;
    descriptor: [LStatus;
    flags: ACC_PRIVATE, ACC_STATIC, ACC_FINAL, ACC_SYNTHETIC

  //编译器添加的values方法
  public static Status[] values();
    descriptor: ()[LStatus;
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: getstatic     #1                  // Field $VALUES:[LStatus;
         3: invokevirtual #2                  // Method "[LStatus;".clone:()Ljava/lang/Object;
         6: checkcast     #3                  // class "[LStatus;"
         9: areturn
      LineNumberTable:
        line 4: 0

  //通过给定的枚举变量名，返回一个枚举变量
  public static Status valueOf(java.lang.String);
    descriptor: (Ljava/lang/String;)LStatus;
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: ldc           #4                  // class Status
         2: aload_0
         3: invokestatic  #5                  // Method java/lang/Enum.valueOf:(Ljava/lang/Class;Ljava/lang/String;)Ljava/lang/Enum;
         6: checkcast     #4                  // class Status
         9: areturn
      LineNumberTable:
        line 4: 0

  //枚举类的构造函数默认是私有的
  private Status();
    descriptor: (Ljava/lang/String;I)V
    flags: ACC_PRIVATE
    Code:
      stack=3, locals=3, args_size=3
         0: aload_0
         1: aload_1
         2: iload_2
         3: invokespecial #6                  // Method java/lang/Enum."<init>":(Ljava/lang/String;I)V
         6: return
      LineNumberTable:
        line 4: 0
    Signature: #30                          // ()V

  //初始化枚举变量以及$VALUES数组
  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=4, locals=0, args_size=0
         0: new           #4                  // class Status
         3: dup
         4: ldc           #7                  // String A
         6: iconst_0
         7: invokespecial #8                  // Method "<init>":(Ljava/lang/String;I)V
        10: putstatic     #9                  // Field A:LStatus;
        13: new           #4                  // class Status
        16: dup
        17: ldc           #10                 // String B
        19: iconst_1
        20: invokespecial #8                  // Method "<init>":(Ljava/lang/String;I)V
        23: putstatic     #11                 // Field B:LStatus;
        26: new           #4                  // class Status
        29: dup
        30: ldc           #12                 // String C
        32: iconst_2
        33: invokespecial #8                  // Method "<init>":(Ljava/lang/String;I)V
        36: putstatic     #13                 // Field C:LStatus;
        39: iconst_3
        40: anewarray     #4                  // class Status
        43: dup
        44: iconst_0
        45: getstatic     #9                  // Field A:LStatus;
        48: aastore
        49: dup
        50: iconst_1
        51: getstatic     #11                 // Field B:LStatus;
        54: aastore
        55: dup
        56: iconst_2
        57: getstatic     #13                 // Field C:LStatus;
        60: aastore
        61: putstatic     #1                  // Field $VALUES:[LStatus;
        64: return
      LineNumberTable:
        line 5: 0
        line 7: 13
        line 9: 26
        line 4: 39
}
Signature: #32                          // Ljava/lang/Enum<LStatus;>;
SourceFile: "Status.java"
```

在上面代码的 `第5行` ，我们可以发现`Status`类继承自`Enum`类，其中的泛型就是`Status`类自己。从 `第13行`开始，我们可以看到`A`，`B`，`C`三个枚举变量被定义为 `final static Status`类型。同时编译器还为我们添加了一个`$VALUES`的数组，这个数组包含`Status`中所有枚举变量。当我们调用`values()`方法时就会通过数组克隆，返回一个枚举变量的数组。

在字节码的`static`初始化块中，我们可以看到：**在类加载的时候，会将每个枚举变量新建一个`Status`并赋值**，同时也会初始化`$VALUES`数组。
