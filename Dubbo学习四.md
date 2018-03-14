---
title: Dubbo源码分析之Consumer
date: 2017-07-17 09:09:25
tags: Dubbo
---
# Dubbo源码分析之Consumer

## 引用提供者的服务

### ReferenceBean

```java
public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean,
                            ApplicationContextAware, InitializingBean, DisposableBean
```

ReferenceBean继承自ReferenceConfig，并实现了FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean。

ReferenceBean.afterPropertiesSet()主要是对没有初始化的几个属性尝试用公共的默认配置进行初始化。由于实现了FactoryBean，当需要初始化或者应用中需要用到时，会调用getObject()方法获取实际的对象：

```java
public Object getObject() throws Exception {
    return get();
}

public synchronized T get() {
    // 如果已销毁抛出异常
    if (destroyed){
        throw new IllegalStateException("Already destroyed!");
    }
    // 如果ref为空则初始化后再返回
    if (ref == null) {
        init();
    }
    return ref;
}
```

> Spring中有两种bean，一种是普通的bean，一种是工厂bean，即FactoryBean。普通bean通过class字符串代表的类直接实例化对象，而工厂bean则是通过class字符串代表的工厂类的getObject()方法来实例化对象。

获取实际的对象由init()方法完成的，代码如下：

```java
private void init() {
    // 先判断initialized标记，如果为true，表示已经初始化过，初始化过则不再初始化
    if (initialized) {
        return;
    }
    initialized = true;
    if (interfaceName == null || interfaceName.length() == 0) {
        throw new IllegalStateException("<dubbo:reference interface=\"\" /> interface not allow null!");
    }
    // 获取消费者全局配置
    checkDefault();
    appendProperties(this);
    // 泛化调用设置，如果是泛化调用则接口类为GeneicService，否则为配置的interfaceName  
    if (getGeneric() == null && getConsumer() != null) {
        setGeneric(getConsumer().getGeneric());
    }
    if (ProtocolUtils.isGeneric(getGeneric())) {
        interfaceClass = GenericService.class;
    } else {
        try {
            interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                    .getContextClassLoader());
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        checkInterfaceAndMethods(interfaceClass, methods);
    }
    String resolve = System.getProperty(interfaceName);
    String resolveFile = null;
    if (resolve == null || resolve.length() == 0) {
        resolveFile = System.getProperty("dubbo.resolve.file");
        if (resolveFile == null || resolveFile.length() == 0) {
            File userResolveFile = new File(new File(System.getProperty("user.home")), "dubbo-resolve.properties");
            if (userResolveFile.exists()) {
                resolveFile = userResolveFile.getAbsolutePath();
            }
        }
        if (resolveFile != null && resolveFile.length() > 0) {
            Properties properties = new Properties();
            FileInputStream fis = null;
            try {
                fis = new FileInputStream(new File(resolveFile));
                properties.load(fis);
            } catch (IOException e) {
                throw new IllegalStateException("Unload " + resolveFile + ", cause: " + e.getMessage(), e);
            } finally {
                try {
                    if(null != fis) fis.close();
                } catch (IOException e) {
                    logger.warn(e.getMessage(), e);
                }
            }
            resolve = properties.getProperty(interfaceName);
        }
    }
    if (resolve != null && resolve.length() > 0) {
        url = resolve;
        if (logger.isWarnEnabled()) {
            if (resolveFile != null && resolveFile.length() > 0) {
                logger.warn("Using default dubbo resolve file " + resolveFile + " replace " + interfaceName + "" + resolve + " to p2p invoke remote service.");
            } else {
                logger.warn("Using -D" + interfaceName + "=" + resolve + " to p2p invoke remote service.");
            }
        }
    }
    // 设置属性
    if (consumer != null) {
        if (application == null) {
            application = consumer.getApplication();
        }
        if (module == null) {
            module = consumer.getModule();
        }
        if (registries == null) {
            registries = consumer.getRegistries();
        }
        if (monitor == null) {
            monitor = consumer.getMonitor();
        }
    }
    if (module != null) {
        if (registries == null) {
            registries = module.getRegistries();
        }
        if (monitor == null) {
            monitor = module.getMonitor();
        }
    }
    if (application != null) {
        if (registries == null) {
            registries = application.getRegistries();
        }
        if (monitor == null) {
            monitor = application.getMonitor();
        }
    }

    checkApplication();
    // 检查local/stub/mock三个配置是否正确
    checkStubAndMock(interfaceClass);
    Map<String, String> map = new HashMap<String, String>();
    Map<Object, Object> attributes = new HashMap<Object, Object>();
    map.put(Constants.SIDE_KEY, Constants.CONSUMER_SIDE);
    map.put(Constants.DUBBO_VERSION_KEY, Version.getVersion());
    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
    if (ConfigUtils.getPid() > 0) {
        map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
    }
    if (! isGeneric()) {
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put("revision", revision);
        }

        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        if(methods.length == 0) {
            logger.warn("NO method found in service interface " + interfaceClass.getName());
            map.put("methods", Constants.ANY_VALUE);
        }
        else {
            map.put("methods", StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }
    map.put(Constants.INTERFACE_KEY, interfaceName);
    appendParameters(map, application);
    appendParameters(map, module);
    appendParameters(map, consumer, Constants.DEFAULT_KEY);
    appendParameters(map, this);
    String prifix = StringUtils.getServiceKey(map);
    if (methods != null && methods.size() > 0) {
        for (MethodConfig method : methods) {
            appendParameters(map, method, method.getName());
            String retryKey = method.getName() + ".retry";
            if (map.containsKey(retryKey)) {
                String retryValue = map.remove(retryKey);
                if ("false".equals(retryValue)) {
                    map.put(method.getName() + ".retries", "0");
                }
            }
            appendAttributes(attributes, method, prifix + "." + method.getName());
            checkAndConvertImplicitConfig(method, map, attributes);
        }
    }
    //attributes通过系统context进行存储.
    StaticContext.getSystemContext().putAll(attributes);
    // 使用map中的参数通过createProxy方法创建代理对象
    ref = createProxy(map);
}
```

