

最近在做项目的时候，有点纠结使用是`ArrayList`还是`LinkedList`，这两种List实现方式不同，应用于不同场景下，不同的遍历方式对性能和代码的可读性也有很大的影响。

<!--more-->

### For I 遍历



第一种遍历方式是最常见的，即使用变量i对List进行遍历，在这里我准备了50000条数据进行性能检测。

```java
private static void fori(List<String> array) {
        long start = System.currentTimeMillis();
        for (int i = 0; i < array.size(); i++) {
            int len = array.get(i).length();

        }
        long end = System.currentTimeMillis();

        System.out.println(array.getClass().getSimpleName() + " fori : " + (end - start));
    }
```

结果如下：

```
ArrayList fori : 4
LinkedList fori : 1898
```

可以看出`LinkedList`使用这种方式遍历的性能极差，原因也显而易见--`LinkedList`底层使用链表实现，在每次`get(i)`的时候会从链表头开始查找。



### Iterator

```java
private static void forIterator(List<String> array) {
        long start = System.currentTimeMillis();
        Iterator<String> iterator = array.iterator();
        while (iterator.hasNext()) {
            int len = iterator.next().length();
        }
        long end = System.currentTimeMillis();

        System.out.println(array.getClass().getSimpleName() + " forIterator : " + (end - start));
    }
```

结果如下：

```
ArrayList forIterator : 2
LinkedList forIterator : 2
```

可以看出在使用Iterator方式遍历时，ArrayList和LinkedList的效率差不多，原因就是由于--LinkedList的实现方式：

```
private class ListItr implements ListIterator<E> {
		//保存上一次返回的节点
        private Node<E> lastReturned;
        private Node<E> next;

		...

        public E next() {
            checkForComodification();
            if (!hasNext())
                throw new NoSuchElementException();

			//从上一次返回的节点开始查找，返回上一次返回节点的下一个节点
            lastReturned = next;
            next = next.next;
            nextIndex++;
            return lastReturned.item;
        }
 }
```

从上面的源码中可以发现，LinkedList通过保存上一次返回节点的方式来提高Iterator遍历的效率。



### For Each

ForEach方式是Java1.5开始提供的一种简洁、安全的新方式，ForEach能够遍历所有实现了Iterable的类，其实它的内部实现也是使用Iterator。

```java
public void test1() {
        List<String> array = new ArrayList<String>();
        for (String s : array) {
            System.out.println(s);
        }
}
```

上面这个方法的字节码如下：

```
public void test1();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=4, args_size=1
         0: new           #2                  // class java/util/ArrayList
         3: dup
         4: invokespecial #3                  // Method java/util/ArrayList."<init>":()V
         7: astore_1
         8: aload_1
         9: invokeinterface #4,  1            // InterfaceMethod java/util/List.iterator:()Ljava/util/Iterator;
        14: astore_2
        15: aload_2
        16: invokeinterface #5,  1            // InterfaceMethod java/util/Iterator.hasNext:()Z
        21: ifeq          44
        24: aload_2
        25: invokeinterface #6,  1            // InterfaceMethod java/util/Iterator.next:()Ljava/lang/Object;
        30: checkcast     #7                  // class java/lang/String
        33: astore_3
        34: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
        37: aload_3
        38: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        41: goto          15
        44: return
```

可以看出forEach也是使用Iterator，只不过语法更加简洁，是一个语法糖。ForEach的测试结果如下：

```
ArrayList foreach : 3
LinkedList foreach : 3
```

另外，For Each 还可以用方法替换要遍历的集合：

```java
public void test1() {
        for (String s : getArray()) {
            System.out.println(s);
        }
}
```


### 总结

在上面的分析中也可以发现每种遍历方式的使用场景，ForEach是在遍历的时候比较推荐的方法，它不容易产生Bug，具体说明见《Effective Java》第46条。但是ForEach也有缺点：不能删除数据。

> 在测试的时候发现 ArrayList add数据比 LinkedList add 数据效率更高，原因待更新
