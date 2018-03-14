## 3、栈和队列

栈、队列和优先队列这三种数据结构与数组的区别：

* 程序员的工具

    它们主要作为构思算法的辅助工具，而不是完全的数据存储工具。这些数据结构的生命周期比那些数据库类型的结构要短得多。在程序操作执行期间他们才被创建，通常用它们去执行某项特殊的任务；当完成任务之后，它们就被销毁。

* 受限访问

    访问是受限制的，即在特定时刻只有一个数据项可以被读取或者被删除。

* 更加抽象

    主要通过接口对栈、队列和优先级队列进行定义，这些接口表明通过它们可以完成的操作，而它们的主要实现机制对用户来说是不可见的。

### 栈

 **栈**是限制插入和删除只能在一个位置上进行的表，该位置是表的未端，叫做栈的顶。栈有时又叫做LIFO(后进先出)表，对栈所能做的基本操作是push(进栈)和pop(出栈)；一般的模型是，存在某个元素位于栈顶，而该元素是唯一的可见元素。

由于栈是一个表，因此任何实现表的方法都能实现栈。两种流行实现栈的方法：

* 栈的链表实现
```java

// 入栈
public void push(E e) {
    addFirst(e);
}

// 设置e为首节点
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}

// 出栈
public E pop() {
    return removeFirst();
}

// 删除首节点指向的节点
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}

```  
* 栈的数组实现

```java
/**
 * Vector类实现可扩展的对象数组。
 */
public class Stack<E> extends Vector<E> {

    // 当首次创建堆栈时，它不包含数据
    public Stack() {}

    // 入栈
    public E push(E item) {
        addElement(item);
        return item;
    }

    // 返回栈顶元素并删除该元素
    public synchronized E pop() {
        E  obj;
        int len = size();

        obj = peek();
        removeElementAt(len - 1);

        return obj;
    }

    // 返回栈顶元素但不删除该元素
    public synchronized E peek() {
        int len = size();

        if (len == 0)
            throw new EmptyStackException();
        return elementAt(len - 1);
    }

    // 测试栈是否为空
    public boolean empty() {
        return size() == 0;
    }

    // 返回一个对象在此堆栈上的基于1(栈顶)的位置。
    public synchronized int search(Object o) {
        int i = lastIndexOf(o);

        if (i >= 0) {
            return size() - i;
        }
        return -1;
    }
}
```

> Deque接口及其实现提供了更完整和一致的LIFO堆栈操作集合，它们应该优先于此类。

### 栈的应用

* 单词逆序
* 递归
* 分隔匹配符

# 队列

**队列**，是先进先出（FIFO, First-In-First-Out）的线性表。在具体应用中通常用链表或者数组来实现。队列只允许在队尾（称为rear）进行插入操作，在队头（称为front）进行删除操作。

> 向队列中插入元素的过程称为入队(Enqueue)，删除元素的过程称为出队(Dequeue)。

## 队列定义的方法

在JDK 1.8中队列（Queue）的方法定义如下：

```java
public interface Queue<E> extends Collection<E> {

    boolean add(E e);

    boolean offer(E e);

    E remove();

    E poll();

    E element();

    E peek();
}
```

Queue除了Collection的操作外，还提供了额外的插入，删除和检查操作。每个方法存在两种形式：

|                  | Throws exception | Returns special value |
| ---------------- | ---------------- | --------------------- |
| Insert           | add(e)           | offer(e)              |
| Remove           | remove()         | poll()                |
| Examine          | element()        | peek()                |

一种操作失败抛出异常，另外一种操作失败返回特定的值或者false。


## 队列的实现


### 双端队列（Deque）

双端队列是“double ended queue”的简称，支持两端的元素插入和移除的线性集合。Deque额外提供了如下方法：

