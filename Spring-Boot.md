---
title: Spring Boot
date: 2017-06-24 19:31:16
tags: Spring Boot
---
# Spring Boot

Spring Boot是由Pivotal团队设计的全新框架，其目的是用来简化Spring应用开发过程。
该框架使用了特定的方式来进行配置，从而使得开发人员不再需要定义一系列样板化的配置文件
，而专注于核心业务开发，项目涉及的一些基础设施则交由Spring Boot来解决。

Spring Boot借助Groovy动态语言实现的，如借助Groovy强大的MetaObject协议、可插拔的AST转换器及内置的依赖解决方案引擎等。

## Spring Boot特点

Spring Boot的作用在于创建和启动新的基于Spring框架的项目，其目的是帮助开发人员快速构建出基于Spring
的应用。

Spring Boot包含如下特性：

* 为开发者提供Spring快速入门体验。

* 内嵌Tomcat和Jetty容器，不需要部署WAR文件到Web容器就可独立运行应用。

* 提供许多基于Maven的pom配置模板来简化工程配置。

* 提供实现自动化配置的基础设施。

* 提供可以直接在生产环境中使用的功能，如性能指标，应用信息和应用健康检查。

* 开箱即用，没有代码生成，也无须XML配置文件，支持修改默认值来满足特定需求。

## Structuring your code（构建程序）

**Using the “default” package**

Spring Boot不需要任何特定的代码布局，但是有一些最佳实践可以帮助您。

通常不鼓励使用“默认包”，应该避免使用。对于使用@ComponentScan，@EntityScan或@SpringBootApplication注释的Spring Boot应用程序，可能会引起特殊问题，因为每个jar的每个类都将被读取。

**Locating the main application class**

我们通常建议您将主应用程序类定位到其他类之上的根包中。

@EnableAutoConfiguration注释通常放置在您的主类上，它隐式定义了某些项目的基本“搜索包”。例如，如果您正在编写JPA应用程序，则@EnableAutoConfiguration注释类的包将用于搜索@Entity项。

使用根包还可以使用@ComponentScan注释，而不需要指定basePackage属性。如果您的主类在根包中，也可以使用@SpringBootApplication注释。

## Configuration classes

Spring Boot支持基于Java的配置。虽然可以使用XML源调用SpringApplication.run()，但我们通常建议您的主源是@Configuration类。通常，定义main方法的类也是一个很好的候选者，作为主要的@Configuration。

**Importing additional configuration classes**

您不需要将所有的@Configuration放在一个类中。@Import注释可用于导入其他配置类。或者，您可以使用@ComponentScan自动拾取所有Spring组件，包括@Configuration类。

**Importing XML configuration**

如果您绝对必须使用基于XML的配置，我们建议您仍然从@Configuration类开始。然后，您可以使用额外的@ImportResource注释来加载XML配置文件。

## Auto-configuration

Spring Boot自动配置尝试根据您添加的jar依赖关系自动配置您的Spring应用程序。例如，如果HSQLDB在您的类路径上，并且您没有手动配置任何数据库连接bean，那么我们将自动配置内存数据库。

您需要通过将@EnableAutoConfiguration或@SpringBootApplication注释添加到您的一个@Configuration类中来选择自动配置。

> If you need to find out what auto-configuration is currently being applied, and why, start your application with the --debug switch. This will enable debug logs for a selection of core loggers and log an auto-configuration report to the console.

**Disabling specific auto-configuration**

如果您发现正在应用不需要的特定自动配置类，可以使用@EnableAutoConfiguration的exclude属性来禁用它们。

```java
import org.springframework.boot.autoconfigure.*;
import org.springframework.boot.autoconfigure.jdbc.*;
import org.springframework.context.annotation.*;

@Configuration
@EnableAutoConfiguration(exclude={DataSourceAutoConfiguration.class})
public class Configuration {
}
```

如果类不在类路径上，则可以使用注释的excludeName属性，并指定完全限定名称。最后，您还可以通过spring.autoconfigure.exclude属性控制要排除的自动配置类列表。

## Spring Beans and dependency injection

您可以自由使用任何标准的Spring Framework技术来定义您的bean及其注入的依赖关系。为了简单起见，我们经常发现使用@ComponentScan找到你的bean，结合@Autowired构造函数注入效果很好。

**Using the @SpringBootApplication annotation**

许多Spring Boot开发人员总是将其主类注释为@Configuration，@EnableAutoConfiguration和@ComponentScan。由于这些注释经常一起使用（特别是如果您遵循上述最佳实践），Spring Boot提供了一个方便的@SpringBootApplication替代。

@SpringBootApplication注释等效于使用@Configuration，@EnableAutoConfiguration和@ComponentScan及其默认属性。

## Hot swapping（热插拔）

由于Spring Boot应用程序只是纯Java应用程序，所以JVM热插拔应该开箱即用。JVM热插拔在一定程度上受到可以替代的字节码的限制，为了更完整的解决方案JRebel或者可以使用Spring Loaded项目。

spring-boot-devtools模块还支持快速重新启动应用程序。

## Spring Boot features

### SpringApplication

SpringApplication类提供了一种方便的方法来引导将从main（）方法启动的Spring应用程序。在许多情况下，您只需委派静态SpringApplication.run方法即可：

```java
public static void main(String[] args) {
    SpringApplication.run(MySpringConfiguration.class, args);
}
```

