# Dubbo源码分析——服务发布

## 了解服务发布

Dubbo官方文档说明了服务提供者暴露服务的主过程，如图所示：

![服务提供者暴露一个服务的详细过程](https://user-gold-cdn.xitu.io/2018/3/7/161fea204f90035a?w=1240&h=795&f=png&s=388676)

>  首先ServiceConfig类拿到对外提供服务的实际类ref(如：HelloWorldImpl),然后通过ProxyFactory类的 getInvoker方法使用ref生成一个AbstractProxyInvoker实例，到这一步就完成具体服务到Invoker的转化。接下来就是Invoker转换到Exporter的过程。**Dubbo 处理服务暴露的关键就在Invoker转换到Exporter的过程，上图中的红色部分**。

## 源码分析

### 入口分析——ServiceBean

服务发布在spring的配置文件中配置如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
	http://code.alibabatech.com/schema/dubbo
	http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
	

	<bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl" />

	<dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService" />
</beans>
```

如果熟悉Spring自定义标签，应该知道`<dubbo:service>`标签会转换成ServiceBean。通过DubboNamespaceHandler.init()方法可知，其源码如下：

```java
    registerBeanDefinitionParser("service", 
            new DubboBeanDefinitionParser(ServiceBean.class, true));
```

ServiceBean类结构如下图所示：

![ServiceBean类结构图](https://user-gold-cdn.xitu.io/2018/3/7/161fea204fe03649?w=1240&h=180&f=png&s=44690)

> dubbo暴露服务有两种情况，一种是设置了延迟暴露（比如delay=”5000”），另外一种是没有设置延迟暴露或者延迟设置为-1（delay=”-1”）：
> 
> 1、设置了延迟暴露，dubbo在Spring实例化bean（initializeBean）的时候会对实现了InitializingBean的类进行回调，回调方法是afterPropertySet()，如果设置了延迟暴露，dubbo在这个方法中进行服务的发布。
>
> 2、没有设置延迟或者延迟为-1，dubbo会在Spring实例化完bean之后，在刷新容器最后一步发布ContextRefreshEvent事件的时候，通知实现了ApplicationListener的类进行回调onApplicationEvent，dubbo会在这个方法中发布服务。


**但是不管延迟与否，都是使用ServiceConfig的export()方法进行服务的暴露。使用export初始化的时候会将Bean对象转换成URL格式，所有Bean属性转换成URL的参数**。

### ServiceConfig.export()方法

ServiceConfig.export()的流程图如下：

![image.png](https://user-gold-cdn.xitu.io/2018/3/7/161fea204faa90a7?w=1240&h=120&f=png&s=38892)

1、export()方法先判断是否需要延迟暴露，如果设置了延迟，则通过一个后台线程调用doExport()方法；反之，直接调用doExport()方法。

2、doExport方法先执行一系列的检查方法，然后调用doExportUrls方法。检查方法会检测dubbo的配置是否在Spring配置文件中声明，没有的话读取properties文件初始化。

3、doExportUrls方法先调用loadRegistries获取所有的注册中心url，然后遍历调用doExportUrlsFor1Protocol方法。

4、doExportUrlsFor1Protocol()先将Bean属性转换成URL对象，然后根据不同协议将服务已URL形式发布。如果scope配置为none则不暴露，如果服务未配置成remote，则本地暴露exportLocal，如果未配置成local，则远程暴露。

暴露服务的核心是由doExportUrlsFor1Protocol()方法处理的，其源码如下：
```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    
    // 1、Bean属性转换URL
    URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);

    if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
            .hasExtension(url.getProtocol())) {
        url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
    }

    String scope = url.getParameter(Constants.SCOPE_KEY);
    
    if (! Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {

        // scope配置不是remote的情况下做本地暴露
        if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
            exportLocal(url);
        }
        // scope配置不是local则暴露为远程服务
        if (! Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope) ){

            if (registryURLs != null && registryURLs.size() > 0
                    && url.getParameter("register", true)) {
                for (URL registryURL : registryURLs) {
                    url = url.addParameterIfAbsent("dynamic", registryURL.getParameter("dynamic"));
                    URL monitorUrl = loadMonitor(registryURL);
                    if (monitorUrl != null) {
                        url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                    }
                    // 2、具体服务到invoker的转换
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                    // 3、invoker转换为exporter
                    Exporter<?> exporter = protocol.export(invoker);
                    exporters.add(exporter);
                }
            } else {
                Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);

                Exporter<?> exporter = protocol.export(invoker);
                exporters.add(exporter);
            }
        }
    }
    this.urls.add(url);
}
```

### 具体服务到Invoker转换

在分析具体服务到Invoker之前，先来看看ServiceConfig的proxyFactory实例。其定义如下：

```java
private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
```

ProxyFactory的自适应扩展点的getInvoker方法源码如下：

```java
public com.alibaba.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, com.alibaba.dubbo.common.URL arg2) throws RpcException {

        if (arg2 == null) throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg2;
        // 如果url没有proxy参数，默认为javassist
        String extName = url.getParameter("proxy", "javassist");
        
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");

        com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        return extension.getInvoker(arg0, arg1, arg2);
    }
