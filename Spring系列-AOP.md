---
title: Spring AOP
date: 2017-06-22 15:17:36
tags: Spring
---
# Spring AOP

* AOP基本知识
* Spring Aop示例
* Spring Aop原理

<!-- more -->

## 1、AOP concepts（AOP术语）

- Aspect/Advisors（切面）
 一个关注点的模块化，这个关注点可能会横切多个对象。在Spring AOP中，切面可以使用基于模式或者基于@Aspect注解的方式来实现。

- Join point（连接点）
 在程序执行期间的一点。在Spring AOP中，连接点总是表示方法执行。

- Advice（通知）
  在切面的某个特定的连接点上执行的动作。许多AOP框架（包括Spring）都是以拦截器做通知模型，并维护一个以连接点为中心的拦截器链。
 
- Pointcut（切入点）
  查找连接点的条件。通知和一个切入点表达式关联，并在满足这个切入点的连接点上运行。

- Introduction（引入）
  给一个类型声明额外的方法或属性。Spring允许引入新的接口（以及一个对应的实现）到任何被代理的对象。  

- Target object（目标对象）
   被一个或者多个切面所通知的对象。也被称做被通知（advised）对象。 既然Spring AOP是通过运行时代理实现的，这个对象永远是一个被代理（proxied）对象。

- AOP proxy
  AOP框架创建的对象，用来实现切面契约（例如通知方法执行等等）。在Spring中，AOP代理可以是JDK动态代理或者CGLIB代理。

- Weaving（织入）
  织入是一个过程，是将切面应用到目标对象从而创建出AOP代理对象的过程，织入可以在编译期、类装载期、运行期进行。

## 1.1 通知类型

- Before advice（前置通知）：在某连接点之前执行的通知，但这个通知不能阻止连接点之前的执行流程（除非它抛出一个异常）。

- After returning advice（后置通知）：在某连接点正常完成后执行的通知：例如，一个方法没有抛出任何异常，正常返回。

- After throwing advice（异常通知）：在方法抛出异常退出时执行的通知。

- After (finally) advice（最终通知）：当某连接点退出的时候执行的通知（不论是正常返回还是异常退出）。

- Around Advice（环绕通知）：包围一个连接点的通知，如方法调用。这是最强大的一种通知类型。环绕通知可以在方法调用前后完成自定义的行为。它也会选择是否继续执行连接点或直接返回它自己的返回值或抛出异常来结束执行。

## 2、Spring AOP

1、Spring AOP使用纯Java实现，它不需要专门的编译过程。Spring AOP不需要控制类加载器层次结构，因此适用于Servlet容器或应用程序服务器。

2、Spring AOP目前仅支持方法执行连接点。

3、Spring实现AOP的方法跟其他的框架不同。Spring并不是要提供最完整的AOP实现（尽管Spring AOP有这个能力），相反的，它其实侧重于提供一种AOP实现和Spring IoC容器之间的整合，用于帮助解决在企业级开发中的常见问题。 

4、Spring AOP从来没有打算通过提供一种全面的AOP解决方案来与AspectJ竞争。我们相信无论是基于代理（proxy-based）的框架如Spring AOP或者是成熟的框架如AspectJ都是很有价值的，他们之间应该是互补而不是竞争的关系。

### 2.1、Spring AOP基于XML的应用程序

1、Jar包依赖

```xml
<dependencies>

    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
    </dependency>

    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjweaver</artifactId>
    </dependency>
    </dependencies>
```

2、定义切面和需要被拦截的对象

```java
public class Student {

    private Integer age;
    private String name;

    public void setAge(Integer age) {
        this.age = age;
    }
    public Integer getAge() {
        System.out.println("Age : " + age );
        return age;
    }

    public void setName(String name) {
        this.name = name;
    }
    public String getName() {
        System.out.println("Name : " + name );
        return name;
    }

    public void printThrowException(){
        System.out.println("Exception raised");
        throw new IllegalArgumentException();
    }
}
```

```java
public class Logging {

    public void beforeAdvice(){
        System.out.println("beforeAdvice.");
    }

    public void afterAdvice(){
        System.out.println("afterAdvice");
    }

    public void afterReturningAdvice(Object retVal){
        System.out.println("afterReturningAdvice:" + retVal.toString() );
    }

    public void afterThrowingAdvice(IllegalArgumentException ex){
        System.out.println("afterThrowingAdvice: " + ex.toString());
    }

}
```

3、配置XML

