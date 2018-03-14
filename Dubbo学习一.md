---
title: Dubbo学习（一） 了解Dubbo
date: 2016-07-12 10:29:35
tags:  Dubbo
---
# 了解Dubbo

## Dubbo介绍

Dubbo是一个分布式服务框架，致力于提供高性能和透明化的RPC远程服务调用方案，以及SOA服务治理方案。其核心部分包含:

* 远程通讯: 提供对多种基于长连接的NIO框架抽象封装，包括多种线程模型，序列化，以及“请求-响应”模式的信息交换方式。

* 集群容错: 提供基于接口方法的透明远程过程调用，包括多协议支持，以及软负载均衡，失败容错，地址路由，动态配置等集群支持。

* 自动发现: 基于注册中心目录服务，使服务消费方能动态的查找服务提供方，使地址透明，使服务提供方可以平滑增加或减少机器。

### Dubbo能做什么？

* 透明化的远程方法调用，就像调用本地方法一样调用远程方法，只需简单配置，没有任何API侵入。

* 软负载均衡及容错机制，可在内网替代F5等硬件负载均衡器，降低成本，减少单点。

* 服务自动注册与发现，不再需要写死服务提供方地址，注册中心基于接口名查询服务提供者的IP地址，并且能够平滑添加或删除服务提供者。

## Dubbo 2.5.3 环境搭建

1、安装opensesame

```powershell
cd /usr/local/src/
git clone https://github.com/alibaba/opensesame.git
cd opensesame/
mvn clean install -Dmaven.test.skip
```

2、获取dubbo源码

```powershell
cd /usr/local/src/
git clone https://github.com/alibaba/dubbo.git
cd /usr/local/src/dubbo
cp -r hessian-lite/ ../
git checkout -b  dubbo-2.5.3 dubbo-2.5.3
cp -r ../hessian-lite/ ./
```

3、修改pom.xml

```xml
<modules>
    <module>hessian-lite</module>   <!-- 添加hessian-lite -->
    <module>dubbo-common</module>
    <module>dubbo-container</module>
    <module>dubbo-remoting</module>
    .........
</modules>

<properties>
    .........
    <fastjson_version>1.1.39</fastjson_version>   <!-- 修改版本为 1.1.39 -->
    .........
</properties>
```

3.1 修改 hessian-lite/pom.xml

```xml
<parent>
    <groupId>com.alibaba</groupId>
    <artifactId>dubbo-parent</artifactId>
    <version>2.5.3</version>   <!-- 修改版本为2.5.3 -->
</parent>
```

3.2 修改 dubbo-admin/pom.xml

```xml
<dependency>
     <groupId>com.alibaba.citrus</groupId>
     <artifactId>citrus-webx-all</artifactId>
     <version>3.1.6</version>
 </dependency>
```

添加velocity的依赖

```xml
<dependency>
    <groupId>org.apache.velocity</groupId>
    <artifactId>velocity</artifactId>
    <version>1.7</version>
</dependency>
```

对依赖项dubbo添加exclusion，避免引入旧spring

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>dubbo</artifactId>
    <version>${project.parent.version}</version>
    <exclusions>
        <exclusion>
            <groupId>org.springframework</groupId>
            <artifactId>spring</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

webx已有spring 3以上的依赖，因此注释掉dubbo-admin里面的spring依赖

```xml
<!--<dependency>-->
    <!--<groupId>org.springframework</groupId>-->
    <!--<artifactId>spring</artifactId>-->
<!--</dependency>-->
```

4、编译

```powershell
 mvn clean install -Dmaven.test.skip
```

## 示例

## dubbo超时和重连机制

dubbo启动时默认有重试机制和超时机制。超时机制的规则是如果在一定的时间内，provider没有返回，则认为本次调用失败，重试机制在出现调用失败时，会再次调用。如果在配置的调用次数内都失败，则认为此次请求异常，抛出异常。

如果出现超时，通常是业务处理太慢，可在服务提供方执行：jstack PID > jstack.log 分析线程都卡在哪个方法调用上，这里就是慢的原因。如果不能调优性能，请将timeout设大。

**某些业务场景下，如果不注意配置超时和重试，可能会引起一些异常**：

* **超时设置**
  dubbo消费端设置超时时间需要根据业务实际情况来设定，如果设置的时间太短，一些复杂业务需要很长时间完成，导致在设定的超时时间内无法完成正常的业务处理。这样消费端达到超时时间，那么dubbo会进行重试机制，不合理的重试在一些特殊的业务场景下可能会引发很多问题，需要合理设置接口超时时间。比如发送邮件，可能就会发出多份重复邮件，执行注册请求时，就会插入多条重复的注册数据。

  (1) 合理配置超时和重连的思路
    1.对于核心的服务中心，去除dubbo超时重试机制，并重新评估设置超时时间。
    2.业务处理代码必须放在服务端，客户端只做参数验证和服务调用，不涉及业务流程处理

  (2) dubbo超时和重连配置示例
  ```xml
  <!-- 服务调用超时设置为6秒,超时不重试-->
  <dubbo:service interface="com.provider.service.DemoService" ref="demoService"  retries="0" timeout="5000"/>
  ```

