---
title: Spring IoC容器
date: 2017-06-28 15:17:36
tags: Spring
---
# IoC容器

## IoC概念

## IoC类型

从注入方法上看，IoC主要可以划分为3种类型：构造函数注入、属性注入和接口注入。Spring支持
构造函数注入和属性注入。

## 资源访问利器

JDK所提供的访问资源的类（如java.net.URL、File等）并不能很好地满足各种底层资源的访问需求，比如缺少从类路劲或者Web容器的上下文中获取资源的操作类。鉴于此，Spring设计了一个Resource接口，它为应用提供了更强的底层资源访问能力，该接口对应不同资源类型的实现类。

**Resource接口提供的方法**：

```java
    public interface Resource extends InputStreamSource {

    // 资源是否存在
    boolean exists();

    // 资源是否可读
    boolean isReadable();

    // 资源是否打开
    boolean isOpen();

    // 如果底层资源可以表示URL，则该方法返回对应的URL对象
    URL getURL() throws IOException;

    // 如果底层资源可以表示URL，则该方法返回对应的URL对象
    URI getURI() throws IOException;

    // 返回资源对应的File对象
    File getFile() throws IOException;

    // 返回资源上次修改时间戳
    long lastModified() throws IOException;

    // 创建资源相对于此资源
    Resource createRelative(String relativePath) throws IOException;

    // 返回资源的文件名
    String getFilename();

    // 资源描述
    String getDescription();

    }
```

Resource在Spring框架中起着不可或缺的作用，Spring框架使用Resource装载各种资源，包括配置文件资源、国际化属性文件资源等。

