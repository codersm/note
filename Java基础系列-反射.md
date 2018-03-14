---
title: Java反射
date: 2017-06-28 15:28:31
tags: Java
---
# Java反射

Java语言允许通过程序化的方式间接对Class进行操作。Class文件由类装载器装载后，在JVM中形成一份描述Class结构的元信息对象，通过该元信息对象可以获知Class的结构信息。Java允许用户借助由这个与Class相关的元信息对象间接调用Class对象的功能。

## Class对象

RTTI（Run-Time Type Identification）运行时类型识别，对于这个词一直是C++中的概念，至于Java中出现RRTI的说法则是源于《Thinking in Java》一书，其作用是在运行时识别一个对象的类型和类的信息，这里分两种：传统的“RRTI”,它假定我们在编译期已知道了所有类型(在没有反射机制创建和使用类对象时，一般都是编译期已确定其类型，如new对象时该类必须已定义好)，另外一种是“反射”机制，它允许我们在运行时发现和使用类型的信息。

类Class的实例表示正在运行的Java应用程序中的类和接口。 枚举是一种类，一个注释是一种接口。 每个数组也属于一个类，它被反映为具有相同元素类型和维数的所有数组共享的Class对象。 原始Java类型（boolean，byte，char，short，int，long，float和double）和关键字void也表示为Class对象。
类没有公共构造函数。 相反，Class对象由Java虚拟机自动构建，因为加载了类，并且通过调用类加载器中的defineClass方法。

所有的类都是在对其第一次使用时动态加载到JVM中的，当程序创建第一个对类的静态成员引用时，就会加载这个被使用的类(实际上加载的就是这个类的字节码文件)，注意，使用new操作符创建类的新实例对象也会被当作对类的静态成员的引用(构造函数也是类的静态方法)，由此看来Java程序在它们开始运行之前并非被完全加载到内存的，其各个部分是按需加载的。

### 如何获取Class对象

* Class<?> forName(String className);

    ```java
    try {
        Class<?> clazz = Class.forName("com.codersm.study.jdk.reflect.Person");
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
    ```

* 通过一个实例对象获取一个类的Class对象，其中的getClass()是从顶级类Object继承而来的，它将返回表示该对象的实际类型的Class对象引用。

    ```java
    Class<?> clazz = new Person().getClass();
    ```

* Class字面常量

    ```java
    Class<Person> clazz = Person.Class;
    ```

**获取方式小结**：

 1. 注意调用forName方法时需要捕获一个名称为ClassNotFoundException的异常，因为forName方法在编译器是无法检测到其传递的字符串对应的类是否存在的，只能在程序运行时进行检查，如果不存在就会抛出ClassNotFoundException异常。

 2. Class字面常量这种方法更加简单，更安全。因为它在编译器就会受到编译器的检查同时由于无需调用forName方法效率也会更高，因为通过字面量的方法获取Class对象的引用不会自动初始化该类。更加有趣的是字面常量的获取Class对象引用方式不仅可以应用于普通的类，也可以应用用接口，数组以及基本数据类型。

### 常用方法

