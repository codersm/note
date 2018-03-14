## 了解服务引用

消费者引用一个服务的主过程，如下图所示：

![图片.png](https://user-gold-cdn.xitu.io/2018/3/8/162064597145fd0f?w=1240&h=664&f=png&s=370292)

> 首先ReferenceConfig类的init方法调用Protocol的refer方法生成Invoker 实例(如上图中的红色部分)，这是服务消费的关键。接下来把Invoker转换为客户端需要的接口(如：HelloWorld)。

## 源码分析

### 入口分析——ReferenceBean

![ReferenceBean类结构图](https://user-gold-cdn.xitu.io/2018/3/8/16206459713dcd9e?w=1240&h=219&f=png&s=48073)

ReferenceBean类实现了FactoryBean接口，通过getBean方法返回的不是FactoryBean本身，而是FactoryBean#getObject方法所返回的对象。

getObject方法返回ReferenceBean类的实例变量ref，其源码如下：

```java
public Object getObject() throws Exception {
    return get();
}

public synchronized T get() {
    if (destroyed){
        throw new IllegalStateException("Already destroyed!");
    }
    if (ref == null) {
        init();
    }
    return ref;
}
```

init方法对ref变量进行赋值，源码如下：

```java
ref = createProxy(map);
```

接下来我们一起看看`createProxy`方法，源码如下：

```java
private T createProxy(Map<String, String> map) {
    // 1、根据referenceBean属性构建tmpUrl
    URL tmpUrl = new URL("temp", "localhost", 0, map);

    final boolean isJvmRefer;
    if (isInjvm() == null) {
        if (url != null && url.length() > 0) { //指定URL的情况下，不做本地引用
            isJvmRefer = false;
        } else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
            //默认情况下如果本地有服务暴露，则引用本地服务.
            isJvmRefer = true;
        } else {
            isJvmRefer = false;
        }
    } else {
        isJvmRefer = isInjvm().booleanValue();
    }

    // 本地引用
    if (isJvmRefer) {
        URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
        invoker = refprotocol.refer(interfaceClass, url);
        if (logger.isInfoEnabled()) {
            logger.info("Using injvm service " + interfaceClass.getName());
        }
    } else {
        // 用户指定URL，指定的URL可能是对点对直连地址，也可能是注册中心URL
        if (url != null && url.length() > 0) { 
            String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
            if (us != null && us.length > 0) {
                for (String u : us) {
                    URL url = URL.valueOf(u);
                    if (url.getPath() == null || url.getPath().length() == 0) {
                        url = url.setPath(interfaceName);
                    }
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                    } else {
                        urls.add(ClusterUtils.mergeUrl(url, map));
                    }
                }
            }
        } else { 
            // 通过注册中心配置拼装URL
            List<URL> us = loadRegistries(false);
            if (us != null && us.size() > 0) {
                for (URL u : us) {
                    URL monitorUrl = loadMonitor(u);
                    if (monitorUrl != null) {
                        map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                    }
                    urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                }
            }
        }
        // 2、调用procotol获取invoker对象
        if (urls.size() == 1) {  // 只存在一个注册中心
            invoker = refprotocol.refer(interfaceClass, urls.get(0));
        } else {
            List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
            URL registryURL = null;
            for (URL url : urls) {
                invokers.add(refprotocol.refer(interfaceClass, url));
                if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) { // 用了最后一个registry url
                    registryURL = url; 
                }
            }
            if (registryURL != null) { // 有 注册中心协议的URL
                // 对有注册中心的Cluster 只用 AvailableCluster
                URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME); 
                invoker = cluster.join(new StaticDirectory(u, invokers));
            }  else { // 不是 注册中心的URL
                invoker = cluster.join(new StaticDirectory(invokers));
            }
        }
    }

    // 3、创建invoker的代理对象
    return (T) proxyFactory.getProxy(invoker);
}
```

createProxy方法的过程如下：

1、根据ReferenceBean的属性拼接URL；

2、调用refprotocol.refer()方法获取invoker对象；

3、创建invoker对象的代理对象。


### 获取invoker对象的过程

`refprotocol.refer`方法的时序图如下：

![WX20180308-113757@2x.png](https://user-gold-cdn.xitu.io/2018/3/8/1620645971dc6f42?w=1240&h=617&f=png&s=74957)

`refprotocol.refer`方法主要逻辑由`doRefer`方法完成的，向注册中心注册消费者的url，然后订阅引用接口的信息，最后通过`cluster.join(directory)`获取invoke对象（这个过程比较简单，直接构造invoker对象）。doRefer方法源码如下：

```java
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, NetUtils.getLocalHost(), 0, type.getName(), directory.getUrl().getParameters());
    if (! Constants.ANY_VALUE.equals(url.getServiceInterface())
            && url.getParameter(Constants.REGISTER_KEY, true)) {
        // 向注册中心注册消费者的url        
        registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                Constants.CHECK_KEY, String.valueOf(false)));
    }
    // 订阅引用接口的提供者
    directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY, 
            Constants.PROVIDERS_CATEGORY 
            + "," + Constants.CONFIGURATORS_CATEGORY 
            + "," + Constants.ROUTERS_CATEGORY));
    return cluster.join(directory);
}
```

注册消费者url和发布服务的注册过程类似，都是创建zookeeper临时节点。我们重点关注其订阅过程：

```java
// this = doRefer()中directory
registry.subscribe(url, this);            
```
通过debug发现会订阅下面三个zookeeper目录：

`/dubbo/com.alibaba.dubbo.demo.DemoService/providers`
`/dubbo/com.alibaba.dubbo.demo.DemoService/configurators`
`/dubbo/com.alibaba.dubbo.demo.DemoService/routers`

当这三个目录有子节点变化时，都会出发点RegisterDirectory的notity方法。源码如下：

```java
public synchronized void notify(List<URL> urls) {
    List<URL> invokerUrls = new ArrayList<URL>();
    List<URL> routerUrls = new ArrayList<URL>();
    List<URL> configuratorUrls = new ArrayList<URL>();
    for (URL url : urls) {
        String protocol = url.getProtocol();
        String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
        if (Constants.ROUTERS_CATEGORY.equals(category) 
                || Constants.ROUTE_PROTOCOL.equals(protocol)) {
            routerUrls.add(url);
        } else if (Constants.CONFIGURATORS_CATEGORY.equals(category) 
                || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
            configuratorUrls.add(url);
        } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
            invokerUrls.add(url);
        } else {
            logger.warn("Unsupported category " + category + " in notified url: " + url + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost());
        }
    }
    // configurators 
    if (configuratorUrls != null && configuratorUrls.size() >0 ){
        this.configurators = toConfigurators(configuratorUrls);
    }
    // routers
    if (routerUrls != null && routerUrls.size() >0 ){
        List<Router> routers = toRouters(routerUrls);
        if(routers != null){ // null - do nothing
            setRouters(routers);
        }
    }
    List<Configurator> localConfigurators = this.configurators; // local reference
    // 合并override参数
    this.overrideDirectoryUrl = directoryUrl;
    if (localConfigurators != null && localConfigurators.size() > 0) {
        for (Configurator configurator : localConfigurators) {
            this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
        }
    }
    // providers
    refreshInvoker(invokerUrls);
}
```

### 创建invoker的代理对象

![WX20180308-113922@2x.png](https://user-gold-cdn.xitu.io/2018/3/8/1620645972090566?w=1240&h=536&f=png&s=62499)

创建invoker的代理对象由JavassistProxyFactory.getProxy方法完成的，源码如下：

```java
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.getProxy(interfaces)
        .newInstance(new InvokerInvocationHandler(invoker));
}
```

生成的代理对象其源码如下：

```java
public class proxy0 implements DC, EchoService, DemoService {
    public static Method[] methods;
    private InvocationHandler handler;

    public String sayHello(String var1) {
        Object[] var2 = new Object[]{var1};
        Object var3 = this.handler.invoke(this, methods[0], var2);
        return (String)var3;
    }

    public Object $echo(Object var1) {
        Object[] var2 = new Object[]{var1};
        Object var3 = this.handler.invoke(this, methods[1], var2);
        return (Object)var3;
    }

    public proxy0() {
    }

    public proxy0(InvocationHandler var1) {
        this.handler = var1;
    }
}
```


RegistryDirectory

RpcInvocation

MockClusterInvoker
```java
public Result invoke(Invocation invocation) throws RpcException {
    Result result = null;
    
    String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim(); 
    if (value.length() == 0 || value.equalsIgnoreCase("false")){
        //no mock
        result = this.invoker.invoke(invocation);
    } else if (value.startsWith("force")) {
        if (logger.isWarnEnabled()) {
            logger.info("force-mock: " + invocation.getMethodName() + " force-mock enabled , url : " +  directory.getUrl());
        }
        //force:direct mock
        result = doMockInvoke(invocation, null);
    } else {
        //fail-mock
        try {
            result = this.invoker.invoke(invocation);
        }catch (RpcException e) {
            if (e.isBiz()) {
                throw e;
            } else {
                if (logger.isWarnEnabled()) {
                    logger.info("fail-mock: " + invocation.getMethodName() + " fail-mock enabled , url : " +  directory.getUrl(), e);
                }
                result = doMockInvoke(invocation, e);
            }
        }
    }
    return result;
}
```

AbstractClusterInvoker
```java
public Result invoke(final Invocation invocation) throws RpcException {

    checkWheatherDestoried();

    LoadBalance loadbalance;
    
    List<Invoker<T>> invokers = list(invocation);
    if (invokers != null && invokers.size() > 0) {
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                .getMethodParameter(invocation.getMethodName(),Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
    } else {
        loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(Constants.DEFAULT_LOADBALANCE);
    }
    // 判断是否异步请求
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    return doInvoke(invocation, invokers, loadbalance);
}



public List<Invoker<T>> list(Invocation invocation) throws RpcException {
    if (destroyed){
        throw new RpcException("Directory already destroyed .url: "+ getUrl());
    }
    List<Invoker<T>> invokers = doList(invocation);
    List<Router> localRouters = this.routers; // local reference
    if (localRouters != null && localRouters.size() > 0) {
        for (Router router: localRouters){
            try {
                if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, true)) {
                    invokers = router.route(invokers, getConsumerUrl(), invocation);
                }
            } catch (Throwable t) {
                logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
            }
        }
    }
    return invokers;
}

public List<Invoker<T>> doList(Invocation invocation) {
    if (forbidden) {
        throw new RpcException(RpcException.FORBIDDEN_EXCEPTION, "Forbid consumer " +  NetUtils.getLocalHost() + " access service " + getInterface().getName() + " from registry " + getUrl().getAddress() + " use dubbo version " + Version.getVersion() + ", Please check registry access list (whitelist/blacklist).");
    }
    List<Invoker<T>> invokers = null;

    // 
    Map<String, List<Invoker<T>>> localMethodInvokerMap = this.methodInvokerMap; // local reference
    
    if (localMethodInvokerMap != null && localMethodInvokerMap.size() > 0) {
        String methodName = RpcUtils.getMethodName(invocation);
        Object[] args = RpcUtils.getArguments(invocation);
        if(args != null && args.length > 0 && args[0] != null
                && (args[0] instanceof String || args[0].getClass().isEnum())) {
            invokers = localMethodInvokerMap.get(methodName + "." + args[0]); // 可根据第一个参数枚举路由
        }
        if(invokers == null) {
            invokers = localMethodInvokerMap.get(methodName);
        }
        if(invokers == null) {
            invokers = localMethodInvokerMap.get(Constants.ANY_VALUE);
        }
        
        if(invokers == null) {
            Iterator<List<Invoker<T>>> iterator = localMethodInvokerMap.values().iterator();
            if (iterator.hasNext()) {
                invokers = iterator.next();
            }
        }
    }
    return invokers == null ? new ArrayList<Invoker<T>>(0) : invokers;
}
```

FailoverClusterInvoker
```java
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    List<Invoker<T>> copyinvokers = invokers;
    checkInvokers(copyinvokers, invocation);
    int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
    if (len <= 0) {
        len = 1;
    }
    // retry loop.
    RpcException le = null; // last exception.
    List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
    Set<String> providers = new HashSet<String>(len);
    for (int i = 0; i < len; i++) {
        //重试时，进行重新选择，避免重试时invoker列表已发生变化.
        //注意：如果列表发生了变化，那么invoked判断会失效，因为invoker示例已经改变
        if (i > 0) {
            checkWheatherDestoried();
            copyinvokers = list(invocation);
            //重新检查一下
            checkInvokers(copyinvokers, invocation);
        }
        Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
        invoked.add(invoker);
        RpcContext.getContext().setInvokers((List)invoked);
        try {
            Result result = invoker.invoke(invocation);
            if (le != null && logger.isWarnEnabled()) {
                logger.warn("Although retry the method " + invocation.getMethodName()
                        + " in the service " + getInterface().getName()
                        + " was successful by the provider " + invoker.getUrl().getAddress()
                        + ", but there have been failed providers " + providers 
                        + " (" + providers.size() + "/" + copyinvokers.size()
                        + ") from the registry " + directory.getUrl().getAddress()
                        + " on the consumer " + NetUtils.getLocalHost()
                        + " using the dubbo version " + Version.getVersion() + ". Last error is: "
                        + le.getMessage(), le);
            }
            return result;
        } catch (RpcException e) {
            if (e.isBiz()) { // biz exception.
                throw e;
            }
            le = e;
        } catch (Throwable e) {
            le = new RpcException(e.getMessage(), e);
        } finally {
            providers.add(invoker.getUrl().getAddress());
        }
    }
    throw new RpcException(le != null ? le.getCode() : 0, "Failed to invoke the method "
            + invocation.getMethodName() + " in the service " + getInterface().getName() 
            + ". Tried " + len + " times of the providers " + providers 
            + " (" + providers.size() + "/" + copyinvokers.size() 
            + ") from the registry " + directory.getUrl().getAddress()
            + " on the consumer " + NetUtils.getLocalHost() + " using the dubbo version "
            + Version.getVersion() + ". Last error is: "
            + (le != null ? le.getMessage() : ""), le != null && le.getCause() != null ? le.getCause() : le);
}
```

**ExchangeClient**

**HeaderExchangeChannel**

**DefaultFuture**

HeaderExchangeChannel

```java
public ResponseFuture request(Object request, int timeout) throws RemotingException {
    if (closed) {
        throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
    }
    // create request.
    Request req = new Request();
    req.setVersion("2.0.0");
    req.setTwoWay(true);
    req.setData(request);
    DefaultFuture future = new DefaultFuture(channel, req, timeout);
    try{
        channel.send(req);
    }catch (RemotingException e) {
        future.cancel();
        throw e;
    }
    return future;
}
```

## 通信


Channel

ChannelHandler

NettyServer为啥做心跳检查

