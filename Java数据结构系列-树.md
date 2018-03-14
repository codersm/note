## 树

树通常结合了另外两种数据结构的优点：一种是有序数组，另一种是链表。在数中查找数据项的速度和在有序数组中查找一样快，并且插入数据项和删除数据项的速度也和链表一样。

树由边连接的节点而构成。

### 树的术语

* 路径
* 根
* 父节点
* 子节点
* 叶节点
* 字树

### 二叉树

如果树中每个节点最多只能有两个子节点，这样的树就称为“二叉树”。二叉树每个节点的两个子节点称为“左子节点”和“右子节点”，二叉树中的节点不是必须有两个子节点；它可以只有一个左子节点，或者只有一个右子节点，或者干脆没有子节点（这种情况下它就是叶节点）。

> 二叉搜索树特征的定义可以这样说：一个节点的左子节点的关键字值小于这个节点，右子节点的关键字值大于或等于这个父节点。

在树中，每个节点都包含数据。而且除了包含数据，除了叶子节点之外，每个节点还包含指向其他节点的引用。

**非平衡树**

大部分的节点在根的一边或者是另一边，如图所示：

![非平衡树](http://i4.eiimg.com/1949/85771928cb2955e5.png)

树变得不平衡是由数据项插入的顺序造成的。如果关键字值是随机插入的，树会或多或少更平衡一点。但是，如果插入序列是升序或是降序，则所有的值都是右子节点或者都是左子节点，这样树就会不平衡了。

#### 用Java实现树

**Node类**

首先，需要有一个节点对象的类。这些对象包含数据，数据代表要存储的内容，而且还有指向节点的两个子节点的引用。

```java
class Node{
    int data;
    Node leftChild;
    Node rightChild;
}
```

**Tree类**

还需要有一个表示树本身的类，由这个类实例化的对象含有所有的节点，这个类是Tree类。它只有一个数据字段：一个表示根的Node变量。它不需要包含其他节点的数据字段，因为其他节点都可以从根开始访问到。

```java
class Tree{
    private Node root;

    public Node find(int key){}

    public void insert(int data){}

    public void delete(int data){}
}
```

**查找节点**

```java
public Node find(int data){
    Node current = root;
    while(current.data != data){
        if(key < current.data){
            current = current.leftChild;
        }else{
            current = current.rightChild;
        }
    }
    if(current == null){
        return null;
    }
    return current;
}
```

**插入一个节点**

要插入节点，必须先找到插入的地方（这很像要找一个不存在的节点的过程）。从根开始查找一个相应的节点，它将是新节点的父节点。当父节点找到了，新的节点就可以连接到它的左子节点或右子节点处，这取决于新节点的值是比父节点的值大还是小。

```java
public void insert(int data){
    Node newNode = new Node();
    newNode.data = data;
    if(root == null){
        root = newNode;
    }else{
        Node current = root;
        Node parent;
        while(true){
            parent = current;
            if(data < current.data){
                current = current.leftChild;
                if(current == null){
                    parent.leftChild = newNode;
                    return;
                }
            }else{
                current = current.rightChild;
                if(current == null){
                    parent.rightChild = newNode;
                    return;
                }
            }
        }
    }
}
```

**遍历树** ？

遍历树的意思是根据一种特定顺序访问树的每一个节点。这个过程不如查找、插入和删除节点常用，其中一个原因是因为遍历的速度不是特别快。

有三种简单的方法遍历树。它们是：前序（preorder）、中序（inorder）、后序（postorder）。二叉搜索树最常用的遍历方法是中序遍历。

* 中序遍历

    中序遍历二叉搜索树会使所有的节点按关键字值升序被访问到。如果希望在二叉树中创建有序的数据序列，这是一种方法。

遍历树的最简单方法是用递归的方法。用递归的方法遍历整棵树要用一个节点作为参数，初始化时这个节点是根。这个方法只需要做三件事：

    1. 调用自身来遍历节点的左子树；
    2. 访问这个节点；
    3. 调用自身来遍历节点的右子树；

遍历可以应用于任何二叉树，而不只是二叉搜索树。这个遍历的原理不关心节点的关键字值；它只是看这个节点是否有子节点。

```java
private void inOrder(Node localRoot){
    if(localRoot != null){
        inOrder(localRoot.leftChild);
        System.out.print(localRoot.data+"");
        inOrder(localRoot.rightChild);
    }
}
```

**前序和后序遍历**【*】

如果要编写程序来解析或分析代数表达式，这两种遍历方法就很有用了。二叉树（不是二叉搜索树）可以用于表示包括二元元算符
号+、-、/、*的算术表达式。根节点保存运算符号，其他节点或者存变量名，或者保存运算符号。每一棵子树都是一个合法的代数表达式。

![表示一个算术表达式的树](http://i4.eiimg.com/1949/108be964a91c92f4.png)

**查找最大值和最小值**

要找最小值时，先走到根的左子节点处，然后接着走到那个子节点的左子节点，如此类推，直到找到一个没有左子节点的节点。
这个节点就是最小值的节点。

```java
public Node minimum(){
    Node current,last;
    current = root;
    while(current != null){
        last = current;
        current = current.leftChild;
    }
    return last;
}
```

按照相同的步骤来查找树中的最大值。


**删除节点**

删除节点要从查找要删的节点开始入手，找到节点后，这个要删除的节点可能会三种情况需要考虑：

1. 该节点是叶节点
2. 该节点有一个子节点
3. 该节点有两个子节点

* 删除没有子节点的节点

要删除叶节点，只需要改变该节点的父节点的对应子字段的值，由指向该节点改为null就可以了。

```java
public boolean delete(int data){
    Node current = root;
    Node parent = root;
    boolean isLeftChild = true;

    while(current.data != data){
        parent = current;
        if(data < current.data){
            isLeftChild = true;
            currrent = current.leftChild;
        }else{
            isLeftChild = false;
        }
    }
    if(current == null){
        return false;
    }
    if(current.leftChild == null && current.rightChild == null){
        root = null;
    }else if(isLeftChild){
        parent.leftChild = null;
    }else{
        parent.rightChild = null;
    }
}
```

### 红黑树

红黑树是为了维护二叉查找树的平衡而产生的一种树，红黑树有以下几个特征：

1. 每一个节点不是红色的就是黑色的；

2. 根总是黑色的；

3. 如果节点是红色的，则它的子节点必须是黑色的；

4. 从根到叶节点或空子节点的每条路径，必须包含相同数目的黑色节点；

> 红黑树并不是高度的平衡树。所谓平衡树指的是一棵空树或它的左右两个子树的高度差的绝对值不超过1。

JDK提供的TreeMap是基于Red-Black树的实现，接下来分析TreeMap是如何实现的。

#### TreeMap中的节点Node定义如下：

```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left; // 左子节点引用
    Entry<K,V> right; // 右字节引用
    Entry<K,V> parent;  // 父节点引用
    boolean color = BLACK;

    
    Entry(K key, V value, Entry<K,V> parent) {
        this.key = key;
        this.value = value;
        this.parent = parent;
    }
}
```

#### put（插入）方法源码如下：

```java
public V put(K key, V value) {
    Entry<K,V> t = root;
    // 根root节点不存在，构造新节点且设置为root节点
    if (t == null) { 
        compare(key, key); // type (and possibly null) check
        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    // comparator
    if (cpr != null) {
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    else {
        // 检测key不能为空，如果为空，则抛出NullPointerException。
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        // 查找要插入节点的父节点parent    
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    Entry<K,V> e = new Entry<>(key, value, parent);
    // cmp < 0，则将其放在左子节点
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    // 插入之后进行修复，包括左旋、右旋、重新着色这些操作，让树保持平衡性
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

#### fixAfterInsertion关键方法

```java
private void fixAfterInsertion(Entry<K,V> x) {
    // 默认情况下，节点x的颜色为红色
    x.color = RED;
    // 节点x的父节点parent.color = RED，需要进行处理。
    while (x != null && x != root && x.parent.color == RED) {
        // x节点为左子节点插入
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            // 
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                // 用于while条件重新判断，保证不会连续出现两个红色节点
                x = parentOf(parentOf(x));
            } else {
                // x节点是否为外侧插入
                if (x == rightOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateLeft(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateRight(parentOf(parentOf(x)));
            }
        } else {
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {        
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    // 根据节点root始终为黑色
    root.color = BLACK;
}
```

左子树内侧插入与左子树外侧插入的区别：

旋转是核心

右旋代码实现如下，

```java
private void rotateRight(Entry<K,V> p) {
    if (p != null) {
        Entry<K,V> l = p.left;
        p.left = l.right;
        // 
        if (l.right != null) l.right.parent = p;

        l.parent = p.parent;

        if (p.parent == null)
            root = l;
        else if (p.parent.right == p)
            p.parent.right = l;
        else p.parent.left = l;
        l.right = p;
        p.parent = l;
    }
}
```

可以看出，旋转过程不涉及颜色变更的。



### 平衡二叉树(AVL树)

平衡二叉树的定义如下：首先符合二叉查找树的定义，其次必须满足任何节点的两个子树的高度最大差为1。查找、插入和删除在平均和最坏情况下都是O（log n）。增加和删除可能需要通过一次或多次树旋转来重新平衡这个树。

~~平衡二叉树多用于内存结构对象中，因此维护的开销相对较小~~。


节点的**平衡因子**是它的左子树的高度减去它的右子树的高度（有时相反）。带有平衡因子1、0或 -1的节点被认为是平衡的。带有平衡因子 -2或2的节点被认为是不平衡的，并需要重新平衡这个树。平衡因子可以直接存储在每个节点中，或从可能存储在节点中的子树高度计算出来。

### B树

在计算机科学中，B树（英语：B-tree）是一种自平衡的树，能够保持数据有序。这种数据结构能够让查找数据、顺序访问、插入数据及删除的动作，都在对数时间内完成。B树，概括来说是一个一般化的二叉查找树（binary search tree），可以拥有多于2个子节点。与自平衡二叉查找树不同，B树为系统大块数据的读写操作做了优化。B树减少定位记录时所经历的中间过程，从而加快存取速度。

B树这种数据结构可以用来描述外部存储。这种数据结构常被应用在数据库和文件系统的实现上。

#### B树定义

![B树定义](http://i2.tiimg.com/1949/13ba3bd44e2cb838.png)

一棵3阶B树

![3阶B树](http://i1.ciimg.com/1949/162e188023f602d2.jpg)

[漫画丨什么是B-树？](https://mp.weixin.qq.com/s?__biz=MzI5OTY0MTMyMg==&mid=2247485356&idx=2&sn=311d5481f26d1b1a551e6348a9567b4b&scene=21#wechat_redirect)

### B+树

B+树由B树和索引顺序访问方法演化而来，但是在现实使用过程中几乎已经没有使用B树的情况了。**B+树是为磁盘或其他直接存取辅助设备设计的一种平衡查找树。在B+树中，所有记录节点都是按键值的大小顺序存放在同一层的叶子结点上，由各叶子节点指针进行连接。**

~~因为B+树结构主要用于磁盘，页的拆分意味着磁盘的操作，所以应该在可能的情况下尽量减少页的拆分操作。~~

**填充因子**


B+树是对B树的一种变形树，它与B树的差异在于：

* 有k个子结点的结点必然有k个关键码；【有疑问】
* 非叶结点仅具有索引作用，跟记录有关的信息均存放在叶结点中。
* 树的所有叶结点构成一个有序链表，可以按照关键码排序的次序遍历全部记录。

B和B+树的区别在于，B+树的非叶子结点只包含导航信息，不包含实际的值，所有的叶子结点和相连的节点使用链表相连，便于区间查找和遍历。

B+ 树的优点在于：

* 由于B+树在内部节点上不好含数据信息，因此在内存页中能够存放更多的key。 数据存放的更加紧密，具有更好的空间局部性。因此访问叶子几点上关联的数据也具有更好的缓存命中率。
* B+树的叶子结点都是相链的，因此对整棵树的便利只需要一次线性遍历叶子结点即可。而且由于数据顺序排列并且相连，所以便于区间查找和搜索。而B树则需要进行每一层的递归遍历。相邻的元素可能在内存中不相邻，所以缓存命中性没有B+树好。
* 但是B树也有优点，其优点在于，由于B树的每一个节点都包含key和value，因此经常访问的元素可能离根节点更近，因此访问也更迅速。

[漫画：什么是B+树？](http://www.jianshu.com/p/8f6a0952a866)

## 树的实现