* **重连机制**

    dubbo在调用服务不成功时，默认会重试2次。Dubbo的路由机制，会把超时的请求路由到其他机器上，而不是本机尝试，所以 dubbo的重试机器也能一定程度的保证服务的质量。但是如果不合理的配置重试次数，当失败时会进行重试多次，这样在某个时间点出现性能问题，调用方再连续重复调用，系统请求变为正常值的retries倍，系统压力会大增，容易引起服务雪崩，需要根据业务情况规划好如何进行异常处理，何时进行重试。

## 参考资料

1、http://www.jianshu.com/p/6541f277f467

<!--
#### com.alibaba.dubbo.config.spring.ServiceBean

![ServiceBean类继承体系](http://i1.buimg.com/598959/a8df47696d2c4c2e.png)

 registerDisposableBeanIfNecessary

Class

RegistryService

RegistryConfig

ExtensionLoader

ProtocolConfig

Protocol

```java
@SPI("dubbo")
public interface Protocol {
    /**
     * 获取缺省端口，当用户没有配置端口时使用。
     * 
     * @return 缺省端口
     */
    int getDefaultPort();

    /**
     * 暴露远程服务：<br>
     * 1. 协议在接收请求时，应记录请求来源方地址信息：RpcContext.getContext().setRemoteAddress();<br>
     * 2. export()必须是幂等的，也就是暴露同一个URL的Invoker两次，和暴露一次没有区别。<br>
     * 3. export()传入的Invoker由框架实现并传入，协议不需要关心。<br>
     * 
     * @param <T> 服务的类型
     * @param invoker 服务的执行体
     * @return exporter 暴露服务的引用，用于取消暴露
     * @throws RpcException 当暴露服务出错时抛出，比如端口已占用
     */
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    /**
     * 引用远程服务：<br>
     * 1. 当用户调用refer()所返回的Invoker对象的invoke()方法时，协议需相应执行同URL远端export()传入的Invoker对象的invoke()方法。<br>
     * 2. refer()返回的Invoker由协议实现，协议通常需要在此Invoker中发送远程请求。<br>
     * 3. 当url中有设置check=false时，连接失败不能抛出异常，并内部自动恢复。<br>
     * 
     * @param <T> 服务的类型
     * @param type 服务的类型
     * @param url 远程服务的URL地址
     * @return invoker 服务的本地代理
     * @throws RpcException 当连接服务提供方失败时抛出
     */
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    /**
     * 释放协议：<br>
     * 1. 取消该协议所有已经暴露和引用的服务。<br>
     * 2. 释放协议所占用的所有资源，比如连接和端口。<br>
     * 3. 协议在释放后，依然能暴露和引用新的服务。<br>
     */
    void destroy();

}
```

```java
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        //export invoker
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
        //registry provider
        final Registry registry = getRegistry(originInvoker);
        final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
        registry.register(registedProviderUrl);
        // 订阅override数据
        // FIXME 提供者订阅时，会影响同一JVM即暴露服务，又引用同一服务的的场景，因为subscribed以服务名为缓存的key，导致订阅信息覆盖。
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
        //保证每次export都返回一个新的exporter实例
        return new Exporter<T>() {
            public Invoker<T> getInvoker() {
                return exporter.getInvoker();
            }
            public void unexport() {
            	try {
            		exporter.unexport();
            	} catch (Throwable t) {
                	logger.warn(t.getMessage(), t);
                }
                try {
                	registry.unregister(registedProviderUrl);
                } catch (Throwable t) {
                	logger.warn(t.getMessage(), t);
                }
                try {
                	overrideListeners.remove(overrideSubscribeUrl);
                	registry.unsubscribe(overrideSubscribeUrl, overrideSubscribeListener);
                } catch (Throwable t) {
                	logger.warn(t.getMessage(), t);
                }
            }
        };
    }
```

Transporter

ServiceConfig

BeanFactoryUtils.beansOfTypeIncludingAncestors

ServiceConfig#doExportUrlsFor1Protocol

ChannelHandlers

javassist

NettyServer

```java
public class NettyServer extends AbstractServer implements Server {
    
    private static final Logger logger = LoggerFactory.getLogger(NettyServer.class);

    private Map<String, Channel>  channels; // <ip:port, channel>

    private ServerBootstrap                 bootstrap;

    private org.jboss.netty.channel.Channel channel;

    public NettyServer(URL url, ChannelHandler handler) throws RemotingException{
        super(url, ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)));
    }

    @Override
    protected void doOpen() throws Throwable {
        NettyHelper.setNettyLoggerFactory();
        ExecutorService boss = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerBoss", true));
        ExecutorService worker = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerWorker", true));
        ChannelFactory channelFactory = new NioServerSocketChannelFactory(boss, worker, getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS));
        bootstrap = new ServerBootstrap(channelFactory);
        
        final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
        channels = nettyHandler.getChannels();
        // https://issues.jboss.org/browse/NETTY-365
        // https://issues.jboss.org/browse/NETTY-379
        // final Timer timer = new HashedWheelTimer(new NamedThreadFactory("NettyIdleTimer", true));
        bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
            public ChannelPipeline getPipeline() {
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec() ,getUrl(), NettyServer.this);
                ChannelPipeline pipeline = Channels.pipeline();
                /*int idleTimeout = getIdleTimeout();
                if (idleTimeout > 10000) {
                    pipeline.addLast("timer", new IdleStateHandler(timer, idleTimeout / 1000, 0, 0));
                }*/
                pipeline.addLast("decoder", adapter.getDecoder());
                pipeline.addLast("encoder", adapter.getEncoder());
                pipeline.addLast("handler", nettyHandler);
                return pipeline;
            }
        });
        // bind
        channel = bootstrap.bind(getBindAddress());
    }

    @Override
    protected void doClose() throws Throwable {
        try {
            if (channel != null) {
                // unbind.
                channel.close();
            }
        } catch (Throwable e) {
            logger.warn(e.getMessage(), e);
        }
        try {
            Collection<com.alibaba.dubbo.remoting.Channel> channels = getChannels();
            if (channels != null && channels.size() > 0) {
                for (com.alibaba.dubbo.remoting.Channel channel : channels) {
                    try {
                        channel.close();
                    } catch (Throwable e) {
                        logger.warn(e.getMessage(), e);
                    }
                }
            }
        } catch (Throwable e) {
            logger.warn(e.getMessage(), e);
        }
        try {
            if (bootstrap != null) { 
                // release external resource.
                bootstrap.releaseExternalResources();
            }
        } catch (Throwable e) {
            logger.warn(e.getMessage(), e);
        }
        try {
            if (channels != null) {
                channels.clear();
            }
        } catch (Throwable e) {
            logger.warn(e.getMessage(), e);
        }
    }
    
    public Collection<Channel> getChannels() {
        Collection<Channel> chs = new HashSet<Channel>();
        for (Channel channel : this.channels.values()) {
            if (channel.isConnected()) {
                chs.add(channel);
            } else {
                channels.remove(NetUtils.toAddressString(channel.getRemoteAddress()));
            }
        }
        return chs;
    }

    public Channel getChannel(InetSocketAddress remoteAddress) {
        return channels.get(NetUtils.toAddressString(remoteAddress));
    }

    public boolean isBound() {
        return channel.isBound();
    }

}
```java

```

```java

private void exportLocal(URL url) {
    if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
        URL local = URL.valueOf(url.toFullString())
                .setProtocol(Constants.LOCAL_PROTOCOL)
                .setHost(NetUtils.LOCALHOST)
                .setPort(0);
        Exporter<?> exporter = protocol.export(
                proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() +" to local registry");
    }
}

```

ProxyFactory

dubbo SPI

Protocol

InetAddress.getLocalHost().getHostAddress()


ReferenceBean


```java

doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs);

if (NetUtils.isInvalidLocalHost(host)) {
    if (registryURLs != null && registryURLs.size() > 0) {
        for (URL registryURL : registryURLs) {
            try {
                Socket socket = new Socket();【*】
                try {
                    SocketAddress addr = new InetSocketAddress(registryURL.getHost(), registryURL.getPort());
                    socket.connect(addr, 1000);
                    host = socket.getLocalAddress().getHostAddress();
                    break;
                } finally {
                    try {
                        socket.close();
                    } catch (Throwable e) {}
                }
            } catch (Exception e) {
                logger.warn(e.getMessage(), e);
            }
        }
    }
    if (NetUtils.isInvalidLocalHost(host)) {
        host = NetUtils.getLocalHost();
    }
}

private void exportLocal(URL url) {
    if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
        URL local = URL.valueOf(url.toFullString())
                .setProtocol(Constants.LOCAL_PROTOCOL)
                .setHost(NetUtils.LOCALHOST)
                .setPort(0);
        Exporter<?> exporter = protocol.export(
                proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() +" to local registry");
    }
}
``` -->