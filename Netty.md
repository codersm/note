---
title: Netty
date: 2017-12-22 16:06:50
tags: Netty
---

TCP粘包和拆包
    Netty半包解码器 *Decoder

TCP以流的方式进行数据传输，上层的应用协议为了对消息进行区分，往往采用如下4种方式：

* 消息长度固定，累计读取到长度总和为定长LEN的报文后，就认为读取到了一个完整的消息；将计数器置位，重新开始读取下一个数据报；

* 将回车换行符作为消息结束符，例如FTP协议，这种方式在文本协议中应用比较广泛；

* 将特殊的分隔符作为消息的结束标志，回车换行符就是一种特殊的结束分隔符；

* 通过在消息头中定义长度字段来标识消息的总长度。

### 编解码技术

Java序列化的目的主要有两个：

1、网络传输

2、对象持久化

Java对象编解码技术

当进行远程跨进程服务调用时，需要把被传输的Java对象编码为字节数组或者ByteBuffer对象。而当远程服务读取到ByteBuffer对象或者字节数组时，需要将其解码为发送时的Java对象。


Java序列化的缺点

Java序列化从JDK 1.1版本就已经提供，它不需要添加额外的类库，只需实现java.io.Serializable并生成序列ID即可，因此，它从诞生之初就得到了广泛的引用。


评判一个编解码框架的优劣时，往往会考虑以下几个因素：

* 是否支持跨语言，支持的语言种类是否丰富；

* 编码后的码流大小；

* 编解码的性能；

* 类库是否小巧，API使用是否方便；

* 使用者需要手工开发的工作量和难度。

在同等情况下，编码后的字节数组越大，存储的时候就越占空间，存储的硬件成本就越高，并且在网络传输时更占宽带，导致系统的吞吐量降低。Java序列化后的码流偏大也一直
被业界所诟病，导致它的应用范围受到了很大限制。


#### 业界主流的编解码框架

##### Google的Protobuf介绍

##### Facebook的Thrift介绍

### WebSocket

### 私有协议开发

只要是能够用于跨进程、跨主机数据交换的非标准协议，都可以称为私有协议。


ServerBootstrap


