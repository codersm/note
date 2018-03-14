---
title: MySQL
date: 2016-09-29 17:34:08
tags: MySQL
---

## MySQL体系结构和存储引擎

MySQL被设计为一个可移植的数据库，几乎在当前所有系统上都能运行，如Linux、Mac和Windows等。尽管各平台在底层实现方面都各有不同，
但是MySQL基本上能够保证在各平台上的物理体系结构的一致性。

**常见数据库术语**

* 数据库
  
  物理操作系统文件或其他形式文件类型的集合。在MySQL数据库中，数据文件可以是frm、MYD、MYI、ibd结尾的文件。

* 实例
 
  MySQL数据库由后台线程以及一个共享内存组成。共享内存可以被运行的后台线程所共享。数据库实例才是真正用于操作数据库文件的。

在MySQL数据库中，实例与数据库的通常关系是一一对应的，即一个实例对应一个数据库，一个数据库对应一个实例。但是，在集群情况下可能存在一个数据库被多个数据实例使用的情况。

MySQL被设计为一个单进程多线程架构的数据库，也就是说，**MySQL数据库实例在系统上的表现就是一个进程**。

当启动实例时，MySQL数据库会去读取配置文件，根据配置文件的参数来启动数据库实例。如果没有配置文件，MySQL会按照编译时的默认参数设置启动实例。可以使用以下命令查看MySQL数据库实例启动时，会在那些位置查找配置文件。

```
mysql --help | grep my.cnf
```

如果几个配置文件中都有同一个参数，MySQL数据库会以读取到最后一个配置文件中的参数为准。

### MySQL体系结构