![Summary of Deque methods](http://upload-images.jianshu.io/upload_images/2889214-e41a37731a1d97c6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当Deque作为队列使用，Queue的方法与Deque方法完全等价，如下表所示
| Queue Method | Deque Method  |
| ------------ | ------------- |
| add(e)       | addLast(e)    |
| offer(e)     | offerLast(e)  |
| remove()     | removeFirst() |
| poll()       | pollFirst()   |
| element()    | getFirst()    |
| peek()       | peekFirst()   |

当Deque作为栈使用，Stack的方法与Deque方法完全等价，如下表所示

| Stack Method | Deque Method  |
| ------------ | ------------- |
| push(e)      | addFirst(e)   |
| pop()        | removeFirst() |
| peek()       | peekFirst()   |

在JDK 1.8中双端队列的实现类有ArrayDeque和LinkedList。

#### ArrayDeque源码分析

**1、数据结构及构造函数**

```java
public class ArrayDeque<E> extends AbstractCollection<E>
                           implements Deque<E>, Cloneable, Serializable
{
    // 使用数组存储元素
    transient Object[] elements;

    // 队头
    transient int head;

    // 队尾
    transient int tail;

    // 数组最小容量8，必须是2幂
    private static final int MIN_INITIAL_CAPACITY = 8;

    // 默认构造函数
    public ArrayDeque() {
        elements = new Object[16];
    }

    // 指定元素数量的构造函数，数组的长度为16
    public ArrayDeque(int numElements) {
        allocateElements(numElements);
    }

     private void allocateElements(int numElements) {
        int initialCapacity = MIN_INITIAL_CAPACITY;
        // 如果numElements > 8,返回大于numElements的最小2的次方
        if (numElements >= initialCapacity) {
            initialCapacity = numElements;
            initialCapacity |= (initialCapacity >>>  1);
            initialCapacity |= (initialCapacity >>>  2);
            initialCapacity |= (initialCapacity >>>  4);
            initialCapacity |= (initialCapacity >>>  8);
            initialCapacity |= (initialCapacity >>> 16);
            initialCapacity++;

            if (initialCapacity < 0)   // Too many elements, must back off
                initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
        }
        elements = new Object[initialCapacity];
    }
}
```
数组elements的长度总是为2的次方，默认长度为16。

**2、入队（addLast）方法**

```java
// 将指定的元素插入队列的末尾
public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[tail] = e;
    // tail+1，重新计算tail，目的是重新利用已删除元素的位置
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        // head == tail，自动扩容
        doubleCapacity();
}
```
入队时当数组没有可用空间，会进行自动扩容，因此ArrayDeque是没有容量限制的。


**2、出队（pollFirst）方法**

```java
// 队列头部元素出列
public E pollFirst() {
    int h = head;
    @SuppressWarnings("unchecked")
    // 获取队头的元素
    E result = (E) elements[h];
    if (result == null)
        return null;
    // 删除元素    
    elements[h] = null;
    // head+1，重新计算head
    head = (h + 1) & (elements.length - 1);
    return result;
}
```

3、**入栈（addFirst）方法**

```java
// 直接将要插入的元素保存在队头的位置
public void addFirst(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[head = (head - 1) & (elements.length - 1)] = e;
    // 扩容判断
    if (head == tail)
        doubleCapacity();
}
```



**4、扩容**
```java
private void doubleCapacity() {
    assert head == tail;
    int p = head;
    int n = elements.length;
    int r = n - p;
    // 数组长度翻倍
    int newCapacity = n << 1;
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
    Object[] a = new Object[newCapacity];
    System.arraycopy(elements, p, a, 0, r);
    System.arraycopy(elements, 0, a, r, p);
    elements = a;
    // 重新计算head和tail
    head = 0;
    tail = n;
}
```

![扩容过程.png](http://upload-images.jianshu.io/upload_images/2889214-0ad6cc26e8660ab8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 阻塞队列

阻塞队列与普通队列的区别在于，当队列是空的时，从队列中获取元素的操作将会被阻塞，或者当队列是满时，往队列里添加元素的操作会被阻塞。

BlockingQueue实现被设计为主要用于生产者 - 消费者队列，但另外支持Collection接口。

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    
       /** The queued items */
    final Object[] items;

     /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;

    // 队列中元素的个数
    int count;
    
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }


    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        // 加锁
        lock.lockInterruptibly();
        try {
            // 循环检查队列是否满了
            while (count == items.length)
                // put阻塞，等待通知
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }

    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        // 队列中有元素，通知notEmpty
        notEmpty.signal();
    }

    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }

    private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        //     
        notFull.signal();
        return x;
    }
}
```

### 优先队列（PriorityQueue）

在优先队列中，元素被赋予优先级。当访问元素时，具有最高优先级的元素最先删除。优先队列具有最高级先出 （first in, largest out）的行为特征。

JDK提供了优先队列的实现类PriorityQueue，接下来分析下其是如何实现：

**1、数据结构及构造函数**


```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {

    private static final int DEFAULT_INITIAL_CAPACITY = 11;

    /**
     * 可以把优先队列看作成平衡二叉树: 节点queue[n]的两个子节点是queue[2*n+1]和queue[2*(n+1)]。* 优先队列由比较器或元素的自然顺序排序，对于堆中的每个节点n和n的每个后代d，n <= d。假设队列是非空，具有最低值的元素在queue[0].
     */
    transient Object[] queue;

    // 队列中元素的个数
    private int size = 0;

    // 排序时使用的比较器
    private final Comparator<? super E> comparator;

    // 优先级队列的结构修改次数
    transient int modCount = 0;

    public PriorityQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }

    public PriorityQueue(int initialCapacity) {
        this(initialCapacity, null);
    }

    public PriorityQueue(Comparator<? super E> comparator) {
        this(DEFAULT_INITIAL_CAPACITY, comparator);
    }

    public PriorityQueue(int initialCapacity,
                         Comparator<? super E> comparator) {
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.queue = new Object[initialCapacity];
        this.comparator = comparator;
    }

    public PriorityQueue(Collection<? extends E> c) {
        if (c instanceof SortedSet<?>) {
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            initElementsFromCollection(ss);
        }
        else if (c instanceof PriorityQueue<?>) {
            PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            initFromPriorityQueue(pq);
        }
        else {
            this.comparator = null;
            initFromCollection(c);
        }
    }

    public PriorityQueue(PriorityQueue<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initFromPriorityQueue(c);
    }

    public PriorityQueue(SortedSet<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initElementsFromCollection(c);
    }
}
```

2、**入队（offer）方法**

```java
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
        // queue容量扩容
        grow(i + 1);
    size = i + 1;
    // 队列尾部为空，直接赋值
    if (i == 0)
        queue[0] = e;
    else
        // 根据比较器进行排序，保证父元素小于插入的元素及queue[0]的值是最小的。
        siftUp(i, e);
    return true;
}

