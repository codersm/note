# Java虚拟机——垃圾收集器

GC需要完成的3件事情

* 哪些内存需要回收

Java内存运行时区域的各个部分，其中程序技术器、虚拟机栈、本地方法栈3个区域随线程而生，随线程而灭：


Java堆和方法区则不一样，一个接口中的多个实现类需要的内存可能不一样，一个方法中的多个分支需要的内存也可能不一样，我们只有在程序处于运行期间时才能知道会创建那些对象，这部分内存的分配和回收都是动态的，垃圾收集器所关注的是这部分内存。

引用计数器算法

> 给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加1；当引用失效时，计数器值就减1；任何时刻计数器为0的对象
就是不可能再被使用的。

主流的Java虚拟机里面没有选用引用计数算法来管理内存，其中最主要的原因是它很解决对象之间相互循环引用的问题。

**如何查看GC日志？**

可达性分析算法

在主流的商用程序语言（Java、C#，甚至包括前面提到的古老的Lisp）的主流实现中，都是称通过可达性分析来判定对象是否存活的。

> 通过一系列的称为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain），当一个对象到GC Roots没有任何引用链相连时，则证明此对象是不可用的，将会被判定为可回收的对象。

在Java语言中，可作为GC Roots的对象包括下面几种：

* 虚拟机栈（栈帧中的本地变量表）中引用的对象。
* 方法区中类静态属性引用的对象
* 方法区中常量引用的对象
* 本地方法栈中国中JNI（即一般说的Native方法）引用的对象

引用传统引用定义（JDK 1.2以前）

> 如果reference类型的数据中存储的数值代表的是另外一块内存的起始地址，就称这块内存代表着一个引用。

在JDK 1.2之后，Java对引用的概念进行了扩充，将引用分为强引用（Strong Reference）、软引用（Soft Reference）、弱引用（Weak Reference）、虚引用（Phantom Reference）4钟，这4钟引用强度依次逐渐减弱。

* 强引用就是指在程序代码之中普遍存在的，类似“Object obj = new Object()”这类的引用，只要强引用还存在，垃圾收集器永远不会回收被引用的对象。

* 软引用是用来描述一些还有用但并非必需的对象。对于软引关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。

* 弱引用也是用来描述非必需对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。

* 虚引用也称为幽灵引用或者幻影引用，它是最弱的一种引用关系。一个对象是否虚拟引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。**为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。JDK 1.2之后，提供了PhantomReference类来实现虚引用。**


#### 回收过程

如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选，筛选的条件是此对象是否有必要执行finalize()方法。当对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。

如果这个对象被判定为有必要执行finalize()方法，那么这个对象将会放置在一个叫做F-Queue的队列之中，并在稍后由一个虚拟机自动建立的、低优先级的Finalizer线程去执行它。

稍后GC将对F-Queue中的对象进行第二次小规模的标记，如果对象要在finalize()中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，那在第二次标记时它将被移除出“即将回收”的集合；如果对象这时候还没有逃脱，那基本上它就真的被回收了。


### 方法区回收

？ 很多人认为方法区（或者Hotspot虚拟机中的永久代）是没有垃圾收集的

永久代的垃圾收集主要回收两部分内容：废弃常量和无用的类。

类需要同时满足下面3个条件才能算是“无用的类”：

* 

虚拟机可以对满足上述3个条件的无用类进行回收，这里说的仅仅是“可以”，而并不是和对象一样，不使用了就必然回收。是否对类进行回收，HotSpot虚拟机提供了-Xnoclassgc参数进行控制。




* 什么时候回收



* 如何回收


### 垃圾收集算法

#### 标记-清除算法（Mark-Sweep）

算法分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。它的主要不足有两个：

* 效率问题
    
    标记和清除两个过程的效率都不高

* 空间问题

    标记清楚之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

#### 复制算法

将可用空间按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后把已使用过的内存空间一次清理掉。这样使得每次都是对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。


现在的商业虚拟机都采用这种收集算法来回收新生代。

新生代 老年代？

#### 标记-整理算法

“标记-整理”算法的标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理端边界以外的内存。

#### 分代收集算法

当前商业虚拟机的垃圾收集都采用“分代收集”算法，根据对象存活周期的不同将内存划分为几块。一般是把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用“标记-清理”或者“标记-整理”算法来进行回收。

### HotSpot虚拟机算法实现

枚举根节点

枚举根节点会导致GC进行时必须停顿所有Java执行线程的一个重要原因。

