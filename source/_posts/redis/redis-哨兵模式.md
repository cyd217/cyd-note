---
title: redis-哨兵模式
tags:
  - redis
  - 面试
categories:
	- redis
toc: true
toc_number: true
cover:  https://img-blog.csdnimg.cn/e80894f5cddd4a198ac1d028ec03f1e2.png
---

# 为什么要有哨兵机制？
在 Redis 的主从架构中，由于主从模式是读写分离的，如果主节点（master）挂了，那么将没有主节点来服务客户端的写操作请求，也没有主节点给从节点（slave）进行数据同步了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/c29962405bc546f2ace83b979e11067d.png)
这时如果要恢复服务的话，需要人工介入，选择一个**从节点**切换为**主节点**，然后让其他从节点指向新的主节点，同时还需要通知上游那些连接 Redis 主节点的客户端，将其配置中的主节点 IP 地址更新为**新主节点的 IP 地址**。
Redis 在 2.8 版本以后提供的**哨兵（Sentinel）机制**，它的作用是实现**主从节点故障转移**。它会监测主节点是否存活，如果发现主节点挂了，它就会选举一个从节点切换为主节点，并且把新主节点的相关信息通知给从节点和客户端。


# 作用和架构
![在这里插入图片描述](https://img-blog.csdnimg.cn/e80894f5cddd4a198ac1d028ec03f1e2.png)
解决的问题是：
哨兵：哨兵的核心功能是主节点的自动故障转移。在主从复制的基础上，哨兵实现了**自动化的故障恢复**。缺陷：**写操作无法负载均衡；存储能力受到单机的限制**。
- **监控**（Monitoring）：哨兵会不断地检查主节点和从节点是否运作正常。
- **自动故障转移**（Automatic failover）：当主节点不能正常工作时，哨兵会开始自动故障转移操作，它会将失效主节点的其中一个从节点升级为新的主节点，并让其他从节点改为复制新的主节点。
- **配置提供者**（Configuration provider）：客户端在初始化时，通过连接哨兵来获得当前Redis服务的主节点地址。	
- **通知**（Notification）：哨兵可以将故障转移的结果发送给客户端。

# 部署主从
主从搭建
```c
#redis-6379.conf
port 6379
daemonize yes
logfile "6379.log"
dbfilename "dump-6379.rdb"
redis-server redis-6379.conf
 
#redis-6380.conf
port 6380
daemonize yes
logfile "6380.log"
dbfilename "dump-6380.rdb"
slaveof 127.0.0.1 6379
redis-server redis-6380.conf
 
#redis-6381.conf
port 6381
daemonize yes
logfile "6381.log"
dbfilename "dump-6381.rdb"
slaveof 127.0.0.1 6379
redis-server redis-6381.conf
```
redis哨兵架构搭建步骤

```c
#sentinel‐26379.conf
port 26379
daemonize yes
pidfile "/var/run/redis‐sentinel‐26379.pid"
logfile "26379.log"
# sentinel monitor <master‐redis‐name> <master‐redis‐ip> <master‐redis‐port> <quorum>
# quorum是一个数字，指明当有多少个sentinel认为一个master失效时(值一般为：sentinel总数/2 +1)，master才算真正失效
sentinel monitor mymaster 192.168.0.60 6379 2 #mymaster名字随便取，客户端访问时会用到
src/redis‐sentinel sentinel‐26379.conf # 启动sentinel哨兵实例

src/redis‐cli ‐p 26379 查看sentinel的info信息
127.0.0.1:26379>info #可以看到Sentinel的info里已经识别出了redis的主从

同理在配置2个哨兵
```

查看下如下配置文件sentinel-26379.conf，如下所示：

```c
cat sentinel-26379.conf
......
# sentinel集群都启动完毕后，会将哨兵集群的元数据信息写入所有sentinel的配置文件里去(追加在文件的
最下面)
1 sentinel known‐replica mymaster 127.0.0.1 6380 #代表redis主节点的从节点信息
2 sentinel known‐replica mymaster 127.0.0.1 6381 #代表redis主节点的从节点信息
3 sentinel known‐sentinel mymaster 127.0.0.1 26380 52d0a5d70c1f90475b4fc03b6ce7c3c569
35760f #代表感知到的其它哨兵节点
4 sentinel known‐sentinel mymaster 127.0.0.1 26381 e9f530d3882f8043f76ebb8e1686438ba8
bd5ca6 #代表感知到的其它哨兵节点
sentinel current-epoch 0
```
`known-slave`和`known-sentinel`显示哨兵已经发现了从节点和其他哨兵；带有`epoch`的参数与配置纪元有关

# Redis哨兵的高可用
## 故障转移
原理：当主节点出现故障时，由`Redis Sentinel`自动完成故障发现和转移，并通知应用方，实现高可用性。
![在这里插入图片描述](https://img-blog.csdnimg.cn/14ad3b6a22e54c869e2e0cb478d6add5.png)
- 哨兵机制建立了多个哨兵节点(进程)，共同监控数据节点的运行状况。
- 同时**哨兵节点之间也互相通信，交换对主从节点的监控**状况。
- 每隔1秒每个哨兵会向整个集群：Master主服务器+Slave从服务器+其他Sentinel（哨兵）进程，发送一次ping命令做一次心跳检测。

这个就是哨兵用来判断节点是否正常的重要依据，涉及两个新的概念：**主观下线和客观下线**。
1) **主观下线**
适用于**主服务器和从服务器**。如果在规定的时间内(配置参数：`down-after-milliseconds`)，`Sentinel `节点没有收到目标服务器的有效回复，则判定该服务器为“主观下线”。比如 Sentinel1 向主服务发送了PING命令，在规定时间内没收到主服务器PONG回复，则 Sentinel1 判定主服务器为“主观下线”。
2) **客观下线**
只适用于**主服务器**。 Sentinel1 发现主服务器出现了故障，它会通过相应的命令，询问其它 Sentinel 节点对主服务器的状态判断。如果超过半数以上的  Sentinel 节点认为主服务器 down 掉，则 Sentinel1 节点判定主服务为“客观下线”。**客观下线是主节点才有的概念；如果从节点和哨兵节点发生故障，被哨兵主观下线后，不会再有后续的客观下线和故障转移操作。**
（4）选举领导者哨兵节点：当主节点被判断客观下线以后，各个哨兵节点会进行协商，选举出一个领导者哨兵节点，并由该领导者节点对其进行故障转移操作。
监视该主节点的所有哨兵都有可能被选为领导者，选举使用的算法是Raft算法；**Raft算法的基本思路是先到先得**：即在一轮选举中，哨兵A向B发送成为领导者的申请，如果B没有同意过其他哨兵，则会同意A成为领导者。选举的具体过程这里不做详细描述，一般来说，**哨兵选择的过程很快，谁先完成客观下线，一般就能成为领导者**。

