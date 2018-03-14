> 特别说明：文章内容是《Java并发编程的艺术》读书笔记

Java是一种多线程语言，从诞生开始就内置了对多线程的支持。正确地使用多线程可以显著提高程序性能，但过多地创建线程和对线程的不当管理也很容易造成问题。

## 线程简介

### 线程定义

现代操作系统在运行一个程序时，会为其创建一个进程。例如，启动一个Java程序，操作系统就会创建一个Java进程。**线程是现代操作系统调度的最小单元，也叫轻量级进程，在一个进程里可以创建多个线程，这些线程都拥有各自的计算器、堆栈和局部变量等属性，并且能够访问共享的内存变量**。处理器在这些线程上高速切换，让使用者感觉到这些线程在同时执行。

Java程序天生就是多线程程序，可以通过JMX查看一个普通的Java程序包含那些线程，代码如下：

```java
public class MutilThread {

    public static void main(String[] args) {
        // 获取Java线程管理MXBean
        ThreadMXBean threadMXBean =  ManagementFactory.getThreadMXBean();
        // 获取线程和线程堆栈信息
        ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false,false);
        // 遍历线程线程，仅打印线程ID和线程名称信息
        for(ThreadInfo threadInfo:threadInfos){
            System.out.println("["+threadInfo.getThreadId()+"]"+threadInfo.getThreadName());
        }
    }
}
```
运行结果如下：

