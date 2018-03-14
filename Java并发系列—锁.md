# Java并发系列—锁

## Lock接口

锁是用来控制多个线程访问共享资源的方式，一般来说，一个锁能够防止多线程同时访问共享资源(但是有些锁可以允许多个线程并发的访问共享资源，比如读写锁)。Lock接口（以及相关实现类）是JavaSE 5之后新增的用来实现锁功能，它提供了与synchronized关键字类似的同步功能，只是在使用时需要显式地获取和释放锁。

### Lock的API

```java
public interface Lock {

    // 获取锁，调用该方法当前线程将会获取锁，当锁获得后，从该方法返回
    void lock();

    // 可中断地获取锁，即在锁的获取中可以中断当前线程
    void lockInterruptibly() throws InterruptedException;

    // 尝试非阻塞获取锁
    boolean tryLock();

    // 超时的获取锁
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    // 释放锁
    void unlock();

    // 获取等待通知组件，提供类似wait()、nofity()和notifyAll的功能
    Condition newCondition();
}
```

### Lock使用的方式

```java
  // 显式创建lock对象
  Lock lock = new ReentrantLock();
  // 加锁
  lock.lock();
  try {
      
  } finally {
    // 释放锁
    lock.unlock();
  }
```

在finally块中释放锁，目的是保证在获取到锁之后，最终能够被释放。不要将获取锁的过程写在try块中，因为如果在获取锁（自定义锁的实现）时发生了异常，异常抛出的同时，也会导致锁无故释放。

### Lock接口提供的synchronized关键字所不具备的主要特性

虽然它缺少了（通过synchronized块或者方法所提供的）隐式获取、释放锁的便捷性，但是却拥有了锁获取与释放的可操作性、可中断的获取锁以及超时获取锁等多种synchronized关键字所不具备的同步特性。

| 特性        | 描述                                                           |
|-----------|--------------------------------------------------------------|
| 尝试非阻塞地获取锁 | 当前线程尝试获取锁，如果这一时刻没有被其他线程获取到，则成功获取并持有锁                         |
| 能被中断地获取锁  | 与synchronized不同，获取到锁的线程能够响应中断，当获取到锁的线程被中断时，中断异常将会被抛出，同时锁会被释放 |
| 超时获取锁     | 在指定的截止时间之前获取锁，如果截止时间到了依旧无法获取锁，则返回。                           |

## 队列同步器（AbstractQueuedSynchronizer）

**队列同步器AbstractQueuedSynchronizer（以下简称同步器），是用来构建锁或者其他同步组件的基础框架，它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作，并发包的作者（Doug Lea）期望它能够成为实现大部分同步需求的基础**。

**同步器的主要使用方式是继承**，子类通过继承同步器并实现它的抽象方法来管理同步状态，在抽象方法的实现过程中免不了要对同步状态进行更改，这时就需要使用同步器提供的3个方法getState()、setState(int newState)和compareAndSetState(int expect,int update)）来进行操作，因为它们能够保证状态的改变是安全的。

子类推荐被定义为自定义同步组件的静态内部类，同步器自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用，同步器既可以支持独占式地获取同步状态，也可以支持共享式地获
取同步状态，这样就可以方便实现不同类型的同步组件（ReentrantLock、
ReentrantReadWriteLock和CountDownLatch等）。

> 同步器是实现锁（也可以是任意同步组件）的关键，在锁的实现中聚合同步器，利用同步器实现锁的语义。可以这样理解二者之间的关系：锁是面向使用者的，它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；同步器面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。锁和同步器很好地隔离了使用者和实现者所需关注的领域。

### 同步器接口与示例

同步器的设计是基于模板方法模式的，使用者需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法。

同步器可重写的方法与描述如下表所示:

