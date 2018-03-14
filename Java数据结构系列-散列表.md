# 散列表

## 简介

散列表（Hash table，也叫哈希表），是根据键（Key）而直接访问在内存存储位置的数据结构。也就是说，它通过计算一个关于键值的函数，将所需查询的数据映射到表中一个位置来访问记录，这加快了查找速度。这个映射函数称做散列函数，存放记录的数组称做散列表。

哈希表是一种数据结构，它可以提供快速的插入操作和查找操作。不论哈希表中有多少数据，插入和删除（有时包括删除）只需要接近常量的时间，即O(1)时间级。哈希表也有一些缺点：

1、它基于数组的，数组创建后难于扩展。某些哈希表被基本填满时，性能下降得非常严重，所以程序员必须要清除表中将要存储多少数据（或者准备好定期地把数据转移到更大的哈希表中，这是个费时的过程）。

2、没有一种简便的方法可以已任何一种顺序（例如从小到大）遍历表中数据项。

## 冲突

对不同的关键字可能得到同一散列地址，这种现象称为冲突。而不发生冲突的可能性是非常之小的，所以通常对冲突进行处理。常用方法有以下几种：

### 开放地址法

在开放地址法中，若数据不能直接放在由哈希函数计算出来的小标所指的单元时，就要寻找数组的其他位置。下面介绍下开发地址法的三种方法：

- 线性探测

    在线性探测中，线性地查找空白单元。如果5421是要插入数据的位置，它已经被占用了，那么就是5422，然后是5423，依次类推，数据下标一直递增，直到找到空位。

- 二次探测
    
    在开放地址法的线性探测中会发生聚集。一旦聚集形成，它会变得越来越大。二次探测是防止聚集产生的一种尝试，思想是探测相隔较远的单元，而不是和原始位置相邻的单元。

    在线性探测中，如果哈希函数计算的原始下标是x，线性探测就是x+1，x+2，x+3，依此类推。而在二次探测中，探测的过程是x+1，x+4，x+9，依此类推，到原始位置的距离是步数平方。

- 再哈希法

    二次探测虽然解决原始聚集，但也带来二次聚集。比如将184，302，420和544依次插入到表中，它们都映射到7。那么302需要以一为步长的探测，420需要以四为步长的探测，544需要以九为步长的探测。只要有一项，其关键字映射到7，就需要更长步长的探测。这个现象叫做二次聚集。

    再哈希法可以解决原始聚集和二次聚集，把关键字用不同的哈希函数再做一遍哈希，用这个结果作为步长。对指定的关键字，步长在整个探测中是不变的，不过不同的关键字使用不同的步长。

### 链地址法

在哈希表每个单元中设置链表，某个数据项的关键字还是通常一样映射到哈希表的单元，而数据项本身到这个单元的链表中，其他同样的映射到这个位置的数据项只需要加到链表中。


### JDK 1.8 HashMap源码实现

#### 数据结构

JDk 1.8 HashMap是数组+链表+红黑树实现的，如下所以

