---
layout: post
title: "jdk源码阅读之ReentrantReadWriteLock"
date: 2015-09-28 19:14:56 +0800
comments: true
categories: jdk源码阅读
---

了解完reentrantlock，再来看看reentrantreadwritelock。在看reentrantreadwritelock源码之前，得了解它的特点。我们都知道读锁跟写锁是互斥的，多个读线程可以同时进行读操作，就是说对锁对读线程来说是共享锁。写锁是独占锁，写锁之间也是互斥的。了解这些基本特性之后，再来看源码。

``` java 

    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

```

reentrantreadwritelock是通过上面的代码初始化的，从上面来看读锁跟写锁公用同一个同步器。因为一个同步器只有一个state变量，我们知道同步器使用state来标识线程的不同状态。**那么，reentrantreadwritelock是怎么来区分读锁、写锁的，后面看过源码后知道,它是用state的高16位表示读锁，低16位表示写锁。** 

先来看读锁的获取，读锁获取首先是下面的代码

``` java

protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail.
             * 2. Otherwise, this thread is eligible for
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            int r = sharedCount(c);
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != current.getId())
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```

从上面的代码来看，首先如果写锁已被其他线程持有，则直接阻塞当前线程。如果进入if分支，分三种情况：没有读线程持有锁，则直接设置firstReader、firstReaderHoldCount；否则，如果当前线程是第一个读线程，则firstReaderHoldCount+1；否则，创建一个HolderCounter对象来记录线程对读锁的持有次数，并将该HolderCounter对象放入ThreadLocal中。其他的情况（CAS失败或readerShouldBlock返回true），则进入fullTryAcquireShared。

``` java 

final int fullTryAcquireShared(Thread current) {
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != current.getId()) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != current.getId())
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```

fullTryAcquireShared首先会判断若其他线程持有写锁，则会阻塞。若readerShouldBlock返回true，则进入if分支，如果当前线程不是第一个读线程，同时当前线程是之前没有持有读锁，则会从ThreadLocal中移除HoldCounter，最后如果count仍是-1，则直接返回-1，阻塞当前线程。接着下来的代码就是对tryAcquireShared CAS state失败的处理，逻辑类似。

下面来看读锁释放的逻辑:

``` java 

protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != current.getId())
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
```

首先，如果当前线程是第一个读线程，先判断读锁重入次数是否为1,如果为1，则直接将firstReader置null;否则，firstReaderHoldCount-1；否则，如果当前线程不是第一个读线程，则对持锁次数减1.下面的for循环是处理CAS state。

下面接着来看写锁的获取锁的过程:

``` java 

protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

首先，从代码来看，**如果是有读线程持有读锁或其他写线程持有锁，则当前写线程会阻塞**，如果是当前线程持有写锁，则将持锁次数加1。如果writerShouldBlock方法返回true或CAS state失败，则阻塞当前写线程；否则，设置当前线程为写锁持有者。

写锁的释放过程：

``` java

  protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
```

释放锁的代码很简单，因为是只有一个线程持有锁，所有是单线程的，sate的变更没有使用CAS。