| 方法名称                                        | 描述                                                  |
|---------------------------------------------|-----------------------------------------------------|
| protected boolean tryAcquire(int arg)       | 独占式获取同步状态，实现该方法需要查询当前状态并判断同步状态是否符合预期，然后再进行CAS设置同步状态 |
| protected boolean tryRelease(int arg)       | 独占式释放同步状态，等待获取同步状态的线程有机会获取同步状态                      |
| protected int tryAcquireShared(int arg)     | 共享式获取同步状态，返回大于等于0的值，表示获取成功，反之，获取失败                  |
| protected boolean tryReleaseShared(int arg) | 共享式释放同步                                             |
| protected boolean isHeldExclusively()       | 当前同步器是否在独占模式下被线程占用，一般该方法表示是否被当前线程所独占                |

**利用同步器实现独占锁的示例**：

```java
class Mutex implements Lock {
    // 静态内部类，自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {
        // 是否处于占用状态
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        // 当状态为0的时候获取锁
        public boolean tryAcquire(int acquires) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        // 释放锁，将状态设置为0
        protected boolean tryRelease(int releases) {
            if (getState() == 0) throw new
                    IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
        // 返回一个Condition，每个condition都包含了一个condition队列
        Condition newCondition() { return new ConditionObject(); }
    }
    // 仅需要将操作代理到Sync上即可
    private final Sync sync = new Sync();

    public void lock() { sync.acquire(1); }
    
    public boolean tryLock() { return sync.tryAcquire(1); }
    
    public void unlock() { sync.release(1); }
    
    public Condition newCondition() { return sync.newCondition(); }
    
    public boolean isLocked() { return sync.isHeldExclusively(); }
    
    public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
    
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
    
    public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
}
```

Mutex中定义了一个静态内部类，该内部类继承了同步器并实现了独占式获取和释放同步状态。在tryAcquire(int acquires)方法中，如果经过CAS设置成功（同步状态设置为1），则代表获取了同步状态，而在tryRelease(int releases)方法中只是将同步状态重置为0。用户使用Mutex时并不会直接和内部同步器的实现打交道，而是调用Mutex提供的方法，在Mutex的实现中，以获取锁的lock()方法为例，只需要在方法实现中调用同步器的模板方法acquire(int args)即可，当前线程调用该方法获取同步状态失败后会被加入到同步队列中等待，这样就大大降低了实现一个可靠自定义同步组件的门槛。

### 队列同步器的实现分析

从实现角度分析同步器是如何完成线程同步的，主要包括：同步队列、独占式同步状态获取与释放、共享式同步状态获取与释放以及超时获取同步状态等同步器的核心数据结构与模板方法。

#### 同步队列

同步器依赖内部的同步队列（一个FIFO双向队列）来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个节点（Node）并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态。

节点说明

节点是构成同步队列的基础，同步器拥有首节点（head）和尾节点（tail），没有成功获取同步状态的线程将会成为节点加入该队列的尾部，同步队列的基本结构如下所示：