| 方法                                                                         | 描述                                                                                                                                                                                                                               |
|----------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| T newInstance()                                                            | 创建由此Class对象表示的类的新实例。如果Class对象表示的类没有空构造函数（没有参数的构造函数），则会抛出异常                                                                                                                                                                       |
| boolean isAssignableFrom(Class<?> cls)                                     | 确定由此Class对象表示的类或接口是否与由指定的Class参数表示的类或接口相同，或者是超类或超级接口。 如果是，则返回true; 否则返回false。                                                                                                                                                    |
| boolean isInstance(Object obj)                                             | 确定指定的对象是否与由此类表示的对象分配兼容。该方法是Java语言instanceof运算符的动态等价物。如果指定的Object参数为非空值，并且可以转换为此Class对象表示的引用类型，而不会引发ClassCastException,则该方法返回true; 否则返回false。                                                                                     |
| boolean isInterface()                                                      | 确定指定的Class对象是否表示接口类型                                                                                                                                                                                                             |
| boolean isArray()                                                          | 确定此Class对象是否表示数组类                                                                                                                                                                                                                |
| boolean isPrimitive()                                                      | 确定指定的Class对象是否表示一个原始类型。有九个预定义的Class对象来表示八个基本类型和void。 这些由Java虚拟机创建，并且具有与它们所代表的基本类型相同的名称。这些对象只能通过以下公共静态最终变量访问，并且是此方法返回true的唯一Class对象。public static final Class<Boolean> TYPE = (Class<Boolean>) Class.getPrimitiveClass("boolean") |
| boolean isAnnotation()                                                     | 确定此Class对象是否表示注解                                                                                                                                                                                                                 |
| Class<? super T> getSuperclass()                                           | 返回表示由此类表示的实体（类，接口，原始类型或void）的超类的类                                                                                                                                                                                                |
| Type getGenericSuperclass()                                                | 返回表示由此类表示的实体的直接超类的类型                                                                                                                                                                                                             |
| Class<?>[] getInterfaces()                                                 | 确定由该对象表示的类或接口实现的接口                                                                                                                                                                                                               |
| Field[] getFields()                                                        | 返回一个包含Field对象的数组，反映由此Class对象表示的类或接口的所有可访问的公共字段                                                                                                                                                                                   |
| Method[] getMethods()                                                      | 返回一个包含Method对象的数组，该对象反映由此Class对象表示的类或接口的所有公共方法，包括由类或接口声明的类和从超类和超级接口继承的类                                                                                                                                                          |
| Constructor<?>[] getConstructors()                                         | 返回一个包含Constructor对象的数组，该对象反映由此Class对象表示的类的所有公共构造函数                                                                                                                                                                               |
| Field getField(String name)                                                | 返回一个Field对象，该对象反映由此Class对象表示的类或接口的指定公共成员字段                                                                                                                                                                                       |
| Method getMethod(String name, Class<?>... parameterTypes)                  | 返回一个方法对象，该对象反映由此Class对象表示的类或接口的指定公共成员方法。 name参数是指定所需方法的简单名称的String。 parameterTypes参数是以声明顺序标识方法的形式参数类型的Class对象数组。 如果parameterTypes为空，则将其视为空数组                                                                                     |
| Constructor<T> getConstructor(Class<?>... parameterTypes)                  | 返回一个Constructor对象，该对象反映由此Class对象表示的类的指定的公共构造函数。 parameterTypes参数是以声明顺序标识构造函数的形式参数类型的Class对象数组。 如果此Class对象表示在非静态上下文中声明的内部类，则形式参数类型将显式包围实例作为第一个参数。                                                                                 |
| Field[] getDeclaredFields()                                                | 返回一个Field对象的数组，反映由此Class对象表示的类或接口声明的所有字段。 这包括public，protected，default（package）访问和private字段，但不包括继承的字段。                                                                                                                            |
| Method[] getDeclaredMethods()                                              | 返回一个包含Method对象的数组，该对象反映了此Class对象所表示的类或接口的所有声明方法，包括public，protected，default（package）访问和私有方法，但不包括继承方法                                                                                                                              |
| `<A extends Annotation> A getAnnotation(Class<A> annotationClass)`         | 如果存在这样的注释，则返回该元素的指定类型的注释，否则返回null。                                                                                                                                                                                               |
| boolean isAnnotationPresent(Class<? extends Annotation> annotationClass)   | 如果此元素上存在指定类型的注释，则返回true，否则返回false。 该方法主要用于方便访问标记注释。                                                                                                                                                                              |
| `<A extends Annotation> A getDeclaredAnnotation(Class<A> annotationClass)` | 如果这样的注释直接存在，则返回指定类型的元素注释，否则返回null。 此方法忽略继承的注释。                                                                                                                                                                                   |

## 反射相关类

### Constructor

