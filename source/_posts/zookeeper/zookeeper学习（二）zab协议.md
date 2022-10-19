---
title: zookeeper学习（二）zab协议
tags:
  - zookeeper 
  - 中间件
  - zab
categories:
	- zookeeper
description: 
toc: true
toc_number: true
cover:  /img/post/98bf9a3c223248f2aa6ca0e66ec0df38.png
---

## 前言
之前介绍了zookeeper是分布式协调框架[添加链接描述](https://blog.csdn.net/weixin_42128977/article/details/127010824?spm=1001.2014.3001.5502)，一个分布式系统必然会存在一个问题：因为**分区容忍性**（partition tolerance）的存在，就必定要求我们需要在系统**可用性**（availability）和**数据一致性**（consistency）中做出权衡 。这就是著名的 CAP 定理。CAP理论中，P（分区容忍性）是必然要满足的，因为毕竟是分布式，不能把所有的应用全放到一个服务器里面，这样服务器是吃不消的。所以，只能从AP（可用性）和CP（一致性）中找平衡。后者就 ZooKeeper 的处理方式，它保证了**CP**（数据一致性）。
![在这里插入图片描述](https://img-blog.csdnimg.cn/40bf242f145646e98eed041b22198c40.png)
**一致性**
- 强一致性：时刻保证客户端看到的数据都是一致的。
- 最终一致性：允许存在中间状态，只要求经过一段时间后，数据最终是一致的。
- 弱一致性：允许存在部分数据不一致。

## ZooKeeper集群
### ZooKeeper集群角色
Zookeeper 集群模式一共有三种类型的角色
Leader :ZooKeeper集群同一时间只会有一个实际工作的Leader，它会发起并维护与各**Follwer及Observer**间的心跳。所有的写操作必须要通**过Leader**完成再由Leader将写操作广播给其它服务器。只要有超过半数节点（不包括 observeer 节点）写入成功，该写请求就会被提交（类 2PC 协议）。
Follower:一个ZooKeeper集群可能同时存在多个Follower，它会**响应Leader的心跳**。Follower可直接处理并返回客户端的读请求，同时会将**写请求转发给Leader处理**，并且负责在Leader处理写请求时对请求进行投票。
Observer：角色与 Follower 类似，但是无投票权。Zookeeper 需保证高可用和强一致性，为了支持更多的客户端，需要增加更多 Server；Server 增多，投票阶段延迟增大，影响性能；引入 Observer，Observer 不参与投票； Observers 接受客户端的连接，并将写请求转发给 leader 节点； 加入更多 Observer 节点，提高伸缩性，同时不影响吞吐率。

### 服务器状态
- LOOKING 不确定Leader状态。该状态下的服务器认为当前集群中没有Leader，会发起Leader选举。
- FOLLOWING 跟随者状态。表明当前服务器角色是Follower，并且它知道Leader是谁。
- LEADING 领导者状态。表明当前服务器角色是Leader，它会维护与Follower间的心跳。
- OBSERVING 观察者状态。表明当前服务器角色是Observer，与Folower唯一的不同在于不参与选举，也不参与集群写操作时的投票。


![在这里插入图片描述](https://img-blog.csdnimg.cn/8213bf242a63497798682cec13420726.png)


## ZAB协议
为了保证写操作的一致性与可用性，ZooKeeper专门设计了一种名为**原子广播**（ZAB）的支持**崩溃恢复**的一致性协议。基于该协议，ZooKeeper实现了一种主从模式的系统架构来保持集群中各个副本之间的数据一致性。

### ZAB特性
- **一致性保证**：一个事务 A 被server提交(committed)了，那么它最终一定会被所有的server提交
- **全局有序**：假设有A、B两个事务，有一台server先执行A再执行B，那么可以保证所有server上A始终都被在B之前执行
- **因果有序**：如果发送者在事务A提交之后再发送B,那么B必将在A之后执行
- 只要大多数节点启动，系统就行正常运行
- 当节点下线后重启，它必须保证能恢复到当前正在执行的事务

### ZAB的具体实现
- ZooKeeper由client、server两部分构成
- client可以在任何一个server节点上进行读操作
- client可以在任何一个server节点上发起写请求，非leader节点会把此次写请求转发到leader节点上。由leader节点执行
- ZooKeeper使用**改编的两阶段提交协议**来保证server节点的事务一致性

###  ZXID

![在这里插入图片描述](https://img-blog.csdnimg.cn/054dd4072e87466687a21036b60e3aa3.png)
ZooKeeper会为每一个事务生成一个唯一且递增长度为64位的ZXID,ZXID由两部分组成：低32位表示计数器(counter)和高32位的纪元号(epoch)。epoch为当前leader在成为leader的时候生成的，且保证会比前一个leader的epoch大

实际上当新的leader选举成功后，会拿到当前集群中最大的一个ZXID(因为数据最新)，并去除这个ZXID的epoch,并将此epoch进行加1操作，作为自己的epoch。
### myid
每个ZooKeeper服务器，都需要在数据文件夹下创建一个名为myid的文件，该文件包含整个ZooKeeper集群唯一的ID（整数）。例如，某ZooKeeper集群包含三台服务器，hostname分别为zoo1、zoo2和zoo3，其myid分别为1、2和3，则在配置文件中其ID与hostname必须一一对应，如下所示。在该配置文件中，server.后面的数据即为myid

> server.1=zoo1:2888:3888
>server.2=zoo2:2888:3888
>server.3=zoo3:2888:3888

### 历史队列
每一个follower节点都会有一个先进先出（FIFO)的队列用来存放收到的事务请求，保证执行事务的顺序。所以：
- **可靠提交**由**ZAB**的事务一致性协议保证
- **全局有序**由TCP协议保证
- **因果有序**由follower的历史队列(history queue)保证

## ZAB工作模式
ZAB 协议是为分布式协调服务 ZooKeeper 专门设计的一种支持**崩溃恢复**的**原子广播**协议。在 ZooKeeper 中，主要依赖 ZAB 协议来实现分布式数据一致性。ZAB协议两种模式：消息广播和崩溃恢复。根据ZAB协议，**所有的写操作都必须通过Leader完成，Leader写入本地日志后再复制到所有的Follower节点。一旦Leader节点无法工作，ZAB协议能够自动从Follower节点中重新选出一个合适的替代者，即新的Leader，该过程即为领导选举。该领导选举过程，是ZAB协议中最为重要和复杂的过程**。基于该协议，Zookeeper 实现了一种 主备模式 的系统架构来保持集群中各个副本之间数据一致性。


### 消息广播模式
当新的Leader出来了，同时已有**过半机器完成同步**之后，ZAB协议将退出恢复模式。进入**消息广播**模式。如果有一台遵守Zab协议的服务器加入集群，因为此时集群中已经存在一个Leader服务器在广播消息，那么该新加入的服务器自动进入恢复模式：找到Leader服务器，并且完成数据同步。同步完成后，作为新的Follower一起参与到消息广播流程中。
如下图所示
![在这里插入图片描述](https://img-blog.csdnimg.cn/98bf9a3c223248f2aa6ca0e66ec0df38.png)

由上图可见，通过Leader进行写操作，主要分为五步：
1. 客户端向zookeeper服务器发起写请求。
1.1 如果服务器是**Follower/Observer**，转发到**Leader**服务器上。
1.2 leader生成一个**新的事务Proposal并为这个事务生成一个唯一的ZXID**。
2. Leader将写请求以**Proposal（事务）**的形式发给所有**Follower**并等待ACK。
3.  Follower收到Proposal后，follower节点将收到的Proposal请求加入到**历史队列**(history queue)中，然后写入本地事务日志，根据自身情况(是否故障)，和zxid的有效性(是否大于本地最大zxid),回复是否同意该事务的ACK。
4. Leader得到过**半数的ACK(Leader对自己默认有一个ACK)** ，自己先添加事务。然后向所有的**Follower和Observer发送Commmit**,当follower收到commit请求时，会判断该事务的ZXID是不是比历史队列中的任何事务的ZXID都小，如果是则提交，如果不是则等待比它更小的事务的commit(保证顺序性)。
5. Leader将处理结果返回给客户端。

**总结：**
1. 所有客户端写入数据都是写入到**主进程Leader**中，然后，由 Leader 复制到**备份进程Follower**中。从而保证数据一致性。
2. 复制过程类似 **2PC**，Z**AB 只需要 Follower 有**一半以上**返回 Ack 信息就可以执行提交**，大大减小了同步阻塞。也提高了可用性。
3. Leader并不需要得到**Observer的ACK**。
4. Observer虽然无投票权，但仍须同步Leader的数据从而在处理读请求时可以返回尽可能新的数据。
5. Follower/Observer接受写请求以后，不能直接处理，而需要将写请求转发给Leader处理， 除了多了一步请求转发，其它流程与直接写Leader无任何区别。
6. Leader处理写请求是通过上面的**消息广播**模式，实质上所有的zkServer都要执行**写操作**，这样数据才会一致。
7. ZAB协议规定了如果一个事务在一台机器上被处理(commit)成功，那么应该在所有的机器上都被处理成功，哪怕机器出现故障崩溃。


		
		

### 崩溃恢复模式
**恢复模式大致可以分为四个阶段  选举、发现、同步、广播。**
- 当leader崩溃后，集群进入选举阶段，开始选举出潜在的新leader(一般为集群中拥有最大ZXID的节点)
- 进入发现阶段，follower与潜在的新leader进行沟通，如果发现超过法定人数的follower同意，则潜在的新leader将epoch加1，进入新的纪元。
- 集群间进行数据同步，保证集群中各个节点的事务一致
- 集群恢复到广播模式，开始接受客户端的写请求

**两个确保**
- ZAB协议需要确保已经在Leader提交的事务最终被所有服务器提交。
![在这里插入图片描述](https://img-blog.csdnimg.cn/4a2db044089f478a93e6b6063bef3085.png)
Leader (server2) 发送 commit 请求，他发送给了 server3，然后要发给 server1 的时候突然挂了。这个时候重新选举的时候我们如果把 server1 作为 Leader 的话，那么肯定会产生数据不一致性，因为 server3 肯定会提交刚刚 server2 发送的 commit 请求的提案，而 server1 根本没收到所以会丢弃。

- ZAB协议需要确保丢弃只在Leader服务器上被提出的事务。
![在这里插入图片描述](https://img-blog.csdnimg.cn/1e7b66ce91074f9aa85f7bbaadeb83d5.png)
 Leader (server2) 此时同意了提案N1，自身提交了这个事务并且要发送给所有 Follower 要 commit 的请求，却在这个时候挂了，此时肯定要重新进行 Leader 的选举，假如此时选 server1 为 Leader 。但是过了一会，这个 挂掉的 Leader 又重新恢复了 ，此时它肯定会作为 Follower 的身份进入集群中，需要注意的是刚刚 server2 已经同意提交了提案N1，但其他 server 并没有收到它的 commit 信息，所以其他 server 不可能再提交这个提案N1了，这样就会出现数据不一致性问题了，所以 该提案N1最终需要被抛弃掉 。

**脑裂问题**
脑裂问题：一个“大脑”被拆分了两个或多个“大脑”。通俗的说，就是比如当你的 cluster 里面有两个节点，它们都知道在这个 cluster 里需要选举出一个 master。那么当它们两之间的通信完全没有问题的时候，就会达成共识，选出其中一个作为 master。但是如果它们之间的通信出了问题，那么两个结点都会觉得现在没有 master，所以每个都把自己选举成 master，于是 cluster 里面就会有两个 master。
ZAB为解决脑裂问题，要求集群内的节点数量为2N+1, 当网络分裂后，始终有一个集群的节点数量过半数，而另一个集群节点数量小于N+1（即小于半数）, 因为选主需要过半数节点同意，所以任何情况下集群中都不可能出现大于一个leader的情况。
因此，有了过半机制，对于一个Zookeeper集群，要么没有Leader，要没只有1个Leader，这样就避免了脑裂问题。


**选举原则**
	Leader选举算法保证新选举出来的Leader服务器拥有集群中所有机器最高编号（ZXID最大）的事务Proposal，那么就能保证新的Leader 一定具有已提交的所有提案，更重要是，如果这么做，可以省去Leader服务器检查Proposal的提交和丢弃工作的这一步。
	
**触发条件**
当整个集群启动过程中，或者当 Leader 服务器出现网络中弄断、崩溃退出或重启等异常时，Zab协议就会 进入崩溃恢复模式，选举产生新的Leader。

**选票数据结构**
每个服务器在进行领导选举时，会发送如下关键信息：

> logicClock 每个服务器会维护一个自增的整数，名为logicClock，它表示这是该服务器发起的第多少轮投票 
> state 当前服务器的状态 
> self_id 当前服务器的myid 
> self_zxid 当前服务器上所保存的数据的最大zxid 
> vote_id被推举的服务器的myid 
> vote_zxid 被推举的服务器上所保存的数据的最大zxid



**投票流程**
1. **自增选举轮次**：ZooKeeper规定所有有效的投票都必须在同一轮次中。每个服务器在开始新一轮投票时，会先对自己维护的logicClock进行自增操作。
2. **初始化选票**：每个服务器在广播自己的选票前，会将自己的投票箱清空。该投票箱记录了所收到的选票。例：服务器2投票给服务器3，服务器3投票给服务器1，则服务器1的投票箱为(2, 3), (3, 1), (1, 1)。票箱中只会记录每一投票者的最后一票，如投票者更新自己的选票，则其它服务器收到该新选票后会在自己票箱中更新该服务器的选票。
3. **发送初始化选票**：每个服务器最开始都是通过广播把票投给自己。
4. **接收外部投票**：服务器会尝试从其它服务器获取投票，并记入自己的投票箱内。如果无法获取任何外部投票，则会确认自己是否与集群中其它服务器保持着有效连接。如果是，则再次发送自己的投票；如果否，则马上与之建立连接。
5. **判断选举轮次**：收到外部投票后，首先会根据投票信息中所包含的logicClock来进行不同处理：     
    5.1. 外部投票的logicClock大于自己的logicClock。说明该服务器的选举轮次落后于其它服务器的选举轮次，立即清空自己的投票箱并将自己的logicClock更新为收到的logicClock，然后再对比自己之前的投票与收到的投票以确定是否需要变更自己的投票，最终再次将自己的投票广播出去。
     5.2. 外部投票的logicClock小于自己的logicClock。当前服务器直接忽略该投票，继续处理下一个投票。
     5.3. 外部投票的logickClock与自己的相等。当时进行选票PK。
6. **选票PK**
选票PK是基于(self_id, self_zxid)与(vote_id, vote_zxid)的对比：
   6.1 外部投票的logicClock大于自己的logicClock，则将自己的logicClock及自己的选票的logicClock变更为收到的logicClock。
  6.2 若logicClock一致，则对比二者的vote_zxid，若外部投票的vote_zxid比较大，则将自己的票中的vote_zxid与vote_myid更新为收到的票中的vote_zxid与vote_myid并广播出去，另外将收到的票及自己更新后的票放入自己的票箱。如果票箱内已存在(self_myid, self_zxid)相同的选票，则直接覆盖。
  6.3 若二者vote_zxid一致，则比较二者的vote_myid，若外部投票的vote_myid比较大，则将自己的票中的vote_myid更新为收到的票中的vote_myid并广播出去，另外将收到的票及自己更新后的票放入自己的票箱.

7. **统计选票**
如果已经确定有过半服务器认可了自己的投票（可能是更新后的投票），则终止投票。否则继续接收其它服务器的投票。

8. **更新服务器状态**
投票终止后，服务器开始更新自身状态。若过半的票投给了自己，则将自己的服务器状态更新为LEADING，否则将自己的状态更新为FOLLOWING。


### 数据同步
当崩溃恢复之后，需要在正式工作之前（接收客户端请求），Leader 服务器首先确认事务是否都已经被过半的 Follwer 提交了，即是否完成了数据同步。目的是为了保持数据一致。
当 Follwer 服务器成功同步之后，Leader 会将这些服务器加入到可用服务器列表中。
实际上，Leader 服务器处理或丢弃事务都是依赖着 ZXID 的，那么这个 ZXID 如何生成呢？
答：在 ZAB 协议的事务编号 ZXID 设计中，ZXID 是一个 64 位的数字，其中低 32 位可以看作是一个简单的递增的计数器，针对客户端的每一个事务请求，Leader 都会产生一个新的事务 Proposal 并对该计数器进行 + 1 操作。而高 32 位则代表了 Leader 服务器上取出本地日志中最大事务 Proposal 的 ZXID，并从该 ZXID 中解析出对应的 epoch 值(leader选举周期)，当一轮新的选举结束后，会对这个值加一，并且事务id又从0开始自增。
![在这里插入图片描述](https://img-blog.csdnimg.cn/44ae907056ef45ba83235c47722337b2.png)

高 32 位代表了每代 Leader 的唯一性，低 32 代表了每代 Leader 中事务的唯一性。同时，也能让 Follwer 通过高 32 位识别不同的Leader。简化了数据恢复流程。
基于这样的策略：当 Follower 连接上 Leader 之后，Leader 服务器会根据自己服务器上最后被提交的 ZXID 和 Follower 上的 ZXID 进行比对，比对结果要么回滚，要么和 Leader 同步。
## 总结
-	ZAB协议是为分布式协调服务Zookeeper专门设计的一种支持崩溃回复的原子广播协议。
- 整个 Zookeeper 就是在这两个模式之间切换。 简而言之，当 Leader 服务可以正常使用，就进入消息广播模式，当 Leader 不可用时，则进入崩溃恢复模式。
- Zookeeper使用单一的主进程来接收并处理客户端的所有事务请求，并采用ZAB的原子广播协议，将服务器数据的状态变更以事务Proposal的形式广播到所有副本进程上去。ZAB协议的主备模型架构保证了同一时刻集群中只能够有一个主进程来广播服务器的状态变更，因此能够更好的处理客户端大量的并发请求。
- 消息广播，类似2PC，Leader和每个Follower之间维护一个FIFO队列。
-  Leader崩溃，或者Leader失去与过半Follower的联系，集群转入崩溃回复模式，回复后重新选举Leader。
- ZAB特性需要确保那些已经在Leader服务器上提交的事务最终被所有服务器都提交
- ZAB协议需要确保丢弃那些只在Leader服务器上被提出的事务
