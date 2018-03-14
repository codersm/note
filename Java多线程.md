---
title: Java并发编程
date: 2016-07-18T16:35:06.000Z
tags: Java并发编程
---
# Java并发编程

 

## Java内存模型

在并发编程中，需要处理两个关键问题：**线程之间如何通信及线程之间如何同步**。通信是指线程之间以何种机制来交换信息，在命令式编程中，线程之间的通信机制有两种：共享内存和消息传递。在共享内存的并发模型里，线程之间共享程序的公用状态，通过写-读内存中的公共状态进行隐式通信。在消息传递的兵法模型里，线程之间没有公共状态，线程之间必须通过发送消息来显式进行通信。

同步是指程序中用于控制不同线程间操作发生相对顺序的机制。在共享内存并发模型里，同步是显式进行的。程序员必须显式指定某个方法或某段代码需要在线程之间互斥执行。在消息传递的并发模型里，同步是隐式进行的（消息的发送必须在消息的接收之前）。

Java的并发采用的共享内存模型，Java线程之间的通信总是隐式进行，整个通信过程对程序员完全透明。

### Java内存模型的抽象结构

在Java中，所有实例域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享。局部变量，方法定义参数和异常处理参数不会在线程之间共享，它们不会有内存可见性问题，也不受内存模型的影响。

Java线程之间的通信由Java内存模型（JMM）控制，JMM决定一个线程对共享变量的写入何时对另一个线程可见。从抽象的角度来看，JMM定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存，本地内存中存储了该线程以读／写共享变量的副本。**本地内存是JMM的一个抽象概念，并不真实存在。**




### 并发编程的模型分类



由于写缓冲区仅对自己的处理器可见，它会导致处理器执行内存操作的顺序可能会与内存实际的操作执行顺序不一致。由于现代的处理器都会使用写缓冲区，因此现代的处理器都会允许对写-读操作进行重排序。

为类保证内存可见性，Java编译器在生成指令序列的适当位置会插入**内存屏障指令**来禁止特定类型的处理器重排序。
                                 
#### happens-before简介

从JDK5开始，Java使用新的JSR-133内存模型。JSR-133使用happens-before的概念来阐述操作之间的内存可见性。

happens-before与JMM的关系如下图所示：

### 重排序

重排序是指编译器和处理器为了优化程序性能而对指令序列进行重新排序的一种手段。从Java源代码到最终实际执行的指令序列，会分别经历下面3种重排序：