```xml
<aop:config>
   <aop:aspect id="log" ref="logging">
     <aop:pointcut id="all" expression="execution(* com.codersm.study.spring.aop.*.*(..))"/>
     <aop:before method="beforeAdvice" pointcut-ref="all"/>
     <aop:after method="afterAdvice" pointcut-ref="all"/>
     <aop:after-returning method="afterReturningAdvice" pointcut-ref="all" returning="retVal"/>
     <aop:after-throwing method="afterThrowingAdvice" pointcut-ref="all" throwing="ex"/>
   </aop:aspect>
</aop:config>

<bean id="student" class="com.codersm.study.spring.aop.Student">
    <property name="name" value="zhangsan"/>
    <property name="age" value="21"/>
</bean>

<bean id="logging" class="com.codersm.study.spring.aop.Logging"/>
```

4、测试

```java
ApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring-aop.xml");
    Student student = (Student) context.getBean("student");
    student.getAge();
```

### 2.3、Spring AOP基于@Aspect的应用程序

1、定义切面

```java

@Aspect
@Component
public class LoggingAspect {

    /**
     * 单独定义切入点，可复用
     */
    @Pointcut("execution(* com.codersm.study.spring.aop.*.*(..))")
    public void pointcut() {
    }

    @Before(value = "pointcut()")
    public void before() {
        System.out.println("Before advice");
    }


    @Around(value = "pointcut()")
    public void around(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        System.out.println("Around advice begin");
        Object ret = proceedingJoinPoint.proceed();
        System.out.println("Around advice end,execute method result is " + ret);
    }

    @After(value = "pointcut()")
    public void after() {
        System.out.println("After advice");
    }

    @AfterThrowing(value = "pointcut()", throwing = "ex")
    public void afterThrowing(Throwable ex) {
        System.out.println("afterThrowing advice exception is " + ex);
    }

    @AfterReturning(value = "pointcut()", returning = "ret")
    public void AfterReturning(Object ret) {
        System.out.println("AfterReturning advice result is :" + ret);
    }
}
```

2、配置xml文件开启@Aspect

```xml
 <context:component-scan base-package="com.codersm.study.spring.*"/>
 <aop:aspectj-autoproxy/>
```

3、测试

```java
 ApplicationContext applicationContext = null;

    @Before
    public void before() {
        applicationContext = 
            new ClassPathXmlApplicationContext("classpath:spring-aop-annotation.xml");
    }

    @Test
    public void testAop() {
        Student student = (Student) applicationContext.getBean("student");
        student.setName("hello world");
        System.out.println("---------------------------------");
        student.printThrowException();
    }
```

### 2.4、 通知类型小结

|通知|	描述|
|--------|--------|
|前置通知	|权限控制(少用)|
|后置通知	|少用|
|环绕通知	|权限控制/性能监控/缓存实现/事务管理|
|异常通知	|发生异常后,记录错误日志|
|最终通知	|释放资源|

## 3、获取通知参数

- 任何通知声明JoinPoint作为通知方法第一个参数，JoinPoint提供一些有用的方法。

**around advice is required to declare a first parameter of type ProceedingJoinPoint, which is a subclass of JoinPoint.**

** ProceedingJoinPoint is only supported for around advice.**

- 传递参数给通知
 To make argument values available to the advice body, you can use the binding form of args.

```java

@Before("execution(* com.codersm.study.spring.aop.*.*(..)) && args(name,..)")
public void before(String name) {
        System.out.println("Before advice,name = " + name);
}
```

另外一种定义方式：

```java
@Pointcut("execution(* com.codersm.study.spring.aop.*.*(..))  && args(name,..)")
public void pointcut(String name) {
}
@Before(value = "pointcut(name)")
public void before(String name) {
       System.out.println("Before advice,name = " + name);
}
```

## 4、AOP proxies

### 4.1、 AOP介绍

Spring AOP使用JDK动态代理或CGLIB创建目标类的代理对象，如果目标类实现了至少一个接口，则使用JDK动态代理；否则，使用CGLIB代理。如果强制使用CGLIB代理，需要考虑这些问题：

- final methods cannot be advised, as they cannot be overridden.

- As of Spring 3.2, it is no longer necessary to add CGLIB to your project classpath, as CGLIB classes are repackaged under org.springframework and included directly in the spring-core JAR. This means that CGLIB-based proxy support 'just works' in the same way that JDK dynamic proxies always have.

- As of Spring 4.0, the constructor of your proxied object will NOT be called twice anymore since the CGLIB proxy instance will be created via Objenesis. Only if your JVM does not allow for constructor bypassing, you might see double invocations and corresponding debug log entries from Spring’s AOP support.

### 4.2、理解AOP代理

any method calls that it may make on itself, such as this.bar() or this.foo(), are going to be invoked against the this reference, and not the proxy. This has important implications. It means that self-invocation is not going to result in the advice associated with a method invocation getting a chance to execute.

**solution**:

-  refactor your code such that the self-invocation does not happen. 

-  You can (choke!) totally tie the logic within your class to Spring AOP by doing this

