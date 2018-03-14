---
title: Netty学习一
date: 2017-07-18 10:04:10
tags: Netty
---
# Netty入门

## 用Netty实现时间服务器

Netty时间服务器服务端程序如下：

```java
public class TimeServer {

    public void bind(int port) throws Exception {
        // 配置服务端的NIO线程组
        EventLoopGroup bossGroup = new NioEventLoopGroup();

        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
                    b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    .childHandler(new ChildChannelHandler());

            // 绑定并开始接受传入连接
            ChannelFuture f = b.bind(port).sync();

            // 等待服务端监听端口关闭
            f.channel().closeFuture().sync();
        } finally {
            // 优雅退出，释放线程池资源
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }

    }


    private class ChildChannelHandler extends ChannelInitializer<SocketChannel> {

        @Override
        protected void initChannel(SocketChannel socketChannel) throws Exception {
            socketChannel.pipeline().addLast(new TimeServerHandler());
        }
    }

    public static void main(String[] args) throws Exception {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                port = 8080;
            }
        }
        new TimeServer().bind(port);
    }
}

public class TimeServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

        ByteBuf buf = (ByteBuf) msg;
        byte[] req = new byte[buf.readableBytes()];
        buf.readBytes(req);
        String body = new String(req, "UTF-8");
        System.out.println("The time server receive order：" + body);
        String currentTime = "QUERY TIME ORDER".equalsIgnoreCase(body)
                ? new Date(System.currentTimeMillis()).toString()
                : "BAD ORDER";
        ByteBuf resp = Unpooled.copiedBuffer(currentTime.getBytes());
        ctx.write(resp);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        super.exceptionCaught(ctx, cause);
        ctx.close();
    }
}
```

Netty时间服务器客户端程序如下：

```java
public class TimeClient {

    public void connect(int port, String host) throws InterruptedException {
        // 配置客户端NIO线程组
        EventLoopGroup group = new NioEventLoopGroup();

        try {
            Bootstrap b = new Bootstrap();
            b.group(group).channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {

                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            socketChannel.pipeline().addLast(new TimeClientHandler());
                        }
                    });

            // 发起异步连接操作
            ChannelFuture f = b.connect(host, port).sync();

            // 等待客户端链路关闭
            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                port = 8080;
            }
        }

        new TimeClient().connect(port,"127.0.0.1");
    }
}

public class TimeClientHandler extends ChannelInboundHandlerAdapter {

    private static final Logger logger = Logger.getLogger(TimeClientHandler.class.getName());

    private final ByteBuf firstMessage;

    public TimeClientHandler(){
        byte[] req = "QUERY TIME ORDER".getBytes();
        firstMessage = Unpooled.buffer(req.length);
        firstMessage.writeBytes(req);
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(firstMessage);
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf buf = (ByteBuf) msg;
        byte[] req = new byte[buf.readableBytes()];
        buf.readBytes(req);
        String body = new String(req,"UTF-8");
        System.out.println("Now is :"+body);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```

## 认识相关类

### NioEventLoopGroup

 NioEventLoopGroup是一个处理I/O操作的多线程事件循环。

 NioEventLoopGroup是个线程组，它包含了一组NIO线程，专门用于网络事件的处理，实际上它们就是Reactor线程组。

### ChannelInboundHandler

它提供了一些方法来接收数据或Channel状态改变时被调用，下面是一些常用方法：

| 方法                                                          | 描述                                          |
|-------------------------------------------------------------|---------------------------------------------|
| channelRegistered(ChannelHandlerContext ctx)                | ChannelHandlerContext的Channel被注册到EventLoop中 |
| channelUnregistered(ChannelHandlerContext ctx)              | ChannelHandlerContext的channel从eventloop中注销  |
| channelActive(ChannelHandlerContext ctx)                    | ChannelHandlerContext的channel已被激活           |
| channelInactive(ChannelHandlerContext ctx)                  | ChannelHandlerContext的channel结束生命周期         |
| channelRead(ChannelHandlerContext ctx, Object msg)          | 当前Channel读取对端消息                             |
| channelReadComplete(ChannelHandlerContext ctx)              | 消息读取完毕                                      |
| userEventTriggered(ChannelHandlerContext ctx, Object evt)   | 一个用户事件被触发                                   |
| channelWritabilityChanged(ChannelHandlerContext ctx)        | 改变通道的可写状态，可以使用Channel.isWritable检查          |
| exceptionCaught(ChannelHandlerContext ctx, Throwable cause) | 重写父类ChannelHandler的方法，处理异常                  |

