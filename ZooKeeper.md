---
title: ZooKeeper
date: 2016-07-12T11:31:57.000Z
tags: ZooKeeper
---

<!-- more -->

# ZooKeeper简介

Zookeeper是一个分布式的、开源的分布式应用协调服务。ZooKeeper是用于维持配置信息、命名、提供分布式同步、并提供分组服务的集中式服务。

Zookeeper并没有直接采用Paxos算法，而是采用了一种被称为ZAB的一致性协议。


## ZooKeeper单机版安装
1、环境要求：各种系统都行，java1.6+；  
2、下载压缩包，解压；  
3、在解压目录下，新建conf/zoo.cfg文件。示例如下：
```
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
```
4、执行解压目录下的 bin/zkServer.sh start 开启zookeeper；  
5、执行解压目录下的 bin/zkCli.sh -server 127.0.0.1:2181 连接zookeeper。

一旦连接上了，你应对会看到这些信息：
```
Welcome to ZooKeeper!
2016-09-04 21:46:19,400 [myid:] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@876] - Socket connection established to 127.0.0.1/127.0.0.1:2181, initiating session  
JLine support is enabled
[zk: 127.0.0.1:2181(CONNECTING) 0] 2016-09-04 21:46:19,516 [myid:] - INFO  [main-SendThread(127.0.0.1:2181):ClientCnxn$SendThread@1299] - Session establishment complete on server 127.0.0.1/127.0.0.1:2181, sessionid = 0x156f56d81520000, negotiated timeout = 30000
```
从shell脚本输入help，可以获取客户端可执行的命令列表，如：
```
help
ZooKeeper -server host:port cmd args
	connect host:port
	get path [watch]
	ls path [watch]
	set path data [version]
	rmr path
	delquota [-n|-b] path
	quit
	printwatches on|off
	create [-s] [-e] path data acl
	stat path [watch]
	close
	ls2 path [watch]
	history
	listquota path
	setAcl path acl
	getAcl path
	sync path
	redo cmdno
	addauth scheme auth
	delete path [version]
	setquota -n|-b val path

```
从这里，你可以尝试一些简单的命令行接口找找感觉。第一，通过发行的列表命令开始，像ls :
```
[zk: 127.0.0.1:2181(CONNECTED) 1] ls /
[zookeeper]

```
下一步，通过运行create /zk_test my_data，创建一个新的znode。这将创建一个新的znode节点和一个相关联的字符串"my_data"。你应该看到：
```
[zk: 127.0.0.1:2181(CONNECTED) 2] create /zk_test my_data
Created /zk_test
```
使用ls / 命令查看目录：
```
[zk: 127.0.0.1:2181(CONNECTED) 3] ls /
[zookeeper, zk_test]
```
注意 zk_test目录现在已经创建了。

接下来，使用 get 命令验证数据是否与znode关联上了
```
[zk: 127.0.0.1:2181(CONNECTED) 6] get /zk_test
my_data
cZxid = 0x1d
ctime = Sun Sep 04 21:48:56 CST 2016
mZxid = 0x1d
mtime = Sun Sep 04 21:48:56 CST 2016
pZxid = 0x1d
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 7
numChildren = 0
```

Zookeeper可以保证如下分布式一致性特性：

顺序一致性

原子性

单一视图

可靠性

实时性


### Zookeeper设计目标

#### 简单的数据模型

#### 可以构建集群

#### 顺序访问

#### 高性能

### Zookeeper基本概念

* 集群角色

  在Zookeeper中，

  Leader：Leader服务器为客户端提供读和写服务
  Follower：
  Observer：

* 会话（Session）

* 数据节点（Znode）

* 版本
* Watcher

* ACL


### ZooKeeper的ZAB协议

ZAB协议的核心是定义了对于那些会改变Zookeeper服务器数据状态的事务请求的处理方式，即：


ZAB协议包括两种基本的模式
#### 消息广播

#### 





# ZooKeeper API使用
Zookeeper客户端和服务端会话的建立是一个异步建立的过程，也就是说在程序中，构造方法会在处理客户端初始化工作后立即返回，在大多数情况下，此时并没有真正建立好一个可用的会话，在会话的生命周期中处于“CONNECTING”状态。  