# 主从故障转移的过程是怎样的？
## 新主节点选举
首先，Sentinel Leader会按照以下条件剔除从节点：

- 主观宕机（SDOWN）或与处于断线状态的从节点；
- 最近5秒内未回复过Sentinel Leader `INFO`命令的从节点；
- 与主节点断开连接超过10倍`down-after-milliseconds`(主从节点断连的最大连接超时时间)的从节点；

筛选过后，剩下的从节点都是数据比较新、与Sentinel Leader通信正常的，可以保证故障转移后最小的数据丢失。
然后，按照以下规则选择新的主节点：
- 选择`replica-priority`最低的节点(优先级最高)。如果存在相同，则继续；
- 选择`复制偏移量最大`的的从节点。如果存在相同，则继续；
- 选择`runId`最小的从节点；

## 将从节点指向新主节点
配置新主节点&新主节点角色提升
更新主从状态：通过`slaveof no one`命令，让选出来的从节点成为主节点；并通过`slaveof`命令让其他节点成为其从节点。
## 通知客户的主节点已更换
客户端和哨兵建立连接后，**客户端会订阅哨兵提供的频道**。主从切换完成后，哨兵就会向` +switch-master `频道发布新主节点的` IP 地址和端口`的消息，这个时候客户端就可以收到这条信息，然后用这里面的`新主节点的 IP 地址和端口`进行通信了。


## 将旧主节点变为从节点
继续监视旧主节点，当旧主节点重新上线时，哨兵集群就会向它发送 `SLAVEOF `命令，让它成为新主节点的从节点。

# 总结
- `sentinel`会为每个被监视的主服务器创建相应的实列，并创建连向主服务器的**命令连接(向主服务器发送命令)和订阅连接（接受指定频道消息）**。
- `sentinel`通过向`master`发送`Info`命令，获取`master`和它下面所有`slave`的当前信息。并为这些从服务器创建相应的实例结构，连向从服务器的**命令连接**和**订阅连接**。
- 每隔10秒向`master`和从服务器发送`info`命令，当主服务器下线时或者`sentinel`对主服务器进行故障迁移操作时，`sentinel`向从服务器发送`info`命令的时间为1秒1次。
- `sentinel`和`sentinel`之间只会有命令连接。`sentinel`与主从服务器创建命令连接和订阅连接。每个`sentinel`通过`__sentinel__:hello`频道发送消息来向其他的sentinel宣告存在。
- `sentinel`每隔1秒向所有服务器发送`ping`命令，如果某台服务器在配置的响应时间内连续返回无效回复，将会被标记为主观下线。

**注意**
**sentinel会为每个被监视的主服务器创建相应的实列，并创建连向主服务器的命令连接和订阅连接，为什么会有2个实例？**
在`redis`的订阅功能中，被发送的消息都不会保存在`redis`服务器里面，如果在信息发送时，想要收到信息的客户端不在，那么就会数据丢失。为了不丢失任何信息，必须专门建立一个频道来接受。另一方面，除了订阅连接，`sentinel`还必须向主服务器发送命令，以此来与主服务器通信，所以还必须有命令连接。因为`sentinel`有多个实例创建的网络连接，所以`sentinel`使用的异步连接。

**sentinel之间不会建立订阅连接**
因为`sentinel`在连接主服务器和从服务器时，会同时创建命令连接和订阅连接，但是在连接其他的`sentinel`时，只会创建命令连接，而不是订阅连接。这是因为`sentinel`需要通过接受主服务器或者从服务器发送的频道信息来发现未知的`sentinel`，所以才需要连接订阅连接。而相互已知的`sentinel`只要使用命令连接就可以了。