## Java中的线程池

Java中的线程池是运用场景最多的并发框架,几乎所有需要异步或并发执行任务的程序都可以使用线程池。在开发过程中,合理地使用线程池能够带来3个好处。

- 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。

- 提高响应速度。当任务到达时,任务可以不需要等到线程创建就能立即执行。

- 提高线程的可管理性。线程是稀缺资源,如果无限制地创建,不仅会消耗系统资源,还会降低系统的稳定性,使用线程池可以进行统一分配、调优和监控。但是,要做到合理利用线程池,必须对其实现原理了如指掌。

### 线程池的创建

```java
public ThreadPoolExecutor(int corePoolSize,
                            int maximumPoolSize,
                            long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue,
                            ThreadFactory threadFactory,
                            RejectedExecutionHandler handler) {
}
```

参数说明：

* corePoolSize

    线程池中的核心线程数，当提交一个任务时，线程池创建一个新线程执行任务，直到当前线程数等于corePoolSize；如果当前线程数为corePoolSize，继续提交的任务被保存到阻塞队列中，等待被执行；如果执行了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有核心线程。

* maximumPoolSize

    线程池中允许的最大线程数。如果当前阻塞队列满了，且继续提交任务，则创建新的线程执行任务，前提是当前线程数小于maximumPoolSize；

* keepAliveTime

    线程空闲时的存活时间，即当线程没有任务执行时，继续存活的时间；默认情况下，该参数只在线程数大于corePoolSize时才有用；

* unit

    keepAliveTime的单位；

* workQueue

    用来保存等待被执行的任务的阻塞队列，且任务必须实现Runable接口，在JDK中提供了如下阻塞队列：

    1、ArrayBlockingQueue：基于数组结构的有界阻塞队列，按FIFO排序任务；
    
    2、LinkedBlockingQuene：基于链表结构的阻塞队列，按FIFO排序任务，吞吐量通常要高于ArrayBlockingQuene；
    
    3、SynchronousQuene：一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQuene；
    
    4、priorityBlockingQuene：具有优先级的无界阻塞队列；

* threadFactory

    创建线程的工厂，通过自定义的线程工厂可以给每个新建的线程设置一个具有识别度的线程名。

* handler

    线程池的饱和策略，当阻塞队列满了，且没有空闲的工作线程，如果继续提交任务，必须采取一种策略处理该任务，线程池提供了4种策略：

    1、AbortPolicy：直接抛出异常，默认策略；
    
    2、CallerRunsPolicy：用调用者所在的线程来执行任务；
    
    3、DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
    
    4、DiscardPolicy：直接丢弃任务；
当然也可以根据应用场景实现RejectedExecutionHandler接口，自定义饱和策略，如记录日志或持久化存储不能处理的任务。


所以Worker被设计成一个AQS是为了根据Worker的锁来判断是否是闲置线程，是否可以被强制中断。

### 线程池源码分析

#### ThreadPoolExecutor状态和属性

**ThreadPoolExecutor线程池有5个状态，分别是**：

1）RUNNING：可以接受新的任务，也可以处理阻塞队列里的任务

2）SHUTDOWN：不接受新的任务，但是可以处理阻塞队列里的任务

3）STOP：不接受新的任务，不处理阻塞队列里的任务，中断正在处理的任务

4）TIDYING：过渡状态，也就是说所有的任务都执行完了，当前线程池已经没有有效的线程，这个时候线程池的状态将会TIDYING，并且将要调用terminated方法

5）TERMINATED：终止状态。terminated方法调用完成以后的状态

**状态之间可以进行转换**：

RUNNING -> SHUTDOWN：手动调用shutdown方法，或者ThreadPoolExecutor要被GC回收的时候调用finalize方法，finalize方法内部也会调用shutdown方法

(RUNNING or SHUTDOWN) -> STOP：调用shutdownNow方法

SHUTDOWN -> TIDYING：当队列和线程池都为空的时候

STOP -> TIDYING：当线程池为空的时候

TIDYING -> TERMINATED：terminated方法调用完成之后

ThreadPoolExecutor内部还保存着线程池的有效线程个数。

线程池初始化状态线程数变量：
```java
// 初始化状态和数量，状态为RUNNING，线程数为0
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```


ThreadPoolExecutor执行任务

ThreadPoolExecutor执行任务的过程如下图所示：