```
通过上述源码我们知道，最终获取invoker是由JavassistProxyFactory.getInvoker方法实现的，其源码如下：

```java
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    // 获取具体服务类的包装对象
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    // 返回构造invoker实例
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName, 
                                    Class<?>[] parameterTypes, 
                                    Object[] arguments) throws Throwable {
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```

**流程总结**：

1、根据url的proxy参数获取对应ProxyFactory类，默认为JavassistProxyFactory；

2、获取具体服务类的包装对象；

3、返回构造invoker实例，将服务具体类的方法调用封装在doInvoke()方法中。

### 远程暴露

通过debug源码，protocol.export(invoker)的时序图如下：

![protocol.export(invoker)时序图](https://user-gold-cdn.xitu.io/2018/3/7/161fea204fb3f439?w=1240&h=697&f=jpeg&s=86036)

**流程总结**

1、**构建invoker过滤链**；

2、**首先DubboProcotol的export方法将invoker转换成DubboExporter，启动Server服务，然后将DubboExporter包装为ListenerExporterWrapper**。

3、**RegistryProtocol的export方法向注册中心注册服务提供者url和订阅url，再次将ListenerExporterWrapper包装为Exporter，覆盖unexport()方法**。


#### RegistryProtocol.export()方法的注册和订阅

下面我们分析下RegistryProtocol.export()方法的注册和订阅，其源码如下：

```java
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //export invoker
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
    //registry provider
    final Registry registry = getRegistry(originInvoker);
    final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
    registry.register(registedProviderUrl);
    // 订阅override数据
    // provider://172.16.6.216:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&category=configurators&check=false&dubbo=2.5.3&interface=com.alibaba.dubbo.demo.DemoService&loadbalance=roundrobin&methods=sayHello&owner=william&pid=2154&side=provider&timestamp=1520235259270
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

#### 注册

由于使用zookeeper作为注册中心，所以registry为ZookeeperRegistry。ZookeeperRegistry.registry过程如下：

![ZookeeperRegistry.registry](https://user-gold-cdn.xitu.io/2018/3/7/161fea204fc8355c?w=548&h=616&f=png&s=28745)

ZookeeperRegistry.doRegister方法如下：

```java
protected void doRegister(URL url) {
    try {
        zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
    } catch (Throwable e) {
        throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```
**doRegister方法将url转换为注册zookeeper的path，创建临时节点。** 临时节点在zookeeper会话失效后会自动删除的，Dubbo如何解决这个问题。
```java
public FailbackRegistry(URL url) {
    super(url);
    int retryPeriod = url.getParameter(Constants.REGISTRY_RETRY_PERIOD_KEY, Constants.DEFAULT_REGISTRY_RETRY_PERIOD);
    this.retryFuture = retryExecutor.scheduleWithFixedDelay(new Runnable() {
        public void run() {
            // 检测并连接注册中心
            try {
                retry();
            } catch (Throwable t) { // 防御性容错
                logger.error("Unexpected error occur at failed retry, cause: " + t.getMessage(), t);
            }
        }
    }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
}
```

#### 订阅

```java
protected void doSubscribe(final URL url, final NotifyListener listener) {
    try {
        if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {
            String root = toRootPath();
            ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
            if (listeners == null) {
                zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                listeners = zkListeners.get(url);
            }
            ChildListener zkListener = listeners.get(listener);
            if (zkListener == null) {
                listeners.putIfAbsent(listener, new ChildListener() {
                    public void childChanged(String parentPath, List<String> currentChilds) {
                        for (String child : currentChilds) {
                            if (! anyServices.contains(child)) {
                                anyServices.add(child);
                                subscribe(url.setPath(child).addParameters(Constants.INTERFACE_KEY, child, 
                                        Constants.CHECK_KEY, String.valueOf(false)), listener);
                            }
                        }
                    }
                });
                zkListener = listeners.get(listener);
            }
            zkClient.create(root, false);
            List<String> services = zkClient.addChildListener(root, zkListener);
            if (services != null && services.size() > 0) {
                anyServices.addAll(services);
                for (String service : services) {
                    subscribe(url.setPath(service).addParameters(Constants.INTERFACE_KEY, service, 
                            Constants.CHECK_KEY, String.valueOf(false)), listener);
                }
            }
        } else {
            List<URL> urls = new ArrayList<URL>();
            // /dubbo/com.alibaba.dubbo.demo.DemoService/configurators
            for (String path : toCategoriesPath(url)) {
                ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                if (listeners == null) {
                    zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                    listeners = zkListeners.get(url);
                }
                ChildListener zkListener = listeners.get(listener);
                if (zkListener == null) {
                    listeners.putIfAbsent(listener, new ChildListener() {
                        public void childChanged(String parentPath, List<String> currentChilds) {
                            // 匿名类内部可以用这种方式得到外部类当前对象
                            ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
                        }
                    });
                    zkListener = listeners.get(listener);
                }
                zkClient.create(path, false);
                List<String> children = zkClient.addChildListener(path, zkListener);
                if (children != null) {
                    urls.addAll(toUrlsWithEmpty(url, path, children));
                }
            }
            notify(url, listener, urls);
        }
    } catch (Throwable e) {
        throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

doSubscribe方法首先将NotifyListener转换为Zookeeper Listener，创建`/dubbo/com.alibaba.dubbo.demo.DemoService/configurators`持久节点并订阅。

为什么要订阅/dubbo/com.alibaba.dubbo.demo.DemoService/configurators这个节点。

