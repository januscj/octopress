---
layout: post
title: "jdk源码阅读之semaphore"
date: 2015-09-04 15:18:34 +0800
comments: true
categories: jdk源码阅读
---

前面看过AQS的源码后，对AQS的原理有了叫深入的理解，现在可以看看基于AQS的各种并发工具了，Semaphore是其中使用最普遍的一个。它的源码如下:

``` java

public class Semaphore implements java.io.Serializable {
    private static final long serialVersionUID = -3222578661600680210L;
    /** All mechanics via AbstractQueuedSynchronizer subclass */
    private final Sync sync;

    /**
     * Synchronization implementation for semaphore.  Uses AQS state
     * to represent permits. Subclassed into fair and nonfair
     * versions.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }

        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }

    /**
     * NonFair version
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }

    /**
     * Fair version
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }

```

跟CountDownLatch一样，Semaphore也是使用组合的方式来使用AQS，内部定义了两个AQS之类，NonfairSync和FairSync。他们的区别是什么呢？稍后详解。使用下面的构造函数，创建的Semaphore对象是使用NonfairSync方式

``` java

    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

```

NonfairSync类

``` java

  static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }
```

Sync类

``` java

   abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }

        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }

```

通过nonfairTryAcquireShared方法知道，线程在未获取到许可的情况下，线程会被方法等待队列。**这种情况下，如果有许可释放出来，另外的线程可能比等待队列的线程先获取到许可**，所以是nonfair的。我们来看看释放许可的方法

``` java

       protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
```

释放许可也是简单for循环.我们知道之前CountDownLatch是会由前面的节点唤醒后面的节点，直到整个线程队列都被唤醒。那Semaphore如果也是这样，不就有问题么？那Semaphore是怎么样的呢？从上面的tryReleaseShared代码知道，每次cas成功,tryReleaseShared都会返回true，这样会"唤醒"第二个节点，从AQS的doAcquireSharedInterruptibly方法知道，第二个节点唤醒后，仍在for循环里，在tryAcquireShared返回大于零(即有可用的许可)之前，第二个节点仍会被park住。**这里我觉得tryReleaseShared方法做得不够好，应该是在有许可的时候，再返回true。这样可以避免线程无意义的unpark、park**

现在，我们来看看nonfair和fair的区别。下面是fair类的代码

``` java

   static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
```
FairSync和NonfairSync的区别是在获取许可的地方，FairSync获取许可的时候，hasQueuedPredecessors使用这个方法判断阻塞队列中是否有线程在排队

``` java

    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }

```
FairSync的做法跟NonfairSync的区别就是，在许可用完的情况下，FairSync是把线程阻塞住，放入等待队列,而NonfairSync的做法是，不管当前是否有线程在排队，NonfairSync始终重试去获取许可，这会导致当前线程可能会优先于等待队列中的线程获得许可。