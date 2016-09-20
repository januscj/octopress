---
layout: post
title: "netty初步分析"
date: 2016-07-28 20:38:43 +0800
comments: true
categories: netty分析 
---
网络通讯是分布式架构的基础，而netty作为网络通讯的基础库，被广泛应用在各种中间件中。深入分析netty很有必要。 

从一个demo来了解netty，先看server端代码
 
``` java
 
 public class TcpServer {

    private int port;

    public TcpServer(int port) {
        this.port = port;
    }

    public void run() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() { // (4)
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new StringEncoder(CharsetUtil.UTF_8), new StringDecoder(CharsetUtil.UTF_8)).
                                    addLast(new ServerHandler(SessionManager.getManager()));
                        }
                    })
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .option(ChannelOption.SO_KEEPALIVE, true)
                    .option(ChannelOption.TCP_NODELAY, true);

            //启动推送消息线程
            Thread t = new Thread(new PushRun(new PushMessage()));
            t.start();
            ChannelFuture f = b.bind(port).sync();
            f.channel().closeFuture().sync();

        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }

    class PushRun implements Runnable {

        private PushMessage pushMessage;

        public PushRun(PushMessage pushMessage) {
            this.pushMessage = pushMessage;
        }

        public void run() {
            int i = 0;
            while (true) {
                i++;
                pushMessage.sendMsg("send msg:" + i, 34445L);
                try {
                    Thread.sleep(500L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

        }
    }

    public static void main(String[] args) throws Exception {
        int port;
        if (args.length > 0) {
            port = Integer.parseInt(args[0]);
        } else {
            port = 4199;
        }
        new TcpServer(port).run();
    }
}
```
 
我们看到先是创建了两个EventLoopGroup，eventLoopGroup是什么，先姑且理解为线程组的概念，后面的文章再详细分析netty的线程模型。然后创建ServerBootStrap，设置上面的两个eventGroup，再就是设置handler，channelInit的时候加入StringEncoder,StringDecoder,业务逻辑处理ServerHandler，再就是一些参数的设置，是否启用NODELAY等。

服务端执行b.bind()之后就开始监听客户端的连接，具体的业务逻辑的处理在ServerHandler里做。我们看下ServerHandler的代码，

``` java

public class ServerHandler extends ChannelInboundHandlerAdapter {

    private SessionManager sessionManager;

    public ServerHandler(SessionManager sessionManager) {
        super();
        this.sessionManager = sessionManager;
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println("receive from client:" + msg);
        TcpProtocol tcpProtocol = JSON.parseObject((String) msg, TcpProtocol.class);
        sessionManager.addSession(String.valueOf(tcpProtocol.getUid()), ctx);
    }

    @Override
    public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        super.channelRegistered(ctx);
        System.out.println("channel registered....");
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        super.channelActive(ctx);
        System.out.println("channel active.....");
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        super.channelInactive(ctx);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) { // (4)
        cause.printStackTrace();
        ctx.close();
    }
}
```

ServerHandler继承自ChannelInboundHandlerAdapter，覆盖了一些方法，channelRegistered、channelActive、channelRead、channelInactive，这些方法分别在channel生命周期的不同阶段调用，由此看来，netty是事件编程模型的。
 
接着看客户端的代码：

``` java

ublic class TcpClient {
    public static void main(String[] args) throws Exception {
        String host = args[0];
        int port = Integer.parseInt(args[1]);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            Bootstrap b = new Bootstrap(); // (1)
            b.group(workerGroup); // (2)
            b.channel(NioSocketChannel.class); // (3)
            b.option(ChannelOption.SO_KEEPALIVE, true);
            b.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new StringEncoder(CharsetUtil.UTF_8),
                            new StringDecoder(CharsetUtil.UTF_8)).addLast(new ClientHandler());
                }
            });

            // Start the client.
            ChannelFuture f = b.connect(host, port).sync(); // (5)
            TcpProtocol tcpProtocol = new TcpProtocol();
            tcpProtocol.setUid(34445L);
            f.channel().writeAndFlush(JSON.toJSONString(tcpProtocol));
            synchronized (TcpClient.class) {
                while (true) {
                    TcpClient.class.wait();
                }
            }
        } finally {
            workerGroup.shutdownGracefully();
        }
    }
}
``` 

客户端因为不需要处理连接的监听，所以只创建了一个EventLoopGroup。同样具体的业务处理的逻辑放在Handler里处理，ClientHandler代码如下:

``` java

package netty; /**
 * Created by Administrator on 2016/7/13.
 */

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

public class ClientHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println(msg);
    }


    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        super.channelActive(ctx);
    }
}
```

覆盖channelRead方法只是输出接受到的消息。


我们看到，不管是Server还是Client，netty都是基于事件来做业务的处理，那事件是传播开来的，事件的触发源在哪里呢？下一篇文章详细分析netty的事件传播。