当该会话真正创建完毕后，ZooKeeper服务端会向会话对应的客户端发送一个事件通知，已告知客户端，客户端只有在获取这个通知之后，才算真正建立了会话。  

## 创建节点

Zookeeper都不支持递归创建，即无法在父节点不存在的情况下创建一个字节点。另外，如果一个节点已经存在了，那么创建同名节点的时候，会抛出NodeExistsException异常。

异步接口创建和同步接口创建的区别：

1、节点的创建过程（包括网络通信和服务端的节点创建过程）是异步的；

2、在同步接口调用过程中，我们需要关注接口抛出异常的可能；但是在异步接口中，接口本身是不会抛出异常的，所有的异常都会在回调函数中通过Result Code（响应码）来体现；

### 删除节点

在ZooKeeper中，只允许删除叶子结点。也就是说，如果一个节点存在至少一个字节点的话，那么该节点将无法被直接，必须先删除掉其所有子节点。

### 读取数据

读取数据，包括子节点列表的获取和节点数据的获取。


默认Watcher

由于Watcher通知是一次性的，即一旦触发一次通知后，该Watcher就失效了，因此客户端需要反复注册Watcher。

节点的数据内容或是节点的数据版本变化，都被看作是ZooKeeper节点的变化。

### 更新节点

version=-1告诉ZooKeeper服务器，客户端需要基于数据的最新版本进行更新操作。如果对Zookeeper数据节点的更新操作没有原子性要求，那么就可以使用“-1”。



# ZooKeeper编程

## 1、Zookeeper数据模型

ZooKeeper有一个层级命名空间，和一个分布式文件系统非常相似。唯一的不同是每个节点可以有关联的数据，子节点也是。就像有一个文件系统并且允许文件是一个目录。一个规范的、绝对的、斜杠分隔的路径来表示一个节点路径，没有相对路径。任何符合下列约束的的unicode字符可以被使用：
 a、null字符串（\u0000）不能是一个路径名称。
 b、下列字符不能被使用，因为不能很好的被展示：\u0001 – \u001F 和 \u007F – \u009F。
 c、下列字符是不允许的：\ud800 – uF8FF, \uFFF0 – uFFFF。
 d、"."字符可以作为另一个名字被使用，但是"."和".."不能单独使用来表示一个节点路径，因为ZooKeeper不使用相对路径，下列是无效的："/a/b/./c"或者 "/a/b/../c"。 e、"zookeeper"标记被保留。

### ZNodes
在ZooKeeper树中的每个节点被称为一个 znode 。Znodes包含了一个stat数据结构。
- ZooKeeper Stat 结构
  czxid：该数据节点被创建时的事务id。
  mzxid：该节点最后一次被更新时的事务id。
  ctime：节点被创建时的时间。
  mtime：节点最后一次被更新时的时间。
  version：这个节点的数据变化的次数。
  cversion：这个节点的子节点 变化次数。
  aversion：这个节点的ACL变化次数。
  ephemeralOwner：如果这个节点是临时节点，表示创建者的会话id。如果不是临时节点，这个值是0。
  dataLength：这个节点的数据长度。
  numChildren：这个节点的子节点个数。
