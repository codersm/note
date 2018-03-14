---
title: junit 不能用于测试多线程
date: 2016-07-11 11:53:19
tags: junit
---
Junit 多线程测试
<!-- more -->

## Junit 不能用于测试多线程
> 原文来自：https://segmentfault.com/a/1190000003762719

线程类:
```java
public class Thread1 implements Runnable {
    @Override
    public void run() {
        // TODO Auto-generated method stub
        for(int i=0;i<10000;i++){
            System.out.print(i+" ");
        }
    }
}
```
测试类:
```java
public class MultiThreadTest {

    @Test
    public void test() throws IOException, InterruptedException{

        Runnable  Thread1 = new Thread1();

        new Thread(Thread1).start();
    //    Thread.sleep(10000);
        System.out.println(" main thread terminate");

    }
}
```
输出结果:
```terminate
main thread terminate
0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 5
```
总结：
在线程类中,循环打印10000 个数字,但是在测试类中只打印了一部分.如果在测试类中把Thread.sleep() 的注释去掉,则可以打印全部的数字.如果把代码放到正常的main() 中执行,也可以打印全部数字出来.

>其实junit是将test作为参数传递给了TestRunner的main函数。并通过main函数进行执行。test函数在main中执行。如果test执行结束，那么main将会调用System.exit(0);即使还有其他的线程在运行，main也会调用System.exit(0);
System.exit()是系统调用，通知系统立即结束jvm的运行，即使jvm中有线程在运行，jvm也会停止的。所以会出现之前的那种情况。其中System.exit(0);的参数如果是0，表示系统正常退出，如果是非0，表示系统异常退出。

junit TestRunner 部分源代码:
```java
junit.textui.TestRunner
 public static void main (String[] args) {
      TestRunner aTestRunner = new TestRunner();
      try {
        TestResult r = aTestRunner.start(args);
       if (!(r.wasSuccessful()))
          System.exit (1);
       System.exit(0);
      } catch (Exception e) {
       System.err.println(e.getMessage());
        System.exit(2);
      }
   }
```

><font color='red'>所以要想编写多线程Junit测试用例，就必须让主线程等待所有子线程执行完成后再退出。想到的办法自然是Thread中的join方法</font>。话又说回来，这样一个简单而又典型的需求，难道会没有第三方的包支持么？通过google，笔者很快就找到了GroboUtils这个Junit多线程测试的开源的第三方的工具包。

GroboUtils下载页面：
  http://groboutils.sourceforge.net/downloads.html

Maven使用
```terminate
mvn install:install-file -Dfile=Groboutils-core-5.jar -DgroupId=net.sourceforge.groboutils -DartifactId=groboutils-core -Dversion=5 -Dpackaging=jar
```
在Maven项目中引入groboutils-core
```xml
<dependency>
  <groupId>net.sourceforge.groboutils</groupId>
  <artifactId>groboutils-core</artifactId>
  <version>5</version>
</dependency>
```
**测试示例**
```java
public void MultiRequestsTest() {
      // 构造一个Runner
      TestRunnable runner = new TestRunnable() {
          @Override
          public void runTest() throws Throwable {
              // 测试内容
          }
      };
      int runnerCount = 10;
      //Rnner数组，想当于并发多少个。
      TestRunnable[] trs = new TestRunnable[runnerCount];
      for (int i = 0; i < runnerCount; i++) {
          trs[i] = runner;
      }
      // 用于执行多线程测试用例的Runner，将前面定义的单个Runner组成的数组传入
      MultiThreadedTestRunner mttr = new MultiThreadedTestRunner(trs);
      try {
          // 开发并发执行数组里定义的内容
          mttr.runTestRunnables();
      } catch (Throwable e) {
          e.printStackTrace();
      }
  }
```