Netty提供了一个实现了ChannelInboundHandler接口并继承ChannelHandlerAdapter的类：**ChannelnboundHandlerAdapter**。它实现了ChannelInboundHandler所有的方法，作用就是处理消息并将消息转发到ChannelPipeline中的下一个channelhandler。ChannelInboundHandlerAdapter的channelRead方法处理完消息后不会自动释放消息，若想自动释放收到的消息，可以使用SimpleChannelInboundHandler。

Usually, channelRead() handler method is implemented like the following:

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    try {
        // Do something with msg
    } finally {
        ReferenceCountUtil.release(msg);
    }
}
```

### ServerBootstrap

    ServerBootstrap是一个帮助类，用于设置服务器。

    ```java

    public ServerBootstrap group(EventLoopGroup group);

    public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup);

    ```

### ByteBuf

  ByteBuf类似于JDK中的java.nio.ByteBuffer对象，不过它提供了更加强大和灵活的功能。

### Channel

网络套接字或能够进行I / O操作（如读取，写入，连接和绑定）的组件的通道。

通道为用户提供：

- 通道当前的状态(比如：打开、连接)；
- 通道配置参数
- 支持I/O操作
- ChannelPipeline处理与通道相关联的所有I / O事件和请求

在Netty中所有的I/O操作都是异步的，这意味着任何I/O调用立即返回，不能保证调用结束时
所有请求的I/O操作已经完成。相反，返回一个ChannelFuture实例，该实例将在请求的I / O操作成功，失败或取消时通知您。

Channel具有层次机构的，层次结构的语义取决于Channel所属的传输实现。

#### 定义：

```java
public interface Channel extends AttributeMap, ChannelOutboundInvoker, Comparable<Channel>
```

#### 方法：

![ChannelOutboundInvoker提供的方法](http://i1.buimg.com/598959/589db0ec08fe6c8e.png)

![Channel提供的方法](http://i1.buimg.com/598959/cdbdff36ff97b90d.png)

### ChannelPipeline

一个ChannelHandler列表，用于处理或拦截Channel的入站事件和出站操作。 ChannelPipeline实现了拦截过滤器模式的高级形式，使用户能够完全控制事件的处理方式以及流水线中的ChannelHandler如何相互交互。

每个Channel都有自己的管道，并在新Channel被创建时会自动创建其管道。

#### How an event flows in a pipeline

I/O事件由ChannelInboundHandler或ChannelOutboundHandler处理，并通过调用ChannelHandlerContext中定义的事件传播方法（如ChannelHandlerContext.fireChannelRead（Object））和ChannelHandlerContext.write（Object）来转发到最接近的处理程序。

```java
                                    I/O Request  via Channel or
                                          ChannelHandlerContext
    +---------------------------------------------------+---------------+
    |                           ChannelPipeline         |               |
    |                                                  \|/              |
    |    +---------------------+            +-----------+----------+    |
    |    | Inbound Handler  N  |            | Outbound Handler  1  |    |
    |    +----------+----------+            +-----------+----------+    |
    |              /|\                                  |               |
    |               |                                  \|/              |
    |    +----------+----------+            +-----------+----------+    |
    |    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
    |    +----------+----------+            +-----------+----------+    |
    |              /|\                                  .               |
    |               .                                   .               |
    | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
    |        [ method call]                       [method call]         |
    |               .                                   .               |
    |               .                                  \|/              |
    |    +----------+----------+            +-----------+----------+    |
    |    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
    |    +----------+----------+            +-----------+----------+    |
    |              /|\                                  |               |
    |               |                                  \|/              |
    |    +----------+----------+            +-----------+----------+    |
    |    | Inbound Handler  1  |            | Outbound Handler  M  |    |
    |    +----------+----------+            +-----------+----------+    |
    |              /|\                                  |               |
    +---------------+-----------------------------------+---------------+
                    |                                  \|/
    +---------------+-----------------------------------+---------------+
    |               |                                   |               |
    |       [ Socket.read() ]                    [ Socket.write() ]     |
    |                                                                   |
    |  Netty Internal I/O Threads (Transport Implementation)            |
    +-------------------------------------------------------------------+
