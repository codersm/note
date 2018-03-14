---
title: Dubbo学习三——服务暴露
date: 2017-07-12 20:52:59
tags: Dubbo
---
# 服务发布

## 使用示例
## 原理

### 服务发布过程

![服务提供者暴露一个服务的详细过程](http://i1.buimg.com/598959/809bd38dbe148369.jpg)

首先ServiceConfig类拿到对外提供服务的实际类ref(如：HelloWorldImpl),然后通过ProxyFactory类的getInvoker方法使用ref生成一个AbstractProxyInvoker实例，到这一步就完成具体服务到Invoker的转化。接下来就是Invoker转换到Exporter的过程。

### 源码分析

* 本地暴露
* 远程暴露

spring classpathxmlapplicationcontext start

BeanFactoryUtils

Wrapper.getWrapper(interfaceClass).getMethodNames();


## 源码分析

### ServiceConfig

```java
private void doExportUrls() {

    List<URL> registryURLs = loadRegistries(true);
    // 支持服务多协议的发布
    for (ProtocolConfig protocolConfig : protocols) {
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}


private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    
    URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);

    if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
            .hasExtension(url.getProtocol())) {
        url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
    }

    String scope = url.getParameter(Constants.SCOPE_KEY);
    //配置为none不暴露
    if (! Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {

        //配置不是remote的情况下做本地暴露 (配置为remote，则表示只暴露远程服务)
        if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
            exportLocal(url);
        }
        //如果配置不是local则暴露为远程服务.(配置为local，则表示只暴露远程服务)
        if (! Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope) ){
            if (logger.isInfoEnabled()) {
                logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
            }
            if (registryURLs != null && registryURLs.size() > 0
                    && url.getParameter("register", true)) {
                for (URL registryURL : registryURLs) {
                    url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
                    URL monitorUrl = loadMonitor(registryURL);
                    if (monitorUrl != null) {
                        url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                    }
                    if (logger.isInfoEnabled()) {
                        logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                    }
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));

                    Exporter<?> exporter = protocol.export(invoker);
                    exporters.add(exporter);
                }
            } else {
                Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url); 【*】

                Exporter<?> exporter = protocol.export(invoker);【*】
                exporters.add(exporter);
            }
        }
    }
    this.urls.add(url);
}
```



### DubboProtocol

```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    // export service.
    String key = serviceKey(url);
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    exporterMap.put(key, exporter);

    //export an stub service for dispaching event
    Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY,Constants.DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice){
        String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0 ){
            if (logger.isWarnEnabled()){
                logger.warn(new IllegalStateException("consumer [" +url.getParameter(Constants.INTERFACE_KEY) +
                        "], has set stubproxy support event ,but no stub methods founded."));
            }
        } else {
            stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
        }
    }

    openServer(url);

    return exporter;
}

private void openServer(URL url) {
    // find server.
    String key = url.getAddress();
    //client 也可以暴露一个只有server可以调用的服务。
    boolean isServer = url.getParameter(Constants.IS_SERVER_KEY,true);
    if (isServer) {
        ExchangeServer server = serverMap.get(key);
        if (server == null) {
            serverMap.put(key, createServer(url));
        } else {
            //server支持reset,配合override功能使用
            server.reset(url);
        }
    }
}

private ExchangeServer createServer(URL url) {
    //默认开启server关闭时发送readonly事件
    url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
    //默认开启heartbeat
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
    String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);

    if (str != null && str.length() > 0 && ! ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str))
        throw new RpcException("Unsupported server type: " + str + ", url: " + url);

    url = url.addParameter(Constants.CODEC_KEY, Version.isCompatibleVersion() ? COMPATIBLE_CODEC_NAME : DubboCodec.NAME);
    ExchangeServer server;
    try {
        server = Exchangers.bind(url, requestHandler);【*】
    } catch (RemotingException e) {
        throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
    }
    str = url.getParameter(Constants.CLIENT_KEY);
    if (str != null && str.length() > 0) {
        Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
        if (!supportedTypes.contains(str)) {
            throw new RpcException("Unsupported client type: " + str);
        }
    }
    return server;
}

public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
    return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));【*】
}
```

NettyTransporter

```java
public Server bind(URL url, ChannelHandler listener) throws RemotingException {
    return new NettyServer(url, listener);
}

public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
    super(url, handler);
    localAddress = getUrl().toInetSocketAddress();
    String host = url.getParameter(Constants.ANYHOST_KEY, false) 
                    || NetUtils.isInvalidLocalHost(getUrl().getHost()) 
                    ? NetUtils.ANYHOST : getUrl().getHost();
    bindAddress = new InetSocketAddress(host, getUrl().getPort());
    this.accepts = url.getParameter(Constants.ACCEPTS_KEY, Constants.DEFAULT_ACCEPTS);
    this.idleTimeout = url.getParameter(Constants.IDLE_TIMEOUT_KEY, Constants.DEFAULT_IDLE_TIMEOUT);
    try {
        doOpen();
        if (logger.isInfoEnabled()) {
            logger.info("Start " + getClass().getSimpleName() + " bind " + getBindAddress() + ", export " + getLocalAddress());
        }
    } catch (Throwable t) {
        throw new RemotingException(url.toInetSocketAddress(), null, "Failed to bind " + getClass().getSimpleName() 
                                    + " on " + getLocalAddress() + ", cause: " + t.getMessage(), t);
    }
    if (handler instanceof WrappedChannelHandler ){
        executor = ((WrappedChannelHandler)handler).getExecutor();
    }
}

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
```

<!-- ### 具体剖析 -->

ProviderConfig


registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&owner=william&pid=28449&registry=zookeeper&timestamp=1510911734884


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