安全点

安全区域

> 内存如何回收是由虚拟机所采用的GC收集器决定的


### 垃圾收集器

#### Serial收集器

Serial收集器是虚拟机运行在Client模式下的默认新生代收集器。它也有着优于其他收集器的地方：简单而高效（与其他收集器的单线程比），对于限定单个CPU的环境来说，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得最高的单线程收集效率。

#### ParNew收集器

ParNew收集器其实就是Serial收集器的多线程版本，除了使用多条线程进行垃圾收集之外，其余行为包括Serial收集器可用的所有控制参数、收集算法、Stop The World对象分配规则、回收策略等都与Serial收集器完全一样，在实现上，这两种收集器也共用了相当多的代码。

ParNew收集器除了多线程收集之外，其他与Serial收集器相比并没有太多创新之处，但它却是许多运行在Server模式下的虚拟机中首选的新生代收集器，其中有一个与性能无关但很重要的原因是，除了Serial收集器外，目前只有它能与CMS收集器配合工作。

> CMS作为老年代的收集器

ParNew收集器也是使用-XX：+UseConcMarkSweepGC选项后的默认新生代收集器，也可以使用-XX:+UseParNewGC选项来强制指定它。


并行和并发


#### Parallel Scavenge收集器

Parallel Scavenge收集器的特点是它的关注点与其他收集器不同，CMS等收集器的关注点是尽可能地缩短垃圾收集时用户线程的停顿时间，而Parallel Scavenge收集器的目标则是达到一个可控制的吞吐量。

停顿时间越短就越适合需要与用户交互的程序，良好的响应速度能提升用户体验，而高吞吐量则可以高效率地利用CPU时间，尽快完成程序的运算任务，主要适合在后台运算任务，主要适合在后台运算而不需要太多交互的任务。

#### Serial Old收集器

Serial Old是Serial收集器的老年代版本，它同样是一个单线程收集器，使用“标记-整理”算法。这个收集器的主要意义也是在于给Client模式下的虚拟机使用。如果在Server模式下 ，那么它主要还有两大用途：

* 在JDK 1.5以及之前的版本中与Parallel Scavenge收集器搭配使用。

* 作为CMS收集器的后备预案，在并发收集发生Concurrent Mode Failure时使用。

#### Parallel Old收集器

Parallel Old是Parallel Scavenge收集器的老年代版本，使用多线程和“标记-整理”算法。这个收集器是在JDK 1.6中才开始提供的。Parallel Old收集器出现后，“吞吐量优先”收集器终于有了比较名副其实的应用组合，在注重吞吐量以及CPU资源敏感的场合，都可以优先考虑Parallel Scavenge加Parallel Old收集器。

#### CMS收集器

CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器。CMS收集器是基于“标记-清除”算法实现的，它的运作过程相对于前面几种收集器来说更复杂一些，整个过程分为4个步骤：

* 初始标记（CMS initial mark）

    标记一下GC Roots能直接关联到的对象，速度很快。

* 并发标记（CMS concurrent mark）

    进行GC Roots Tracing的过程

* 重新标记（CMS remark）

    为了修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短。

* 并发清除（CMS concurrent sweep）

其中，初始标记、重新标记这两个步骤仍然需要“Stop The World”。整个过程中耗时最长的并发标记和并发清楚过程收集器线程都可以与用户线程一起工作，所以，从总体上来说，CMS收集器的内存回收过程是与用户线程一起并发执行的。


### G1 收集器

G1是一款面向服务端应用的垃圾收集器。HotSpot开发团队赋予它的使命是未来可以替换掉JDK 1.5 中发布的CMS收集器。与其他GC收集器相比，G1具备如下特点：

* 

* 

* 


#### 内存分配和回收策略

##### 大对象直接进入老年代

所谓的大对象是指需要大量连续内存空间的Java对象，最典型的大对象就是那种很长的字符串以及数组。

虚拟机提供了一个`-XX:PretenureSizeThreshold`参数，令大于这个设置的对象直接在老年代分配。这样做的目的是
避免在Eden区及两个Survivor区之间发生大量的内存复制（复习一下：新生代采用复制算法收集内存）。


> PretenureSizeThreshold参数只对Serial和ParNew两款收集器有效，Parallel Scavenge收集器不认识这个参数，Parallel Scavenge收集器一般并不需要设置。如果遇到必须使用此参数的场合，可以考虑ParNew加CMS的收集器组合。