| 方法                                       | 描述                                                                    |
|------------------------------------------|-----------------------------------------------------------------------|
| Type[] getGenericParameterTypes()        | 返回一个Type对象的数组，它以声明顺序表示由该对象表示的可执行文件的形式参数类型。 如果底层可执行文件不带参数，则返回长度为0的数组。  |
| Class<?>[] getExceptionTypes()           | 返回一个Class对象数组，它们以声明顺序表示由该对象表示的可执行文件的形式参数类型。 如果底层可执行文件不带参数，则返回长度为0的数组。 |
| T newInstance(Object ... initargs)       | 使用指定的初始化参数来创建和初始化构造函数声明类的新实例。                                         |
| Annotation[] getDeclaredAnnotations()    | 返回此元素上直接显示的注释                                                         |
| AnnotatedType getAnnotatedReturnType()   | 返回一个AnnotatedType对象，它表示使用一个类型来指定此可执行文件所表示的方法/构造函数的返回类型。               |
| AnnotatedType getAnnotatedReceiverType() | 返回一个AnnotatedType对象，该对象表示使用类型来指定此可执行文件对象所表示的方法/构造函数的接收器类型。            |

### Field

| 方法                                    | 描述                                                     |
|---------------------------------------|--------------------------------------------------------|
| String getName()                      | 返回此Field对象表示的字段的名称。                                    |
| Class<?> getDeclaringClass()          | 返回表示声明由此Field对象表示的字段的类或接口的Class对象。                     |
| Object get(Object obj)                | 返回指定对象上由此Field表示的字段的值。 如果该对象具有原始类型，则该值将自动包含在对象中。       |
| set(Object obj, Object value)         | 将指定对象参数上的此Field对象表示的字段设置为指定的新值。 如果基础字段具有原始类型，则新值将自动展开。 |
| Annotation[] getDeclaredAnnotations() | 返回此元素上直接显示的注释。                                         |

### Method

| 方法                                        | 描述                                                                    |
|-------------------------------------------|-----------------------------------------------------------------------|
| String getName()                          | 返回由此Method对象表示的方法的名称                                                  |
| Class<?> getReturnType()                  | 返回一个Class对象，该对象表示此Method对象表示的方法的正式返回类型。                               |
| Class<?>[] getParameterTypes()            | 返回一个Class对象数组，它们以声明顺序表示由该对象表示的可执行文件的形式参数类型。 如果底层可执行文件不带参数，则返回长度为0的数组。 |
| Object invoke(Object obj, Object... args) | 在具有指定参数的指定对象上调用此Method对象表示的底层方法                                       |

### 示例

```java
public class Person {

    private String name;

    public Person(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }


    public static void main(String[] args) throws Exception {

        Class<?> clazz = Person.class;
        // 获取构造函数
        Constructor<?> constructor = clazz.getConstructor(String.class);
        // 创建Person示例
        Person person = (Person) constructor.newInstance("张三");

        Field field = clazz.getDeclaredField("name");
        Method method = clazz.getDeclaredMethod("getName");
        // 通过Field、Method获取name的值
        System.out.println(field.get(person) + "," + method.invoke(person));
        // 通过Field、Method设置name的值
        field.set(person, "李四");
        System.out.println(person.getName());
        Method setMethod = clazz.getDeclaredMethod("setName", String.class);
        setMethod.invoke(person, "王五");
        System.out.println(person.getName());
    }
}
```

## 动态代理

### 什么是代理？

代理是一种常用的设计模式，其目的就是为其他对象提供一个代理以控制对某个对象的访问。代理类负责为委托类预处理消息，过滤消息并转发消息，以及进行消息被委托类执行后的后续处理。

代理模式UML图：

