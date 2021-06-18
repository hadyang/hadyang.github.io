
有一定开发经验的人一定都听说过单例模式，可以说单例模式是一种最简单的设计模式，同时使用的也是很多的。单例指仅仅被实例化一次的类，单例模式通常被用来代替那些本质上唯一的系统组件，比如文件系统、管理器等等。

<!--more-->

虽然说单例模式理解起来很简单，但是其实现方法有很多，比如：饿汉单例模式、懒汉单例模式、静态内部类单例模式等等，下面我们就一一介绍各种实现方法，并说明他们的适用情况和优缺点。

### 饿汉单例模式

饿汉单例模式将单例对象作为私有静态类常量，这样可以保证`instance`的`new`过程是线程安全的，因为类域是在类加载阶段进行初始化，在这个阶段JVM会保证线程安全。在这种实现方式下，类加载速度较慢，但是获取单例的速度较快。

```Java
public class App {
  private static final App instance = new App();

  private App(){};

  public static App getInstance(){
    return instance;
  }
}
```

### 懒汉单例模式

懒汉单例模式将`instance`的初始化延迟到客户端第一次调用`getInstance`方法时，这就是懒的精髓。但是这种实现方式在每次获取单例对象的时候都要加锁，所以在大量线程使用单例的情况下比较费时。

> 懒汉单例模式和饿汉单例模式的优缺点正好相反。

```Java
public class App {
  private static final App instance = null;

  private App(){};

  public static synchronized App getInstance(){
    if (instance == null) {
      instance = new App();
    }
    return instance;
  }
}
```

### 静态内部类单例模式

这种方式同样使用了类加载机制，保证了初始化instance时线程安全。同时这种方式还能做到延迟加载的目的，只有第一次调用`App getInstance()`时才会初始化`instance`，静态内部类的实现方式集合了懒汉和饿汉两种方法的优点，这是一种比较推荐的单例模式的实现方法。

```Java
public class App {
  private App(){};

  public static App getInstance(){

    return AppHolder.instance;
  }

  private static class AppHolder{
    private static final App instance = new App();
  }
}
```

### 枚举单例

 这种方式是Effective Java作者Josh Bloch 提倡的方式，它不仅能避免多线程同步问题，而且还能防止反序列化重新创建新的对象，由于枚举是Java1.5之后才推出的，使用的比较少。

 在上面所有的实现方式中，反序列化都是不安全的——会出现重新创建对象的情况。通过反序列化可以将一个单例的实例对象写到磁盘，然后再读取出来，从而获取一个实例。即使构造函数是私有的，反序列化时依然可以通过特殊途径去创建类的一个新的实例，相当于调用该类的构造函数。反序列化操作提供了一个特别的函数——`readResolve()`，这个方法可以让开发人员控制对象的反序列化。例如，上述几个实现方法中如果要杜绝单例对象在反序列化的时候重新生成对象，那么必须加入以下方法：

 ```Java
 private Object readResolve() throw ObjectStreamException{
   return instance;
 }
 ```

 也就是在`readResolve()`中将`instance`对象返回，而不是默认重新生成一个新的对象。对于枚举，并不存在这个问题，因为即使反序列化也不会重新生成对象。

```Java
public enum App {
  INSTANCE

  public void doSomething(){
    ...
  }
}
```
