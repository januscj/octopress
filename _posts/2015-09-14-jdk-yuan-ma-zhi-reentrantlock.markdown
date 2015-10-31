---
layout: post
title: "jdk源码阅读之ReentrantLock"
date: 2015-09-14 11:06:57 +0800
comments: true
categories: jdk源码阅读
description: java ReentrantLock
---

接着来看并发工具类库另外一个常用的工具ReentrantLock。Reentrantlock也有公平策略和非公平策略,先来看非公平策略。

``` java 

static final class NonfairSync extends ReentrantLock.Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        NonfairSync() {
        }

        final void lock() {
            if(this.compareAndSetState(0, 1)) {
                this.setExclusiveOwnerThread(Thread.currentThread());
            } else {
                this.acquire(1);
            }

        }

        protected final boolean tryAcquire(int acquires) {
            return this.nonfairTryAcquire(acquires);
        }
    }

```

非公平策略优先尝试cas state，如果没成功，则进入AQS的aquire。先来看Sync的nonfairTryAcquire。

``` java 

    final boolean nonfairTryAcquire(int acquires) {
            Thread current = Thread.currentThread();
            int c = this.getState();
            if(c == 0) {
                if(this.compareAndSetState(0, acquires)) {
                    this.setExclusiveOwnerThread(current);
                    return true;
                }
            } else if(current == this.getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if(nextc < 0) {
                    throw new Error("Maximum lock count exceeded");
                }

                this.setState(nextc);
                return true;
            }

            return false;
        }

```

这个方法首先判断是否线程持有锁，如果没有则获取锁，并把线程所有者设置为当前线程。否则，判断当前线程是否是持有锁，如果是，则state+1，返回true。这里说明ReentrantLock是可重入的。如果别的线程已经获取锁，则返回false。如果返回false，这个时候，我们再来看acquire方法:

``` java

    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    } 
```

这个时候进入acquireQueued。

``` java

 final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

从上面的代码来看，这个时候线程会被park住，至此获取锁的过程分析完毕。下面，来看释放锁的过程。首先unlock调用的是release方法

``` java

    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

```
先来看tryRelease

``` java

 protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

从上面的代码来看，当state变为0，tryRelease才会返回true，release才会unpark第二个线程节点。这个时候回到acquireQueued方法，第二节点被unpark之后。仍在for循环里，这个时候把如果tryRelease返回true，即等待队列的第二个节点获得锁，进入if分支，把头结点设置为当前节点，并直接返回。等待队列的其他节点，依次类推。

下面来说说公平策略好非公平策略的区别。跟Semaphore一样，公平策略会判断线程等待队列中是否有线程再等待，如果有则将当前线程放入等待队列，直到被被唤醒。但非公平策略是，不过当前是否有线程再等待，都尝试去获取锁，这样当前线程可能会优先获取到锁，所以是不公平的。

另外，condition也是并发场景中用的较多的工具之一。我们来看看condition是怎么做的。首先看await方法

``` java 

        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

首先，addConditionWaiter再condition队列为空的时候，构建condition队列，若condition队列不为空，则把当前线程节点添加到队尾。后面while循环里判断，当前节点是否已经已到线程同步队列。如果没有移动到线程同步队列，则park当前线程节点，至此线程则被阻塞住了，直到被唤醒，才会往下走。

``` java

    private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }
```

如果线程没有被中断，则会继续循环。再次判断当前节点是否在线程同步队列，如果在同步队列，则会结束循环。从后面代码知道，线程会在**signal的时候移动到同步队列**。接着，会调用acquireQueued判断同步队列能否获得锁，不能则会被阻塞住。否则，会继续往下走。

await的整个流程就如上面分析，下面来看看sigal的流程。

``` java

    public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
```

简单的判断当前线程是否持有锁，如果是则往下走。

``` java

        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```

从上面的代码来看，doSignal从第一个节点开始调用transferForSignal


``` java 

  final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

首先，尝试将node的state由-2更新为0，如果更新不成功，则继续循环，尝试唤醒下个节点;如果，更新成功，则将当前节点添加到线程同步队列。然后，把节点状态更新为-1，返回true，这样doSignal就做完了。

ReentrantLock其实就是实现了线程间最基本的互斥和通信机制。lock作为一种互斥机制比内置互斥机制更灵活