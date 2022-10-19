---
title: zookeeper学习（一） 入门篇
tags:
  - zookeeper 
  - 中间件
  - 分布式
categories:
	- zookeeper
description: 
toc: true
toc_number: true
cover:  /img/post/bb04ee5adabd4a4f9d79d54001d17bdd.png
---


## ZooKeeper简介
ZooKeeper 是一个开源的**分布式协调框架**，它的定位是为**分布式应用提供一致性**服务ZooKeeper 会封装好复杂易出错的关键服务，将高效、稳定、易用的服务提供给用户使用。

**ZooKeeper = 文件系统 + 监听通知机制。**（重点）

### 1.zookeeper文件系统
![在这里插入图片描述](https://img-blog.csdnimg.cn/bb04ee5adabd4a4f9d79d54001d17bdd.png)
Zookeeper 提供一个**多层级**的节点命名空间（节点称为 znode）。这些节点都可以设置**关联的数据**。**每个znode都可以通过其路径唯一标识**。
Zookeeper 为了保证**高吞吐和低延迟**，在**内存**中维护了这个树状的目录结构，这种特性使得 Zookeeper 不能用于存放大量的数据，每个节点的存放**数据上限为1M**。

#### 1.1节点类型
- 持久化目录节点：客户端与zookeeper断开连接后，该节点依旧存在。
- 持久化顺序编号目录节点：客户端与zookeeper断开连接后，该节点依旧存在，只是Zookeeper给该节点名称进行顺序编号。
- 临时目录节点 ：客户端与zookeeper断开连接后，该节点被删除。
- 临时顺序编号目录节点 ：客户端与zookeeper断开连接后，该节点被删除，只是Zookeeper给该节点名称进行顺序编号。
- 容器 节点：3.5.3 版本新增，如果Container节点下面没有子节点，则Container节点在未来会被Zookeeper自动清除,定时任务默认60s 检查一次
- TTL 节点： 默认禁用，只能通过系统配置  zookeeper.extendedTypesEnabled=true  开启，不稳定。

#### 1.2监听通知机制
Watcher 监听机制是 Zookeeper 中非常重要的特性，我们基于 Zookeeper上创建的**节点**，可以对这些节点绑定监听事件，比如可以监听节点**数据变更、节点删除、子节点状态变更**等事件，通过这个事件机制，可以基于 Zookeeper 实现**分布式锁、集群管理**等多种功能。**客户端向服务端注册指定的 watcher ，当服务端符合了 watcher 的某些事件或要求则会 向客户端发送事件通知 ，客户端收到通知后找到自己定义的 Watcher 然后 执行相应的回调方法** 。

**ZooKeeper 的 Watcher 机制，总的来说可以分为三个过程：**

客户端注册 Watcher，注册 watcher 有 3 种方式，getData、exists、getChildren。
服务器处理 Watcher 。
客户端回调 Watcher 客户端。
![在这里插入图片描述](https://img-blog.csdnimg.cn/66a697c4ce8d4ddc8b0f4a98e8515288.png)


**Watcher 特性总结：**
- 一次性：无论是服务端还是客户端，一旦一个 Watcher 被 触 发 ，Zookeeper 都会将其从相应的存储中移除。这样的设计有效的减轻了服务端的压力。
- 客户端串行执行：客户端 Watcher 回调的过程是一个串行同步的过程。
- 轻量：
    - Watcher 通知非常简单，只会告诉客户端发生了事件，而不会说明事件的具体内容。
    - 客户端向服务端注册 Watcher 的时候，并不会把客户端真实的 Watcher 对象实体传递到服务端，仅仅是在客户端请求中使用 boolean 类型属性进行了标记。
- watcher event 异步发送 watcher 的通知事件从 server 发送到 client 是异步的，这就存在一个问题，不同的客户端和服务器之间通过 socket 进行通信，由于网络延迟或其他因素导致客户端在不通的时刻监听到事件，由于 Zookeeper 本身提供了 ordering guarantee，即客户端监听事件后，才会感知它所监视 znode发生了变化。所以我们使用 Zookeeper 不能期望能够监控到节点每次的变化。Zookeeper 只能保证最终的一致性，而无法保证强一致性。
- 注册 watcher getData、exists、getChildren
- 触发 watcher create、delete、setData
- 当一个客户端连接到一个新的服务器上时，watch 将会被以任意会话事件触发。当与一个服务器失去连接的时候，是无法接收到 watch 的。而当 client 重新连接时，如果需要的话，所有先前注册过的 watch，都会被重新注册。通常这是完全透明的。只有在一个特殊情况下，watch 可能会丢失：对于一个未创建的 znode的 exist watch，如果在客户端断开连接期间被创建了，并且随后在客户端连接上之前又删除了，这种情况下，这个 watch 事件可能会被丢失。

**Zookeeper事件类型：**
            None: 连接建立事件
            NodeCreated： 节点创建
            NodeDeleted： 节点删除
            NodeDataChanged：节点数据变化
            NodeChildrenChanged：子节点列表变化
            DataWatchRemoved：节点监听被移除
            ChildWatchRemoved：子节点监听被移除

**Zookeeper 对节点的 watch 监听通知是永久的吗？为什么不是永久的?**
不是。如果服务端变动频繁，而监听的客户端很多情况下，每次变动都要通知到所有的客户端，给网络和服务器造成很大压力。在实际应用中，很多情况下，我们的客户端不需要知道服务端的每一次变动，我只要最新的数据即可。

### 2.zk指令
- 创建zookeeper 节点命令

>  create [‐s] [‐e] [‐c] [‐t ttl] path [data] [acl]
>    -s: 顺序节点   -e: 临时节点   -c: 容器节点
-t:  可以给节点添加过期时间，默认禁用，需要通过系统参数启用
//举例
> [zk: localhost:2181(CONNECTED) 10] create /data-test data1   创建一个持久化节点
>  Created /data-test
> 
- 查看节点：
>   get [-s] [-w ]  path 
>   -s 查看节点状态
>    -w 
> [zk: localhost:2181(CONNECTED) 1] get /data-test 
> data1

- 修改节点数据：

> set [-v] path [data]
> -v 当前cversion版本号  乐观锁机制
> [zk: localhost:2181(CONNECTED) 2] set  /data-test test

- 查看节点状态信息：

> [zk: localhost:2181(CONNECTED) 1] stat /data-test
cZxid = 0x8   创建znode的事务ID（Zxid的值）。
ctime = Fri Sep 23 10:23:59 UTC 2022  znode创建时间
mZxid = 0xb  最后修改znode的事务ID。
mtime = Mon Sep 26 08:56:12 UTC 2022  znode最近修改时间
pZxid = 0x8 最后添加或删除子节点的事务ID（子节点列表发生变化才会发生改变）。
cversion = 0   znode的子节点结果集版本（一个节点的子节点增加、删除都会影响这个版本）。
dataVersion = 1   znode的当前数据版本。
aclVersion = 0   表示对此znode的acl版本。
ephemeralOwner = 0x0  znode是临时znode时，表示znode所有者的 session ID。 如果
znode不是临时znode，则该字段设置为零。 
dataLength = 4  znode数据字段的长度
numChildren = 0  znode的子znode的数量

- 查看子节点信息

>ls [-R][-w] /
>-R 子目录
>-w 监听
> [zk: localhost:2181(CONNECTED) 2] ls /
[data-test, runoob0000000000, zookeeper]

- 注册监听的同时获取数据
> get ‐w /path // 注册监听的同时获取数据
> [zk: localhost:2181(CONNECTED) 5] get -w /data-test
test
[zk: localhost:2181(CONNECTED) 6] set /data-test test2
WATCHER::
WatchedEvent state:SyncConnected type:NodeDataChanged path:/data-test
>监听根目录以其子目录变化
>[zk: localhost:2181(CONNECTED) 7] ls ‐R ‐w /
[data-test, runoob0000000000, zookeeper]
[zk: localhost:2181(CONNECTED) 8] delete data-test
Path must start with / character
[zk: localhost:2181(CONNECTED) 9] delete /data-test
WATCHER::
WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/

### 3.Zookeeper特性
上面我们说到ZooKeeper 是一个的**分布式协调服务框架**，为分布式系统提供一致性服务。
什么是分布式，什么是集群，为什么需要协调各服务？
将一套系统拆分成不同子系统部署在不同服务器上（这叫分布式），大的问题拆分为多个小的问题，并分别解决，最终协同合作，分布式的主要工作是**分解任务，将职能拆解**。
 然后部署多个相同的子系统在不同的服务器上（这叫集群）， 集群主要是简单加机器解决问题，对于问题本身不做任何分解，集群主要的使用场景是为了**分担请求的压力**。
大白话：
分布式 ：多个人在一起作不同的事 。
集群：多个人在一起作同样的事 。

**集群**
![集群](https://img-blog.csdnimg.cn/f590f10729c049568a85d208b797696d.png)
**分布式**
![在这里插入图片描述](https://img-blog.csdnimg.cn/cb0e8dfaaf9144c3b1f33c1994a21887.png)
对于集群来说，多加几台服务器就行（当然还得解决session共享，负载均衡等问题），而对于分布式来说，你首先需要将业务进行拆分，然后再加服务器，同时还要去解决分布式带来的一系列问题。比如各个分布式组件如何协调起来，如何减少各个系统之间的耦合度，如何处理**分布式事务**，如何去配置整个分布式系统，如何解决各分布式**子系统的数据不一致问题**等等。ZooKeeper 主要就是解决这些问题的。

#### 3.1 Zookeeper保证了如下分布式一致性特性:
- 顺序一致性：同一个客户端发起的事务请求，最终将会严格的按照其发起的顺序被应用到Zookeeper中，对于客户端的每个请求，Zookeeper都会分配一个全局唯一的递增编号，这个编号反应了所有事务操作的先后顺序。
- 原子性：所有事务的请求处理在整个集群中所有的机器上的应用情况是一致的。对于一个事务，要么集群中所有机器都成功应用，要么都没有应用，不会出现部分应用的情况。
- 单一视图：无论客户端连接哪个Zookeeper服务器，看到服务端的数据模型都一致的。
- 可靠性：一旦事务完成，就会被保留下来，除非另一个事务修改。
- 实时性：Zookeeper仅仅保证在一定的时间段内，客户端最终一定能从服务器端读取到最新数据。
- 有序性是zookeeper中非常重要的一个特性，所有的更新都是全局有序的，每个更新都有一个 唯一的时间戳，这个时间戳称为zxid。而读请求只会相对于更新有序，也就是读请求的返回结果中会带有这个zookeeper最新的zxid。

#### 3.2 ZooKeeper 操作特性
- Znode:包含ACL权限控制、修改/访问时间、最后一次操作的事务Id(zxid)等等
- 所有运行时数据存储在内存中，在内存中维护这么一颗树
- 每次对Znode节点修改都是保证顺序和原子性的操作。写操作是原子性操作
- 生命周期：当客户端会话结束的时候，是否清理掉这个会话创建的节点，持久-不清理，临时-清理
- 会话节点：每一个会话，创建单独的节点。


 #### 3.3 ZooKeeper  数据持久化
由上面可知，zookeeper数据的组织形式为一个**类似文件系统的数据结构**，而这些数据都是存储在内存中的，所以我们可以认为，Zookeeper是一个基于**内存的小型数据库**
	
**数据类型**：内存数据库
			
**事务日志**：每次数据写操作完成会保存到内存数据库的同时，会记录到事务日志中 ，频繁进行磁盘IO操作，事务日志的不断追加写操作会触发底层磁盘IO为文件开辟新的磁盘块，即磁盘Seek。因此，为了提升磁盘IO的效率，Zookeeper在创建事务日志文件的时候就进行文件空间的预分配- 即在创建文件的时候，就向操作系统申请一块大一点的磁盘块。
**快照日志**：数据快照用于记录Zookeeper服务器上某一时刻的全量数据，并将其写入到指定的磁盘文件中。可以通过配置**snapCount**配置每间隔事务请求个数，生成快照，数据存储在dataDir 指定的目录中。
**有了事务日志，为啥还要快照数据。**：快照数据主要时为了**快速恢复**， 事务日志文件是**每次事务请求都会进行追加的操作**，而快照是达到某种设定条件下的内存全量数据。所以通常快照数据是**反应当时内存数据的状态**。事务日志是**更全面的数据，所以恢复数据的时候，可以先恢复快照数据，再通过增量恢复事务日志中的数据即可**。
**完整流程**：
1.接收写操作，ZooKeeper集群中的每个服务器节点每次接收到写操作请求时，都会先将这次请求发送给leader。
2.协调写操作，leader将这次写操作转换为带有状态的事务，然后leader会对这次写操作广播出去以便进行协调。
3.记录数据，当协调通过(大多数节点允许这次写)后，leader通知所有的服务器节点，让它们将这次写操作应用到内存数据库中，并将其记录到事务日志中
4.记录快照数据，当事务日志记录的次数达到一定数量后(默认10W次)，将内存数据库序列化一次，持久化保存到磁盘上，序列化后的文件为快照文件
5.基于事务日志和快照，就可以让任意节点恢复到任意时间点(只要没有清理事务日志和快照)

#### 3.4 Zookeeper会话（Session）
Session 可以看作是 ZooKeeper 服务器与客户端的之间的一个 **TCP 长连接**，客户端与服务端之间的任何交互操作都和Session 息息相关，其中包含zookeeper的**临时节点的生命周期、客户端请求执行以及Watcher通知机制**等。

**session会话状态有：**
- connecting：**连接中**，session 一旦建立，状态就是 connecting 状态，时间很短。
- connected：**已连接**，连接成功之后的状态。
- closed：**已关闭**，发生在 session 过期，一般由于网络故障客户端重连失败，服务器宕机或者客户端主动断开。

**作用：**
- tcp长连接
- 心跳检测
- 发送请求
- 接受watch事件


### 4.总结
简介了介绍zookeeper数据模型，监听机制，持久化，指令以及一些特性。
- zk是基于**内存**进行读写操作的，有时候会进行消息广播，因此不建议在节点存取容量比较大的数据。
- dataDir（快照）目录、dataLogDir（事务日志）两个目录会随着时间推移变得庞大，容易造成硬盘满了。
- 类似文件系统，一直存放在内存中，由一系列Znode组成，Znode之间的层级关系就像文件系统的目录结构一样。
- 监听节点**数据变更、节点删除、子节点状态变更**等事件，而且监听是一次性的。