---
layout: post
title: "netty事件模型分析"
date: 2016-08-02 17:49:50 +0800
comments: true
categories: netty分析
---

netty中以channel来表示每个连接，用handler来做事件的处理，pipline则是将handler组成了双向链表。事件借助pipline在链表中传播，pipline的整个结构如下：

{% img /images/netty-pipline.png netty pipline结构%}

其中的context指的是ChannelHandlerContext，它维护了channel、hander的引用。

那事件是怎么在pipline中传播的呢？是从头结点开始，还是从尾节点开始呢？又是从哪里出发的呢？

我们从读的事件着手分析，先看事件的源头是哪里。

netty里有不同的io编程模型实现，以Nio为例，对io事件的处理是在NioEventLoop里做的，事件的注册，是下面的这个方法

``` java

private static void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final NioUnsafe unsafe = ch.unsafe();
        if (!k.isValid()) {
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
            return;
        }

        try {
            int readyOps = k.readyOps();
            // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
            // to a spin loop
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
                if (!ch.isOpen()) {
                    // Connection already closed - no need to handle write.
                    return;
                }
            }
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
                ch.unsafe().forceFlush();
            }
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
                // See https://github.com/netty/netty/issues/924
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);

                unsafe.finishConnect();
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }
```

我们看到，不同的事件调用unsafe的不同方法，netty对底层socket的操作都是通过unsafe来做的。而unsafe主要由两种不同的实现NioMessageUnsafe和NioByteUnsafe。NioServerSocketChannel使用的是NioMessageUnsafe来做socket操作，NioSocketChannel使用NioByteUnsafe来做soket操作。以服务端的NioMessageUnsafe为例来看下read()方法的实现：
``` java

        @Override
        public void read() {
            assert eventLoop().inEventLoop();
            final ChannelConfig config = config();
            if (!config.isAutoRead() && !isReadPending()) {
                // ChannelConfig.setAutoRead(false) was called in the meantime
                removeReadOp();
                return;
            }

            final int maxMessagesPerRead = config.getMaxMessagesPerRead();
            final ChannelPipeline pipeline = pipeline();
            boolean closed = false;
            Throwable exception = null;
            try {
                try {
                    for (;;) {
                        int localRead = doReadMessages(readBuf);
                        if (localRead == 0) {
                            break;
                        }
                        if (localRead < 0) {
                            closed = true;
                            break;
                        }

                        // stop reading and remove op
                        if (!config.isAutoRead()) {
                            break;
                        }

                        if (readBuf.size() >= maxMessagesPerRead) {
                            break;
                        }
                    }
                } catch (Throwable t) {
                    exception = t;
                }
                setReadPending(false);
                int size = readBuf.size();
                for (int i = 0; i < size; i ++) {
                    pipeline.fireChannelRead(readBuf.get(i));
                }

                readBuf.clear();
                pipeline.fireChannelReadComplete();

                if (exception != null) {
                    if (exception instanceof IOException && !(exception instanceof PortUnreachableException)) {
                        // ServerChannel should not be closed even on IOException because it can often continue
                        // accepting incoming connections. (e.g. too many open files)
                        closed = !(AbstractNioMessageChannel.this instanceof ServerChannel);
                    }

                    pipeline.fireExceptionCaught(exception);
                }

                if (closed) {
                    if (isOpen()) {
                        close(voidPromise());
                    }
                }
            } finally {
                // Check if there is a readPending which was not processed yet.
                // This could be for two reasons:
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
                //
                // See https://github.com/netty/netty/issues/2254
                if (!config.isAutoRead() && !isReadPending()) {
                    removeReadOp();
                }
            }
        }


```

关键在读数据的时候的那个for循环，调用pipline.fireChannelRead()这个是读事件的触发源头，读完成的触发也在这个方法里触发 pipeline.fireChannelReadComplete()。

下面来分析事件是如何在pipline中传播的，先来看pipline的fireChannelRead()方法代码：

``` java

    @Override
    public ChannelPipeline fireChannelRead(Object msg) {
        head.fireChannelRead(msg);
        return this;
    }
```
由此可见读事件是从head开始触发的(**所有的inbound事件都是从head开始触发的**)，接着分析head节点的fireChannelRead,head节点是AbstractChannelHandlerContext的一个实例，具体实现在抽象类已经做了：

``` java

    @Override
    public ChannelHandlerContext fireChannelRead(final Object msg) {
        if (msg == null) {
            throw new NullPointerException("msg");
        }

        final AbstractChannelHandlerContext next = findContextInbound();
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeChannelRead(msg);
        } else {
            executor.execute(new OneTimeTask() {
                @Override
                public void run() {
                    next.invokeChannelRead(msg);
                }
            });
        }
        return this;
    }
```
findContextInbound()回从pipline的handlerContext链表里获取一个inboundHander来调用invokeChannelRead()，findContextInbound()的逻辑如下：

``` java

    private AbstractChannelHandlerContext findContextInbound() {
        AbstractChannelHandlerContext ctx = this;
        do {
            ctx = ctx.next;
        } while (!ctx.inbound);
        return ctx;
    }
```

从代码可以看出，它会从头节点开始直到找到InboundHandler节点，接着调用节点的invokeChannelRead()方法：

``` java

    private void invokeChannelRead(Object msg) {
        try {
            ((ChannelInboundHandler) handler()).channelRead(this, msg);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
     }
```
AbstractChannelHandlerContext的invokeChannelRead方法实际上是调用这个context里的handler的channelRead()方法。我们知道ChannelInboundHandlerAdapter的channelRead方法代码如下：

``` java

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws    Exception {
        ctx.fireChannelRead(msg);
    }
```
它只是接着调用AbstractChannelHandlerContext的fireChannelRead()，fireChannelRead()又会从当前节点开始寻找下一个inbound context，这样事件就传递下去了。