**Resource接口实现类**：
![Resource接口实现类](http://i2.kiimg.com/598959/627b89a024cbcc0f.png)

### 资源加载

为了访问不同类型的资源，必须使用相应的Resource实现类，这是比较麻烦的。Spring提供了一个强大的加载资源的机制，不但能够通过“classpath:”、“file:”等资源地址前缀识别不同的资源类型，还支持Ant风格带通配符的资源地址。

**资源地址表达式**：

| 地址前缀       | 对应的资源类型                                                                          |
|------------|----------------------------------------------------------------------------------|
| classpath: | 从类路径中加载资源，classpath:和classpath:/是等价的，都是相对于类的根路径。资源文件可以在标准的文件系统中，也可以在JAR或ZIP的类包中。 |
| file       | 使用UrlResource从文件系统目录中装载资源，可采用绝对或相对路径                                             |
| http://    | 使用UrlResource从Web服务器中装载资源                                                        |
| ftp://     | 使用UrlResource从FTP服务器中装载资源                                                        |
| 没有前缀       | 根据ApplicationContext的具体实现类采用对应类型的Resource                                        |

> “`classpath:`”和“`classpath*`:”的区别：假设有多个JAR包或文件系统类路径都拥有一个相同的包名(如com.smart)，“`classpath:`”只会在第一个加载com.smart包的类路径下查找；而“`classpath*:`”会扫描所有这些JAR包及类路径下出现的com.smart类路径。

**Ant风格的资源地址支持3种匹配符**：
   `?`：匹配文件名中的一个字符。
   `*`：匹配文件名中的任意字符。
   `**`：匹配多层路径。

#### 资源加载器(ResourceLoader)

ResourceLoader接口定义:

```java
public interface ResourceLoader {

    // 根据一个资源地址加载文件资源。不过，资源地址仅支持资源类型前缀的表达式，不支持Ant风格的资源路径表达式。
    Resource getResource(String location);

    // 返回ResourceLoader使用的ClassLoader。
    ClassLoader getClassLoader();
}
```

ResourceLoader接口具体实现类：

![ResourceLoader接口具体实现类](http://i4.piimg.com/598959/d9358fc8d72245e0.png)

> 用Resource操作文件时，如果资源配置文件在项目发布时会被打包到JAR中，那么不能使用Resource#getFile()方法，否则会抛出FileNotFoundException。但可以使用Resource#getInputStream()方法读取。

## BeanFactory和ApplicationContext

Spring通过一个配置文件描述Bean及Bean之间的依赖关系，利用Java语言的反射功能实例化Bean并建立Bean之间的依赖关系。Spring的IoC容器在完成这些底层工作的基础上，还提供了Bean实例缓存、生命周期管理、Bean实例代理、事件发布、资源装载等高级服务。

Bean工厂是Spring框架最核心的接口，它提供了高级IoC的配置机制。BeanFactory使管理不同类型的Java对象成为可能，应用上下文建立在BeanFactory基础之上，提供了更多面向应用的功能，它提供了国际化支持和框架事件体系，更易于创建实际应用。我们一般称BeanFactory为IoC容器，而称ApplicationContext为应用上下文。

### BeanFactory

BeanFactory是类的通用工厂，它可以创建并管理各种类的对象。这些可被创建和管理的对象本身没有什么特别之处，仅是一个POJO，Spring称这些被创建和管理的Java对象为Bean。

#### BeanFactory的类体系结构

![BeanFactory的类体系结构](http://i1.buimg.com/598959/0dcd523cf7c35026.png)

### ApplicationContext

ApplicationContext由BeanFactory派生而来，提供了更多面向实际应用的功能。在BeanFactory中，很多功能需要以编程的方式实现，而在ApplicationContext中则可以通过配置的方式实现。

**ApplicationContext的类体系结构**：

![ApplicationContext的类体系结构](http://i1.buimg.com/598959/52daeea831a3fc43.png)

### 父子容器

通过HierarchicalBeanFactoryj接口，Spring的IoC容器可以建立父子层级关联的容器体系，子容器可以访问父容器中的Bean，但父容器不能访问子容器中的Bean。在容器内，Bean的id必须是唯一的，但子容器可以拥有一个和父容器id相同的Bean。父子容器层级体系增强了Spring容器架构的扩展性和灵活性，因为第三方可以通过编程的方式为一个已经存在的容器添加一个或多个特殊用途的子容器，以提供一些额外的功能。

### Bean的声明周期

在Spring中，可以从两个层面定义Bean的声明周期：第一个层面是Bean的作用范围；第二个层面是实例化Bean时所经历的一系列阶段。

Bean的完整声明周期从Spring容器着手实例化Bean开始，直到最终销毁Bean。可以将这些方法大致划分为4类：

* Bean自身的方法
* Bean级生命周期接口方法
* 容器级生命周期接口方法
* 工厂后处理器接口方法：工厂后处理器也是容器级的，在应用上下文装配配置文件后立即调用。

> Bean级生命周期接口和容器生命周期接口是个性和共性辩证统一思想的体现，前者解决Bean个性化处理的问题，而后者解决容器中某些Bean共性化处理的问题。

ApplicationContextAware

```java
public interface ApplicationContextAware extends Aware {

    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```

如果在配置文件中声明了工厂后处理器接口BeanFactoryPostProcessor的实现类，则应用上下文在装载配置文件之后，初始化Bean实例之前将调用这些BeanFactoryPostProcessor对配置信息进行加工处理。

工厂后处理器是容器级的，仅在应用上下文初始化时调用一次，其目的是完成一些配置文件的加工处理工作。

ApplicationContext和BeanFactory另一个最大的不同之处在于：前者会利用Java反射机制自动识别出配置文件中定义的BeanPostProcessor、InstantiationAwareBeanPostProcessor和BeanFactoryPostProcessor，并自动将它们注册到应用上下文中；而后者需要在代码中通过手工调用addBeanPostProcessor()方法进行注册。

Bean配置信息是Bean的元数据信息，它由以下4个方面组成：

* Bean的实现类
* Bean的属性信息
* Bean的依赖关系
* Bean的行为配置

Spring容器内部协作解构

![Spring容器内部协作解构](http://i1.buimg.com/598959/3181c7f77ec8fed4.png)

Bean配置信息首先定义了Bean的实现及依赖关系，Spring容器根据各种形式的Bean配置信息在容器内部建立Bean定义注册表；然后后根据注册表加载、实例化Bean，并建立Bean和Bean之间的依赖关系；最后将这些准备就绪的Bean放到Bean缓存池中，以提供外层的应用程序进行调用。

### Bean作用域

| 类型            | 说明  |
|---------------|-----|
| singleton     | B1  |
| prototype     | B2  |
| request       | B3  |
| session       | B3  |
| globalSession | B3  |

### FactoryBean

### Groovy DSL

### Spring扩展自定义标签

在Spring中，自定义组件标签非常方便，只需经过以下几个步骤：

1. 采用XSD描述自定义标签的元素属性；

2. 编写Bean定义的解析器

3. 注册自定义标签解析器

4. 绑定命名空间解析器

### 内部工作机制

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        // 加载配置文件
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // 注册bean处理器拦截bean创建
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

ApplicationContextAwareProcessor

```java
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        customizeBeanFactory(beanFactory);
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException(
                "I/O error parsing XML document for application context [" + getDisplayName() + "]", ex);
    }
}
```

#### DefaultListableBeanFactory

![DefaultListableBeanFactory类继承结构图](http://i1.buimg.com/598959/7cffa4a3e4d89258.png)

DocumentBuilderFactory

BeanDefinitionParserDelegate

#### BeanDefinition

![BeanDefinition类继承结构图](http://i1.buimg.com/598959/3c1da40c9ed57511.png)

RootBeanDefinition是最常用的实现类，它对应一般性的`<bean>`标签。在配置文件中可以定义父`<bean>`和子`<bean>`，父`<bean>`用RootBeanDefinition表示，子`<bean>`用ChildBeanDefinition表示，而没有父`<bean>`的`<bean>`则用RootBeanDefinition表示。

#### BeanDefinitionRegistry

#### InstantiationStrategy

![InstantiationStrategy](http://i1.buimg.com/598959/981da970f5092e86.png)

InstantiationStrategy仅负责实例化Bean的操作，相当于执行Java语言中new的功能，它并不会参与Bean属性的设置工作。

#### BeanWrapper

![BeanWrapper](http://i4.piimg.com/598959/42708e81ebf91859.png)

### 属性编辑器

#### JavaBean的编辑器

JavaBean规范通过java.beans.PropertyEditor定义了设置JavaBean属性的方法，通过BeanInfo描述了
JavaBean的那些属性是可定制的，此外还描述了可定制属性与PropertyEditor的对应关系。

* PropertyEditor

```java
public interface PropertyEditor {

    void setValue(Object value);

    Object getValue();

    boolean isPaintable();

    void paintValue(java.awt.Graphics gfx, java.awt.Rectangle box);

    String getJavaInitializationString();

    String getAsText();

    void setAsText(String text) throws java.lang.IllegalArgumentException;

    String[] getTags();

    java.awt.Component getCustomEditor();

    boolean supportsCustomEditor();

    void addPropertyChangeListener(PropertyChangeListener listener);

    void removePropertyChangeListener(PropertyChangeListener listener);

}
```

* BeanInfo

```java
public interface BeanInfo {

    BeanDescriptor getBeanDescriptor();

    EventSetDescriptor[] getEventSetDescriptors();

    int getDefaultEventIndex();

    PropertyDescriptor[] getPropertyDescriptors();

    int getDefaultPropertyIndex();

    MethodDescriptor[] getMethodDescriptors();

    BeanInfo[] getAdditionalBeanInfo();

    Image getIcon(int iconKind);

}
```

### PropertyEditorSupport

### 自定义属性编辑器

Spring的大部分默认属性编辑器都直接扩展与java.beans.PropertyEditorSupport类，开发者也可以扩展PropertyEditorSupport实现自己的属性编辑器。

注册自定义属性编辑器
CustomEditorConfigurer

#### 使用外部属性文件

PropertyPlaceholderConfigurer

#### 使用加密的属性文件

对于敏感信息，一般情况下则希望以密文的方式保存。PropertyPlaceholderConfigurer继承自PropertyResourceConfigurer类，后者有几个有用的protected方法，用于在属性使用之前对属性列表中
的属性进行转换。

* void convertProperties(Properties props)：属性文件中的所有属性值都封装在props中，覆盖此方法，可以对所有属性值进行转换处理。

* String convertProperty(String propertyName, String propertyValue)：在加载属性文件并读取文件中的每个属性时，都会调用此方法进行转换处理。

* String convertPropertyValue(String originalValue)：和上一个方法类似，只不过没有传入属性名。

在默认情况下，这3个方法内部方法都是空的，即不会对属性值进行任何转换。可以扩展PropertyPlaceholderConfigurer，覆盖相应的属性转换方法，就可以支持加密版的属性文件了。

### 国际化信息

“国际化信息”也称为“本地化信息”，一般需要两个条件才可以确定一个特定类型的本地化信息，分别是“语言类型”
和“国家/地区类型”。

#### Locale

java.util.Locale是表示语言和国家/地区信息的本地化类，它是创建国际化应用的基础。

#### 本地化工具类

JDK的java.util包中提供了几个支持本地化的格式化操作工具类，如NumberFormat、
DateFormat、MessageFormat。

MessageFormat在NumberFormat和DateFormat的基础上提供了强大的占位符字符串的格式化功能，
支持时间、货币、数字及对象属性的格式化操作。

#### ResourceBoundle

如果应用系统中的某些信息需要支持国际化功能，则必须为期望支持的不同本地化类型分别提供对应的资源文件，并以规范的方式进行命名。国际化资源文件的命名规范规定资源名称采用以下方式进行命名：

`<资源名>` _ `<语言代码>`_`<国家/地区代码>`.properties

其中，语言代码和国家/地区代码都是可选的。`<资源名>`.properties命名的国际化资源文件是默认的资源文件。

ResourceBoundle在加载资源时，如果指定的本地化资源文件不存在，则按以下顺序尝试加载其他资源：本地系统默认本地化对象对应的资源--->默认的资源。如果这些资源文件都不存在，则将抛出java.util.MissingResourceException异常。

#### MessageSource

Spring定义了访问国际化信息的MessageSource接口，并提供了若干个易用的实现类。

```java
public interface MessageSource {

    /**
        * Try to resolve the message. Return default message if no message was found.
        * @param code the code to lookup up, such as 'calculator.noRateSet'. Users of
        * this class are encouraged to base message names on the relevant fully
        * qualified class name, thus avoiding conflict and ensuring maximum clarity.
        * @param args an array of arguments that will be filled in for params within
        * the message (params look like "{0}", "{1,date}", "{2,time}" within a message),
        * or {@code null} if none.
        * @param defaultMessage a default message to return if the lookup fails
        * @param locale the locale in which to do the lookup
        * @return the resolved message if the lookup was successful;
        * otherwise the default message passed as a parameter
        * @see java.text.MessageFormat
        */
    String getMessage(String code, Object[] args, String defaultMessage, Locale locale);

    /**
        * Try to resolve the message. Treat as an error if the message can't be found.
        * @param code the code to lookup up, such as 'calculator.noRateSet'
        * @param args an array of arguments that will be filled in for params within
        * the message (params look like "{0}", "{1,date}", "{2,time}" within a message),
        * or {@code null} if none.
        * @param locale the locale in which to do the lookup
        * @return the resolved message
        * @throws NoSuchMessageException if the message wasn't found
        * @see java.text.MessageFormat
        */
    String getMessage(String code, Object[] args, Locale locale) throws NoSuchMessageException;

    /**
        * Try to resolve the message using all the attributes contained within the
        * {@code MessageSourceResolvable} argument that was passed in.
        * <p>NOTE: We must throw a {@code NoSuchMessageException} on this method
        * since at the time of calling this method we aren't able to determine if the
        * {@code defaultMessage} property of the resolvable is {@code null} or not.
        * @param resolvable the value object storing attributes required to resolve a message
        * @param locale the locale in which to do the lookup
        * @return the resolved message
        * @throws NoSuchMessageException if the message wasn't found
        * @see java.text.MessageFormat
        */
    String getMessage(MessageSourceResolvable resolvable, Locale locale) throws NoSuchMessageException;

}
```

![MessageSource的类继承结构图](http://i4.piimg.com/598959/f3282f33074832bc.png)