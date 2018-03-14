---
title: Spring Validation
date: 2017-06-21 15:59:26
tags: Spring
---

 Spring Validation
<!-- more  -->

Spring Framework 4.0 supports Bean Validation 1.0 (JSR-303) and Bean Validation 1.1 (JSR-349) in terms of setup support, also adapting it to Spring’s Validator interface.

Specifically validation should not be tied to the web tier, should be easy to localize and it should be possible to plug in any validator available. Considering the above, Spring has come up with a Validator interface that is both basic and eminently usable in every layer of an application.

Data binding is useful for allowing user input to be dynamically bound to the domain model of an application (or whatever objects you use to process user input). Spring provides the so-called DataBinder to do exactly that. The Validator and the DataBinder make up the validation package, which is primarily used in but not limited to the MVC framework.

- Validator

```java
public interface Validator {

    boolean supports(Class<?> clazz);

    void validate(Object target, Errors errors);
}
```

- DataBinder

- BeanWrapper

```java
public interface BeanWrapper extends ConfigurablePropertyAccessor {

    void setAutoGrowCollectionLimit(int autoGrowCollectionLimit);

    int getAutoGrowCollectionLimit();

    Object getWrappedInstance();

    Class<?> getWrappedClass();

    PropertyDescriptor[] getPropertyDescriptors();

    PropertyDescriptor getPropertyDescriptor(String propertyName) throws InvalidPropertyException;

}
```

- PropertyEditor

Spring uses the concept of PropertyEditors to effect the conversion between an Object and a String.

> PropertyEditorManager

- ValidationUtils

- MessageSource

- MessageCodesResolver

## To learn how to setup a Bean Validation provider as a Spring bean

- Configuring a Bean Validation Provider

Spring provides full support for the Bean Validation API.This allows for a javax.validation.ValidatorFactory or javax.validation.Validator to be injected wherever validation is needed in your application.

 Use the LocalValidatorFactoryBean to configure a default Validator as a Spring bean:

```xml
  <bean id="validator"
    class="org.springframework.validation.beanvalidation.LocalValidatorFactoryBean"/>
```

The basic configuration above will trigger Bean Validation to initialize using its default bootstrap mechanism. A JSR-303/JSR-349 provider, such as Hibernate Validator, is expected to be present in the classpath and will be detected automatically.

- Injecting a Validator

LocalValidatorFactoryBean implements both javax.validation.ValidatorFactory and javax.validation.Validator, as well as Spring’s org.springframework.validation.Validator. You may inject a reference to either of these interfaces into beans that need to invoke validation logic.

## Configuring Custom Constraints

Each Bean Validation constraint consists of two parts. First, a @Constraint annotation that declares the constraint and its configurable properties. Second, an implementation of the javax.validation.ConstraintValidator interface that implements the constraint’s behavior. To associate a declaration with an implementation, each @Constraint annotation references a corresponding ValidationConstraint implementation class. 

```java
import javax.validation.Validator;

@Service
public class MyService {

    @Autowired
    private Validator validator;
```

By default, the LocalValidatorFactoryBean configures a SpringConstraintValidatorFactory that uses Spring to create ConstraintValidator instances. This allows your custom ConstraintValidators to benefit from dependency injection like any other Spring bean.

## Spring-driven Method Validation

The method validation feature supported by Bean Validation 1.1, and as a custom extension also by Hibernate Validator 4.3, can be integrated into a Spring context through a MethodValidationPostProcessor bean definition:

```xml
<bean class="org.springframework.validation.beanvalidation.MethodValidationPostProcessor"/>
```

 In order to be eligible for Spring-driven method validation, all target classes need to be annotated with Spring’s @Validated annotation, optionally declaring the validation groups to use.

### Spring方法参数实例

1. 准备工作

```java

@Validated
public interface UserService {

    @Validated
    void save(@NotNull @Valid User user);

    // ModifyUser标识接口，用于分组
    @Validated(value = {ModifyUser.class})
    void update(@NotNull @Valid User user);
}

public class UserServiceImpl implements UserService {

    @Override
    public void save(User user) {
        // do something
    }

    @Override
    public void update(User user) {
        // do something
    }
}
```

 2. Spring配置文件

```xml
<bean id="validator" class="org.springframework.validation.beanvalidation.LocalValidatorFactoryBean"/>

<bean class="org.springframework.validation.beanvalidation.MethodValidationPostProcessor"/>
```

3. 测试

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:spring-validator.xml"})
public class UserServiceTest {

    @Resource
    private UserService userService;

    @Test
    public void add() {

        User user2 = new User();
        user2.setName("zhang san");
        userService.save(user2);


        User user = new User();
        user.setName("zhang san");
        userService.update(user);
    }
}
```

### Spring方法参数分析

MethodValidationPostProcessor委托JSR-303提供者执行方法注解验证。

1、MethodValidationPostProcessor类结构图

![MethodValidationPostProcessor类结构图](http://upload-images.jianshu.io/upload_images/2889214-d56677e2bc335714.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2、创建切面

```java
public void afterPropertiesSet() {
    Pointcut pointcut = new AnnotationMatchingPointcut(this.validatedAnnotationType, true);
    Advice advice = (this.validator != null ? new MethodValidationInterceptor(this.validator) :
    new MethodValidationInterceptor());
    this.advisor = new DefaultPointcutAdvisor(pointcut, advice);
    }
```

3、为目标类添加DefaultPointcutAdvisor

```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (bean instanceof AopInfrastructureBean) {
        // Ignore AOP infrastructure such as scoped proxies.
            return bean;
    }

    if (bean instanceof Advised) {
        Advised advised = (Advised) bean;
        if (!advised.isFrozen() && isEligible(AopUtils.getTargetClass(bean))) {
        // Add our local Advisor to the existing proxy's Advisor chain...
        if (this.beforeExistingAdvisors) {
            advised.addAdvisor(0, this.advisor);
        }
        else {
            advised.addAdvisor(this.advisor);
        }
            return bean;
        }
    }

    if (isEligible(bean, beanName)) {
        ProxyFactory proxyFactory = prepareProxyFactory(bean, beanName);
        if (!proxyFactory.isProxyTargetClass()) {
            evaluateProxyInterfaces(bean.getClass(), proxyFactory);
        }
        proxyFactory.addAdvisor(this.advisor);
            customizeProxyFactory(proxyFactory);
            return proxyFactory.getProxy(getProxyClassLoader());
        }

        // No async proxy needed.
        return bean;
    }
```