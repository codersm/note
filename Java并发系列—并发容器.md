## Java并发容器和框架

### Copy-On-Write容器

CopyOnWrite容器即写时复制的容器。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。

从JDK 1.5开始，Java并发包里提供了两个使用CopyOnWrite机制实现的并发容器,它们是CopyOnWriteArrayList和CopyOnWriteArraySet。CopyOnWrite容器非常有用，可以在非常多的并发场景中使用到。

#### CopyOnWriteArrayList的实现原理

```java
// 添加元素
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 获取原容器
        Object[] elements = getArray();
        int len = elements.length;
        // 将原容器进行Copy，复制出一个新的容器
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        // 将原容器的引用指向新的容器
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}

// 获取元素
public E get(int index) {
    return get(getArray(), index);
}
```

> CopyOnWrite并发容器用于读多写少的并发场景。比如白名单，黑名单，商品类目的访问和更新场景，假如我们有一个搜索网站，用户在这个网站的搜索框中，输入关键字搜索内容，但是某些关键字不允许被搜索。这些不能被搜索的关键字会被放在一个黑名单当中，黑名单每天晚上更新一次。当用户搜索时，会检查当前关键字在不在黑名单当中，如果在，则提示不能搜索。

#### 参考资料

[图解集合3：CopyOnWriteArrayList](http://www.cnblogs.com/xrq730/p/5020760.html)

[JAVA中的COPYONWRITE容器](https://coolshell.cn/articles/11175.html)

### ConcurrentHashMap

与HashTable有什么区别

* 同一个时刻，迭代器只能被一个线程所使用；

* 聚合方法只有在没有其他线程进行更新操作时，才有用，这些方法的结果反映了可能足以进行监视或估计的瞬态目的，但不适用于程序控制。


通过使用java.util.concurrent.atomic.LongAdder值和通过computeIfAbsent进行初始化，可以将ConcurrentHashMap用作可伸缩频率映射.


Like Hashtable but unlike HashMap, this class does not allow null to be used as a key or value.

初始化
```java
/**
 * Initializes table, using the size recorded in sizeCtl.
 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```


```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    // 获取插入到数组的槽（index）
    int hash = spread(key.hashCode());

    int binCount = 0;
    // 
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 如果tab没有初始化，则初始化。
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();

        // 数组槽没有元素占用，直接插入
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 将要插入的槽锁定，提高了并发性
            synchronized (f) {
                // 再次检测，防止其他线程更新操作
                if (tabAt(tab, i) == f) {

                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 如果key相等，覆盖旧值
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // 如果hash值相等，key不相等时，直接插入链表末尾。
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,value, null);
                                break;
                            }
                        }
                    }
                    // 如果Tree结构，直接添加
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                        value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            
            if (binCount != 0) {
                // 如果链表元素大于8，将链表转换为Tree结构
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

counterCells

BASECOUNT

baseCount




ConcurrentHashMap所使用的锁分段技术。首先将数据分成一段一段地存储,然后给每一段数据配一把锁,当一个线程占用锁访问其中一个段数据的时候,其他段的数据也能被其他线程访问。

ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成。Segment是一种可重入(ReentrantLock)，在ConcurrentHashMap里扮演锁的角色；HashEntry则用于存储键值对数据。一个ConcurrentHashMap里包含个Segment数组。Segment的结构和HashMap类似,是一种数组和链表结构。一个Segment里包含一个HashEntry数组,每个HashEntry是一个链表结构的元素,每个Segment守护着一个HashEntry数组里的元素,当对HashEntry数组的数据进行修改时,必须首先获得与它对应的Segment锁。

### ConcurrentLinkedQueue

ConcurrentLinkedQueue是一个基于链接节点的无界线程安全队列,它采用先进先出的规
则对节点进行排序,当我们添加一个元素的时候,它会添加到队列的尾部;当我们获取一个元
素时,它会返回队列头部的元素。

### Java中的阻塞队列

阻塞队列(BlockingQueue)是一个支持两个附加操作的队列。这两个附加的操作支持阻塞
的插入和移除方法。

阻塞队列常用于生产者和消费者的场景,生产者是向队列里添加元素的线程,消费者是
从队列里取元素的线程。阻塞队列就是生产者用来存放元素、消费者用来获取元素的容器。

JDK 7提供了7个阻塞队列,如下

- ArrayBlockingQueue：由数组结构组成的有界阻塞队列。

- LinkedBlockingQueue：由链表结构组成的有界阻塞队列。

- PriorityBlockingQueue：支持优先级排序的无界阻塞队列。

- DelayQueue

    DelayQueue是一个支持延时获取元素的无界阻塞队列。队列使用PriorityQueue来实现。队列中的元素必须实现Delayed接口,在创建元素时可以指定多久才能从队列中获取当前元素。
    只有在延迟期满时才能从队列中提取元素。

    DelayQueue非常有用,可以将DelayQueue运用在以下应用场景。

    - 缓存系统的设计:可以用DelayQueue保存缓存元素的有效期,使用一个线程循环查询
    DelayQueue,一旦能从DelayQueue中获取元素时,表示缓存有效期到了。

    - 定时任务调度:使用DelayQueue保存当天将会执行的任务和执行时间,一旦从
    DelayQueue中获取到任务就开始执行,比如TimerQueue就是使用DelayQueue实现的。

    如何实现Delayed接口

    如何实现延时阻塞队列


- SynchronousQueue

    SynchronousQueue是一个不存储元素的阻塞队列。每一个put操作必须等待一个take操作,否则不能继续添加元素。

    SynchronousQueue可以看成是一个传球手,负责把生产者线程处理的数据直接传递给消费者线程。队列本身并不存储任何元素,非常适合传递性场景。

- LinkedTransferQueue

    LinkedTransferQueue是一个由链表结构组成的无界阻塞TransferQueue队列。相对于其他阻
    塞队列,LinkedTransferQueue多了tryTransfer和transfer方法。

    (1)transfer方法

    如果当前有消费者正在等待接收元素(消费者使用take()方法或带时间限制的poll()方法
    时),transfer方法可以把生产者传入的元素立刻transfer(传输)给消费者。如果没有消费者在等
    待接收元素,transfer方法会将元素存放在队列的tail节点,并等到该元素被消费者消费了才返
    回。

    (2)tryTransfer方法

    tryTransfer方法是用来试探生产者传入的元素是否能直接传给消费者。如果没有消费者等
    待接收元素,则返回false。和transfer方法的区别是tryTransfer方法无论消费者是否接收,方法
    立即返回,而transfer方法是必须等到消费者消费了才返回。

    对于带有时间限制的tryTransfer(E e,long timeout,TimeUnit unit)方法,试图把生产者传入
    的元素直接传给消费者,但是如果没有消费者消费该元素则等待指定的时间再返回,如果超
    时还没消费元素,则返回false,如果在超时时间内消费了元素,则返回true。


- LinkedBlockingDeque：由链表结构组成的双向阻塞队列。


**阻塞队列的实现原理**

使用通知模式实现。所谓通知模式,就是当生产者往满的队列里添加元素时会阻塞住生
产者,当消费者消费了一个队列中的元素后,会通知生产者当前队列可用。