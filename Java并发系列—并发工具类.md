## Java中的并发工具类

## CountDownLatch

CountDownLatch也叫闭锁，在JDK1.5被引入，允许一个或多个线程等待其他线程完成操作后再执行。

CountDownLatch内部会维护一个初始值为线程数量的计数器，主线程执行await方法，如果计数器大于0，则阻塞等待。当一个线程完成任务后，计数器值减1。当计数器为0时，表示所有的线程已经完成任务，等待的主线程被唤醒继续执行。

### 使用示例

```java
public class CountDownLatchDemo {
    final static SimpleDateFormat sdf=new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch=new CountDownLatch(2);//两个工人的协作
        Worker worker1=new Worker("zhang san", 5000, latch);
        Worker worker2=new Worker("li si", 8000, latch);
        worker1.start();//
        worker2.start();//
        latch.await();//等待所有工人完成工作
        System.out.println("all work done at "+sdf.format(new Date()));
    }


    static class Worker extends Thread{
        String workerName;
        int workTime;
        CountDownLatch latch;
        public Worker(String workerName ,int workTime ,CountDownLatch latch){
             this.workerName=workerName;
             this.workTime=workTime;
             this.latch=latch;
        }
        public void run(){
            System.out.println("Worker "+workerName+" do work begin at "+sdf.format(new Date()));
            doWork();//工作了
            System.out.println("Worker "+workerName+" do work complete at "+sdf.format(new Date()));
            latch.countDown();//工人完成工作，计数器减一

        }

        private void doWork(){
            try {
                Thread.sleep(workTime);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }


}
```

### 实现原理

CountDownLatch实现主要基于java同步器AQS，其内部维护一个AQS子类，并重写了相关方法。

```java
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

await()实现：
```java
public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
}
```

countDown()实现：
```java
public void countDown() {
    sync.releaseShared(1);
}
```

### CyclicBarrier

CyclicBarrier的字面意思是可循环使用(Cyclic)的屏障(Barrier)。它要做的事情是,让一
组线程到达一个屏障(也可以叫同步点)时被阻塞,直到最后一个线程到达屏障时,屏障才会
开门,所有被屏障拦截的线程才会继续运行。

CyclicBarrier还提供一个更高级的构造函数CyclicBarrier(int parties,Runnable barrier-
Action),用于在线程到达屏障时,优先执行barrierAction,方便处理更复杂的业务场景。

CyclicBarrier可以用于多线程计算数据,最后合并计算结果的场景。

**CyclicBarrier和CountDownLatch的区别**

CountDownLatch的计数器只能使用一次,而CyclicBarrier的计数器可以使用reset()方法重
置。所以CyclicBarrier能处理更为复杂的业务场景。

### Semaphore

Semaphore(信号量)是用来控制同时访问特定资源的线程数量,它通过协调各个线程,以保证合理的使用公共资源。

Semaphore的用法也很简单,首先线程使用Semaphore的acquire()方法获取一个许可证,使用完之后调用release()方法归还许可证。

### Exchanger

Exchanger(交换者)是一个用于线程间协作的工具类。Exchanger用于进行线程间的数据交
换。它提供一个同步点,在这个同步点,两个线程可以交换彼此的数据。这两个线程通过
exchange方法交换数据,如果第一个线程先执行exchange()方法,它会一直等待第二个线程也
执行exchange方法,当两个线程都到达同步点时,这两个线程就可以交换数据,将本线程生产
出来的数据传递给对方。