![Netty服务端创建时序图](http://upload-images.jianshu.io/upload_images/2889214-6060d62e795cebeb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1、创建ServerBootstrap实例。ServerBootstrap是Netty服务端的启动辅助类，它提供了一系列的方法用于设置服务端启动相关的参数。底层通过门面模式对各种能力进行抽象和封装，尽量不需要用户跟过多的底层API打交道，以降低用户的开发难度。

EventLoop的职责是处理所有注册到本线程多路复用器Selector上的Channel，Selector的轮询操作由绑定的EventLoop线程run方法驱动，在一个循环体内循环执行。

EventLoop的职责不仅仅是处理网络I／O事件，用户自定义的Task和定时任务Task也统一由EventLoop负责处理，这样线程模型就实现统一。

Netty的ServerBoostrap方法提供了channel方法用于指定服务端Channel的类型。

ChannelPipeline并不是NIO服务端必需的，它本质就是一个负责处理网络事件的职责链，负责管理和执行ChannelHandler。网络事件以事件流的形式在ChannelPipeline中流转，由ChannelPipeline根据ChannelHandler的执行策路调度ChannelHandler的执行。

NioEventLoopGroup实际就是Reactor线程池，负责调度和执行客户端的接入、网络都写事件的处理、用户自定义任务和定时任务的执行。









用户可以为启动辅助类和其父类分别指定Handler。两类Handler的用途不同：子类钟Handler是NioServerSocketChannel对应的ChannelPipeline的Handler；父类中的Handler是客户端新接入的连接SocketChannel对应的ChannelPipeline的Handler。**两者的区别**：


本质区别就是：ServerBootstrap中的handler是NioServerSocketChannel使用的，所有连接该监听端口的客户端都会执行它；父类AbstractBootstrap中的Handler是个工厂类，它为了每个新接入的客户端都会创建一个新的Handler。


**Netty客户端创建流程分析**：

1、用户线程创建Bootstrap实例，通过API设置创建客户端相关的参数，异步发起客户端连接。

2、创建处理客户端连接、I／O读写的Reactor线程组NioEventLoopGroup。可以通过构造函数指定I／O线程的个数，默认为CPU内核数的2倍。

3、通过Bootstrap的ChannelFactory和用户指定的Channnel类型创建用于客户端连接的NioSocketChannnel，它的功能类似于JDK NIO类库提供的SocketChannel；

4、创建默尔的Channel Handler Pipeline，用于调度和执行网络事件；

5、异步发起TCP连接，判断连接是否成功。如果成功，则直接将NioSocketChannel注册到多路复用器上，监听读操作位，用于数据报读取和消息发送；如果没有立即连接成功，则注册连接监听位到多路复用器，等待连接结果；

6、注册对应的网络监听状态到多路复用器；

7、由多路复用器在I/O现场中轮训各Channel，处理连接结果；

8、如果连接成功，设置Future结果，发送连接成功事件，触发ChannelPipeline执行；

9、由ChannelPipeline调度执行系统和用户的ChannelHandler，执行业务逻辑。



设置Handler接口：Bootstrap为了简化Handler的编排，提供了ChannelInitiallizer，它继承了ChannelHanderAdapter，当TCP链路注册成功之后，调用initChannel接口，用于设置用户ChannelHandler。

### ByteBuf和相关辅助类

当进行网络传输的时候，往往需要使用到缓冲区，常用的缓冲区就是JDK NIO类库提供的java.nio.Buffer.它的实现类如下图：

对于NIO编程而言，我们主要使用的是ByteBuffer。ByteBuffer也有其局限性：

![1519872123395.jpg](http://upload-images.jianshu.io/upload_images/2889214-24d2c9c487ae569c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ByteBuf的基本功能应该与JDK的ByteBuffer一致，提供以下几类基本功能。

- 7种Java基础类型、byte数组、ByteBuffer（ByteBuf）等的读写；

- 缓冲区自身的copy和slice等；

- 设置网络字节序；

- 构造缓冲区实例；

- 操作位置指针等方法。

由于JDK的ByteBuffer已经提供了这些基础能力的实现，因此，Netty ByteBuf的实现可以有两种策略：

- 参考JDK ByteBuffer的实现，增加额外的功能，解决原ByteBuffer的缺点；

- 聚合JDK ByteBuffer，通过Facade模式对其进行包装，可以减少自身的代码量，降低实现成本。


ByteBuf通过两个位置指针来协助缓冲区的读写操作，读操作使用readerIndex，写操作使用writerIndex。用于写操作不修改readerIndex指针，这极大地简化了缓冲区的读写操作，避免了由于遗漏或者不熟悉flip()操作导致的功能异常。

ByteBuf如何实现动态扩展的

通常情况下，当我们对ByteBuffer进行put操作的时候，如果缓冲区剩余可写空间不够，就会发生BufferOverFlowException异常。为了避免发生这个问题，通常在进行put操作的时候对剩余可用空间进行校验。如果剩余空间不足，需要重新创建一个新的ByteBuffer，并将之前的ByteBuffer复制到新创建的ByteBuffer中，最后释放老的ByteBuffer。


#### ByteBuf的功能介绍

1、顺序读操作（read）

2、顺序写操作

3、readerIndex和writerIndex

4、**Discardable bytes**

相比与其他的Java对象，缓冲区的分配和释放是个耗时的操作，因此，我们需要尽量重用它们。由于缓冲区的动态扩张需要进行字节数组的复制，它是个耗时的操作，因此，为了最大程序地提升性能，往往需要尽最大努力提升缓冲区的重用率。

**调用discardReadBtes会发生字节数组的内存复制，所以，频繁调用将会导致性能下降，因此在调用它之前要确认你确实需要这样做，例如牺牲性能换取更多的可用内存。**


5、ReadAble Bytes和Writeable bytes

6、Clear操作

用来操作readerIndex和writeIndex，将它们还原为初始分配值。

7、Mark和reset

8、查找操作

9、Derived buffers

10、转换成标准的ByteBuffer

11、随机读写

### Channel和Unsafe

类似于NIO的Channle，Netty提供了自己的Channel和其子类实现，用于异步I／O操作和其他相关的操作。Unsafe是个内部接口，聚合在Channel中协助进行网络读写相关的操作，因为它设计初衷就是Channel的内部辅助类，不应该被Netty框架的上层使用者调用，所以被命名为Unsafe。

io.neety.channel.Channel是Netty网络操作抽象类，它聚合 了一组功能，包括但不限于网络的读、写，客户端发起连接，主动关闭连接，链路关闭，获取通信双方的网络地址等。它也包含了Netty框架相关的一些功能，包括获取该Channel的EventLoop，获取缓冲分配器ByteBufAllocator和pipeline等。


功能介绍

1、网络I／O操作


#### ChannelPipeline和ChannelHandler

Netty的ChannelPipeline和ChannelHandler机制类似于Servlet和Filter过滤器，这类拦截器实际上是职责链模式的一种变形，主要是为了方便事件的拦截和用户业务逻辑的定制。

Netty的Channel过滤器实现原理与Servlet Filter机制一致，它将Channel的数据管道抽象为ChannelPipeline，消息在ChannelPipeline中流动和传递。ChannelPipeline持有I／O事件拦截器ChannelHandler的链表，由ChannelHandler对I／O事件进行拦截和处理，可以方便地通过新增和删除ChannelHandler来实现不同的业务逻辑定制，不需要对已有的ChannnelHandler进行修改，能够实现对修改封闭和对扩展的支持。

#### ChannelPipeline

ChannelPipeline是ChannelHandler的容器，它负责ChannelHandler的管理和事件拦截与调度。

![ChannelPipeline对事件流的拦截和处理流程](http://upload-images.jianshu.io/upload_images/2889214-18a55e75124affab.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


Netty中的事件分为inbound事件和outbound事件。inbound事件通常由I／O线程触发，例如TCP链路建立事件、链路关闭事件、读事件、异常通知事件等。

Outbound事件通常是由用户主动发起的网络I／O操作，例如用户发起的连续操作、绑定操作、消息发送等操作，它对应上图的右半部分。

#### 自定义拦截器

ChannelPipeline通过ChannelHandler接口来实现事件的拦截和处理，由于ChannnelHandler中的事件种类繁多，不同的ChannelHandler可能只需要关心其中的某一个或者几个事件，所以，通常ChannelHandler只需要继承ChannelHandlerAdapter类覆盖自己关心的方法即可。

> Netty中的事件分为inbound事件和outbound事件。inbound事件通常由I／O线程触发，例如TCP链路建立事件、链路关闭事件、读事件、异常通知事件等。outbound事件通常是由用户主动发起的网络I／O操作，例如用户发起的连接操作、绑定操作、消息发送等操作。

ChannelPipeline的主要特性

ChannelPipeline支持运行时动态的添加或删除ChannelHandler，在某些场景下这个特性非常实用。ChannelPipeline是线程安全的，这意味着N个业务线程可以并发地操作ChannelPipeline而不存在多线程并发问题。但是，ChannelHandler却不是线程安全的，这意味着尽管ChannelPipeline是线程安全的，但是用户仍然需要自己保证ChannelHandler的线程安全。




#### 构建pipeline

事实上，用户不需要自己创建pipeline，因为使用ServerBootstrap或者Bootstrap启动服务端或者客户端时，Netty会为每个Channnel连接创建一个独立的pipeline。对于使用者而言，只需要将自定义的拦截器加入到pipeline中即可。

```java
pipeline = ch.pipeline();
pipeline.addLast("decoder",new MyProtocolDecoder());
pipeline.addLast("encoder",new MyProtocolEncoder());
```

ChannelPipeline支持运行态的添加或者删除ChannelHandle，在某些场景下这个特征非常实用。


#### ChannelHandler功能

ChannelHandler类似于Servlet的Filter过滤器，负责对I／O事件或者I／O操作进行拦截和处理，它可以选择性地拦截和处理，它可以选择地拦截和处理自己感兴趣的事件，也可以透传和终止事件的传递。


ByteToMessageDecoder并没有考虑TCP粘包和组包等场景，读半包需要用户解码器自己负责处理。



## Netty的线程模型

### Reactor单线程模型

Reactor单线程模型，是指所有的I／O操作都在同一个NIO线程上面完成。NIO线程的职责如下：

* 作为NIO服务端，接收客户端的TCP连接；

* 作为NIO客户端，向服务端发起TCP连接；

* 读取通信对端的请求或者应答消息；

* 向通信对端发送消息请求或者应答消息；

### Reactor多线程模型

Reactor多线程模型与单线程模型最大的区别就是有一组NIO线程来处理I/O操作，它的原理如下：

### 主从Reactor多线程模型


Netty的线程模型并不是一成不变的，它实际取决于用户的启动参数配置。通过设置不同的启动参数，Netty可以同时支持Reactor单线程模型、多线程模型和主从Reactor多线程模型。

## Future和Promise

ChannelFutrue超时并不代表I/O超时。

### Promise功能介绍

Promise是可写的Future，Futrue自身并没有写操作相关的接口，Netty通过Promise对Futrue进行扩展，用于设置I／O操作的结果。

Netty发起I／O操作的时候，会创建一个新的Promise对象。当I／O操作发生异常或者完成时，设置Promise的结果。


## Netty架构剖析