![从Java源代码到最终实际执行的指令序列](http://upload-images.jianshu.io/upload_images/2889214-cca873778e80ec53.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1）编译器优化的重排序：编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。

2）指令级并行的重排序：现代处理器采用了并行技术来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。

3）内存系统的重排序：由于处理器使用缓存和读／写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。

> 2和3属于处理器重排序。

这些重排序可能会导致多线程程序出现内存可见性问题。对于编译器，JMM的编译器重排序规则会禁止特定类型的编译器重排序；对于处理器重排序，JMM的处理器重排序规则会要求Java编译器在生成指令序列时，插入特定类型的内存屏障指令，通过内存屏障指令来禁止特定类型的处理器重排序。

常见处理器允许的重排序类型的列表：

![处理器的重排序规则](http://upload-images.jianshu.io/upload_images/2889214-cca4be8062882ea7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

常见的处理器都允许Store-load重排序；常见的处理器都不允许对存在数据依赖的操作做重排序。

内存屏障

内存屏障（Memory Barrier，或有时叫做内存栅栏，Memory Fence）是一种CPU指令，用于控制特定条件下的重排序和内存可见性问题。JMM把内存屏障指令分为4类，如下图所示：

![内存屏障类型表](http://upload-images.jianshu.io/upload_images/2889214-db1af11931bf94fd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


JMM属于语言级的内存模型，它确保在不同编译器和不同的处理器平台之上，通过禁止特定类型的编译器重排序和处理器重排序，为程序员提供一致的内存可见性保证。

#### 数据依赖性

如果两个操作访问同一个变量，且这两个操作中有一个为写操作，此时这两个操作之间就存在数据依赖性。


编译器和处理器在重排序时，会遵守数据依赖性，编译器和处理器不会改变存在数据依赖关系的两个操作的执行顺序。**这里所说的数据依赖性仅针对单个处理器中执行的指令序列和单个线程中执行的操作，不同处理器之间和不同线程之间的数据依赖性不被编译器和处理器考虑**。

#### as-if-serial语义

as-if-serial语义：不管怎重排序（编译器和处理器为了提高并行度），（单线程）程序的执行结果不能被改变。编译器、runtime和处理器都必须遵守as-if-serial语义。

#### 程序顺序规则

在单线程程序中，对存在控制依赖的操作重排序，不会改变执行的结果；但在多线程程序中，对存在控制依赖的操作做重排序，可能会改变程序的执行结果。

#### 顺序一致性

顺序一致性内存模型是一个理论参考模型，在设计的时候，处理器的内存模型和编程语言的内存模型都会以顺序一致性内存模型作为参照。

数据竞争的定义：
    在一个线程中写一个变量，在另一个线程读同一个变量，而且写和读没有通过同步来排序。

当代码中包含数据竞争时，程序的执行往产生违反直觉的结果。如果一个多线程程序能够正确同步，这个程序将是一个没有数据竞争的程序。

JMM对正确同步的多线程程序的内存一致性做了如下保证。

#### volatile

volatile变量自身具有下列特性：

* 可见性

    对一个volatile变量的读，总是能看到（任意线程）对这个变量最后的写入。

* 原子性

    对任意单个volatile变量的读／写具有原子性，但类似于volatile++这种复合操作不具有原子性。

volatile写-读的内存语义

为了实现volatile内存语义，JMM会分别限制这两种类型的重排序类型。JMM针对编译器制定的volatile重排序规则表如下：

#### 锁的内存语义

锁释放和锁获取的内存语义：

#### final域的内存语义

对于final域，编译器和处理器要遵守两个重排序规则：

* 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用变量赋值给一个引用变量，这两个操作之间不能重排序。

* 初次读一个包含final域的对象引用，与随后初次读这个final域，这两个操作之间不能重排序。

JMM把happens-befor要求禁止的重排序分为了下面两类：

* 会改变程序执行结果的重排序

* 不会改变程序执行结果的重排序

happens-before关系定义如下：

如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二操作可见，而且第一个操作的执行顺序排在第二个操作之前。

两个操作之间存在happens-before关系，并不意味着Java平台的具体实现必须按照happens-before关系指定的顺序来执行。如果重排序之后的执行结果，与按happens-before关系来执行的结果一致，那么这种重排序并不非法。

as-if-serial语义保证单线程内程序的执行结果不被改变，happens-before关系保证正确同步的多线程程序的执行结果不被改变。

happens-before规则：

* 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。

* 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。

* volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。

* 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。

* start()规则：如果线程A执行操作ThreadB.start(),那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。

* join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。



## 线程安全性

  编写线程安全的代码，其核心在于要对状态访问操作进行管理，特别是对共享的和可变状态的访问。

  “共享”意味着变量可以由多个线程同时访问，而“可变”则意味着变量的值在其生命周期内可以发生变化。

  当多个线程访问某个状态变量并且其中有一个线程执行写入操作时，必须采用同步机制来协同这些线程对变量的访问。Java中的主要同步机制是关键字synchronized，它提供了一种独占的加锁方式，但“同步”这个术语还包括volatile类型的变量，显式锁以及原子变量。

如果当多个线程访问一个可变的状态变量时没有使用合适的同步，那么程序就会出现错误。有三种方式可以修复这个问题：

* 不在线程之间共享该状态变量;
* 将状态变量修改不可变的变量；
* 在访问状态变量时使用同步;

### Java内存模型的抽象结构

在Java中，所有实例域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享。局部变量，方法定义参数和异常处理器参数不会在线程之间共享，它们不会有内存可见性问题，也不受内存模型的影响。

Java线程之间的通信由Java内存模型（简称为JMM）控制，JMM决定一个线程对共享变量的写入何时对另一个线程可见。从抽象的角度来看，JMM定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存(Local Memory)，本地内存中存储了该线程以读/写共享变量的副本。

本地内存是JMM的抽象概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。

### 从源代码到指令序列的重排序

在执行程序时，为了提供性能，编译器和处理器常常会对指令做重排序。重排序分3种类型：

- 编译器优化的重排序

    编译器在不改变单线程程序语义的前提下，可以重新安语句的执行顺序。

- 指令级并行的重排序

    现代处理器采用了指令级并行技术来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。

- 内存系统的重排序

    由于

## happen-before原则


## 什么是线程安全性

 当多个线程访问某个类时，这个类始终都能表现出正确的行为，那么就称这个类是线程安全的。

> 在线程安全类中封装了必要的同步机制，因此客户端无须进一步采取同步机制。

`无状态对象一定是线程安全的；`

### 竞态条件

  由于不恰当的执行时序而出现不正确的结果。

### 重入

一种常见错误的认为，只有在写入共享变量时才需要使用同步，然而事实并非如此。

## 内存可见性

### 重排序

### 非原子的64位操作

Java内存模型要求，变量的读取操作和写入操作都必须是原子操作，但对于非volatile类型long和double变量，JVM允许将64位的读操作或写操作分解为两个32位的操作。
在多线程程序中使用共享且可变的long和double等类型的变量也是不安全的，除非用关键字volatile来声明它们，或者用锁保护起来。

## 并发

## 并行

## 公有资源

wait/notify的区别

wait/notify必须存在于synchronized块中。并且，这三个关键字针对的是同一个监视器（某对象的监视器）。这意味着wait之后，其他线程可以进入同步块执行。

当某代码并不持有监视器的使用权时（如图中5的状态，即脱离同步块）去wait或notify，会抛出java.lang.IllegalMonitorStateException。也包括在synchronized块中去调用另一个对象的wait/notify，因为不同对象的监视器不同，同样会抛出此异常。

## 典型场景——生产者和消费者

## volatile

多线程的内存模型：main memory（主存）、working memory（线程栈），在处理数据时，线程会把值从主存load到本地栈，完成操作后再save回去(volatile关键词的作用：每次针对该变量的操作都激发一次load and save)。

## 中断

## Callable

futrue模式：并发模式的一种，可以有两种形式，即无阻塞和阻塞。







## 什么是Fork/Join框架

Fork/Join框架是Java 7提供的一个用于并行执行任务的框架,是一个把大任务分割成若干
个小任务,最终汇总每个小任务结果后得到大任务结果的框架。

**工作窃取算法**

工作窃取(work-stealing)算法是指某个线程从其他队列里窃取任务来执行。那么,为什么
需要使用工作窃取算法呢?假如我们需要做一个比较大的任务,可以把这个任务分割为若干
互不依赖的子任务,为了减少线程间的竞争,把这些子任务分别放到不同的队列里,并为每个
队列创建一个单独的线程来执行队列里的任务,线程和队列一一对应。比如A线程负责处理A
队列里的任务。但是,有的线程会先把自己队列里的任务干完,而其他线程对应的队列里还有
任务等待处理。干完活的线程与其等着,不如去帮其他线程干活,于是它就去其他线程的队列
里窃取一个任务来执行。而在这时它们会访问同一个队列,所以为了减少窃取任务线程和被
窃取任务线程之间的竞争,通常会使用双端队列,被窃取任务线程永远从双端队列的头部拿
任务执行,而窃取任务的线程永远从双端队列的尾部拿任务执行。


工作窃取算法的优点:充分利用线程进行并行计算,减少了线程间的竞争。

工作窃取算法的缺点:在某些情况下还是存在竞争,比如双端队列里只有一个任务时。并且该算法会消耗了更多的系统资源,比如创建多个线程和多个双端队列。

Fork/Join框架的设计

我们已经很清楚Fork/Join框架的需求了,那么可以思考一下,如果让我们来设计一个
Fork/Join框架,该如何设计?这个思考有助于你理解Fork/Join框架的设计。
步骤1 分割任务。首先我们需要有一个fork类来把大任务分割成子任务,有可能子任务还
是很大,所以还需要不停地分割,直到分割出的子任务足够小。
步骤2 执行任务并合并结果。分割的子任务分别放在双端队列里,然后几个启动线程分
别从双端队列里获取任务执行。子任务执行完的结果都统一放在一个队列里,启动一个线程
从队列里拿数据,然后合并这些数据。

使用Fork/Join框架

- ForkJoinTask:我们要使用ForkJoin框架,必须首先创建一个ForkJoin任务。它提供在任务中执行fork()和join()操作的机制。通常情况下,我们不需要直接继承ForkJoinTask类,只需要继承它的子类。Fork/Join框架提供了以下两个子类：
    - RecursiveAction:用于没有返回结果的任务。
    - RecursiveTask:用于有返回结果的任务。

- ForkJoinPool:ForkJoinTask需要通过ForkJoinPool来执行。
任务分割出的子任务会添加到当前工作线程所维护的双端队列中,进入队列的头部。当一个工作线程的队列里暂时没有任务时,它会随机从其他工作线程的队列的尾部获取一个任务。

Fork/Join框架的异常处理

ForkJoinTask在执行的时候可能会抛出异常,但是我们没办法在主线程里直接捕获异常,
所以ForkJoinTask提供了isCompletedAbnormally()方法来检查任务是否已经抛出异常或已经被
取消了,并且可以通过ForkJoinTask的getException方法获取异常。

## Java中13个原子操作类


## Executor框架

在Java中，使用线程来异步执行任务。Java线程的创建与销毁需要一定的开销，如果我们
为每一个任务创建一个新线程来执行，这些线程的创建与销毁将消耗大量的计算资源。同时，
为每一个任务创建一个新线程来执行，这种策略可能会使处于高负荷状态的应用最终崩溃。

Java的线程既是工作单元，也是执行机制。从JDK 5开始，把工作单元与执行机制分离开
来。工作单元包括Runnable和Callable，而执行机制由Executor框架提供。

### Executor框架简介

#### Executor框架的两级调度模型

在HotSpot VM的线程模型中，Java线程（java.lang.Thread）被一对一映射为本地操作系统线
程。Java线程启动时会创建一个本地操作系统线程；当该Java线程终止时，这个操作系统线程
也会被回收。操作系统会调度所有线程并将它们分配给可用的CPU。

在上层，Java多线程程序通常把应用分解为若干个任务，然后使用用户级的调度器
（Executor框架）将这些任务映射为固定数量的线程；在底层，操作系统内核将这些线程映射到
硬件处理器上。这种两级调度模型的示意图如图所示:

![任务的两级调度模型](http://upload-images.jianshu.io/upload_images/2889214-04cea835707871e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

应用程序通过Executor框架控制上层的调度；而下层的调度由操作系统
内核控制，下层的调度不受应用程序的控制。

#### Executor框架的结构与成员

* Executor框架主要由3大部分组成如下：

    1、任务。包括被执行任务需要实现的接口：Runnable接口或Callable接口。

    2、任务的执行。包括任务执行机制的核心接口Executor，以及继承自Executor的ExecutorService接口。Executor框架有两个关键类实现了ExecutorService接口（ThreadPoolExecutor和ScheduledThreadPoolExecutor）。

    3、异步计算的结果。包括接口Future和实现Future接口的FutureTask类。

##### Executor框架的使用

![Executor框架的使用示意图](http://upload-images.jianshu.io/upload_images/2889214-836b01d460f6e027.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Executor框架的成员

Executor框架的主要成员：ThreadPoolExecutor、ScheduledThreadPoolExecutor、
Future接口、Runnable接口、Callable接口和Executors。

##### ThreadPoolExecutor

ThreadPoolExecutor通常使用工厂类Executors来创建。Executors可以创建3种类型的ThreadPoolExecutor：SingleThreadExecutor、FixedThreadPool和CachedThreadPool。

* FixedThreadPool

    FixedThreadPool适用于为了满足资源管理的需求，而需要限制当前线程数量的应用场景，它适用于负载比较重的服务器。

* SingleThreadExecutor

    SingleThreadExecutor适用于需要保证顺序地执行各个任务；并且在任意时间点，不会有多个线程是活动的应用场景。

* CachedThreadPool

    CachedThreadPool是大小无界的线程池，适用于执行很多的短期异步任务的小程序，或者是负载较轻的服务器

##### ScheduledThreadPoolExecutor

ScheduledThreadPoolExecutor通常使用工厂类Executors来创建。Executors可以创建2种类型的ScheduledThreadPoolExecutor，如下：

* ScheduledThreadPoolExecutor

    包含若干个线程的ScheduledThreadPoolExecutor。ScheduledThreadPoolExecutor适用于需要多个后台线程执行周期任务，同时为了满足资源管理的需求而需要限制后台线程的数量的应用场景。

* SingleThreadScheduledExecutor

    只包含一个线程的ScheduledThreadPoolExecutor。SingleThreadScheduledExecutor适用于需要单个后台线程执行周期任务，同时需要保证顺序地执行各个任务的应用场景。

##### Future接口

Future接口和实现Future接口的FutureTask类用来表示异步计算的结果。当我们把Runnable接口或Callable接口的实现类提交（submit）给ThreadPoolExecutor或ScheduledThreadPoolExecutor时，ThreadPoolExecutor或ScheduledThreadPoolExecutor会向我们返回一个FutureTask对象。

##### Runnable接口和Callable接口

Runnable接口和Callable接口的实现类，都可以被ThreadPoolExecutor或ScheduledThreadPoolExecutor执行。它们之间的区别是**Runnable不会返回结果，而Callable可以返回结果**。

除了可以自己创建实现Callable接口的对象外，还可以使用工厂类Executors来把一个Runnable包装成一个Callable。

### ThreadPoolExecutor详解

Executor框架最核心的类是ThreadPoolExecutor,它是线程池的实现类,主要由下列4个组件构成。

* corePool:核心线程池的大小。
* maximumPool:最大线程池的大小。
* BlockingQueue:用来暂时保存任务的工作队列。
* RejectedExecutionHandler:当ThreadPoolExecutor已经关闭或ThreadPoolExecutor已经饱和
时(达到了最大线程池大小且工作队列已满),execute()方法将要调用的Handler。

```java
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory,ejectedExecutionHandler handler)
```

#### FixedThreadPool详解

FixedThreadPool被称为可重用固定线程数的线程池。下面是FixedThreadPool的源代码实现。
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
    0L, TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<Runnable>());
}
```
FixedThreadPool的corePoolSize和maximumPoolSize都被设置为创建FixedThreadPool时指定的参数nThreads。
当线程池中的线程数大于corePoolSize时,keepAliveTime为多余的空闲线程等待新任务的最长时间,超过这个时间后多余的线程将被终止。这里把keepAliveTime设置为0L,意味着多余的空闲线程会被立即终止。


1)如果当前运行的线程数少于corePoolSize,则创建新线程来执行任务。

2)在线程池完成预热之后(当前运行的线程数等于corePoolSize),将任务加入LinkedBlockingQueue。

3)线程执行完1中的任务后,会在循环中反复LinkedBlockingQueue获取任务来执行。FixedThreadPool使用无界队列LinkedBlockingQueue作为线程池的工作队列(队列的容量为
Integer.MAX_VALUE)。

**使用无界队列作为工作队列会对线程池带来如下影响**：

1)当线程池中的线程数达到corePoolSize后,新任务将在无界队列中等待,因此线程池中的线程数不会超过corePoolSize。

2)由于1,使用无界队列时maximumPoolSize将是一个无效参数。

3)由于1和2,使用无界队列时keepAliveTime将是一个无效参数。

4)由于使用无界队列,运行中的FixedThreadPool(未执行方法shutdown()或shutdownNow())不会拒绝任务(不会调用RejectedExecutionHandler.rejectedExecution方法)。

#### SingleThreadExecutor详解

SingleThreadExecutor是使用单个worker线程的Executor。下面是SingleThreadExecutor的源代码实现。

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
    (new ThreadPoolExecutor(1, 1,
    0L, TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<Runnable>()));
}
```

SingleThreadExecutor的corePoolSize和maximumPoolSize被设置为1。其他参数与FixedThreadPool相同。SingleThreadExecutor使用无界队列LinkedBlockingQueue作为线程池的工作队列(队列的容量为Integer.MAX_VALUE)。SingleThreadExecutor使用无界队列作为工作队列对线程池带来的影响与FixedThreadPool相同。

#### CachedThreadPool详解

CachedThreadPool是一个会根据需要创建新线程的线程池。下面是创建CachedThreadPool的源代码。

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
    60L, TimeUnit.SECONDS,
    new SynchronousQueue<Runnable>());
}
```

CachedThreadPool的corePoolSize被设置为0,即corePool为空;maximumPoolSize被设置为Integer.MAX_VALUE,即maximumPool是无界的。这里把keepAliveTime设置为60L,意味着CachedThreadPool中的空闲线程等待新任务的最长时间为60秒,空闲线程超过60秒后将会被终止。

FixedThreadPool和SingleThreadExecutor使用无界队列LinkedBlockingQueue作为线程池的工作队列。CachedThreadPool使用没有容量的SynchronousQueue作为线程池的工作队列,但CachedThreadPool的maximumPool是无界的。这意味着,如果主线程提交任务的速度高于maximumPool中线程处理任务的速度时,CachedThreadPool会不断创建新线程。极端情况下,CachedThreadPool会因为创建过多线程而耗尽CPU和内存资源。

#### ScheduledThreadPoolExecutor详解

ScheduledThreadPoolExecutor继承自ThreadPoolExecutor。它主要用来在给定的延迟之后运行任务,或者定期执行任务。ScheduledThreadPoolExecutor的功能与Timer类似,但
ScheduledThreadPoolExecutor功能更强大、更灵活。Timer对应的是单个后台线程,而ScheduledThreadPoolExecutor可以在构造函数中指定多个对应的后台线程数。


#### FutrueTask详解

可以把FutureTask交给Executor执行;也可以通过ExecutorService.submit(...)方法返回一个
FutureTask,然后执行FutureTask.get()方法或FutureTask.cancel(...)方法。除此以外,还可以单独
使用FutureTask。

当一个线程需要等待另一个线程把某个任务执行完后它才能继续执行,此时可以使用
FutureTask。假设有多个线程执行若干任务,每个任务最多只能被执行一次。当多个线程试图
同时执行同一个任务时,只允许一个线程执行任务,其他线程需要等待这个任务执行完后才
能继续执行。