使用map中的参数通过createProxy方法创建代理对象如下：

```java
private T createProxy(Map<String, String> map) {
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
        } else { // 通过注册中心配置拼装URL
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
            if (urls == null || urls.size() == 0) {
                throw new IllegalStateException("No such any registry to reference " + interfaceName  + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
            }
        }
        // 当只有一个注册中心时，使用refprotocol.refer创建invoker
        if (urls.size() == 1) {
            invoker = refprotocol.refer(interfaceClass, urls.get(0));
        } else {
            List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
            URL registryURL = null;
            for (URL url : urls) {
                invokers.add(refprotocol.refer(interfaceClass, url));
                if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                    registryURL = url; // 用了最后一个registry url
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
    // 检查服务提供者是否存在
    Boolean c = check;
    if (c == null && consumer != null) {
        c = consumer.isCheck();
    }
    if (c == null) {
        c = true; // default true
    }
    if (c && ! invoker.isAvailable()) {
        throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
    }
    // 返回提供者的服务代理实例
    return (T) proxyFactory.getProxy(invoker);
}
```

上述代码主要功能就是获取Invoke实例，然后创建Invoke代理实例。

refprotocol

```java
package com.alibaba.dubbo.rpc;

import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {

    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws java.lang.Class {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }

    public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
        if (arg0 == null) throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }
}
```

可知refprotocol为RegistryProtocol实例。

```java
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
    ／／ 获取
    Registry registry = registryFactory.getRegistry(url);
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }

    // group="a,b" or group="*"
    Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(Constants.REFER_KEY));
    String group = qs.get(Constants.GROUP_KEY);
    if (group != null && group.length() > 0 ) {
        if ( ( Constants.COMMA_SPLIT_PATTERN.split( group ) ).length > 1
                || "*".equals( group ) ) {
            return doRefer( getMergeableCluster(), registry, type, url );
        }
    }
    return doRefer(cluster, registry, type, url);
}


private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, NetUtils.getLocalHost(), 0, type.getName(), directory.getUrl().getParameters());
    if (! Constants.ANY_VALUE.equals(url.getServiceInterface())
            && url.getParameter(Constants.REGISTER_KEY, true)) {
        registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
                Constants.CHECK_KEY, String.valueOf(false)));
    }
    directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY, 
            Constants.PROVIDERS_CATEGORY 
            + "," + Constants.CONFIGURATORS_CATEGORY 
            + "," + Constants.ROUTERS_CATEGORY));
    return cluster.join(directory);
}
```

cluster == FailoverCluster

cluster.join(directory)返回的是MockClusterInvoker。

