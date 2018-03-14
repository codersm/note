Dubbo采用**微内核+插件**模式的设计原则，微内核负责组装插件，也就是Dubbo的所有功能点都可被用户自定义扩展所替换。微内核是由Dubbo SPI机制实现的，因此了解Dubbo SPI是非常重要的。

## Dubbo SPI简介

Dubbo SPI从JDK标准的SPI (Service Provider Interface) 扩展点发现机制加强而来，改进了JDK标准的 SPI 的以下问题：

*  **无法获取指定的扩展实现 。**~~JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。~~

* 如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK标准的ScriptEngine，通过getName() 获取脚本类型的名称，但如果RubyScriptEngine因为所依赖的 jruby.jar不存在导致 RubyScriptEngine 类加载失败，这个失败原因被吃掉了，和ruby对应不起来，当用户执行ruby脚本时，会报不支持ruby，而不是真正失败的原因。

* 增加了对扩展点IoC和AOP的支持，一个扩展点可以直接setter注入其它扩展点。

> 如果不了解JDK SPI机制，可以看我写的[JDK SPI源码详解](https://juejin.im/post/5a6036d5518825734107f96d)。

## Dubbo SPI机制源码分析

JDK SPI机制是由java.util.ServiceLoader这个工具类实现的，同样Dubbo也提供了类似的类(com.alibaba.dubbo.common.extension.ExtensionLoader)来实现的Dubbo SPI机制。

下面的内容主要讲解ExtensionLoader如何获取指定的扩展点和自适应扩展点

### 获取指定的扩展点

通过获取DubboProtocol为例，其代码如下：

```java
Protocol protocol  = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("dubbo");
```

通过具体的源码分析，其运行过程如下：

1、创建ExtensionLoader实例

```java
// Dubbo SPI只支持接口并且被@SPI修饰
ExtensionLoader loader = new ExtensionLoader(
    Protocol.class,
    ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()
)
```

2、获取扩展点**所有实现类**的Class对象

```java
 /**
  * 加载META-INF/services/、META-INF/dubbo/internal/、META-INF/dubbo/目录下type.getName文件
  * 并解析内容，然后将type基本实现类（不包括包装类，没有Adaptive注解）存储在extensionClasse中。
  */
private Map<String, Class<?>> loadExtensionClasses() {
    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
    loadFile(extensionClasses, DUBBO_DIRECTORY);
    loadFile(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}
```

3、创建DubboProtocol对象并装配其依赖的对象

```java
// 获取名为dubbo对应的扩展点实现类
Class<?> clazz = getExtensionClasses().get(name);
// 创建对象
T instance = clazz.newInstance();
// 自动装配
injectExtension(instance);
```

4、返回ProtocolListenerWrapper对象

```java
Set<Class<?>> wrapperClasses = cachedWrapperClasses;
// 循环遍历初始化包装类
if (wrapperClasses != null && wrapperClasses.size() > 0) {
    for (Class<?> wrapperClass : wrapperClasses) {
        instance = injectExtension((T) wrapperClass.getConstructor(type)
            .newInstance(instance));
    }
}
return instance;
```

> 自动注入过程获取需要依赖的对象，是通过ExtensionLoader.objectFactory对象获取的。

### 获取自适应扩展点

通过获取Procotol自适应扩展点为例，其代码如下：
```java
Protocol protocol  = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

其过程如下：

```java
// 创建Protocol自适应扩展点实例
private T createAdaptiveExtension() {
    return injectExtension((T) getAdaptiveExtensionClass().newInstance());
}

/**
  * 之所以贴出这段代码，因为获取自适应扩展点会触发获取扩展点所有实现类的Class对象。目前
  * dubbo只有被@Adaptive注解的类仅有AdaptiveCompiler和AdaptiveExtensionFactory，因此除了
  * Complie和ExtensionFactory外都需要动态创建其自适应扩展点的Class
  */
private Class<?> getAdaptiveExtensionClass() {
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
    
// 创建自适应扩展点类Class对象
private Class<?> createAdaptiveExtensionClass() {
    String code = createAdaptiveExtensionClassCode();
    ClassLoader classLoader = findClassLoader();
    com.alibaba.dubbo.common.compiler.Compiler compiler = 
    ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class)
    .getAdaptiveExtension();
    return compiler.compile(code, classLoader);
}
```

动态生成的自适应扩展点Protocol$Adpative类源码如下：

```java
public class Protocol$Adpative implements com.alibaba.dubbo.rpc.Protocol {

    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
    }

    public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws java.lang.Class {
        if (arg1 == null) 
            throw new IllegalArgumentException("url == null");
        com.alibaba.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
       
        com.alibaba.dubbo.rpc.Protocol extension = 
        (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader
            .getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class)
            .getExtension(extName);
        return extension.refer(arg0, arg1);
    }

    public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.Invoker {
        if (arg0 == null)
         throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
        com.alibaba.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        
        com.alibaba.dubbo.rpc.Protocol extension = 
        (com.alibaba.dubbo.rpc.Protocol) ExtensionLoader
            .getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class)
            .getExtension(extName);
        return extension.export(arg0);
    }
}
```

可以发现自适应扩展点的作用：通过URL的参数信息获取对应扩展点，从而实现方法动态调用。对应Dubbo基本设计原二：
> 采用 URL 作为配置信息的统一格式，所有扩展点都通过传递 URL 携带配置信息。