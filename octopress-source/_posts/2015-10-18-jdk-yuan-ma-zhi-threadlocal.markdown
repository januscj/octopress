---
layout: post
title: "jdk-yuan-ma-zhi-threadlocal"
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

将当前ThreadLocal和当前值传给了ThreadLocalMap的构造函数。