#### ZNodes特性
- Watches
客户端可以在znodes上设置监听器，znode的改变触发这个监听器然后清空这个监听器。当一个监听器被触发，ZooKeeper发送给客户端一个通知。更多信息可以查看ZooKeeper Watches章节。
- 数据访问
每个znode的上存储的数据读写都是原子的，读操作取出所有的和这个znode有关的所有数据，写操作替换所有的数据。每个节点有一个访问权限列表（ACL）来限制谁可以做这些事情。
ZooKeeper没有被设计成一个一般的数据库或一个大型对象存储。它管理协调数据，数据可以是配置、状态信息、集合点等的形式。各种各样的数据有一个共同的属性就是他们都很小：以千字节为标准。
ZooKeeper客户端和服务器有一个健康检查来确保znodes的数据少于1M，但是数据平均应该更小。操作较大的数据将导致一些操作花费更多的时间，并且会影响一些操作的延迟，因为在网络和存储媒介中移动更多的数据将需要额外的时间。如果需要存储大数据，通常的处理是把数据存储在一个大容量存储系统中，并把存储位置的指针存储到ZooKeeper上。
- 临时节点
ZooKeeper也临时节点的概念。这些znodes存活的时间和创建这个节点的会话有效期是一样的。当会话结束，节点被删除。因为这种临时节点的特性，临时节点不允许有子节点。
- 顺序节点——唯一名称
当创建一个节点的时候，也可以请求ZooKeeper在路径后面增加一个自增的计数器。对父节点来说，这个计数器是 唯一的。计数器是%010d的格式——是一个十位数，比如：<path>0000000001。
查看 Queue Recipe 使用这个特性的示例，注意：这个计数器用来存储下一个序列号是一个4字节的数，当增加到2147483647 之后，计数器会溢出。
- ZooKeeper中的时间
ZooKeeper以多种方式跟踪时间：
 Zxid：ZooKeeper状态的每次变化都接收一个 zxid（ ZooKeeper事务id）形式的标记。这个展示了所有的ZooKeeper的变更顺序。每次变更会有一个唯一的zxid，如果zxid1小于zxid2说明zxid1在zxid2之前发生。
 Version numbers：节点的每次变化都会引起这个节点版本号之一的一次增加。这三个版本号是：version（一个节点的数据变化次数），cversion（一个节点的子节点变化次数），aversion（一个节点的ACL 变化次数）。
 Tricks：当使用多个ZooKeeper服务，服务器使用ticks来确定事件的时间，比如说状态上传、会话超时、连接超时等。这个tick时间仅仅通过最小会话超时时间间接的暴露出来；如果一个客户端请求会话的超时时间小于最小超时时间，服务器将会告诉客户端实际的会话超时时间是最小超时时间。

- Real Time：ZooKeeper不使用实时、时钟时间。除了把时间戳放在stat结构中

- Zookeeper会话
  创建客户端会话，应用代码必须提供一个以逗号分隔开的host:port的列表。Zookeeper客户端会选择任意一个服务并尝试连接它。如果这个连接失败，或者客户端变为disconected，客户端会自动的尝试连接列表里的下一个服务器，直到建立连接。

  客户端得到Zookeeper服务handle时，Zookeeper创建一个Zookeeper session，用64位的数字代表，分配到客户端。如果客户端连接到不同的Zookeeper服务器，他将发送session id作为连接握手的一部分。作为安全措施，服务器为session id创建一个任何Zookeeper服务器可以验证的密码。当建立会话是连同session id一起发送密码到客户端。客户端每当重建会话时都发送这个session id和密码。

  session逾期由Zookeeper集群自己管理，并不是客户端。当ZK客户端建立一个与集群的session时它提供一个上面描述的"timeout"值。这个值由集群确定什么时候客户端session逾期。当集群在指定的session超时周期内没有听到客户端(没有心跳)时发生。session逾期，集群会删除全部session的临时节点并立即通知其他链接的客户端(watch这些znode的客户端)。这时候逾期session的客户端仍然是disconnected的，它不会被通知session逾期知道他能够重新连接到集群。客户端将一直待在disconnected状态知道重新建立与集群的TCP连接，届时逾期session的watcher会收到"session expired"的通知。
- ZooKeeper Watches
- 一致性保证
- 构建块：ZooKeeper操作指南
- 绑定
- 程序结构：简单例子
- 陷阱：常见问题和故障排查


发布/订阅系统一般有两种设计模式，分别是推(Push)模式和拉(Pull)模式。在推模式中，服务端主动


### 典型应用场景

#### 数据发布／订阅

数据发布／订阅系统，即所谓的配置中心，实现配置信息的集中式管理和数据的动态更新。

发布／订阅系统一般有两种设计模式，分别是推（Push）模式和拉（Pull）模式。通常客户端采用定时进行拉取的方式。


在我们平常的应用系统开发中，经常会碰到这样的需求：系统中需要使用一些通用的配置信息，例如机器列表信息、运行时的开关配置、数据库配置信息等。这些
全局配置信息通常具备以下3个特性：

* 数据量通常比较小；

* 数据内容在运行时会发生动态变化；

* 集群中各机器共享，配置一致；

#### 负载均衡

基于Zookeeper实现的动态DNS方案

#### 命名服务