> 如果启动失败，注册的FailureAnalyzers有机会提供专门的错误消息和具体的措施来解决问题;如果没有故障分析仪能够处理异常，您仍然可以显示完整的自动配置报告，以更好地了解出现的问题。为此，您需要启用调试属性或启用对于org.springframework.boot.autoconfigure.logging.AutoConfigurationReportLoggingInitializer的DEBUG日志记录。

**Customizing the Banner** 【*】

The banner that is printed on start up can be changed by adding a banner.txt file to your classpath, or by setting banner.location to the location of such a file. If the file has an unusual encoding you can set banner.charset (default is UTF-8). In addition to a text file, you can also add a banner.gif, banner.jpg or banner.png image file to your classpath, or set a banner.image.location property. Images will be converted into an ASCII art representation and printed above any text banner.

**Customizing SpringApplication** 【*】

**Fluent builder API**

If you need to build an ApplicationContext hierarchy (multiple contexts with a parent/child relationship), or if you just prefer using a ‘fluent’ builder API, you can use the SpringApplicationBuilder.

The SpringApplicationBuilder allows you to chain together multiple method calls, and includes parent and child methods that allow you to create a hierarchy.

```java
new SpringApplicationBuilder()
        .sources(Parent.class)
        .child(Application.class)
        .bannerMode(Banner.Mode.OFF)
        .run(args);
```

**Application events and listeners**

除了常见的Spring Framework事件（例如ContextRefreshedEvent）之外，SpringApplication还会发送一些其他应用程序事件。

> Some events are actually triggered before the ApplicationContext is created so you cannot register a listener on those as a @Bean. You can register them via the SpringApplication.addListeners(…​) or SpringApplicationBuilder.listeners(…​) methods.
>
> If you want those listeners to be registered automatically regardless of the way the application is created you can add a META-INF/spring.factories file to your project and reference your listener(s) using the org.springframework.context.ApplicationListener key.

随着应用程序的运行，应用程序事件按照以下顺序发送：

1. An ApplicationStartingEvent is sent at the start of a run, but before any processing except the registration of listeners and initializers.
2. An ApplicationEnvironmentPreparedEvent is sent when the Environment to be used in the context is known, but before the context is created.
3. An ApplicationPreparedEvent is sent just before the refresh is started, but after bean definitions have been loaded.
4. An ApplicationReadyEvent is sent after the refresh and any related callbacks have been processed to indicate the application is ready to service requests.
5. An ApplicationFailedEvent is sent if there is an exception on startup.

**Web environment**

A SpringApplication will attempt to create the right type of ApplicationContext on your behalf. By default, an AnnotationConfigApplicationContext or AnnotationConfigEmbeddedWebApplicationContext will be used, depending on whether you are developing a web application or not.

The algorithm used to determine a ‘web environment’ is fairly simplistic (based on the presence of a few classes). You can use setWebEnvironment(boolean webEnvironment) if you need to override the default.

It is also possible to take complete control of the ApplicationContext type that will be used by calling setApplicationContextClass(…​).

**Accessing application arguments**

If you need to access the application arguments that were passed to SpringApplication.run(…​) you can inject a org.springframework.boot.ApplicationArguments bean.

```java
import org.springframework.boot.*
import org.springframework.beans.factory.annotation.*
import org.springframework.stereotype.*

@Component
public class MyBean {

    @Autowired
    public MyBean(ApplicationArguments args) {
        boolean debug = args.containsOption("debug");
        List<String> files = args.getNonOptionArgs();
        // if run with "--debug logfile.txt" debug=true, files=["logfile.txt"]
    }

}
```

**Using the ApplicationRunner or CommandLineRunner**

一旦SpringApplication启动，您需要运行一些特定的代码，就可以实现ApplicationRunner或CommandLineRunner接口。

您可以额外实现org.springframework.core.Ordered接口，或者使用org.springframework.core.annotation.Order注释，如果定义了若干CommandLineRunner或ApplicationRunner bean，这些bean必须按特定顺序调用。

**Application exit**

每个SpringApplication将注册一个与JVM关联的关闭钩子，以确保ApplicationContext在退出时正常关闭。

另外，如果在调用SpringApplication.exit（）时希望返回一个特定的退出代码，bean可以实现org.springframework.boot.ExitCodeGenerator接口。

此外，ExitCodeGenerator接口可能会由异常实现。遇到这样的异常时，Spring Boot将返回由实现的getExitCode（）方法提供的退出代码。

**Admin features（管理功能）**【*】

## Externalized Configuration（外部配置）

Spring Boot允许您外部化您的配置，以便您可以在不同的环境中使用相同的应用程序代码。您可以使用属性文件，YAML文件，环境变量和命令行参数来外部化配置。

可以使用@Value注释将属性值直接注入到您的bean中，该注释可通过Spring环境抽象访问，或通过@ConfigurationProperties绑定到结构化对象。

Spring Boot使用非常特殊的PropertySource命令，旨在允许明智地覆盖值。属性按以下顺序考虑：

**Profile-specific properties**

**Type-safe Configuration Properties**

**Developing web applications**


## Spring Boot核心

### 入口类和@SpringBootApplication

### 定制Banner

### Spring Boot的配置文件

Spring Boot使用一个全局的配置文件application.properties或application.yml，放置在src/main/resources目录或者类路径的/config下。

Spring Boot的全局配置文件的作用是对一些默认配置的配置值进行修改。


@ConfigurationProperties

## Spring Boot运行原理


## 添加过滤器

![spring-boot使用教程(二)：增加自定义filter（新）](http://www.jianshu.com/p/05c8be17c80a)

![Spring boot 注册Servlet和Filter](https://www.pocketdigi.com/20170527/1576.html)