```java
private ExchangeClient[] getClients(URL url){
    //是否共享连接
    boolean service_share_connect = false;
    int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
    //如果connections不配置，则共享连接，否则每服务每连接
    if (connections == 0){
        service_share_connect = true;
        connections = 1;
    }

    ExchangeClient[] clients = new ExchangeClient[connections];
    for (int i = 0; i < clients.length; i++) {
        if (service_share_connect){
            clients[i] = getSharedClient(url);
        } else {
            clients[i] = initClient(url);
        }
    }
    return clients;
}
```

#### cluster.join(new StaticDirectory(u, invokers))

// 不怎么清楚

```java
List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
URL registryURL = null;
for (URL url : urls) {
    invokers.add(refprotocol.refer(interfaceClass, url));
    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
        registryURL = url; // 用了最后一个registry url
    }
}
if (registryURL != null) { // 有 注册中心协议的URL
    // 对有注册中心的Cluster 只用 AvailableCluster
    URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME); 
    invoker = cluster.join(new StaticDirectory(u, invokers));
}  else { // 不是 注册中心的URL
    invoker = cluster.join(new StaticDirectory(invokers));
}
```

- FailoverCluster

```java
public class FailoverCluster implements Cluster {

    public final static String NAME = "failover";

    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new FailoverClusterInvoker<T>(directory);
    }

}
```

- StaticDirectory

```java
public StaticDirectory(URL url, List<Invoker<T>> invokers, List<Router> routers) {
    super(url == null && invokers != null && invokers.size() > 0 ? invokers.get(0).getUrl() : url, routers);
    if (invokers == null || invokers.size() == 0)
        throw new IllegalArgumentException("invokers == null");
    this.invokers = invokers;
}


public AbstractDirectory(URL url, URL consumerUrl, List<Router> routers) {
    if (url == null)
        throw new IllegalArgumentException("url == null");
    this.url = url;
    this.consumerUrl = consumerUrl;
    setRouters(routers);
}

protected void setRouters(List<Router> routers){
        // copy list
        routers = routers == null ? new  ArrayList<Router>() : new ArrayList<Router>(routers);
        // append url router
        String routerkey = url.getParameter(Constants.ROUTER_KEY);
        if (routerkey != null && routerkey.length() > 0) {
            RouterFactory routerFactory = ExtensionLoader.getExtensionLoader(RouterFactory.class).getExtension(routerkey);
            routers.add(routerFactory.getRouter(url));
        }
        // append mock invoker selector
        routers.add(new MockInvokersSelector());
        Collections.sort(routers);
        this.routers = routers;
}
```

### 服务代理类具体过程

```java
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}

public static Proxy getProxy(Class<?>... ics){
    return getProxy(ClassHelper.getCallerClassLoader(Proxy.class), ics);
}

public static Proxy getProxy(ClassLoader cl, Class<?>... ics)
{
        ...
        // create ProxyInstance class.
        String pcn = pkg + ".proxy" + id;
        ccp.setClassName(pcn);
        ccp.addField("public static java.lang.reflect.Method[] methods;");
        ccp.addField("private " + InvocationHandler.class.getName() + " handler;");
        ccp.addConstructor(Modifier.PUBLIC, new Class<?>[]{ InvocationHandler.class },
                                         new Class<?>[0], "handler=$1;");
        ccp.addDefaultConstructor();
        Class<?> clazz = ccp.toClass();
        clazz.getField("methods").set(null, methods.toArray(new Method[0]));

        // create Proxy class.
        String fcn = Proxy.class.getName() + id;
        ccm = ClassGenerator.newInstance(cl);
        ccm.setClassName(fcn);
        ccm.addDefaultConstructor();
        ccm.setSuperClass(Proxy.class);
        ccm.addMethod("public Object newInstance(" + 
                        InvocationHandler.class.getName() + " h){ return new " + pcn + "($1); }");
        Class<?> pc = ccm.toClass();
        proxy = (Proxy)pc.newInstance();
    }
    ...
    return proxy;
}
```

可以看到Dubbo是通过Javassist字节码技术动态创建代理类，通过反编译工具可以得到代理类的代码如下：

```java
public class Proxy0 extends Proxy implements DC {
    public Object newInstance(InvocationHandler var1) {
        return new proxy0(var1);
    }

    public Proxy0() {
    }
}
```

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

## 调用提供者的服务

