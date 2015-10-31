---
layout: post
title: "AQS探究"
date: 2015-08-13 
comments: true
categories: 
---

总所周知，java concurrent包的工具类是构建在AbstractQueuedSynchronizer类上的基础上的，而这个类是Doug Lea大神基于CHL队列实现的同步器。这个强大的同步器是怎样实现的呢？我们来一探究竟。

因为AQS的代码比较难以理解，我们从concurrent包下的并发工具类着手开始研究。从最简单的CountDownLatch开始，首先看它的源码

``` java

   public class CountDownLatch {
    /**
     * Synchronization control For CountDownLatch.
     * Uses AQS state to represent count.
     */
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
```

CountDownlatch类定义了一个Sync类继承自AQS，实现的了AQS的tryAcquireShared和tryReleaseShared方法，share顾名思义是共享锁。首先从await方法入手：

``` java 

    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```

await方法调用的AQS的acquireSharedInterruptibly

``` java

    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

从这个方法看，await方法是可中断的，如果当前线程被中断，则直接向上抛InterruptedException。如果正常执行，则会调用tryAcquireShared方法，这个是在之类中实现的。现在回到CountDownLatch，看tryAcquireShared的实现：

``` java

   protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
```

很简单，如果state为0则返回1，否则，返回-1。state是构造函数里传进来的。我们都知道使用CountDownlatch时传进来的数字表示并发执行的线程数，由此联想state就是持有锁的线程数。从acquireSharedInterruptibly方法可以看到，当前state!=0，即并发任务线程还没执行完时，会进入doAcquireSharedInterruptibly:

``` java

private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

首先看addWaiter方法

``` java

 private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }

``` 
也就是说在pred为null的时候会初始化队列

``` java 

    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

从上面代码看初始化之后的队列是这样的：

{% img /images/queue.jpg  线程队列初始化%}  

head只是指向一个空节点，这一点对于理解后面的代码很重要，再回到doAcquireSharedInterruptibly,p的前继节点就是head,所以会进入下面的if分支（**至于为什么有这个if判断后面再详解**），对于CountDownLatch，在并发任务还没完成的时候，tryAcquireShared返回值为-1，所以就不会往下走。直接进入shouldParkAfterFailedAcquire

``` java

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

pred就是head，初始化之后waitStatus=0，进入else分支，故head的waitStatus被更新为SIGNAL，再回到doAcquireSharedInterruptibly,这个时候如果线程没有被中断，那么会接着循环，再次进入shouldParkAfterFailedAcquire，这个是进入第一个if分支，返回true，那么就是进入parkAndCheckInterrupt，将当前线程阻塞住,这就是CountDownlatch调用await后阻塞住的原因。
>从上面的分析可以知道，对于CountDownlatch，在并发任务还没结束的时候，如果另外一个**线程B**再调用await方法，那么当前线程会放到等待队列的最后面。第一个节点park住的时候，它的waitStatus还是0，所以这次，shouldParkAfterFailedAcquire会把第一个节点的waitStatus设置为SIGNAL，同时下次循环会park住**线程B**

**AQS获取锁**的过程已经了解清楚了，下面来看看**AQS释放锁的过程**。还是从CountDownLatch的countdown()方法入手。countdown()是直接调用AQS的releaseShared

``` java 

    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

从代码看，tryReleaseShared是在子类中实现的:

``` java

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```

从tryReleaseShared方法代码来看，只有等所有并发任务执行完，tryReleaseShared才会返回true，才会执行doReleaseShared

``` java 

private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

如果head节点的waitStatus为SIGNAL，则先把head节点的status设置为0，然后进入unparkSuccessor

``` java

private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

在通常情况下，s!=null并且s.waitStatus为SIGNAL，所以head节点的后继节点会被唤醒。就是说每次调用releaseShared只会唤醒等待队列中head节点之后的线程。
>分析到这里，试想这个使用CountDownLatch场景，线程A和线程B，都调用await方法等待线程B、线程C完成任务。那么在线程B、线>程C完成任务的时候，主线程调用releaseShared进入doReleaseShared唤醒head节点之后的节点线程。因为原来的线程是在doAcquireSharedInterruptibly里的for循环最后park住，现在仍然回到该处，继续下次循环。这个时候会进入上面提到的if分支，进入setHeadAndPropagate。
``` java 

    private void setHeadAndPropagate(Node node, long propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        if (propagate > 0 || h == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```
 >从代码来看，setHeadAndPropagate就是把当前当前head节点remove掉，设置当前线程节点为head节点（也就是第二个节点）。同时在共享锁的模式下，会调用doReleaseShared，唤醒当前节点的后继节点,这就是propagate的概念。同理后续节点又会再唤醒它后面的节点，直到整个队列都被唤醒。
 
 至此，已基本了解AQS的工作原理的，为了加深印象，我们来看下面的线程队列的变化过程图。

 线程thread1调用acquireSharedInterruptibly之后，线程队列如下图，同时thread1被park住

 {% img /images/queue-init.png %}  

另外一个线程thead2再次调用acquireSharedInterruptibly之后，线程队列如下图，同时thread2被park住

 {% img /images/queue-two.png %}  

这个时候，另一个线程触发releaseShared，线程队列如下图，同时thread1被unpark

 {% img /images/queue-three.png %}  

thread1被unpark之后，会进入setHeadAndPropagate，setHead之后，线程队列如下图

 {% img /images/queue-four.png %}  

thread1调用doReleaseShared唤醒thread2后，线程队列如下图

 {% img /images/queue-five.png %}

thread2 进入setHeadAndPropagate，setHead之后，线程队列如下图

 {% img /images/queue-six.png %}