```java
public class SimplePojo implements Pojo {

    public void foo() {
        // this works, but... gah!
        ((Pojo) AopContext.currentProxy()).bar();
    }

    public void bar() {
        // some logic...
    }
}
```

```java
 public static void main(String[] args) {

        ProxyFactory factory = new ProxyFactory(new SimplePojo());
        factory.adddInterface(Pojo.class);
        factory.addAdvice(new RetryAdvice());
        factory.setExposeProxy(true);

        Pojo pojo = (Pojo) factory.getProxy();

        // this is a method call on the proxy!
        pojo.foo();
    }
```

## 5、AOP源码分析

spring.handlers

```xml
http\://www.springframework.org/schema/aop=org.springframework.aop.config.AopNamespaceHandler
```

```java
public class AopNamespaceHandler extends NamespaceHandlerSupport {

	/**
	 * Register the {@link BeanDefinitionParser BeanDefinitionParsers} for the
	 * '{@code config}', '{@code spring-configured}', '{@code aspectj-autoproxy}'
	 * and '{@code scoped-proxy}' tags.
	 */
	@Override
	public void init() {
		// In 2.0 XSD as well as in 2.1 XSD.
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());

		// Only in 2.0 XSD: moved to context namespace as of 2.1
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}

}
```

ConfigBeanDefinitionParser.parse( )方法：

```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
		CompositeComponentDefinition compositeDef =
				new CompositeComponentDefinition(element.getTagName(), parserContext.extractSource(element));
		parserContext.pushContainingComponent(compositeDef);

		configureAutoProxyCreator(parserContext, element);

		List<Element> childElts = DomUtils.getChildElements(element);
		for (Element elt: childElts) {
			String localName = parserContext.getDelegate().getLocalName(elt);
			if (POINTCUT.equals(localName)) {
				parsePointcut(elt, parserContext);
			}
			else if (ADVISOR.equals(localName)) {
				parseAdvisor(elt, parserContext);
			}
			else if (ASPECT.equals(localName)) {
				parseAspect(elt, parserContext);
			}
		}

		parserContext.popAndRegisterContainingComponent();
		return null;
	}
```

**configureAutoProxyCreator(parserContext, element)**:

```java
private void configureAutoProxyCreator(ParserContext parserContext, Element element) {
		AopNamespaceUtils.registerAspectJAutoProxyCreatorIfNecessary(parserContext, element);
	}
```

```java
	public static void registerAspectJAutoProxyCreatorIfNecessary(
			ParserContext parserContext, Element sourceElement) {

		BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAutoProxyCreatorIfNecessary(
				parserContext.getRegistry(), parserContext.extractSource(sourceElement));
		useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
		registerComponentIfNecessary(beanDefinition, parserContext);
	}
```

Spring对XML文件解析

```java
/**
	 * Parse the elements at the root level in the document:
	 * "import", "alias", "bean".
	 * @param root the DOM root element of the document
	 */
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
```

```java
public static BeanDefinition registerAspectJAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object source) {
		return registerOrEscalateApcAsRequired(AspectJAwareAdvisorAutoProxyCreator.class, registry, source);
	}
```

AspectJAwareAdvisorAutoProxyCreator

![AspectJAwareAdvisorAutoProxyCreator层次结构.png](http://upload-images.jianshu.io/upload_images/2889214-9f2cf7dfc88178a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/900)

```java
public interface BeanPostProcessor {

	Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

	Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

> BeanPostProcessor接口定义回调方法，允许修改新的实例化Bean，例如检查标记接口或用代理进行包装。 

- postProcessBeforeInitialization

在任何bean初始化回调（如InitializingBean的afterPropertiesSet或自定义init方法）之前，将此BeanPostProcessor应用于给定的新bean实例。

- postProcessAfterInitialization

在任何bean初始化回调之后，将此BeanPostProcessor应用于给定的新Bean实例（如InitializingBean的afterPropertiesSet或自定义init方法）。

> ApplicationContext 会自动检测由 BeanPostProcessor 接口的实现定义的 bean，注册这些 bean 为后置处理器，然后通过在容器中创建 bean，在适当的时候调用它。

AspectJAwareAdvisorAutoProxyCreator这两个方法的实现：

```java
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean;
	}

    @Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (!this.earlyProxyReferences.contains(cacheKey)) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

继续跟踪源码，发现了**createProxy方法**：

```java
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```

**createProxy方法**:

```java
protected Object createProxy(
			Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}

		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

		if (!proxyFactory.isProxyTargetClass()) {
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		for (Advisor advisor : advisors) {
			proxyFactory.addAdvisor(advisor);
		}

		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		return proxyFactory.getProxy(getProxyClassLoader());
	}
```

创建AopProxy代理对象，具体流程：

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```

> ObjenesisCglibAopProxy继承CglibAopProxy。方法调用原理可以查看CglibAopProxy和JdkDynamicAopProxy。