在分布式系统中，被命名的实体通常可以是集群中的机器，提供的服务地址或远程对象等——这些我们都可以统称它们为名字（Name），其中较为常见
的就是一些分布式服务框架（如RPC、RMI）中的服务地址列表，通过使用命名服务，客户端应用能够根据指定名字来获取资源的实体、服务地址
和提供者的信息等。

使用Zookeeper来实现一套分布式全局唯一ID的分配机制。

顺序节点

#### 分布式协调/通知

#### Master选举

### 技术内幕

#### 系统模型

##### 数据模型

Zookeeper的视图结构和标准的Unix文件系统非常类似，但没有引入传统文件系统中目录和文件等相关概念，而是使用了其特有的
“数据节点”概念，称之为ZNode。ZNode是Zookeeper中数据的最小单元，每个ZNode上都可以保存数据，同时还可以挂载子节点，
因此构成了一个层次化的命名空间，我们称之为树。

在Zookeeper中，每一个数据节点都被称为一个ZNode，所有ZNode按层次化结构进行组织，形成一棵树。ZNode的节点路径标识方式和Unix文件系统
路径非常相似，都是由一系列使用斜杠（／）进行分割的路径表示，开发人员可以向这个节点中写入数据，也可以在节点下面创建子节点。

![Zookeeper数据模型](http://upload-images.jianshu.io/upload_images/2889214-2797882c1734a91a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 事务ID

在Zookeeper中，事务是指能够改变Zookeeper服务器状态的操作，我们也称之为事务操作或更新操作，一般包括数据节点创建与删除、数据节点内容更新和客户端会话创建与失效等操作。对于每一个事务请求，Zookeeper都会为其分配一个全局唯一的事务ID，用ZXID来表示，通常是一个64位的数字。每一个ZXID对应一次更新操作，从这些ZXID中可以间接地识别出Zookeeper处理这些更新操作请求的全局顺序。

##### 节点特性

* 节点类型

  在Zookeeper中，每个数据节点都是有生命周期的，其生命周期的长短取决于数据节点的节点类型。在Zookeeper中，节点类型可以分为持久节点（PERSISTENT）、临时节点（EPHEMERAL）和顺序节点（SEQUENTIAL）三大类，具体在节点创建过程中，通过组合使用，可以生成以下四种组合型节点类型：

  * 持久节点

  * 持久顺序节点

  * 临时节点

  * 临时顺序节点

* 状态信息

	每个数据节点除了存储了数据内容之外，还存储了数据节点本身的一些状态信息（Stat），包含类一个数据节点的所有状态信息。Stat对象状态属性说明

	| 状态属性  | 说明                             |
	| ----- | ------------------------------ |
	| czxid | 即Created ZXID，表示该数据节点被创建时的事务ID |
	| mzxid      |即Modified ZXID，表示该节点最后一次被更新时的事务ID                                |
	|ctime|即Created Time，表示节点被创建的时间|
	|mtime|即Modified Time，表示该节点最后一次被更新的时间|
	|version|数据节点的版本号。|
	|cversion|子节点的版本号|
	|aversion|节点的ACL版本号|
	|ephemeralOwner|创建该临时节点的会话的sessionID。如果该节点是持久节点，那么这个属性值为0|
	|dataLength|数据内容的长度|
	|numChildren|当前节点的子节点个数|
	|pzxid|表示该节点的子节点列表最后一次被修改时的事务ID。注意，只有自节点列表变更了才会变更pzxid，子节点内容变更不会影响pzxid|

* 版本——保证分布式数据原子操作

在ZooKeeper中，version属性正是用来实现乐观锁机制中的“写入校验”的。

* Watcher——数据变更的通知

	Watcher注册流程

	* Watcher特性
	一次性
	客户端串行执行
	轻量

* ACL——保障数据的安全


#### 序列化与协议

#### 客户端

#### 会话

##### 服务器启动

##### Leader选举

##### 各服务器角色介绍

##### 会话激活

##### 请求处理

##### 数据与存储

在Zookeeper中，数据存储分为两部分：内存数据存粗与磁盘数据存储。Zookeeper的数据模型是一棵树，而从使用角度看，Zookeeper就像一个内存数据库一样。在这个内存数据库中，存储类整颗树的内容，包括所有的节点路径、节点数据及其ACL信息等，Zookeeper会定时将这个数据存储到磁盘上。