![MySQL体系结构图](http://upload-images.jianshu.io/upload_images/2889214-74b1cd3a008b8c9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由上图可知，MySQL由以下几部分组成：
- 连接池组件
- 管理服务和工具组件
- SQL接口组件
- 查询分析器组件
- 优化器组件
- 缓冲组件
- 插件式存储引擎
- 物理文件

MySQL数据库区别于其他数据库的最重要的一个特点就是其插件式的表存储引擎。MySQL插件式的存储引擎架构提供了一系列标准的管理和服务支持，这些标准与存储引擎本身无关，可能是每个数据库系统本身都是必需的，如SQL分析期和优化器等，而存储引擎是底层物理结构的实现，每个存储引擎开发者可以按照自己的意愿来进行开发。**存储引擎是基于表的，而不是数据库。**

### MySQL存储引擎

#### InnoDB存储引擎

  InnoDB存储引擎支持事务，其设计目标主要面向在线事务处理（OLTP）的应用。其特点是行锁设计、支持外键、并支持类似于Oracle的非锁定读，即默认读取操作不会产生锁。从MySQL数据库5.5.8版本开始，InnoDB存储引擎是默认的存储引擎。

   InnoDB存储引擎将数据放在一个逻辑的表空间中，这个表空间就像黑盒一样由InnoDB存储引擎自身进行管理。从MySQL4.1（包括4.1）版本开始，它可以将每个InnoDB存储引擎的表单独存放到一个独立的ibd文件中。此外，InnoDB存储引擎支持用裸设备用来建立其表空间。
  
  InnoDB通过使用多版本并发控制（MVCC）来获得高并发性，并且实现了SQL标准的4种隔离级别，默认REPEATABLE级别。同时，使用一种被称为next-keylocking的策略来避免幻读现象的产生。除此之外，InnoDB存储引擎还提供了插入缓冲、二次写、自适应哈希索引、预读等高性能和高可用的功能。

  对于表中数据的存储，InnoDB存储引擎采用了聚集的方式，因此每张表的存储都是按主键的顺序进行存放。如果没有显式地在表定义时指定主键，InnoDB存储引擎会为每一行生成一个6字节的ROWID，并以此作为主键。
  
#### MyISAM存储引擎

 MyISAM存储引擎不支持事务、表锁设计、支持全文索引，主要面向一些OLAP数据库应用。在
 MySQL5.5.8版本之前，MyISAM存储引擎是默认的存储引擎（除Windows版本外）。
 
 **数据库系统与文件系统很大的一个不同之处在与对事务的支持，然后MyISAM存储引擎是不支持事务的**。这不很难理解。试想用户是否在所有的应用中都需要事务呢？在数据仓库中，如果没有ETL这些操作，只是简单的报表查询是否还需要事务的支持呢？此外，MyISAM存储引擎的另外一个与从不同的地方是它的缓存池只缓存索引文件，而不缓冲数据文件，这点和大多数的数据库都非常不同。

#### NDB存储引擎

NDB存储引擎是一个集群存储引擎，类似于Oracle的RAC集群，不过与Oracle RAC share everything架构不同的是，其结构是share nothing的集群架构，因此能提供更高的可用性。NDB的特定是数据全部放在内存中（从MySQL5.1版本开始，可以将非索引数据放在磁盘上），因此主键查找的速度极快，并且通过添加NDB数据存储节点（Data Node）可以线性地提高数据库性能，是高可用、高性能的集群系统。

关于NDB存储引擎，有一个问题值得注意，那就是NDB存储引擎的连接操作（JOIN）是在MySQL数据库层完成的，而不是在存储引擎完成的。这意味着，复杂的连接操作需要巨大的网络开销，因此查询速度很慢。如果解决了这个问题，NDB存储引擎的市场应该是非常巨大的。

## InnoDB存储引擎

InnoDB是事务安全的MySQL存储引擎，设计上采用了类似与Oracle数据库的构架。通常来说，InnoDB存储引擎是OLTP应用中核心表的首选存储引擎。

InnoDB存储引擎被包含于所有MySQL数据库的二进制发行版本中。早期其版本随着MySQL数据库的更新而更新。从MySQL5.1版本时，MySQL数据库允许存储引擎开发商以动态方式加载引擎，这样存储引擎的更新可以不受MySQL数据库版本的限制。

### InnoDB体系架构

~~InnoDB存储引擎有多个内存块，可以认为这些内存块组成了一个大的内存池，负责如下工作：
- 维护所有进程/线程需要访问的多个内部数据结构。
- 缓存磁盘上的数据，方便快速地读取，同时在对磁盘文件的数据修改之前在这里缓存。
- 重做日志缓冲。~~

#### 后台线程
InnoDB存储引擎是多线程的模型，因此其后台有多个不同的后台线程，负责处理不同的任务。
- Master Thread

  Master Thread是一个非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲、UNDO页的回收等。

- IO Thread

  在InnoDB存储引擎中大量使用了AIO（Async IO）来处理写I/O请求，这样可以极大提高数据库的性能。而IO Thread的工作主要是负责这些IO请求的回调。

- Purge Thread

  事务被提交后，其所使用的undolog可能不再需要，因此需要PurgeThread来回收已经使用并分配的undo页。在InnoDB 1.1版本之前，purge操作仅在InnoDB存储引擎的Master Thread中完成。而从InnoDB 1.1版本开始，purge操作可以独立到单独的线程中进行，以此减轻Master Thread的工作，从而提高CPU的使用率以及提升存储引擎性能。

- Page Cleaner Thread

  Page Cleaner Thread是在InnoDB 1.2.x版本中引入的。其作用是将之前版本中脏页刷新操作都放入到单独的线程中来完成。而其目的是为了减轻原Master Thread的工作以及用户查询线程的阻塞，进一步提高InnoDB存储引擎的性能。

#### 内存

- 缓冲池

  InnoDB存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。因此可将其视为基于磁盘的数据库系统。在数据库系统中，由于CPU速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用**缓冲池技术**来提高数据库的整体性能。

  缓冲池简单来说就是一块内存区域，通过内存的速度来弥补磁盘速度较慢对数据库性能的影响。在数据库中进行读取页的操作，首先将从磁盘读到的页存放在缓冲池中，这个过程称为将页“FIX”在缓冲池中。下一次再读相同的页时，首先判断该页是否在缓冲池中。若在缓冲池中，称该页在缓冲池中被命中，直接读取该页。否则，读取磁盘上的页。

  对于数据库中页的修改操作，则首先修改再缓冲池中的页，然后再以一定的频率刷新到磁盘上。这里需要注意的是，页从缓冲池刷新回磁盘的操作并不是在每次页发生更新时触发，而是通过一种称为Checkpoint的机制刷新回磁盘。
  
  > 缓存池的大小直接影响着数据库的整体性能。

  具体来看，缓冲池中缓存的数据页类型有：索引页、数据页、undo页、插入缓冲、自适应哈希索引、InnoDB存储的锁信息、数据字典信息等。

  > 不能简单地认为，缓冲池只是缓存索引页和数据页，它们只是占缓冲池很大的一部分而已。

  从InnoDB 1.0.x版本开始，允许有多个缓冲池实例。每个页根据哈希值平均分配到不同缓冲池实例中。这样做的好处是减少数据库内部的资源竞争，增加数据库的并发处理能力。

- 缓冲池管理

  通常来说，数据库中的缓冲池是通过LRU（Latest Reccent Used，最近最少使用）算法来进行管理的。即最频繁使用的页在LRU列表的前端，而最少使用的页在LRU列表的尾端。当缓冲池不能存放新读取的页时，将首先释放LRU列表中尾端的页。

- 重做日志缓冲

  InnoDB存储引擎首先将重做日志信息先放入到重做日志缓冲这个缓冲区，然后按一定频率将其刷新到重做日志文件。重做日志缓冲一般不需要设置很大，因为每一秒钟会将重做日志缓冲刷新到日志文件，因此用户只需要保证每秒产生的事务量在这个缓冲大小之内即可。该值可由配置参数innodb_log_buffer_size控制，默认为8MB。

  通常情况下，8MB的重做日志缓冲池足以满足绝大部分的应用，因为重做日志在下列三种情况下会将重做日志缓冲中的内容刷新到外部磁盘的重做日志文件中。

- 额外的内存池

#### Checkpoint技术

Checkpoint（检查点）技术的目的时解决以下几个问题：

- 缩短数据库的恢复时间；
- 缓冲池不够用时，将脏页刷新到磁盘；
- 重做日志不可用时，刷新脏页。  

  当数据库发生宕机时，数据库不需要重做所有的日志，因为Checkpoint之前的页都已经刷新回磁盘。故数据库只需对Checkpoint后的重做日志进行恢复。这样就大大缩短了恢复的时间。

在InnoDB存储引擎内部，又两种Checkpoint，分别为：

- Sharp Checkpoint

  Sharp Checkpoint发生在数据库关闭时将所有的脏页都刷新回磁盘，这是默认的工作方式，即参数innodb_fast_shutdown = 1。

- Fuzzy Checkpoint

Fuzzy Checkpoint刷新一部分脏页，而不是刷新所有的脏页回磁盘。

在InnoDB存储引擎中可能发生如下几种情况的Fuzzy Checkpoint：

- Master Thread Checkpoint
- FLUSH_LRU_LIST Checkpoint
- Async/Sync Flush checkpoint
- Dirty Page too much Checkpoint

### InnoDB关键特性

InnoDB存储引擎的关键特性包括：
- 插入缓冲（Insert Buffer）

非聚集索引

InnoDB存储引擎开创性地设计了Insert Buffer，对于非聚集索引的插入或更新操作，不是每一次直接插入到索引页中，而是先胖段插入的非聚集索引页是否缓冲池中，若在，则直接插入；若不在，则先放入到一个Insert Buffer对象中，好似欺骗。数据库这个非聚集的索引已经插到叶子节点，而实际并没有，只是存放在另一个位置。然后再以一定的频率和情况进行Insert Buffer和辅助索引页子节点的merge（合并）操作，这时通常能将多个插入合并到一个操作中，这就大大提高类对于非聚集索引插入的性能。

Insert Buffer的使用需要同时满足以下两个条件：

- 索引是辅助索引
- 索引不是唯一的

Change Buffer


Insert Buffer内部实现

Insert Buffer是一棵B+树，因此也由叶结点和非叶结点组成。非叶结点存放的是查询的search key（键值）

Insert Buffer Bitmap

用来标记每个辅助索引页的可用空间。每个Insert Buffer Bitmap页用来追踪16384个辅助索引页，也就是256个区。

Merge Insert Buffer

Insert Buffer中的记录何时合并到真正的辅助索引中？





- 两次写

提高数据页的可靠性。

当发生数据库宕机时，可能InnoDB存储引擎正在写入某个页到表中，儿这个页只写了一部分，之后就发生了宕机，这种情况被成为部分写失效。在InnoDB存储引擎未使用doublewrite技术前，曾经出现过因为部分写失效而导致数据丢失的情况。



- 自适应哈希索引

InnoDB存储引擎会监控对表上各索引页的查询。如果观察到建立哈希索引可以带来速度提升，则建立哈希索引，称之为**自适应哈希索引（Adaptive Hash Index，AHI）**。

AHI是通过缓存池的B+树页构造而来，因此建立的速度很快，而且不需要对整张表构建哈希索引。InnoDB存储引擎会自动根据访问的频率和模式来自动地为某些热点页建立哈希索引。




- 异步IO（Async IO）
- 刷新邻接页

其工作原理为：当刷新一个脏页时，InnoDB存储引擎会检测该页所在区的所有页，如果是脏页，那么一起进行刷新。这样做的好处显而易见，通过AIO可以将多个IO写入操作合并为一个IO操作，故该工作机制在传统机械磁盘下有着显薯的优势。


## 表

简单来说，表就是关于特定实体的数据集合，是关系型数据库模型核心。

### 索引组织表

在InnoDB存储引擎中，表都是根据主键顺序组织存放的，这种存储方式的表称为索引组织表。在InnoDB存储引擎表中，每张表都有个主键，如果在创建表时没有显式地定义主键，则InnoDB存储引擎按如下方式选择或创建主键：

- 首先判断表中是否有非空的唯一索引（Unique NOT NULL），如果有，则该列即为主键。
- 如果不符合上述条件，InnoDB存储引擎自动创建一个6字节大小的指针。

当表中有多个非空唯一索引时，InnoDB存储引擎将选择建表时第一个定义的非空唯一索引为主键。_rowid可以显示表的主键，但_rowid只能用于查看单个列为主键的情况，对于多列组成的主键就显得无能为力了。


### InnoDB逻辑存储结构

从InnoDB存储引擎的逻辑存储结构看，所有数据都被逻辑地存放在一个空间中，称之为表空间（tablespace）。表空间又由段(segment)、区(extent)、页(page)组成。InnoDB存储引擎的逻辑存储结构大致如图所示：

![InnoDB逻辑存储引擎](http://upload-images.jianshu.io/upload_images/2889214-1f6e25931897b530.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 表空间

表空间可以看做是InnoDB存储引擎逻辑结构的最高层，所有的数据都存放在表空间中。在默认情况下InnoDB存储引擎有一个共享表空间ibdata1，即所有数据都存放在这个表空间内。如果用户启用了参数innodb_file_per_table，则每张表内的数据可以单独放到一个表空间内。

如果启用了innodb_file_per_table的参数，需要注意的是每张表的表空间内存放的只是数据、索引和插入缓冲Bitmap页，其他类的数据还是存放在原来的共享表空间内。

### 段

常见的段有数据段、索引段、回滚段等。数据段即为B+树的叶子节点，索引段即为B+树的非索引节点。？

### 区

区是由连续页组成的空间，在任何情况下每个区的大小都为1MB。为了保证区中页的连续性，InnoDB存储引擎一次从磁盘申请4～5个区。在默认情况下，InnoDB存储引擎页的大小为16KB，即一个区中一共有64个连续的页。

### 页

页是InnoDB磁盘管理的最小单位。在InnoDB存储引擎中，默认每个页的大小为16KB。而从InnoDB 1.2.x版本开始，可以通过参数innodb_page_size将页的大小设置为4K、8K、16K。若设置完成，则所有表中页的大小都为innodb_page_size，不可以对其再次进行修改。除非通过mysqldump导入和导出操作来产生新的库。

在InnoDB存储引擎中，常见的页类型有：

- 数据页
- undo页
- 系统页
- 事务数据页
- 插入缓冲位图页
- 插入缓冲空闲列表页
- 未压缩的二进制大对象页
- 压缩的二进制大对象页

### 行

InnoDB存储引擎是面向列的，也就说数据是按行进行存放的。每个页存放的行记录也是有硬性定义的，最多允许存放16KB／2-200行的记录，即7992行记录。

#### InnoDB行记录格式

InnoDB存储引擎提供了Compact和Redundant两种格式来存放行记录数据，Redundant格式是为兼容之前版本而保留，在MySQL 5.1版本中，默认设置为Compact行格式。

可以通过如下命令查看当前表使用的行格式，其中row_format属性表示当前所使用的行记录结构类型。
```java
SHOW TABLE STATUS LIKE 'table_name';
```

#### Compact行记录格式

Compact行记录是在MySQL 5.0中引入的，其设计目标是高效地存储数据。简单来说，一个页中存放的行数据越多，其性能就越高。

InnoDB存储引擎在页内部是通过一种链表的结构来串连各个行记录的。

在compact格式下NULL值都不占用任何存储空间。

Redundant行记录格式

Redundant是MySQL 5.0版本之前InnoDB的行记录存储方式，MySQL 5.0支持Redundant是为了兼容之前版本的页格式。

InnoDB存储引擎表是索引组织的，即B+Tree的结构，这样每个页中至少应该有两条行记录（否则失去了B+Tree的意义，变成链表了）。因此，如果页中只能存放下一条记录，那么InnoDB存储引擎会自动将行数据存放到溢出页中。

InnoDB 1.0.x版本开始引入了新的文件格式，以前支持的Compact和Redundant格式称为Antelope文件，新的文件格式为Barracuda文件格式。Barracuda文件格式下拥有两种新的行记录格式：Compressed和Dynamic。

### 约束

约束是一个逻辑的概念，用来保证数据的完整性，而索引是一个数据结构，既有逻辑上的概念，在数据库还代表着物理存储的方式。

在某些默认设置下，MySQL数据库允许非法的或不正确的数据的插入或更新，又或者可以在数据库内部将其转化为一个合法的值，如向NOT NULL的字段插入一个NULL值，MySQL数据库会将其更改为0再进行插入，因此数据库本身没有对数据的正确性进行约束。

如果用户想通过约束对于数据库非法数据的插入或更新，即MySQL数据库提示报错而不是警告，那么用户必须设置参数sql_mode,用来严格审核输入的参数。

#### 视图

在MySQL数据库中，视图是一个命名的虚表，它由一个SQL查询来定义，可以当做表使用。与持久表不同的是，视图中的数据没有实际的事物存储。

#### 分区表

分区功能并不是在存储引擎层完成的，MySQL数据库在5.1版本时添加了对分区的支持。分区的过程是将一个表或索引分解为多个更小、更可管理的部分。就访问数据库的应用而言，从逻辑上讲，只有一个表或一个索引，但是在物理上这个表或索引可能由十个物理分区组成。

MySQL数据库支持的分区类型为水平分区，并不支持垂直分区。此外，MySQL的分区是局部分区索引，一个分区中即存放了数据又存放了索引。而全局分区指，数据存放在各个分区中，但是所有数据的索引放在一个对象中。


## 索引与算法

索引是应用程序设计和开发的一个重要方面。若索引太多，应用程序的性能可能会受到影响。而索引太少，对查询性能又会产生影响。要找到一个合适的平衡点，这对应用程序的性能至关重要。

InnoDB存储引擎支持以下几种常见的索引：
- B+树索引
- 全文索引
- 哈希索引

B+树索引并不能找到一个给定键值的具体行。B+树索引能找到的只是被查找数据行所在的页。然后数据库通过把页读人到内存，再在内存中进行查找，最后得到要查找的数据。

### B+树索引

B+树索引的本质就是B+树在数据库中的实现。但是B+索引在数据库中有一个特点是 _高扇出性_，因此在数据库中，B+树的高度一般都在2-4层。

数据库中的B+树索引可以分为聚集索引和辅助索引，但是不管是聚集索引还是辅助索引，其内部都是B+树的，叶子节点存放着所有的数据。聚集索引与辅助索引不同的是，叶子节点存放的是否是一整行的信息。

#### 聚集索引

聚集索引就是按照每张表的主键构造一颗B+树，同时叶子结点中存放的即为整张表的行记录数据，也将聚集索引的叶子节点称为数据页。聚集索引的这个特性决定了索引组织表中数据也是索引的一部分。同B+树数据结构一样，每个数据页都通过一个双向链表来进行链接。

#### 辅助索引

辅助索引叶子节点并不包含行记录的全部数据，叶子节点除了包含键值以外，每个叶子节点中的索引行中还包含了一个书签。该书签用来告诉InnoDB存储引擎哪里可以找到与索引相对应的行数据。由于InnoDB存储引擎表是索引组织表，因此InnoDB存储引擎的辅助索引的书签就是相应行数据的聚集索引键。

当通过辅助索引来寻找数据时，InnoDB存储引擎会遍历辅助索引并通过叶级别的指针获得指向主键索引的主键，然后再通过主键索引来找到一个完整的行记录。

Page Header

通过命令SHOW INDEX FROM table_name可以查看索引：


![Jietu20171221-113413.jpg](http://upload-images.jianshu.io/upload_images/2889214-dff0a3d2076f038c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

SELECT DISTINCT
    TABLE_NAME,
    INDEX_NAME
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = 'your_schema';


索引展示字段说明：

| 字段           | 说明                                                                       |
| ------------ | ------------------------------------------------------------------------- |
| Table        | 索引所在的表名                                                                   |
| Non_unique   | 非唯一的索引                                                                    |
| Key_name     |                                                                           |
| Seq_in_index | 索引中该列的位置                                                                  |
| Colunm_name  | 索引列的名称                                                                    |
| Collation    | 列以什么方式存储在索引中。可以是A或NULL。B+树索引总是A，即排序的                                     |
| Cardinality  | 非常关键的值，表示索引中唯一值的数目的估计值。Cardinality表的行数应尽可能接近1，如果非常小，那么用户需要考虑是否可以删除此索引 |
| Sub_part     | 是否是列的部分被索引。                                                               |
| Packed       | 关键字如何被压缩。如果没有被压缩，则为NULL。                                                  |
| Null         | 是否索引的列含有NULL值。                                                            |
| Index_type   | 索引的类型。InnoDB存储引擎只支持B+树索引，所以这里显示的都是BTREE。                                  |
| Comment      | 注释                                                                        |


在MySQL5.5版本之前，Mysql数据库对于索引的添加或者删除的这类DDL操作，MySQL数据库的操作过程为：

- 首先创建一张新的临时表，表结构为通过命令ALTER TABLE新定义的结构。
- 然后把原表中数据导入到临时表。
- 接着删除原表。
- 最后把临时表重名为原来的表名。

若用户对于一张大表进行索引的添加和删除操作，那么这会需要很长的时间。更关键的是，若有大量事务需要访问正在被修改的表，这意味着数据库服务不可用。


InnoDB存储引擎从InnoDB 1.0.x版本开始支持一种称为Fast Index Creation（快速索引创建）的索引创建方式——简称FIC。

#### Online DDL

MySQL 5.6版本开始支持Online DDL（在线数据定义）操作，其允许辅助索引创建的同时，还允许其他诸如INSERT、UPDATE、DELETE这类DML操作，这极大地提高了MySQL数据库在生产环境中的可用性。

此外，不仅是辅助索引，以下这几类DDL操作都可以通过“在线”的方式进行操作：
- 辅助索引的创建与删除
- 改变自增长值
- 添加或删除外键约束
- 列的重命名

#### B+树索引的使用

1、OLTP和OLAP应用

2、联合索引

联合索引是指对表上的多个列进行索引。

3、覆盖索引

InnoDB存储引擎支持覆盖索引，即从辅助索引中就可以得到查询的记录，而不需要查询聚集索引中的记录。使用覆盖索引的一个好处是覆盖索引不包含整行记录的所有信息，故其大小要远小于聚集索引，因此可以减少大量的IO操作。

4、索引提示

MySQL数据库支持索引提示（INDEX HINT），显式地告诉优化器使用哪个索引。

FORCE INDEX

5、Multi-Range Read优化

MySQL5.6版本开始支持Multi-Range Read（MRR）优化。其目的就是为了减少磁盘的随机访问，并且将随机访问转化为较为顺序的数据访问，这对于IO-bound类型的SQL查询语句可带来性能极大的提升。Multi-Range Read优化可适用于range，ref，eq_ref类型的查询。

6、Index Condition Pushdown（ICP）优化

#### 自适应哈希

通过如下命令可以查看当前自适应哈希索引的使用状况：
```java
SHOW ENGINE INNODB STATUS;
```

哈希索引只能用来搜索等值的查询，而对于其他查找类型，如范围查找，是不能使用哈希索引的。

#### 全文检索

全文索引是将存储于数据库中的整本书或整篇文章中的任意内容信息查找出来的技术。从nnoDB 1.2.x版本开始，InnoDB存储引擎开始支持全文索引。

全文索引通常使用倒排索引来实现。




# MySQL记录

## 1、select into from 和 insert into select 的用法和区别

> Insert是T-sql中常用语句，但我们在开发中经常会遇到需要表复制的情况，如将一个table1的数据的部分字段复制到table2中，或者将整个table1复制到table2中，这时候我们就要使用SELECT INTO 和 INSERT INTO SELECT 表复制语句了

### INSERT INTO SELECT语句

```sql
  Insert into Table2(field1,field2,...) select value1,value2,... from Table1
```

> **注意地方**
>
> （1）要求目标表Table2必须存在，并且字段field,field2...也必须存在
>
> （2）注意Table2的主键约束，如果Table2有主键而且不为空，则 field1， field2...中必须包括主键
>
>（3）注意语法，不要加values，和插入一条数据的sql混了，不要写成:
>
>（4）由于目标表Table2已经存在，所以我们除了插入源表Table1的字段外，还可以插入常量。

### SELECT INTO FROM语句

```sql
  SELECT vale1, value2 into Table2 from Table1
```

要求目标表Table2不存在，因为在插入时会自动创建表Table2，并将Table1中指定字段数据复制到Table2中 。

select into from 和 insert into select都是用来复制表，两者的主要区别为：select into from 要求目标表不存在，因为在插入时会自动创建。insert into select from 要求目标表存在。

## MySQL多表关联数据同时删除

1、从数据表t1中把那些id值在数据表t2里有匹配的记录全删除掉

```sql
  DELETE t1 FROM t1,t2 WHERE t1.id=t2.id 或 DELETE FROM t1 USING t1,t2 WHERE t1.id=t2.id
```

2、从数据表t1里在数据表t2里没有匹配的记录查找出来并删除掉

```sql
   DELETE t1 FROM t1 LEFT JOIN T2 ON t1.id=t2.id WHERE t2.id IS NULL 或 DELETE FROM t1,USING t1 LEFT JOIN T2 ON t1.id=t2.id WHERE t2.id IS NUL
```

3、从两个表中找出相同记录的数据并把两个表中的数据都删除掉

```sql
   DELETE t1,t2 from t1 LEFT JOIN t2 ON t1.id=t2.id
```

## A LEFT JOIN B ON 条件表达式(WHERE)

  1、ON 条件（“A LEFT JOIN B ON 条件表达式”中的ON）用来决定如何从 B 表中检索数据行。如果 B 表中没有任何一行数据匹配 ON 的条件,将会额外生成一行所有列为 NULL 的数据。
  2、在匹配阶段 WHERE 子句的条件都不会被使用。仅在匹配阶段完成以后，WHERE 子句条件才会被使用。它将从匹配阶段产生的数据中检索过滤。

## Mysql 列转行统计查询 、行转列统计查询

[Mysql 列转行统计查询 、行转列统计查询 ](http://www.cnblogs.com/lhj588/p/3315876.html)

## MySQL查询和修改auto_increment的方法

查询表名为tableName的auto_increment值：

```sql
SELECT AUTO_INCREMENT FROM information_schema.tables WHERE table_name="tableName";
```

修改表名为tableName的auto_increment值：

```sql
ALTER TABLE tableName auto_increment=number;
```

## mysql的auto_increment_offset和auto_increment_increment配置

mysql中有自增长字段，在做数据库的主主同步时需要设置自增长的两个相关配置：auto_increment_offset和auto_increment_increment。

auto_increment_offset表示自增长字段从那个数开始，他的取值范围是1 .. 65535
auto_increment_increment表示自增长字段每次递增的量，其默认值是1，取值范围是1 .. 65535
在主主同步配置时，需要将两台服务器的auto_increment_increment增长量都配置为2，而要把auto_increment_offset分别配置为1和2.

这样才可以避免两台服务器同时做更新时自增长字段的值之间发生冲突。

更多信息请参考官方文档： http://dev.mysql.com/doc/refman/5.0/en/replication-options-master.html  

## MySQL中删除数据的两种方法

如果要清空表中的所有记录，可以使用下面的两种方法：

　　DELETE FROM table1
　　TRUNCATE TABLE table1

　　其中第二条记录中的TABLE是可选的。

　　如果要删除表中的部分记录，只能使用DELETE语句。

　　DELETE FROM table1 WHERE ...;

　　如果DELETE不加WHERE子句，那么它和TRUNCATE TABLE是一样的，但它们有一点不同，那就是DELETE可以返回被删除的记录数，而TRUNCATE TABLE返回的是0。

　　如果一个表中有自增字段，使用TRUNCATE TABLE和没有WHERE子句的DELETE删除所有记录后，这个自增字段将起始值恢复成1.如果你不想这样做的话，可以在DELETE语句中加上永真的WHERE，如WHERE 1或WHERE true。

　　DELETE FROM table1 WHERE 1;

　　上面的语句在执行时将扫描每一条记录。但它并不比较，因为这个WHERE条件永远为true。这样做虽然可以保持自增的最大值，但由于它是扫描了所有的记录，因此，它的执行成本要比没有WHERE子句的DELETE大得多。

　　DELETE和TRUNCATE TABLE的最大区别是DELETE可以通过WHERE语句选择要删除的记录。但执行得速度不快。而且还可以返回被删除的记录数。而TRUNCATE TABLE无法删除指定的记录，而且不能返回被删除的记录。但它执行得非常快。

　　和标准的SQL语句不同，DELETE支持ORDER BY和LIMIT子句，通过这两个子句，我们可以更好地控制要删除的记录。如当我们只想删除WHERE子句过滤出来的记录的一部分，可以使用LIMIB，如果要删除后几条记录，可以通过ORDER BY和LIMIT配合使用。假设我们要删除users表中name等于"Mike"的前6条记录。可以使用如下的DELETE语句：

　　DELETE FROM users WHERE name = 'Mike' LIMIT 6;

　　一般MySQL并不确定删除的这6条记录是哪6条，为了更保险，我们可以使用ORDER BY对记录进行排序。

　　DELETE FROM users WHERE name = 'Mike' ORDER BY id DESC LIMIT 6;

## MySQL 添加列，修改列，删除列的sql语句写法

# mysql or条件可以使用索引而避免全表

http://blog.csdn.net/hguisu/article/details/7106159
http://xianglp.iteye.com/blog/869892

# MySQL: Combining the AND and OR Conditions

https://www.techonthenet.com/mysql/and_or.php

## Disable ONLY_FULL_GROUP_BY

``` sql
SET GLOBAL sql_mode=(SELECT REPLACE(@@sql_mode,'ONLY_FULL_GROUP_BY',''))
```