![一个普通的Java程序包含那些线程](https://user-gold-cdn.xitu.io/2018/1/22/1611e594b8c93a76?w=1138&h=298&f=png&s=46299)


### 使用多线程的原因

正确使用多线程，总是能够给开发人员带来显著的好处，而使用多线程的原因主要有以下几点：

1、更多的处理器核心

随着处理器上的核心数量越来越多，以及超线程技术的广泛运用，现在大多数计算机都比以往更加擅长并行计算，而处理器性能的提升方式，也从更高的主频向更多的核心发展。

2、更快的响应时间

有时我们会编写一些业务逻辑比较复杂的代码，例如，一笔订单的创建，它包括插入订单数据、生成订单快照、发送邮件通知卖家和记录货品销售数量等。用户从单击“订购”按钮开始，就要等待这些操作全部完成才能看到订购成功的结果。但是这么多业务操作，如何能够让其更快地完成呢？

在上面的场景中，可以使用多线程技术，即将数据一致性不强的操作派发给其他线程处理（也可以使用消息队列），如生成订单快照、发送邮件等。这样做的好处是响应用户请求的线程能够尽可能快地处理完成，缩短了响应时间，提升了用户体验。

3、 更好的编程模型

Java为多线程编程提供了<!--良好、考究并且-->一致的编程模型，使开发人员能够更加专注于问题的解决，即为所遇到的问题建立合适的模型，而不是绞尽脑汁地考虑如何将其多线程化。<!-- 一旦开发人员建立好了模型，稍做修改总是能够方便地映射到Java提供的多线程编程模型上。-->

### 线程优先级

现代操作系统基本采用时分的形式调度运行的线程，操作系统会分出一个个时间片，线程会分配到若干时间片，当线程的时间片用完了就会发生线程调度，并等待着下次分配。线程分配到的时间片多少也就决定了线程使用处理器资源的多少，而线程优先级就是决定线程需要多或者少分配一些处理器资源的线程属性。

在Java线程中，通过一个整型成员变量priority来控制优先级，优先级的范围从1~10，在线程构建的时候可以通过setPriority(int)方法来修改优先级，默认优先级是5，优先级高的线程分配时间片的数量要多于优先级低的线程。

设置线程优先级时，针对频繁阻塞（休眠或者I/O操作）的线程需要设置较高优先级，而偏重计算（需要较多CPU时间或者偏运算）的线程则设置较低的优先级，确保处理器不会被独占。

**注意**：线程优先级不能作为程序正确性的依赖，因为操作系统可以完全不用理会Java线程对于优先级的设定。

### 线程的状态

Java线程在运行的生命周期中可能处于下表所示的6种不同的状态，在给定的一个时刻，线程只能处于其中的一个状态。

| 状态名称           | 说明                                                   |
| -------------- | ---------------------------------------------------- |
| NEW            | 初始状态，线程被构建，但是还没有调用start()方法                          |
| RUNNABLE       | 运行状态，Java线程将操作系统中的就绪和运行两种状态笼统地称作“运行中”                |
| BLOCKED        | 阻塞状态，表示线程阻塞于锁                                        |
| WAITING        | 等待状态，表示线程进入等待状态，进入该状态表示当前线程需要等待其他线程做出一些特定动作（通知或中断）   |
| TIME_WAITING   | 超时等待状态，该状态不同于WAITING，它是可以在指定的时间自行返回的                 |
| TERMINATED     | 终止状态，表示当前线程已经执行完毕                                    |


线程在自身的生命周期中，并不是固定地处于某个状态，而是随着代码的执行在不同的状态之间进行切换，Java线程状态变迁如下图：

![Java线程状态变迁](https://user-gold-cdn.xitu.io/2018/1/22/1611e594b8b48283?w=1240&h=850&f=png&s=414524)


Java将操作系统中的运行和就绪两个状态合并称为运行状态。阻塞状态是线程阻塞在进入synchronized关键字修饰的方法或代码块（获取锁）时的状态，但是阻塞在java.concurrent包中Lock接口的线程状态却是等待状态，因为java.concurrent包中Lock接口对于阻塞的实现均使用了LockSupport类中的相关方法。

### Daemon线程

Daemon线程是一种支持型线程，因为它主要被用作程序中后台调度以及支持性工作。当一个Java虚拟机中不存在非Daemon线程的时候，Java虚拟机将会退出。可以通过调用Thread.setDaemon(true)将线程设置为Daemon线程。**Daemon属性需要在启动线程之前设置，不能在启动线程之后设置**。

在构建Daemon线程时，不能依靠finally块中的内容来确保执行关闭或清理资源的逻辑。如下代码：

```java
public class Daemon {

    public static void main(String[] args) {
        Thread thread = new Thread(new DeamonRunner(),"DeamonRunner");
        thread.setDaemon(true);
        thread.start();
    }

    static class DeamonRunner implements Runnable{

        @Override
        public void run() {
            try {
                Thread.sleep(2000l);
            } catch (InterruptedException e) {
                //
            }finally {
                System.out.println("DeamonThread finally run.");
            }
        }
    }
}
```
运行Deamon程序，可以看到在终端或者命令提示符没有任何输出。

## 启动线程

在运行线程之前首先要构造一个线程对象，线程对象在构造的时候需要提供线程所需要的属性，如线程所属的线程组、线程优先级、是否是Daemon线程等信息。

```java
private void init(ThreadGroup g, Runnable target, String name,long stackSize,AccessControlContext acc) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }
    // 当前线程就是该线程的父线程
    Thread parent = currentThread();
    this.group = g;
    // 将daemon、priority属性设置为父线程的对应属性
    this.daemon = parent.isDaemon();
    this.priority = parent.getPriority();
    this.name = name.toCharArray();
    this.target = target;
    setPriority(priority);
    // 将父线程的InheritableThreadLocal复制过来
    if (parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals=ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    // 分配一个线程ID
    tid = nextThreadID();
}
```

在上述过程中，一个新构造的线程对象是由其parent线程来进行空间分配的，而child线程继承了parent是否为Deamon、优先级和加载资源的ContextClassLoader以及可继承的ThreadLocal，同时还会分配一个唯一的ID来标识这个child线程。

线程对象在初始化完成之后，调用start()方法就可以启动这个线程。**线程start()方法的含义是：当前线程（即parent线程）同步告知Java虚拟机，只要线程规划器空闲，应立即启动调用start()方法的线程。**

> 启动一个线程前，最好为这个线程设置线程名称，因为这样在使用jstack分析程序或者进行问题排查时，就会给开发人员提供一些提示，自定义的线程最好能够起个名字。

### 理解中断

中断可以理解为线程的一个标识位属性，它表示一个运行中的线程是否被其他线程进行了中断操作。中断好比其他线程对该线程打了个招呼，其他线程通过调用该线程的interrupt()方法对其进行中断操作。

线程通过检查自身是否被中断来进行响应，线程通过方法isInterrupted()来进行判断是否被中断，也可以调用静态方法Thread.interrupted()对当前线程的中断标识位进行复位。如果该线程已经处于终结状态，即使该线程被中断过，在调用该线程对象的isInterrupted()时依旧会返回false。

从Java的API中可以看到，许多声明抛出InterruptedException的方法（例如Thread.sleep(longmillis)方法）这些方法在抛出InterruptedException之前，Java虚拟机会先将该线程的中断标识位清除，然后抛出InterruptedException，此时调用isInterrupted()方法将会返回false。

### 过期的suspend()、resume()和stop()

suspend()、resume()和stop()方法完成了线程的暂停、恢复和终止工作，而且非常“人性化”。但是这些API是过期的，也就是不建议使用的。

不建议使用的原因主要有：以suspend()方法为例，在调用后，线程不会释放已经占有的资源（比如锁），而是占有着资源进入睡眠状态，这样容易引发死锁问题。同样，stop()方法在终结一个线程时不保证线程的资源正常释放，通常是没有给予线程完成资源释放工作的机会，因此会导致程序可能工作在不确定状态下。

> 因为suspend()、resume()和stop()方法带来的副作用，这些方法才被标注为不建议使用的过期方法，而暂停和恢复操作可以用等待/通知机制来替代。


### 安全地终止线程

中断操作是一种简便的线程间交互方式，而这种交互方式最适合用来取消或停止任务。除了中断以外，还可以利用一个boolean变量来控制是否需要停止任务并终止该线程。

```java
public class Shutdown {
    public static void main(String[] args) throws Exception {
        Runner one = new Runner();
        Thread countThread = new Thread(one, "CountThread");
        countThread.start();
        // 睡眠1秒，main线程对CountThread进行中断，使CountThread能够感知中断而结束
        TimeUnit.SECONDS.sleep(1);
        countThread.interrupt();
        Runner two = new Runner();
        countThread = new Thread(two, "CountThread");
        countThread.start();
        // 睡眠1秒，main线程对Runner two进行取消，使CountThread能够感知on为false而结束
        TimeUnit.SECONDS.sleep(1);
        two.cancel();
    }

    private static class Runner implements Runnable {
        private long i;
        private volatile boolean on = true;

        @Override
        public void run() {
            while (on && !Thread.currentThread().isInterrupted()) {
                i++;
            }
            System.out.println("Count i = " + i);
        }

        public void cancel() {
            on = false;
        }
    }
}
```

main线程通过中断操作和cancel()方法均可使CountThread得以终止。这种通过标识位或者中断操作的方式能够使线程在终止时有机会去清理资源，而不是武断地将线程停止，因此这种终止线程的做法显得更加安全和优雅。

## 线程间通信

线程开始运行，拥有自己的栈空间，就如同一个脚本一样，按照既定的代码一步一步地执行，直到终止。但是，每个运行中的线程，如果仅仅是孤立地运行，那么没有一点儿价值，或者说价值很少，如果多个线程能够相互配合完成工作，这将会带来巨大的价值。

### volatile和synchronized关键字

Java支持多个线程同时访问一个对象或者对象的成员变量，由于每个线程可以拥有这个变量的拷贝（虽然对象以及成员变量分配的内存是在共享内存中的，但是每个执行的线程还是可以拥有一份拷贝，这样做的目的是加速程序的执行，这是现代多核处理器的一个显著特性），所以程序在执行过程中，一个线程看到的变量并不一定是最新的。

关键字volatile可以用来修饰字段（成员变量），就是告知程序任何对该变量的访问均需要从共享内存中获取，而对它的改变必须同步刷新回共享内存，它能保证所有线程对变量访问的可见性。

关键字synchronized可以修饰方法或者以同步块的形式来进行使用，它主要确保多个线程在同一个时刻，只能有一个线程处于方法或者同步块中，它保证了线程对变量访问的**可见性和排他性**。

通过使用javap工具查看生成的class文件信息来分析synchronized关键字的实现细节，代码如下

```java
public class Synchronized {

    public static void main(String[] args) {
        synchronized (Synchronized.class){
            m();
        }
    }

    public static synchronized void m(){

    }
}
```

执行javap -v Synchronized.class，部分相关输出如下所示：

![javap -v Synchronized.class部分输出内容](https://user-gold-cdn.xitu.io/2018/1/22/1611e594b8837837?w=1240&h=1141&f=png&s=527787)

对于同步块的实现使用了monitorenter和monitorexit指令，而同步方法则是依赖方法修饰符上的ACC_SYNCHRONIZED来完成。无论采用哪种方式，其本质是对一个对象的监视器进行获取，而这个获取过程是排他的，也就是同一时刻只能有一个线程获取到由synchronized所保护对象的监视器。

任意一个对象都拥有自己的监视器，当这个对象由同步块或者这个对象的同步方法调用时，执行方法的线程必须先获取到该对象的监视器才能进入同步块或者同步方法，而没有获取到监视器（执行该方法）的线程将会被阻塞在同步块和同步方法的入口处，进入BLOCKED状态。

<!-- 对象、对象的监视器、同步队列和执行线程之间的关系如下：-->

### 等待／通知机制

等待／通知机制是指一个线程A调用了对象O的wait()方法进入等待状态，而另一个线程B调用了对象O的notify()或notifyAll()方法，线程A收到通知后从对象O的wait()方法返回，进而执行后续操作。上述两个线程对象O来完成交互，而对象上的wait()和notify/notifyAll()的关系就如同开关信号一样，用来完成等待方通知方之间的交互工作。

> 等待/通知的相关方法是任意Java对象都具备的，这些方法被定义在所有对象的超类java.lang.Object上。

1、实现生产者-消费者模型，代码如下：

```java
public class WaitNotify {

    private final static int CONTAINER_MAX_LENGTH = 3;

    private static Queue<Integer> resources = new LinkedList<Integer>();

    //作为synchronized的对象监视器
    private static final Object lock = new Object();

    /**
     * 消息者
     */
    static class Consumer implements Runnable {

        @Override
        public void run() {
            synchronized (lock) {
                // 不能使用if判断，防止过早唤醒
                while (resources.isEmpty()) {
                    try {
                        // 当前释放锁，线程进入等待状态。
                        lock.wait(); 
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println(Thread.currentThread().getName() + " get number is " + resources.remove());
                // 唤醒所有等待状态的线程
                lock.notifyAll();
            }
        }
    }


    /**
     * 生产者
     */
    static class Producer implements Runnable {

        @Override
        public void run() {
            synchronized (lock) {
                while (resources.size() == CONTAINER_MAX_LENGTH) {
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                int number = (int) (Math.random() * 100);
                System.out.println(Thread.currentThread().getName() + " produce number is " + number);
                resources.add(number);
                lock.notifyAll();
            }
        }
    }


    public static void main(String[] args) {
        for (int i = 0; i < 50; i++) {
            new Thread(new Consumer(), "consumer-" + i).start();
        }

        for (int i = 0; i < 50; i++) {
            new Thread(new Producer(), "producer-" + i).start();
        }
    }
}
```

> > 调用wait()、notify()以及notifyAll()时需要注意的细节，如下:
> 
> * 使用wait()、notify()和notifyAll()时需要先对调用对象加锁。
> 
> * 调用wait()方法后，线程状态由RUNNING变为WAITING，并将当前线程放置到对象的等待队列。
> 
> * notify()或notifyAll()方法调用后，等待线程依旧不会从wait()返回，需要调用notify()或notifAll()的线程释放锁之后，等待线程才有机会从wait()返回。
> 
> * notify()方法将等待队列中的一个等待线程从等待队列中移到同步队列中，而notifyAll()方法则是将等待队列中所有的线程全部移到同步队列，被移动的线程状态由WAITING变为BLOCKED。
> 
> * 从wait()方法返回的前提是获得了调用对象的锁。

2、面试题：设计一个程序，启动三个线程A,B,C,各个线程只打印特定的字母，各打印10次，例如A线程只打印‘A’。要求在控制台依次显示“ABCABC…”

```java
public class WaitNotify02 {


    public static void main(String[] args) {
        Print print = new Print(15);
        new Thread(print, "A").start();
        new Thread(print, "B").start();
        new Thread(print, "C").start();
    }


    private final static Object lock = new Object();

    static class Print implements Runnable {

        private int max_print;

        private int count = 0;

        private String str = "A";

        public Print(int max_print) {
            this.max_print = max_print;
        }

        @Override
        public void run() {
            synchronized (lock) {
                String name = Thread.currentThread().getName();
                while (count < max_print) {
                    if (str.equals(name)) {
                        System.out.print(name);
                        if (str.equals("A")) {
                            str = "B";
                        } else if (str.equals("B")) {
                            str = "C";
                        } else {
                            count++;
                            str = "A";
                        }
                        lock.notifyAll();
                    } else {
                        try {
                            lock.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }
    }
}
```