![](https://upload-images.jianshu.io/upload_images/2184951-18c425ebd3877453.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/639)

1）如果当前运行的线程小于corePoolSize，则创建新线程来执行；

2）如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue；

3）如果无法将任务加入BlockingQueue（队列已满），则创建新的线程来处理任务；

4）如果创建新线程将使当前运行的线程超出maximumPoolSize,任务将被拒绝，并调用handler.rejectedExecution(command, this)方法。


execute()方法源码如下： 

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    // 过程1
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    // 过程2
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 再次检查线程池状态
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 线程池中没有可用的工作线程时    
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false)) // 过程3
        // 过程4
        reject(command);
}
```

在进入addWorker(Runnable firstTask, boolean core)方法之前，我们先来认识下Worker类。其源码如下：
```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    /**
        * This class will never be serialized, but we provide a
        * serialVersionUID to suppress a javac warning.
        */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
        * Creates with given first task and thread from ThreadFactory.
        * @param firstTask the first task (null if none)
        */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }

    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```


```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
                firstTask == null &&
                ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            // 1、线程的数量大于线程最大值；2、
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // workerCount加1成功，跳出循坏。    
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            // 线程池状态发生变化，继续循坏。
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    // worker是否执行标识
    boolean workerStarted = false;
    // worker是否添加成功标识
    boolean workerAdded = false;
    // 保存创建的worker变量
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        // 检查线程是否创建成功
        if (t != null) {
            // 加锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // 将w存储到workers容器中
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    // 添加成功标识
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                // 执行任务
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

worker执行任务过程：
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                    (Thread.interrupted() &&
                    runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

线程池中的线程执行任务分两种情况，如下：

1）在execute()方法中创建一个线程时，会让这个线程执行当前任务。

2）这个线程执行完当前任务，会反复从BlockingQueue获取任务来执行。



**向线程池提交任务**  
>  
>  
>  
> - execute()方法  
>  
>     execute()方法用于提交不需要返回值的任务,所以无法判断任务是否被线程池执行成功。  
>  
> - submit()方法  
>  
>     submit()方法用于提交需要返回值的任务。线程池会返回一个future类型的对象,通过这个future对象可以判断任务是否执行成功,并且可以通过future的get()方法来获取返回值,get()方  
>     法会阻塞当前线程直到任务完成,而使用get(long timeout,TimeUnit unit)方法则会阻塞当前线程一段时间后立即返回,这时候有可能任务没有执行完。

### 关闭线程池

可以通过调用线程池的shutdown或shutdownNow方法来关闭线程池。它们的原理是遍历线程池中的工作线程,然后逐个调用线程的interrupt方法来中断线程,所以无法响应中断的任务可能永远无法终止。
但是它们存在一定的区别，shutdownNow首先将线程池的状态设置成STOP,然后尝试停止所有的正在执行或暂停任务的线程,并返回等待执行任务的列表,而shutdown只是将线程池的状态设置成SHUTDOWN状态,然后中断所有没有正在执行任务的线程。    

只要调用了这两个关闭方法中的任意一个，isShutdown方法就会返回true。当所有的任务都已关闭后，才表示线程池关闭成功，这时调用isTerminaed方法会返回true。


[Java线程池ThreadPoolExecutor源码分析](http://fangjian0423.github.io/2016/03/22/java-threadpool-analysis/)

### 合理地配置线程池

要想合理地配置线程池,就必须首先分析任务特性,可以从以下几个角度来分析。

- 任务的性质：CPU密集型任务、IO密集型任务和混合型任务。

- 任务的优先级：高、中和低。

- 任务的执行时间：长、中和短。

- 任务的依赖性：是否依赖其他系统资源,如数据库连接。

性质不同的任务可以用不同规模的线程池分开处理：

1）CPU密集型任务应配置尽可能小的线程,如配置N cpu +1个线程的线程池。

2）IO密集型任务线程并不是一直在执行任务,则应配置尽可能多的线程,如2*N cpu 。

3）混合型的任务,如果可以拆分,将其拆分成一个CPU密集型任务和一个IO密集型任务,只要这两个任务执行的时间相差不是太大,那么分解后执行的吞吐量将高于串行执行的吞吐量。如果这两个任务执行时间相差太大,则没必要进行分解。

> 可以通过Runtime.getRuntime().availableProcessors()方法获得当前设备的CPU个数。

#### 线程池监控

如果在系统中大量使用线程池，则有必要对线程池进行监控，方便在出现问题时，可以根据线程池的使用状况快速定位问题。可以通过线程池提供的参数进行监控，在监控线程池的时机可以使用以下属性：

* taskCount：线程池需要执行的任务数量。

* completedTaskCount：线程池在运行过程中已完成的任务数量，小于或等于taskCount。

* largestPoolSize：线程池里曾经创建过的最大线程数量。通过这个数据可以直到线程池是否曾经满过。

* getPoolSize：线程池的线程数量。

* getActiveCount：获取活动的线程数。

通过扩展线程池进行监控。可以通过继承线程池来自定义线程池，重写线程池的beforeExecute、afterExecute和terminated方法，也可以在任务执行前、执行后和线程池关闭之前执行一些代码来进行监控。
