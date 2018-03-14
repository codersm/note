# JDK SPI源码详解

## 认识JDK SPI

SPI是Service Provider Interface的缩写，可以使用它扩展框架和更换的组件。**JDK提供了java.util.ServiceLoader工具类，在使用某个服务接口时，它可以帮助我们查找该服务接口的实现类，加载和初始化，前提条件是基于它的约定**。

大多数开发人员可能不熟悉，却经常使用它。举个例子，获取MySQL数据库连接，代码如下：

```java
public class MySQLConnect {

    private final static String url ="jdbc:mysql://localhost:3306/test";

    private final static String username = "root";

    private final static String password = "root";

    public static Connection getConnection() throws SQLException {
       // Class.forName("com.mysql.jdbc.Driver");
       return DriverManager.getConnection(url,username,password);
    }

    public static void main(String[] args) throws SQLException {
        System.out.println(getConnection());
    }
}
```

上述代码是可以运行成功的。接下来我们就分析下DriverManager.getConnection(url,username,password)的过程，探究其如何获取到数据库连接Connection？

1、首先分析loadInitialDrivers方法，其源码如下：

```java
private static void loadInitialDrivers() {
        
    /**
     * 获取ServiceLoader实例，loadedDrivers = 
     * new ServiceLoader(Driver.class,Thread.currentThread().getContextClassLoader())
     */
    ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
    /**
      * 获取serviceLoader实例的迭代器,driversIterator = new LazyIterator(service, loader)
      */
    Iterator<Driver> driversIterator = loadedDrivers.iterator();

    /**
      * 应用程序加载器（Thread.currentThread().getContextClassLoader()）
      * 加载classpath下META-INF/services/java.sql.Driver文件，
      * 解析java.sql.Driver文件的内容将其存储在LazyIterator.pending实例变量中。
      */
    while(driversIterator.hasNext()) {
        /**
        * 通过反射实例化java.sql.Driver文件中的类（com.mysql.jdbc.Driver，
            com.mysql.fabric.jdbc.FabricMySQLDriver）
        */
        driversIterator.next();
    }
}
```

> **约定**：当服务的提供者，提供了服务接口（java.sql.Driver）的一种实现之后，在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。

2、com.mysql.jdbc.Driver类的初始化

```java
 static {
    try {
        // registeredDrivers存储Dirver实例
        java.sql.DriverManager.registerDriver(new Driver());
    } catch (SQLException E) {
        throw new RuntimeException("Can't register driver!");
    }
}
```

3、获取数据库连接

```java
for(DriverInfo aDriver : registeredDrivers) {

    if(isDriverAllowed(aDriver.driver, callerCL)) {
        try {
            // 获取Connection
            Connection con = aDriver.driver.connect(url, info);
            if (con != null) {
                return (con);
            }
        } catch (SQLException ex) {
            if (reason == null) {
                reason = ex;
            }
        }

    } else {
        println("skipping: " + aDriver.getClass().getName());
    }

}
```

## JDK SPI简单使用

1、目录结构

![目录结构](https://user-gold-cdn.xitu.io/2018/1/18/16107d7894f8aa77?w=524&h=289&f=png&s=21497)

2、示例代码

```java

public interface HelloService {

    String sayHello();

}

public class CHelloService implements HelloService {

    @Override
    public String sayHello() {
        return "Welcome to C world";
    }

}

public class JavaHelloService implements HelloService {

    @Override
    public String sayHello() {
        return "Welcome to Java world";
    }

}

```

3、com.codersm.study.jdk.spi.HelloService文件内容

```text

com.codersm.study.jdk.spi.impl.CHelloService
com.codersm.study.jdk.spi.impl.JavaHelloService

```

4、测试

```java

@Test
public void testSpi() {

    ServiceLoader<HelloService> loaders = ServiceLoader.load(HelloService.class);
    for (HelloService loader : loaders) {
        System.out.println(loader.sayHello());
    }
}

```



-> getConnection(url, info, Reflection.getCallerClass())
--> loadInitialDrivers
---> ServiceLoader.load(Driver.class) // 创建ServiceLoader实例
-----> load(Class<S> service)
------> ClassLoader cl = Thread.currentThread().getContextClassLoader();

-----> ServiceLoader.load(service, cl)
-----> loadedDrivers.iterator(); // 创建Iterator实例


-----> loop：driversIterator.hasNext() // 
------> lookupIterator.hasNext() // lookupIterator = new LazyIterator(service, loader);
-------> hasNextService() //  由应用程序加载器加载／META-INF/services/java.sql.Driver
--------> pending = parse(service, configs.nextElement())
---------> parseLine(service, u, r, lc, names) // 获取／META-INF/services/java.sql.Driver文件的内容

将解释内容的第一个赋值给nextName

-----> driversIterator.next();
------> nextService();
-------> hasNextService()
--------> c = Class.forName(cn, false, loader); // cn = nextName
---------> p = c.newInstance()
----------> java.sql.DriverManager.registerDriver(new Driver())
-----------> registeredDrivers.addIfAbsent(new DriverInfo(driver, da))
---------> providers.put(cn, p)；

-> loop：registeredDrivers
--> Connection con = aDriver.driver.connect(url, info)


registeredDrivers.addIfAbsent(new DriverInfo(driver, da))

Reflection.getCallerClass()

T cast(Object obj)

？isAssignableFrom
```
