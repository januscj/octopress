---
layout: post
title: "jdk源码阅读之ThreadLocal"
date: 2015-10-18 20:40:24 +0800
comments: true
categories: jdk源码阅读
---

并发编程中，另一个常用的工具是Threadlocal.今天来一探其神秘面纱。
我们最常用的是Threadlocal的set方法，下面从set方法的源码入手:

``` java
    
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocal.ThreadLocalMap map = this.getMap(t);
        if(map != null) {
            map.set(this, value);
        } else {
            this.createMap(t, value);
        }

    }
```

从上面的代码来看，先回调用getMap判断ThreadLocalMap是否存在，getMap的代码如下:

``` java

  ThreadLocal.ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
它实际上就是Thread对象的一个成员变量。再回到set方法，如果当前Thread对象的threadLocals没有设置，则会调用createMap创建新的ThreadlocalMap。

``` java

  void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocal.ThreadLocalMap(this, firstValue);
```

将当前ThreadLocal对象和传给ThreadLocalMap。

下面来看看ThreadLocalMap的构造函数

```java
        ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
```
ThreadLocalMap实际上只是一个数组,第一个entry的放入位置是根据firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1)来计算的。ThreadLocal的threadLocalHashCode是0x61c88647累加得到的,而0x61c88647是黄金比例Math.sqrt(5) - 1左移31位得到。

再回到set方法，如果Thread的threadlocal已存在，则直接调用ThreadLocalMap的set，源码如下:

```java
private void set(ThreadLocal key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

set方法会先判断map中是否有同一个ThreadLocal，如果有就使用当前的value覆盖调原来的value。如果原来的threadlocal为null(被回收掉了)，则直接使用当前的key/value替换掉原来的threadlocal。否则，找到一个空的位置把当前的Entry填进去。

下面看看get方法的源码
```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }
```
get方法很简单就是从Thread的ThreadLocalMap中取，如果没有取到，则初始化数据到map中。

ThreadLocal的remove是直接调用ThreadLocalMap的remove方法，源码如下
```java
        private void remove(ThreadLocal key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        
```
从代码来看，如果找到则会调用Entry的clear，接着调用expungeStaleEntry移除entry。

通读源码知道，ThreadLocalMap的设计思想跟HashMap是不一样的，HashMap实用链表来解决冲突，而ThreadLocalMap实用是开放地址的算法来解决冲突，同时在set的时候会移除stale entry，来保证数组不会太慢，导致多次rehash。