![JDK 1.8 HashMap内部数据结构](https://tech.meituan.com/img/java-hashmap/hashMap%E5%86%85%E5%AD%98%E7%BB%93%E6%9E%84%E5%9B%BE.png)

（1）从源码可知HashMap中使用数组为Node<K,V>[] table，即哈希桶数组。Node源码如下：
```java
static class Node<K,V> implements Map.Entry<K,V> {
    // 用来定位数组索引位置
    final int hash;
    final K key;
    V value;
    // 链表的下一个node
    Node<K,V> next;
}
```

Node是HashMap的一个内部类，实现了Map.Entry接口，本质是就是一个映射(键值对)。

（2）Java中HashMap采用了链地址法解决哈希冲突。

#### 初始化过程

HashMap提供了4种构造函数，其源码如下：
```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                            initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                            loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}

// 返回最接近（大于或等于）指定cap大小的2次方，比如cap = 7，则返回8
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

可以发现构造函数的作用仅是对loadFactor（负载因子）或threshold赋值，并没有初始化哈希桶。分析源码可知初始化是由resize()实现的，源码如下：
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;

    // oldCap > 0表示table已初始化了
    if (oldCap > 0) {
        
    }
    // 设定了threshold的值
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // 默认情况即没有设定threshold的值  
    else {               
        // DEFAULT_INITIAL_CAPACITY = 16,DEFAULT_LOAD_FACTOR = 0.75f
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    // 触发扩容的阀值
    threshold = newThr;
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    return newTab;
}
```
如果指定了threshold的值，table的length（长度）就是threshold；如果没有，则table的length默认为16。初始化过程就是创建table数组，并计算触发下一次扩容的阀值。

在HashMap中，哈希桶数组table的长度length大小必须为2的n次方(一定是合数)，这是一种非常规的设计，常规的设计是把桶的大小设计为素数。相对来说素数导致冲突的概率要小于合数，具体证明可以参考[http://blog.csdn.net/liuqiyao_01/article/details/14475159]()，Hashtable初始化桶大小为11，就是桶大小设计为素数的应用（Hashtable扩容后不能保证还是素数）。HashMap采用这种非常规设计，主要是为了在取模和扩容时做优化，同时为了减少冲突，HashMap定位哈希桶索引位置时，也加入了高位参与运算的过程。

#### put方法

HashMap的put方法执行过程如下：

（1）.判断键值对数组table[i]是否为空或为null，否则通过resize()进行初始化；

（2）.根据键值key的hash值计算得到插入的数组索引i，如果table[i]==null，直接新建节点添加，如果table[i]不为空，转向（3）；

（3）.判断table[i]的首个元素是否和key一样，如果相同直接覆盖value，否则转向（4）

（4）.判断table[i]是否为treeNode，即table[i]是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向（5）；

（5）.遍历table[i]，如果遍历过程中若发现key已经存在直接覆盖value即可；如果没有，则在链表末尾插入键值对并判断链表长度是否大于8，大于8的话把链表转换为红黑树；

（6）.插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // table为null或空，则进行初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 对应哈希槽tab[i]为null，直接添加 
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 1、tab[i]存储node节点的key与当前要存储的key是否相同
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 2、tab[i]存储node节点的类型是否为TreeNode类型   
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 3、如果不满足以上两种条件，则遍历node节点会存在两种情况。（1）遍历过程中发现存在与当前要存储key相同的节点；
        // （2）不存在与当前要存储key相同节点，将当前节点存储链表的末尾
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 如果链表长度大于TREEIFY_THRESHOLD，则将其转换为红黑树结构。
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 当e不为null时表示当前要存储key已存在，更新原来key的value值并直接返回之前的value值。
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;

    // size表示map中存储键值对的个数，是否扩容
    if (++size > threshold)
        resize();
    // 空实现    
    afterNodeInsertion(evict);
    return null;
}
```

#### get方法

get()方法执行流程如下：

（1）、根据关键字key的hash值计算得到存储在哈希槽的节点first，如果first.key等于关键字key，则直接返回；如果不是，转向（2）；

（2）、判断first节点类型是否TreeNode（前提是first存在next节点，不存在直接返回null），如果是，直接从红黑树中查找；如果不是，遍历查找；

（3）、如果以上情况没有找到对应的node，直接返回null。

```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
            
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

#### 自动扩容

扩容(resize)就是重新计算容量，向HashMap对象里不停的添加元素，而HashMap对象内部的数组无法装载更多的元素时，对象就需要扩大数组的长度，以便能装入更多的元素。当然Java里的数组是无法自动扩容的，方法是使用一个新的数组代替已有的容量小的数组，就像我们用一个小桶装水，如果想装更多的水，就得换大水桶。

下面举个例子说明下扩容过程。假设了我们的hash算法就是简单的用key mod 一下表的大小（也就是数组的长度）。其中的哈希桶数组table的size=2， 所以key = 3、7、5，put顺序依次为 5、7、3。在mod 2以后都冲突在table[1]这里了。这里假设负载因子 loadFactor=1，即当键值对的实际大小size 大于 table的实际大小时进行扩容。接下来的三个步骤是哈希桶数组 resize成4，然后所有的Node重新rehash的过程。

![](https://tech.meituan.com/img/java-hashmap/jdk1.7%E6%89%A9%E5%AE%B9%E4%BE%8B%E5%9B%BE.png)

经过观测可以发现，我们使用的是2次幂的扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。

#### 线程安全性

在多线程使用场景中，应该尽量避免使用线程不安全的HashMap，而使用线程安全的ConcurrentHashMap。那么为什么说HashMap是线程不安全的？

在并发下HashMap的自动扩展操作可能使Node链表形成环形数据结构，一旦形成环形数据结构，Node的next节点永远不为空，就会在获取Node时产生死循环。

### 参考资料

[Java 8系列之重新认识HashMap](https://tech.meituan.com/java-hashmap.html)