![代理模式UML图](http://i1.buimg.com/598959/2ea60dd828341dc8.png)

为了保持行为的一致性，代理类和委托类通常会实现相同的接口，所以在访问者看来两者没有丝毫的区别。通过代理类这中间一层，能有效控制对委托类对象的直接访问，也可以很好地隐藏和保护委托类对象，同时也为实施不同控制策略预留了空间，从而在设计上获得了更大的灵活性。Java 动态代理机制以巧妙的方式近乎完美地实践了代理模式的设计理念。

### 静态代理

JDK 5引入的动态代理机制，允许开发人员在运行时刻动态的创建出代理类及其对象。在运行时刻，可以动态创建出一个实现了多个接口的代理类。

### Java 动态代理

Java动态代理类位于java.lang.reflect包下，一般主要涉及到以下两个类：

#### InvocationHandler

```java
/**
 *处理代理实例上的方法调用并返回结果。 当在与之关联的代理实例上调用方法时，将在调用处理程序上调用此方法
 **/
public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
```

在实际使用时，第一个参数proxy一般是指代理类，method是被代理的方法，如上例中的request()，args为该方法的参数数组。

#### Proxy：该类即为动态代理类，其中主要包含以下内容：

* protected Proxy(InvocationHandler h)

构造函数，用于给内部的h赋值。

* public static Object newProxyInstance(ClassLoader loader,
    Class<?>[] interfaces,
    InvocationHandler h) throws IllegalArgumentException
获得一个代理类，其中loader是类装载器，interfaces是真实类所拥有的全部接口的数组。

* public static InvocationHandler getInvocationHandler(Object proxy)

返回代理类的一个实例，返回后的代理类可以当作被代理类使用(可使用被代理类的在Subject接口中声明过的方法)

所谓DynamicProxy是这样一种class：它是在运行时生成的class，在生成它时你必须提供一组interface给它，然后该class就宣称它实现了这些interface。你当然可以把该class的实例当作这些interface中的任何一个来用。当然，这个DynamicProxy其实就是一个Proxy，它不会替你作实质性的工作，在生成它的实例时你必须提供一个handler，由它接管实际的工作。在使用动态代理类时，我们必须实现InvocationHandler接口。

通过这种方式，被代理的对象(RealSubject)可以在运行时动态改变，需要控制的接口(Subject接口)可以在运行时改变，控制的方式(DynamicSubject类)也可以动态改变，从而实现了非常灵活的动态代理关系。

### JDK的动态代理怎么使用

1、需要动态代理的接口

```java
public interface OneInterface {

    void doSomething(String msg);
}
```

2、需要代理的实际对象

```java
public class OneInterfaceImpl implements OneInterface{

    @Override
    public void doSomething(String msg) {
        System.out.println(msg);
    }
}
```

3、代理工厂并实现调用处理器实现类

```java
public class DynamicProxyFactory implements InvocationHandler {


    private static DynamicProxyFactory factory = new DynamicProxyFactory();

    public static DynamicProxyFactory getFactory() {
        return factory;
    }

    public Object getProxy(Object proxied) {
        this.proxied = proxied;
        return Proxy.newProxyInstance(proxied.getClass().getClassLoader(),
                proxied.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        return method.invoke(proxied, args);
    }

    private Object proxied;
}
```

4、测试

```java
@Test
public void test() {

    DynamicProxyFactory factory =  DynamicProxyFactory.getFactory();
    OneInterface proxy = (OneInterface) factory.getProxy(new OneInterfaceImpl());
    proxy.doSomething("Welcome to Java World");
}
```

[说说Java代理模式](https://www.jianshu.com/p/a1d094fc6c00)

## Java字节码生成开源框架介绍--Javassist

Javassist是一个开源的分析、编辑和创建Java字节码的类库。是由东京工业大学的数学和计算机科学系的 Shigeru Chiba （千叶 滋）所创建的。它已加入了开放源代码JBoss 应用服务器项目,通过使用Javassist对字节码操作为JBoss实现动态AOP框架。javassist是jboss的一个子项目，其主要的优点，在于简单，而且快速。直接使用java编码的形式，而不需要了解虚拟机指令，就能动态改变类的结构，或者动态生成类。