根据上述引用提供者的服务分析过程可知，调用提供者的服务实际上是由InvokerInvocationHandler.invoke()方法完成的。invoke方法如下：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    String methodName = method.getName();
    Class<?>[] parameterTypes = method.getParameterTypes();
    if (method.getDeclaringClass() == Object.class) {
        return method.invoke(invoker, args);
    }
    if ("toString".equals(methodName) && parameterTypes.length == 0) {
        return invoker.toString();
    }
    if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
        return invoker.hashCode();
    }
    if ("equals".equals(methodName) && parameterTypes.length == 1) {
        return invoker.equals(args[0]);
    }
    return invoker.invoke(new RpcInvocation(method, args)).recreate();
}
```

### DubboInvoker

```java
public Result invoke(Invocation inv) throws RpcException {
    if(destroyed) {
        throw new RpcException("Rpc invoker for service " + this + " on consumer " + NetUtils.getLocalHost() 
                                        + " use dubbo version " + Version.getVersion()
                                        + " is DESTROYED, can not be invoked any more!");
    }
    RpcInvocation invocation = (RpcInvocation) inv;
    invocation.setInvoker(this);
    if (attachment != null && attachment.size() > 0) {
        invocation.addAttachmentsIfAbsent(attachment);
    }
    Map<String, String> context = RpcContext.getContext().getAttachments();
    if (context != null) {
        invocation.addAttachmentsIfAbsent(context);
    }
    if (getUrl().getMethodParameter(invocation.getMethodName(), Constants.ASYNC_KEY, false)){
        invocation.setAttachment(Constants.ASYNC_KEY, Boolean.TRUE.toString());
    }
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

    try {
        return doInvoke(invocation);
    } catch (InvocationTargetException e) { // biz exception
        Throwable te = e.getTargetException();
        if (te == null) {
            return new RpcResult(e);
        } else {
            if (te instanceof RpcException) {
                ((RpcException) te).setCode(RpcException.BIZ_EXCEPTION);
            }
            return new RpcResult(te);
        }
    } catch (RpcException e) {
        if (e.isBiz()) {
            return new RpcResult(e);
        } else {
            throw e;
        }
    } catch (Throwable e) {
        return new RpcResult(e);
    }
}


protected Result doInvoke(final Invocation invocation) throws Throwable {
    RpcInvocation inv = (RpcInvocation) invocation;
    final String methodName = RpcUtils.getMethodName(invocation);
    inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
    inv.setAttachment(Constants.VERSION_KEY, version);

    ExchangeClient currentClient;
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        currentClient = clients[index.getAndIncrement() % clients.length];
    }
    try {
        boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY,Constants.DEFAULT_TIMEOUT);
        if (isOneway) {[*]
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            currentClient.send(inv, isSent);
            RpcContext.getContext().setFuture(null);
            return new RpcResult();
        } else if (isAsync) {[*]
            ResponseFuture future = currentClient.request(inv, timeout) ;
            RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
            return new RpcResult();
        } else {
            RpcContext.getContext().setFuture(null);
            return (Result) currentClient.request(inv, timeout).get();
        }
    } catch (TimeoutException e) {
        throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    } catch (RemotingException e) {
        throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " `+` getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

### FailoverClusterInvoker

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
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    return doInvoke(invocation, invokers, loadbalance);
}

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

RegistryDirectory$InvokerDelegete

```java
private static class InvokerDelegete<T> extends InvokerWrapper<T>{
    private URL providerUrl;
    public InvokerDelegete(Invoker<T> invoker, URL url, URL providerUrl) {
        super(invoker, url);
        this.providerUrl = providerUrl;
    }
    public URL getProviderUrl() {
        return providerUrl;
    }
}


// ListenerInvokerWrapper
public Result invoke(Invocation invocation) throws RpcException {
    return invoker.invoke(invocation);
}
```

发送请求

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

public void send(Object message, boolean sent) throws RemotingException {
    if (closed) {
        throw new RemotingException(this.getLocalAddress(), null, "Failed to send message " + message + ", cause: The channel " + this + " is closed!");
    }
    if (message instanceof Request
            || message instanceof Response
            || message instanceof String) {
        channel.send(message, sent);
    } else {
        Request request = new Request();
        request.setVersion("2.0.0");
        request.setTwoWay(false);
        request.setData(message);
        channel.send(request, sent);
    }
}
```

// 过滤器