// 使用元素的自然顺序排序，进行插入
private void siftUpComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    while (k > 0) {
        // 获取该插入元素（子元素）的父元素
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        if (key.compareTo((E) e) >= 0)
            break;
        // 如果父元素小于子元素，将父元素向下移位    
        queue[k] = e;
        k = parent;
    }
    queue[k] = key;
}
```

3、**出队（poll）方法**

```java
public E poll() {
    if (size == 0)
        return null;
    int s = --size;
    modCount++;
    // 移除queue[0]元素
    E result = (E) queue[0];
    // 获取queue[size-1]（最后添加）元素
    E x = (E) queue[s];
    queue[s] = null;
    // 如果堆中只有一个元素直接删除，否则删除元素后对堆进行调整
    if (s != 0) 
        siftDown(0, x);
    return result;
}

// siftDown(0, x)方法从堆的第一个元素往下比较，保证最小元素在堆顶
private void siftDownComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>)x;
    // loop while a non-leaf
    int half = size >>> 1;        
    while (k < half) {
        // assume left child is least
        int child = (k << 1) + 1; 
        Object c = queue[child];
        int right = child + 1;
        if (right < size &&
            ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            c = queue[child = right];
        if (key.compareTo((E) c) <= 0)
            break;
        queue[k] = c;
        k = child;
    }
    queue[k] = key;
}
```

4、**扩容**

```java
// 如果队列中的元素数量大于或等于数组的长度，则需要进行扩容
if (i >= queue.length)
    grow(i + 1);

private void grow(int minCapacity) {
    int oldCapacity = queue.length;
    // 如果oldCapacity < 64,则翻倍扩容；相反则进行1.5倍进行扩容
    int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                        (oldCapacity + 2) :
                                        (oldCapacity >> 1));
    // 扩容的最大值为Integer.MAX_VALUE
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    queue = Arrays.copyOf(queue, newCapacity);
}
```


### 延迟队列

DelayQueue是一个无界的BlockingQueue，用于放置实现了Delayed接口的对象，其中的对象只能在其到期时才能从队列中取走。这种队列是有序的，即队头对象的延迟到期时间最长。注意：不能将null元素放置到这种队列中。 

1、 数据结构及构造函数
```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {

    // 并发控制
    private final transient ReentrantLock lock = new ReentrantLock();

    // 使用优先队列存储元素
    private final PriorityQueue<E> q = new PriorityQueue<E>();

    // 
    private Thread leader = null;

    // 条件队列
    private final Condition available = lock.newCondition();

    public DelayQueue() {}

    public DelayQueue(Collection<? extends E> c) {
        this.addAll(c);
    }
}
```

2、offer()方法

```java
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 向优先队列添加元素
        q.offer(e);
        // 如果元素e为队头，激活等待线程take元素
        if (q.peek() == e) {
            leader = null;
            available.signal();
        }
        return true;
    } finally {
        lock.unlock();
    }
}
```

3、take()方法

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lockInterruptibly();
    try {
        // 死循环
        for (;;) {
            // 获取队头元素
            E first = q.peek();
            // 如果队列中没有元素，则当前线程等待
            if (first == null)
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                // 如果队头元素过期，则poll队头元素
                if (delay <= 0)
                    return q.poll();
                first = null; // don't retain ref while waiting
                // leader != null，表示有其他线程正在进行take操作，则阻塞当前线程
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        // 当前线程阻塞指定时间后（数据过期），重新竞争锁
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
```


在我们的业务中通常会有一些需求是这样的：
1. 淘宝订单业务:下单之后如果三十分钟之内没有付款就自动取消订单。
2. 饿了吗订餐通知:下单成功后60s之后给用户发送短信通知。 

[怎么用DelayQueue来解决这类的问题](http://blog.csdn.net/u011001723/article/details/51882887)

### 同步队列

http://cmsblogs.com/?p=2418

http://www.cnblogs.com/leesf456/p/5560362.html

http://www.dczou.com/viemall/322.html