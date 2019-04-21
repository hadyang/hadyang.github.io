---
title: 缓存算法--LRU && LFU
date: 2016-11-30
categories:
    - 算法
type: post
tags:
    - Java
    - 缓存
---

今天在熟悉公司业务的时候，看到[LFU](http://baike.baidu.com/link?url=njKbC1j8pqxN7q75i8U0db8AAE7uOKCbDWgmG1hVR6Ucm8xR5jiTAR_ACFDUiurrOMAUfIICIKtUziWtTYsvBa)算法，十分困惑。觉得和LRU算法差不多，于是仔细研究了一番，发现其实他们的思想是类似的--通过一种机制来标记缓存中的元素，当缓存满时进行“末位”淘汰。

<!--more-->

### LRU

LRU（Least Recently Used），即最近最少使用算法，LRU算法是使用最多的缓存算法，它的实现更加符合业务实际。它关注的是 **缓存数据的最近一次使用的时间** ，该时间越长越容易被淘汰掉。下面给出LRU算法的Java实现。

```
...
public class MemoryLRUCache<K, V> {

    private LinkedHashMap<K, V> cache;

    /**
     * 缓存元素的个数
     */
    private int capacity;

    private int size;

    private static final int DEFAULT_CAPACITY = 10;

    public MemoryLRUCache() {
        this(DEFAULT_CAPACITY);
    }

    public MemoryLRUCache(int capacity) {
        this.capacity = capacity;
        //设置accessOrder为true，LinkedHashMap会自动将访问过的元素放置到队列到末尾
        cache = new LinkedHashMap<K, V>(capacity, 0.75f, true);
        size = 0;
    }

    public V get(K key) {
        return cache.get(key);
    }

    public void put(K key, V value) {
        if (size == capacity) {
            Iterator<Map.Entry<K, V>> iterator = cache.entrySet().iterator();
            iterator.next();
            iterator.remove();
            --size;
        }

        cache.put(key, value);
        ++size;
    }

    public void clear() {
        cache.clear();
    }

    public int size() {
        return size;
    }

}

```

### LFU

LFU（Least Frequently Used）即最近最少使用算法，LFU关注的是 **缓存数据被使用的次数** ，但是如果有一个曾经使用很多次的缓存数据，那么它就很难被淘汰出去，即使最近很长时间没有使用它。这就是LFU算法的缺点，很可能导致 “老数据” 迟迟不能淘汰，而一直被缓存。下面给出LFU算法的Java实现。

```
...
public class MemoryLFUCache<K, V> {
    private HashMap<K, Entry<K, V>> cache;

    private int capacity;

    private static final int DEFAULT_CAPACITY = 10;

    private int size;


    public MemoryLFUCache() {
        this(DEFAULT_CAPACITY);
    }

    public MemoryLFUCache(int capacity) {
        this.capacity = capacity;
        cache = new HashMap<K, Entry<K, V>>(capacity);
        size = 0;
    }

    public V get(K key) {
        Entry<K, V> value = cache.get(key);
        value.count++;
        return value.value;
    }

    public void put(K key, V value) {
        if (size == capacity) {
            Entry<K, V> entry = Collections.min(cache.values());
            cache.remove(entry.key);
            --size;
        }

        cache.put(key, new Entry<K, V>(key, value));
        ++size;
    }

    public int size() {
        return size;
    }

    public void clear() {
        cache.clear();
    }

    private static class Entry<K, V> implements Comparable<Entry> {
        K key;
        V value;
        int count;

        public Entry(K key, V value) {
            this.key = key;
            this.value = value;
            this.count = 0;
        }

        public int compareTo(Entry o) {
            return this.count - o.count;
        }
    }
}
```