```

入站事件由自下而上方向的入站处理程序处理，如图所示左侧所示。 入站处理程序通常处理由图底部的I/O线程生成的入站数据。入站数据通常通过实际的输入操作（如SocketChannel.read（ByteBuffer））从远程对等体读取。 如果入站事件超出了顶部入站处理程序，则会以静默方式丢弃，如果需要您注意，则记录该事件。

出站事件由自上而下方向的出站处理程序处理，如图的右侧所示。 出站处理程序通常生成或转换出站流量，如写入请求。 如果出站事件超出了底部出站处理程序，则它将由与通道相关联的I/O线程处理。 I/O线程通常执行实际的输出操作，如SocketChannel.write（ByteBuffer）。

#### Forwarding an event to the next handler

处理程序必须调用ChannelHandlerContext中的事件传播方法将事件转发到其下一个处理程序。有如下方法：

- Inbound event propagation methods:
    * ChannelHandlerContext.fireChannelRegistered()
    * ChannelHandlerContext.fireChannelActive()
    * ChannelHandlerContext.fireChannelRead(Object)
    * ChannelHandlerContext.fireChannelReadComplete()
    * ChannelHandlerContext.fireExceptionCaught(Throwable)
    * ChannelHandlerContext.fireUserEventTriggered(Object)
    * ChannelHandlerContext.fireChannelWritabilityChanged()
    * ChannelHandlerContext.fireChannelInactive()
    * ChannelHandlerContext.fireChannelUnregistered()

- Outbound event propagation methods:
    * ChannelHandlerContext.bind(SocketAddress, ChannelPromise)
    * ChannelHandlerContext.connect(SocketAddress, SocketAddress, ChannelPromise)
    * ChannelHandlerContext.write(Object, ChannelPromise)
    * ChannelHandlerContext.flush()
    * ChannelHandlerContext.read()
    * ChannelHandlerContext.disconnect(ChannelPromise)
    * ChannelHandlerContext.close(ChannelPromise)
    * ChannelHandlerContext.deregister(ChannelPromise)

#### Building a pipeline

A user is supposed to have one or more ChannelHandlers in a pipeline to receive I/O events (e.g. read) and to request I/O operations (e.g. write and close). For example, a typical server will have the following handlers in each channel's pipeline, but your mileage may vary depending on the complexity and characteristics of the protocol and business logic:

* Protocol Decoder - translates binary data (e.g. ByteBuf) into a Java object.
* Protocol Encoder - translates a Java object into binary data.
* Business Logic Handler - performs the actual business logic (e.g. database access).

and it could be represented as shown in the following example:

```java
static final EventExecutorGroup group = new DefaultEventExecutorGroup(16);
...

ChannelPipeline pipeline = ch.pipeline();

pipeline.addLast("decoder", new MyProtocolDecoder());
pipeline.addLast("encoder", new MyProtocolEncoder());

// Tell the pipeline to run MyBusinessLogicHandler's event handler methods
// in a different thread than an I/O thread so that the I/O thread is not blocked by
// a time-consuming task.
// If your business logic is fully asynchronous or finished very quickly, you don't
// need to specify a group.
pipeline.addLast(group, "handler", new MyBusinessLogicHandler());
```

#### Thread safety

A ChannelHandler can be added or removed at any time because a ChannelPipeline is thread safe

### ChannelInitializer

一个特殊的ChannelInboundHandler，它提供了一个简单的方法来初始化一个Channel，一旦它注册到它的EventLoop。

实现通常用于Bootstrap.handler（ChannelHandler），ServerBootstrap.handler（ChannelHandler）和ServerBootstrap.childHandler（ChannelHandler）的上下文中，以设置通道的ChannelPipeline。

```java
public class MyChannelInitializer extends ChannelInitializer {
    public void initChannel(Channel channel) {
        channel.pipeline().addLast("myHandler", new MyHandler());
    }
}

ServerBootstrap bootstrap = ...;
...
bootstrap.childHandler(new MyChannelInitializer());
...
```

### ChannelOption

ChannelOption允许以类型安全的方式配置ChannelConfig。 支持哪个ChannelOption取决于ChannelConfig的实际实现，并且可能取决于它所属的传输的性质。

## Dealing with a Stream-based Transport

在基于流的传输（如TCP/IP）中，接收的数据被存储到套接字接收缓冲器中。不幸的是，基于流的传输的缓冲区不是数据包的队列，而是字节队列。因此，您无法保证您所读取的内容正是您远程对等人写的内容。

因此，无论服务器端或客户端如何，接收部分都应将接收到的数据进行碎片整理到一个或多个有意义的框架中，这些框架可以被应用程序逻辑。

应用协议

TCP以流的方式进行数据传输，上层的应用协议为了对消息进行区分，往往采用如下4种方式。

（1）消息长度固定，累计读取到长度总和为定长LEN的报文后，就认为读取到了一个完整的消息；
将计数器置位，重新开始读取下一个数据报；
（2）将回车换行符作为消息结束符，例如FTP协议，这种方式在文本协议中应用比较广泛；
（3）将特殊的分隔符作为消息的结束标志，回车换行符就是一种特殊的结束分隔符；
（4）通过在消息头中定义长度字段来标识消息的总长度。

Netty对上面4种应用做了统一的抽象，提供了4种解码器来解决对应的问题，使用起来非常方便。有了这些解码器，用户不需要自己对读取的报文进行人工解码，也不需要考虑TCP的粘包和拆包。

- ByteToMessageDecoder

- ReplayingDecoder

## Speaking in POJO instead of ByteBuf