![同步队列的基本结构.png](http://upload-images.jianshu.io/upload_images/2889214-59cacb3cd4632956.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同步器包含了两个节点类型的引用，一个指向头节点，而另一个指向尾节点。
试想一下，当一个线程成功地获取了同步状态（或者锁），其他线程将无法获取到同步状态，转而被构造成为节点并加入到同步队列中，而这个加入队列的过程必须要保证线程安全，因此同步器提供了一个基于CAS的设置尾节点的方法：compareAndSetTail(Node expect,Node update)，它需要传递当前线程“认为”的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立关联。

同步队列遵循FIFO，首节点是获取同步状态成功的节点，首节点的线程在释放同步状态时，将会唤醒后继节点，而后继节点将会在获取同步状态成功时将自己设置为首节点，该过程如下图所示：

![首节点设置.png](http://upload-images.jianshu.io/upload_images/2889214-d35d9b9930ed01f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 独占式同步状态获取与释放

通过调用同步器的acquire(int arg)方法可以获取同步状态，_该方法对中断不敏感_？，也就是由于线程获取同步状态失败后进入同步队列中，后续对线程进行中断操作时，线程不会从同步队列中移出。该方法代码如下：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 防止获取锁成功的线程调用中断方法
        selfInterrupt();
}
```

其主要逻辑是：首先调用自定义同步器实现的tryAcquire(int arg)方法，该方法保证线程安全的获取同步状态，如果同步状态获取失败，则构造同步节点（独占式Node.EXCLUSIVE，同一时刻只能有一个线程成功获取同步状态）并通过addWaiter(Node node)方法将该节点加入到同步队列的尾部，最后调用acquireQueued(Node node,int arg)方法，使得该节点以“死循环”的方式获取同步状态。如果获取不到则阻塞节点中的线程，而被阻塞线程的唤醒主要依靠前驱节点的出队或阻塞线程被中断来实现。

下面分析一下相关工作。首先是节点的构造以及加入同步队列,代码如下：

```java

private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    // 尾节点存在时
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    
    enq(node);
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

在enq(final Node node)方法中，同步器通过“死循环”来保证节点的正确添加，在“死循环”中只有通过CAS将节点设置成为尾节点之后，当前线程才能从该方法返回，否则，当前线程不断地尝试设置。可以看出，enq(final Node node)方法将并发添加节点的请求通过CAS变得“串行化”了。

节点进入同步队列之后，就进入了一个自旋的过程，每个节点（或者说每个线程）都在自省地观察，当条件满足，获取到了同步状态，就可以从这个自旋过程中退出，否则依旧留在这个自旋过程中（并会阻塞节点的线程）。源码如下：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

当前线程获取同步状态并执行了相应逻辑之后，就需要释放同步状态，使得后续节点能够继续获取同步状态。通过调用同步器的release(int arg)方法可以释放同步状态，该方法在释放了同步状态之后，会唤醒其后继节点（进而使后继节点重新尝试获取同步状态）。源码如下：

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

该方法执行时，会唤醒头节点的后继节点线程，unparkSuccessor(Node node)方法使用LockSupport来唤醒处于等待状态的线程。

### 共享式同步状态获取与释放

共享式获取与独占式获取最主要的区别在于同一时刻能否有多个线程同时获取到同步状态。以文件的读写为例，如果一个程序在对文件进行读操作，那么这一时刻对于该文件的写操作均被阻塞，而读操作能够同时进行。写操作要求对资源的独占式访问，而读操作可以是共享式访问。

通过调用同步器的acquireShared(int arg)方法可以共享式地获取同步状态,源码如下：

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

在acquireShared(int arg)方法中，同步器调用tryAcquireShared(int arg)方法尝试获取同步状态，tryAcquireShared(int arg)方法返回值为int类型，当返回值大于等于0时，表示能够获取到同步状态。因此，在共享式获取的自旋过程中，成功获取到同步状态并退出自旋的条件就是tryAcquireShared(int arg)方法返回值大于等于0。可以看到，在doAcquireShared(int arg)方法的自旋过程中，如果当前节点的前驱为头节点时，尝试获取同步状态，如果返回值大于等于0，表示该次获取同步状态成功并从自旋过程中退出。

与独占式一样，共享式获取也需要释放同步状态，通过调用releaseShared(int arg)方法可以释放同步状态，源码如下：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

该方法在释放同步状态之后，将会唤醒后续处于等待状态的节点。对于能够支持多个线程同时访问的并发组件（比如Semaphore），它和独占式主要区别在于tryReleaseShared(int arg)方法必须确保同步状态（或者资源数）线程安全释放，一般是通过循环和CAS来保证的，因为释放同步状态的操作会同时来自多个线程。

通过调用同步器的doAcquireNanos(int arg,long nanosTimeout)方法可以超时获取同步状态，即在指定的时间段内获取同步状态，如果获取到同步状态则返回true，否则，返回false。该方法提供了传统Java同步操作（比如synchronized关键字）所不具备的特性。

在分析该方法的实现前，先介绍一下响应中断的同步状态获取过程。在Java 5之前，当一个线程获取不到锁而被阻塞在synchronized之外时，对该线程进行中断操作，此时该线程的中断标志位会被修改，但线程依旧会阻塞在synchronized上，等待着获取锁。在Java 5中，同步器提供了acquireInterruptibly(int arg)方法，这个方法在等待获取同步状态时，如果当前线程被中断，会立刻返回，并抛出InterruptedException。

超时获取同步状态过程可以被视作响应中断获取同步状态过程的“增强版”，
doAcquireNanos(int arg,long nanosTimeout)方法在支持响应中断的基础上，增加了超时获取的特性。针对超时获取，主要需要计算出需要睡眠的时间间隔nanosTimeout，为了防止过早通知，nanosTimeout计算公式为：nanosTimeout-=now-lastTime，其中now为当前唤醒时间，lastTime为上
次唤醒时间，如果nanosTimeout大于0则表示超时时间未到，需要继续睡眠nanosTimeout纳秒，反之，表示已经超时,源码如下：

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    long lastTime = System.nanoTime();
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (; ; ) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            if (nanosTimeout <= 0)
                return false;
            if (shouldParkAfterFailedAcquire(p, node)
                    && nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            long now = System.nanoTime();
            //计算时间，当前时间now减去睡眠之前的时间lastTime得到已经睡眠
            //的时间delta，然后被原有超时时间nanosTimeout减去，得到了
            //还应该睡眠的时间
            nanosTimeout -= now - lastTime;
            lastTime = now;
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

该方法在自旋过程中，当节点的前驱节点为头节点时尝试获取同步状态，如果获取成功则从该方法返回，这个过程和独占式同步获取的过程类似，但是在同步状态获取失败的处理上有所不同。如果当前线程获取同步状态失败，则判断是否超时（nanosTimeout小于等于0表示已经超时），如果没有超时，重新计算超时间隔nanosTimeout，然后使当前线程等待nanosTimeout纳秒（当已到设置的超时时间，该线程会从LockSupport.parkNanos(Object blocker,long nanos)方法返回）。

如果nanosTimeout小于等于spinForTimeoutThreshold（1000纳秒）时，将不会使该线程进行超时等待，而是进入快速的自旋过程。原因在于，非常短的超时等待无法做到十分精确，如果这时再进行超时等待，相反会让nanosTimeout的超时从整体上表现得反而不精确。因此，在超时非常短的场景下，同步器会进入无条件的快速自旋。

### 重入锁（ReentrantLock）

重入锁ReentrantLock，顾名思义，就是支持重进入的锁，它表示该锁能够支持一个线程对资源的重复加锁。除此之外，该锁的还支持获取锁时的公平和非公平性选择。

ReentrantLock虽然没能像synchronized关键字一样支持隐式的重进入，但是在调用lock()方法时，已经获取到锁的线程，能够再次调用lock()方法获取锁而不被阻塞。

关于锁获取的公平性问题，如果在绝对时间上，先对锁进行获取的请求一定先被满足，那么这个锁是公平的，反之，是不公平的。公平的获取锁，也就是等待时间最长的线程最优先获取锁，也可以说锁获取是顺序的。ReentrantLock提供了一个构造函数，能够控制锁是否是公平的。

事实上，公平的锁机制往往没有非公平的效率高，但是，并不是任何场景都是以TPS作为唯一的指标，公平锁能够减少“饥饿”发生的概率，等待越久的请求越是能够得到优先满足。

#### 实现重进入

重进入是指任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞，该特性的实现需要解决以下两个问题：

1）线程再次获取锁。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再次成功获取。

2）锁的最终释放。线程重复n次获取了锁，随后在第n次释放该锁后，其他线程能够获取到该锁。锁的最终释放要求锁对于获取进行计数自增，计数表示当前锁被重复获取的次数，而锁被释放时，计数自减，当计数等于0时表示锁已经成功释放。

以非公平性（默认的）实现为例，获取同步状态的代码如下：

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

该方法增加了再次获取同步状态的处理逻辑：通过判断当前线程是否为获取锁的线程来决定获取操作是否成功，如果是获取锁的线程再次请求，则将同步状态值进行增加并返回true，表示获取同步状态成功。

成功获取锁的线程再次获取锁，只是增加了同步状态值，这也就要求ReentrantLock在释放同步状态时减少同步状态值，该方法的代码如下：

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

如果该锁被获取了n次，那么前(n-1)次tryRelease(int releases)方法必须返回false，而只有同步状态完全释放了，才能返回true。可以看到，该方法将同步状态是否为0作为最终释放的条件，当同步状态为0时，将占有线程设置为null，并返回true，表示释放成功。

#### 公平与非公平获取锁的区别

公平性与否是针对获取锁而言的，如果一个锁是公平的，那么锁的获取顺序就应该符合请求的绝对时间顺序，也就是FIFO。

公平锁的tryAcquire()方法如下：

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

hasQueuedPredecessors()方法，即加入了同步队列中当前节点是否有前驱节点的判断，如果该方法返回true，则表示有线程比当前线程更早地请求获取锁，因此需要等待前驱线程获取并释放锁之后才能继续获取锁。

### 读写锁（ReentrantReadWriteLock）

读写锁在同一时刻可以允许多个读线程访问，但是在写线程访问时，所有的读线程和其他写线程均被阻塞。读写锁维护了一对锁，一个读锁和一个写锁，通过分离读锁和写锁，使得并发性相比一般的排他锁有了很大提升。

一般情况下，读写锁的性能都会比排它锁好，因为大多数场景读是多于写的。在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量。

ReentrantReadWriteLock的特性?

#### 使用方式

```java
public class Cache {

    static Map<String,Object> map = new HashMap<String,Object>();

    static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

    static Lock r = rwl.readLock();

    static Lock w = rwl.writeLock();

    // 获取一个key对应的value
    public static final Object get(String key) {
        r.lock();
        try {
            return map.get(key);
        }finally {
            r.unlock();
        }
    }

    // 设置key对应的value，并返回旧的value
    public static final Object put(String key,Object value) {
        w.lock();
        try {
            return map.put(key, value);
        }finally {
            w.unlock();
        }
    }

    // 清空所有的内容
    public static final void clear() {
        w.lock();
        try {
            map.clear();
        }finally {
            w.unlock();
        }
    }
}
```

#### 读写锁的实现分析

读写锁同样依赖自定义同步器来实现同步功能，而读写状态就是其同步器的同步状态。读写锁的自定义同步器需要在同步状态（一个整型变量）上维护多个读线程和一个写线程的状态，使得该状态的设计成为读写锁实现的关键。

如果在一个整型变量上维护多种状态，就一定需要“按位切割使用”这个变量，读写锁将变量切分成了两个部分，高16位表示读，低16位表示写，划分方式如图所示：

![读写锁状态的划分方式](http://upload-images.jianshu.io/upload_images/2889214-558e6db1fea25c07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当前同步状态表示一个线程已经获取了写锁，且重进入了两次，同时也连续获取了两次读锁。读写锁是如何迅速确定读和写各自的状态呢？答案是通过位运算。假设当前同步状态值为S，写状态等于S&0x0000FFFF（将高16位全部抹去），读状态等于S>>>16（无符号补0右移
16位）。当写状态增加1时，等于S+1，当读状态增加1时，等于S+(1<<16)，也就是
S+0x00010000。

> 根据状态的划分能得出一个推论：S不等于0时，当写状态（S&0x0000FFFF）等于0时，则读状态（S>>>16）大于0，即读锁已被获取。

#### 写锁的获取与释放

写锁是一个支持重进入的排它锁。如果当前线程已经获取了写锁，则增加写状态。如果当前线程在获取写锁时，读锁已经被获取（读状态不为0）或者该线程不是已经获取写锁的线程，则当前线程进入等待状态，获取写锁的代码如下：

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // 如果c != 0 and w == 0时，说明读锁已存在。
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 写锁重人情况
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        setState(c + acquires);
        return true;
    }
    // 如果当前线程节点Node存在前驱节点或CAS操作失败，直接返回。
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

该方法除了重入条件（当前线程为获取了写锁的线程）之外，增加了一个读锁是否存在的判断。如果存在读锁，则写锁不能被获取，原因在于：读写锁要确保写锁的操作对读锁可见，如果允许读锁在已被获取的情况下对写锁的获取，那么正在运行的其他读线程就无法感知到当前写线程的操作。因此，只有等待其他读线程都释放了读锁，写锁才能被当前线程获取，而写锁一旦被获取，则其他读写线程的后续访问均被阻塞。

写锁的释放与ReentrantLock的释放过程基本类似，每次释放均减少写状态，当写状态为0时表示写锁已被释放，从而等待的读写线程能够继续访问读写锁，同时前次写线程的修改对后续读写线程可见。

#### 读锁的获取与释放

读锁是一个支持重进入的共享锁，它能够被多个线程同时获取，在没有其他写线程访问（或者写状态为0）时，读锁总会被成功地获取，而所做的也只是（线程安全的）增加读状态。如果当前线程已经获取了读锁，则增加读状态。如果当前线程在获取读锁时，写锁已被其他线程获取，则进入等待状态。

```java
final boolean tryReadLock() {
    Thread current = Thread.currentThread();
    for (;;) {
        int c = getState();
        // 当存在写锁时且当前线程不是拥有写锁的线程时，则直接返回获取失败；
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return false;
        int r = sharedCount(c);
        if (r == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (r == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
            }
            return true;
        }
    }
}
```

在tryAcquireShared(int unused)方法中，如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态。如果当前线程获取了写锁或者写锁未被获取，则当前线程（线程安全，依靠CAS保证）增加读状态，成功获取读锁。

读锁的每次释放（线程安全的，可能有多个读线程同时释放读锁）均减少读状态，减少的值是（1<<16）。

#### 锁降级

锁降级指的是写锁降级成为读锁，一个拥有写锁的线程，再获取到读锁，随后释放（先前拥有的）写锁的过程。如果当前线程拥有写锁，然后将其释放，最后再获取读锁，这种分段完成的过程不能称之为锁降级。


锁降级示例？

锁降级的示例,代码如下：

```java
public void processData() {
    readLock.lock();
    if (!update) {
        // 必须先释放读锁
        readLock.unlock();
        // 锁降级从写锁获取到开始
        writeLock.lock();
        try {
            if (!update) {
                // 准备数据的流程（略）
                update = true;
            }
            readLock.lock();
        } finally {
            writeLock.unlock();
        }
        // 锁降级完成，写锁降级为读锁
    }
    try {
        // 使用数据的流程（略）
    } finally {
        readLock.unlock();
    }
}
```

当数据发生变更后，update变量（布尔类型且volatile修饰）被设置为false，此时所有访问processData()方法的线程都能够感知到变化，但只有一个线程能够获取到写锁，其他线程会被阻塞在读锁和写锁的lock()方法上。当前线程获取写锁完成数据准备之后，再获取读锁，随后释放写锁，完成锁降级。

锁降级中读锁的获取是否必要呢？答案是必要的。主要是为了保证数据的可见性，如果当前线程不获取读锁而是直接释放写锁，假设此刻另一个线程（记作线程T）获取了写锁并修改了数据，那么当前线程无法感知线程T的数据更新。如果当前线程获取读锁，即遵循锁降级的步骤，则线程T将会被阻塞，直到当前线程使用数据并释放读锁之后，线程T才能获取写锁进行数据更新。

#### LockSupport工具

LockSupport定义了一组的公共静态方法,这些方法提供了最基本的线程阻塞和唤醒功能,而LockSupport也成为构建同步组件的基础工具。

LockSupport定义了一组以park开头的方法用来阻塞当前线程,以及unpark(Thread thread)方法来唤醒一个被阻塞的线程。

在Java 6中,LockSupport增加了park(Object blocker)、parkNanos(Object blocker,long nanos)
和parkUntil(Object blocker,long deadline)3个方法,用于实现阻塞当前线程的功能,其中参数
blocker是用来标识当前线程在等待的对象(以下称为阻塞对象),该对象主要用于问题排查和系统监控。

### Condition接口

任意一个Java对象，都拥有一组监视器方法（定义在java.lang.Object上），主要包括wait()、wait(long timeout)、notify()以及notifyAll()方法，这些方法与synchronized同步关键字配合，可以实现等待/通知模式。Condition接口也提供了类似Object的监视器方法，与Lock配合可以实现等待/通知模式，但是这两者在使用方式以及功能特性上还是有差别的。

Object的监视器方法与Condition接口的对比如下：

| 对比项| Object Monitor Methods | Condition|
| ---| ---------------------- | --------- |
| 前置条件 | 获取对象的锁                 | 1）调用Lock.lock()获取锁；2）调用Lock.newCondition()获取Condition对象 |
| 调用方式| 直接调用 | 直接调用|
| 等待队列个数 | 一个| 多个|
|当前线程释放锁并进入等待状态| 支持| 支持|
|当前线程释放锁并进入等待状态，在等待状态中响应中断 | 不支持| 支持||当前线程释放锁并进入超时等待状态    | 支持 | 支持|  
|当前线程释放锁并进入等待状态到将来的某个时间    | 不支持 | 支持| |唤醒等待队列中的一个线程| 支持  | 支持|  
| 唤醒等待队列中的全部线程 | 支持 | 支持|

#### Condition接口与示例

Condition定义了等待/通知两种类型的方法，当前线程调用这些方法时，需要提前获取到Condition对象关联的锁。Condition对象是由Lock对象（调用Lock对象的newCondition()方法）创建出来的，换句话说，Condition是依赖Lock对象的。

一般都会将Condition对象作为成员变量。当调用await()方法后,当前线程会释放锁并在此等待,而其他线程调用Condition对象的signal()方法,通知当前线程后,当前线程才从await()方法返回,并且在返回前已经获取了锁。

Condition的(部分)方法以及描述:

* void await() throws InterruptedException 

    当前线程进入等待状态直到被通知（signal）或中断，当前线程将进入运行状态且await()方法返回的情况，包括：
    其他线程调用该Condition的signal()或signalAll()方法。而当前线程被选中唤醒：

    * 其他线程（调用interrupt()方法）中断当前线程
    * 如果当前等待线程从await()方法返回，那么表明该线程已经获取了Condition对象所对应的锁

* void awaitUninterruptibly()

    当前线程进入等待状态直到被通知，从方法名称上可以看出该方法对中断不敏感

* long awaitNanos(long nanosTimeout) throws InterruptedException

    当前线程进入等待状态直到被通知、中断或者超时。返回值表示剩余的时间，如果在nanosTimeout纳秒之前被唤醒，那么返回值就是(nanosTimeout - 实际耗时)。如果返回值是0或者负数，那么可以认定已经超时了。

* boolean awaitUntil(Date deadline) throws InterruptedException

    当前线程进入等待状态直到被通知、中断或者到某个时间。如果没有到指定时间被通知，方法返回true，否则，表示到了指定时间，方法返回false

* void signal()

    唤醒一个等待在Condition上的线程，该线程从等待方法返回前必须获得与Condition相关联的锁

* void signalAll()

    唤醒所有等待在Condition上的线程，能够从等待方法返回的线程必须获得与Condition相关联的锁

下面通过一个有界队列的示例来深入了解Condition的使用方式。有界队列是一种特殊的队列,当队列为空时,队列的获取操作将会阻塞获取线程,直到队列中有新增元素,当队列已满时,队列的插入操作将会阻塞插入线
程,直到队列出现“空位”,代码如下所示：

```java
public class BoundedQueue<T> {
    private Object[] items;
    // 添加的下标，删除的下标和数组当前数量
    private int addIndex, removeIndex, count;
    private Lock lock = new ReentrantLock();
    private Condition notEmpty = lock.newCondition();
    private Condition notFull = lock.newCondition();
    public BoundedQueue(int size) {
        items = new Object[size];
    }
    // 添加一个元素，如果数组满，则添加线程进入等待状态，直到有"空位"
    public void add(T t) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)
                notFull.await();
            items[addIndex] = t;
            if (++addIndex == items.length)
                addIndex = 0;
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }
    // 由头部删除一个元素，如果数组空，则删除线程进入等待状态，直到有新添加元素
    @SuppressWarnings("unchecked")
    public T remove() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await();
            Object x = items[removeIndex];
            if (++removeIndex == items.length)
                removeIndex = 0;
            --count;
            notFull.signal();
            return (T) x;
        } finally {
            lock.unlock();
        }
    }
}
```

#### Condition的实现分析

ConditionObject是同步器AbstractQueuedSynchronizer的内部类，因为Condition的操作需要获取相关联的锁，所以作为同步器的内部类也较为合理。每个Condition对象都包含着一个队列（以下称为等待队列），该队列是Condition对象实现等待/通知功能的关键。

Condition的实现,主要包括:等待队列、等待和通知。

##### 等待队列

等待队列是一个FIFO的队列,在队列中的每个节点都包含了一个线程引用,该线程就是在Condition对象上等待的线程,如果一个线程调用了Condition.await()方法,那么该线程将会释放锁、构造成节点加入等待队列并进入等待状态。事实上,节点的定义复用了同步器中节点
的定义,也就是说,同步队列和等待队列中节点类型都是同步器的静态内部类
AbstractQueuedSynchronizer.Node。

一个Condition包含一个等待队列,Condition拥有首节点(firstWaiter)和尾节点(lastWaiter)。当前线程调用Condition.await()方法,将会以当前线程构造节点,并将节点从尾部加入等待队列,等待队列的基本结构如图所示：

![等待队列的基本结构.png](http://upload-images.jianshu.io/upload_images/2889214-3505672edc67405f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Condition拥有首尾节点的引用,而新增节点只需要将原有的尾节点nextWaiter指向它,并且更新尾节点即可。上述节点引用更新的过程并没有使用CAS保证,原因在于调用await()方法的线程必定是获取了锁的线程,也就是说该过程是由锁来保证线程安全的。

在Object的监视器模型上,一个对象拥有一个同步队列和等待队列,而并发包中的Lock(更确切地说是同步器)拥有一个同步队列和多个等待队列,其对应关系如图所示：

![同步队列与等待队列.png](http://upload-images.jianshu.io/upload_images/2889214-62410ede8009b5ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Condition的实现是同步器的内部类,因此每个Condition实例都能够访问同步器提供的方法,相当于每个Condition都拥有所属同步器的引用。

##### 等待

调用Condition的await()方法(或者以await开头的方法),会使当前线程进入等待队列并释放锁,同时线程状态变为等待状态。当从await()方法返回时,当前线程一定获取了Condition相关联的锁。

如果从队列(同步队列和等待队列)的角度看await()方法,当调用await()方法时,相当于同步队列的首节点(获取了锁的节点)移动到Condition的等待队列中。

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

调用该方法的线程成功获取了锁的线程，也就是同步队列中的首节点，该方法会将当前线程构造成节点并加入等待队列中，然后释放同步状态，唤醒同步队列中的后继节点，然后当前线程会进入等待状态。

![当前线程加入等待队列](http://upload-images.jianshu.io/upload_images/2889214-0b404b5566a0de3c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当等待队列中的节点被唤醒，则唤醒节点的线程开始尝试获取同步状态。如果不是通过其他线程调用Condition.signal()方法唤醒，而是对等待线程进行中断，则会抛出InterruptedException。

##### 通知

调用Condition的signal()方法，将会唤醒在等待队列中等待时间最长的节点（首节点），在唤醒节点之前，会将节点移到同步队列中。

```java
public final void signal() {
    if (!isHeldExclusively())
    throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
    doSignal(first);
}
```

调用该方法的前置条件是当前线程必须获取了锁，可以看到signal()方法进行了isHeldExclusively()检查，也就是当前线程必须是获取了锁的线程。接着获取等待队列的首节点，将其移动到同步队列并使用LockSupport唤醒节点中的线程。

![节点从等待队列移动到同步队列](http://upload-images.jianshu.io/upload_images/2889214-2989e6f81c07f76f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过调用同步器的enq(Node node)方法，等待队列中的头节点线程安全地移动到同步队列。当节点移动到同步队列后，当前线程再使用LockSupport唤醒该节点的线程。被唤醒后的线程，将从await()方法中的while循环中退出（isOnSyncQueue(Node node)方法返回true，节点已经在同步队列中），进而调用同步器的acquireQueued()方法加入到获取同步状态的竞争中。

成功获取同步状态（或者说锁）之后，被唤醒的线程将从先前调用的await()方法返回，此时该线程已经成功地获取了锁。
Condition的signalAll()方法，相当于对等待队列中的每个节点均执行一次signal()方法，效果就是将等待队列中所有节点全部移动到同步队列中，并唤醒每个节点的线程。

