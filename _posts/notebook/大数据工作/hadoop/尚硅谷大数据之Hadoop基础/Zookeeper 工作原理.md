  
Zookeeper 简介、基本概念和工作原理 - 内方外圆 - CSDN 博客

ZooKeeper 是一个分布式的，开放源码的分布式应用程序协调服务，它包含一个简单的原语集，分布式应用程序可以基于它实现同步服务，配置维护和命名服务等。Zookeeper 是 hadoop 的一个子项目，其发展历程无需赘述。在分布式应用中，由于工程师不能很好地使用锁机制，以及基于消息的协调机制不适合在某些应用中使用，因此需要有一种可靠的、可扩展的、分布式的、可配置的协调机制来统一系统的状态。Zookeeper 的目的就在于此。本文简单分析 zookeeper 的工作原理，对于如何使用 zookeeper 不是本文讨论的重点。

# 1 Zookeeper 的基本概念

## 1.1 角色

Zookeeper 中的角色主要有以下三类，如下表所示：

![](http://static.oschina.net/uploads/img/201308/08171344_cqXs.jpg)

系统模型如图所示：

![](http://static.oschina.net/uploads/img/201308/08171345_l5K3.jpg)

## 1.2 设计目的

1. 最终一致性：client 不论连接到哪个 Server，展示给它都是同一个视图，这是 zookeeper 最重要的性能。

2 . 可靠性：具有简单、健壮、良好的性能，如果消息 m 被到一台服务器接受，那么它将被所有的服务器接受。

3 . 实时性：Zookeeper 保证客户端将在一个时间间隔范围内获得服务器的更新信息，或者服务器失效的信息。但由于网络延时等原因，Zookeeper 不能保证两个客户端能同时得到刚更新的数据，如果需要最新数据，应该在读数据之前调用 sync() 接口。

4 . 等待无关（wait-free）：慢的或者失效的 client 不得干预快速的 client 的请求，使得每个 client 都能有效的等待。

5. 原子性：更新只能成功或者失败，没有中间状态。

6 . 顺序性：包括全局有序和偏序两种：全局有序是指如果在一台服务器上消息 a 在消息 b 前发布，则在所有 Server 上消息 a 都将在消息 b 前被发布；偏序是指如果一个消息 b 在消息 a 后被同一个发送者发布，a 必将排在 b 前面。

# 2 ZooKeeper 的工作原理

Zookeeper 的核心是原子广播，这个机制保证了各个 Server 之间的同步。实现这个机制的协议叫做 Zab 协议。Zab 协议有两种模式，它们分别是恢复模式（选主）和广播模式（同步）。当服务启动或者在领导者崩溃后，Zab 就进入了恢复模式，当领导者被选举出来，且大多数 Server 完成了和 leader 的状态同步以后，恢复模式就结束了。状态同步保证了 leader 和 Server 具有相同的系统状态。

为了保证事务的顺序一致性，zookeeper 采用了递增的事务 id 号（zxid）来标识事务。所有的提议（proposal）都在被提出的时候加上了 zxid。实现中 zxid 是一个 64 位的数字，它高 32 位是 epoch 用来标识 leader 关系是否改变，每次一个 leader 被选出来，它都会有一个新的 epoch，标识当前属于那个 leader 的统治时期。低 32 位用于递增计数。

每个 Server 在工作过程中有三种状态：

-   LOOKING：当前 Server 不知道 leader 是谁，正在搜寻
    
-   LEADING：当前 Server 即为选举出来的 leader
    
-   FOLLOWING：leader 已经选举出来，当前 Server 与之同步
    

## 2.1 选主流程

当 leader 崩溃或者 leader 失去大多数的 follower，这时候 zk 进入恢复模式，恢复模式需要重新选举出一个新的 leader，让所有的 Server 都恢复到一个正确的状态。Zk 的选举算法有两种：一种是基于 basic paxos 实现的，另外一种是基于 fast paxos 算法实现的。系统默认的选举算法为 fast paxos。先介绍 basic paxos 流程：

1.  1 . 选举线程由当前 Server 发起选举的线程担任，其主要功能是对投票结果进行统计，并选出推荐的 Server；
    
2.  2 . 选举线程首先向所有 Server 发起一次询问 (包括自己)；
    
3.  3 . 选举线程收到回复后，验证是否是自己发起的询问 (验证 zxid 是否一致)，然后获取对方的 id(myid)，并存储到当前询问对象列表中，最后获取对方提议的 leader 相关信息 (id,zxid)，并将这些信息存储到当次选举的投票记录表中；
    
4.  4. 收到所有 Server 回复以后，就计算出 zxid 最大的那个 Server，并将这个 Server 相关信息设置成下一次要投票的 Server；
    
5.  5. 线程将当前 zxid 最大的 Server 设置为当前 Server 要推荐的 Leader，如果此时获胜的 Server 获得 n/2 + 1 的 Server 票数， 设置当前推荐的 leader 为获胜的 Server，将根据获胜的 Server 相关信息设置自己的状态，否则，继续这个过程，直到 leader 被选举出来。
    

通过流程分析我们可以得出：要使 Leader 获得多数 Server 的支持，则 Server 总数必须是奇数 2n+1，且存活的 Server 的数目不得少于 n+1.

每个 Server 启动后都会重复以上流程。在恢复模式下，如果是刚从崩溃状态恢复的或者刚启动的 server 还会从磁盘快照中恢复数据和会话信息，zk 会记录事务日志并定期进行快照，方便在恢复时进行状态恢复。选主的具体流程图如下所示：

![](http://static.oschina.net/uploads/img/201308/08171345_J3LF.jpg)

fast paxos 流程是在选举过程中，某 Server 首先向所有 Server 提议自己要成为 leader，当其它 Server 收到提议以后，解决 epoch 和 zxid 的冲突，并接受对方的提议，然后向对方发送接受提议完成的消息，重复这个流程，最后一定能选举出 Leader。其流程图如下所示：

![](http://static.oschina.net/uploads/img/201308/08171346_zLlp.jpg)

## 2.2 同步流程

选完 leader 以后，zk 就进入状态同步过程。

1.  1. leader 等待 server 连接；
    
2.  2 .Follower 连接 leader，将最大的 zxid 发送给 leader；
    
3.  3 .Leader 根据 follower 的 zxid 确定同步点；
    
4.  4 . 完成同步后通知 follower 已经成为 uptodate 状态；
    
5.  5 .Follower 收到 uptodate 消息后，又可以重新接受 client 的请求进行服务了。
    

流程图如下所示：

![](http://static.oschina.net/uploads/img/201308/08171346_oExa.jpg)

## 2.3 工作流程

### 2.3.1 Leader 工作流程

Leader 主要有三个功能：

1.  1 . 恢复数据；
    
2.  2 . 维持与 Learner 的心跳，接收 Learner 请求并判断 Learner 的请求消息类型；
    
3.  3 .Learner 的消息类型主要有 PING 消息、REQUEST 消息、ACK 消息、REVALIDATE 消息，根据不同的消息类型，进行不同的处理。
    

PING 消息是指 Learner 的心跳信息；REQUEST 消息是 Follower 发送的提议信息，包括写请求及同步请求；ACK 消息是 Follower 的对提议的回复，超过半数的 Follower 通过，则 commit 该提议；REVALIDATE 消息是用来延长 SESSION 有效时间。  
Leader 的工作流程简图如下所示，在实际实现中，流程要比下图复杂得多，启动了三个线程来实现功能。[](http://static.oschina.net/uploads/img/201308/08171346_87iA.jpg)

![](http://static.oschina.net/uploads/img/201308/08171346_87iA.jpg)

### 2.3.2 Follower 工作流程

Follower 主要有四个功能：

1.  1. 向 Leader 发送请求（PING 消息、REQUEST 消息、ACK 消息、REVALIDATE 消息）；
    
2.  2 . 接收 Leader 消息并进行处理；
    
3.  3 . 接收 Client 的请求，如果为写请求，发送给 Leader 进行投票；
    
4.  4 . 返回 Client 结果。
    

Follower 的消息循环处理如下几种来自 Leader 的消息：

1.  1 .PING  消息： 心跳消息；
    
2.  2 .PROPOSAL  消息：Leader 发起的提案，要求 Follower 投票；
    
3.  3 .COMMIT  消息：服务器端最新一次提案的信息；
    
4.  4 .UPTODATE  消息：表明同步完成；
    
5.  5 .REVALIDATE  消息：根据 Leader 的 REVALIDATE 结果，关闭待 revalidate 的 session 还是允许其接受消息；
    
6.  6 .SYNC  消息：返回 SYNC 结果到客户端，这个消息最初由客户端发起，用来强制得到最新的更新。
    

Follower 的工作流程简图如下所示，在实际实现中，Follower 是通过 5 个线程来实现功能的。

![](http://static.oschina.net/uploads/img/201308/08171346_slOJ.jpg)

对于 observer 的流程不再叙述，observer 流程和 Follower 的唯一不同的地方就是 observer 不会参加 leader 发起的投票。

  
主流应用场景：

Zookeeper 的主流应用场景实现思路（除去官方示例）  
  
(1) 配置管理集中式的配置管理在应用集群中是非常常见的，一般商业公司内部都会实现一套集中的配置管理中心，应对不同的应用集群对于共享各自配置的需求，并且在配置变更时能够通知到集群中的每一个机器。  
  
Zookeeper 很容易实现这种集中式的配置管理，比如将 APP1 的所有配置配置到 /APP1 znode 下，APP1 所有机器一启动就对 /APP1 这个节点进行监控(zk.exist("/APP1",true)), 并且实现回调方法 Watcher，那么在 zookeeper 上 /APP1 znode 节点下数据发生变化的时候，每个机器都会收到通知，Watcher 方法将会被执行，那么应用再取下数据即可 (zk.getData("/APP1",false,null));  
  
以上这个例子只是简单的粗颗粒度配置监控，细颗粒度的数据可以进行分层级监控，这一切都是可以设计和控制的。[](http://static.oschina.net/uploads/img/201312/20143856_Yv5J.jpg)

![](http://static.oschina.net/uploads/img/201312/20143856_Yv5J.jpg)

  
(2) 集群管理  
应用集群中，我们常常需要让每一个机器知道集群中（或依赖的其他某一个集群）哪些机器是活着的，并且在集群机器因为宕机，网络断链等原因能够不在人工介入的情况下迅速通知到每一个机器。  
  
Zookeeper 同样很容易实现这个功能，比如我在 zookeeper 服务器端有一个 znode 叫 /APP1SERVERS, 那么集群中每一个机器启动的时候都去这个节点下创建一个 EPHEMERAL 类型的节点，比如 server1 创建 /APP1SERVERS/SERVER1(可以使用 ip, 保证不重复)，server2 创建/APP1SERVERS/SERVER2，然后 SERVER1 和 SERVER2 都 watch /APP1SERVERS 这个父节点，那么也就是这个父节点下数据或者子节点变化都会通知对该节点进行 watch 的客户端。因为 EPHEMERAL 类型节点有一个很重要的特性，就是客户端和服务器端连接断掉或者 session 过期就会使节点消失，那么在某一个机器挂掉或者断链的时候，其对应的节点就会消失，然后集群中所有对 /APP1SERVERS 进行 watch 的客户端都会收到通知，然后取得最新列表即可。  
另外有一个应用场景就是集群选 master, 一旦 master 挂掉能够马上能从 slave 中选出一个 master, 实现步骤和前者一样，只是机器在启动的时候在APP1SERVERS 创建的节点类型变为 EPHEMERAL_SEQUENTIAL 类型，这样每个节点会自动被编号  
我们默认规定编号最小的为 master, 所以当我们对 /APP1SERVERS 节点做监控的时候，得到服务器列表，只要所有集群机器逻辑认为最小编号节点为master，那么 master 就被选出，而这个 master 宕机的时候，相应的 znode 会消失，然后新的服务器列表就被推送到客户端，然后每个节点逻辑认为最小编号节点为 master，这样就做到动态 master 选举。[](http://static.oschina.net/uploads/img/201312/20143856_6ZXG.jpg)

![](http://static.oschina.net/uploads/img/201312/20143856_6ZXG.jpg)

  

### Zookeeper 监视（Watches） 简介

Zookeeper C API 的声明和描述在 include/zookeeper.h 中可以找到，另外大部分的 Zookeeper C API 常量、结构体声明也在 zookeeper.h 中，如果如果你在使用 C API 是遇到不明白的地方，最好看看 zookeeper.h，或者自己使用 doxygen 生成 Zookeeper C API 的帮助文档。

Zookeeper 中最有特色且最不容易理解的是监视 (Watches)。Zookeeper 所有的读操作——getData(), getChildren(), 和 exists() 都 可以设置监视 (watch)，监视事件可以理解为一次性的触发器， 官方定义如下： a watch event is one-time trigger, sent to the client that set the watch, which occurs when the data for which the watch was set changes。对此需要作出如下理解：

-   （一次性触发）One-time trigger
    
    当设置监视的数据发生改变时，该监视事件会被发送到客户端，例如，如果客户端调用了 getData("/znode1", true) 并且稍后 /znode1 节点上的数据发生了改变或者被删除了，客户端将会获取到 /znode1 发生变化的监视事件，而如果 /znode1 再一次发生了变化，除非客户端再次对 /znode1 设置监视，否则客户端不会收到事件通知。
    
-   （发送至客户端）Sent to the client
    
    Zookeeper 客户端和服务端是通过 socket 进行通信的，由于网络存在故障，所以监视事件很有可能不会成功地到达客户端，监视事件是异步发送至监视者的，Zookeeper 本身提供了保序性 (ordering guarantee)：即客户端只有首先看到了监视事件后，才会感知到它所设置监视的 znode 发生了变化 (a client will never see a change for which it has set a watch until it first sees the watch event). 网络延迟或者其他因素可能导致不同的客户端在不同的时刻感知某一监视事件，但是不同的客户端所看到的一切具有一致的顺序。
    
-   （被设置 watch 的数据）The data for which the watch was set
    
    这意味着 znode 节点本身具有不同的改变方式。你也可以想象 Zookeeper 维护了两条监视链表：数据监视和子节点监视 (data watches and child watches) getData() and exists() 设置数据监视，getChildren() 设置子节点监视。 或者，你也可以想象 Zookeeper 设置的不同监视返回不同的数据，getData() 和 exists() 返回 znode 节点的相关信息，而 getChildren() 返回子节点列表。因此， setData() 会触发设置在某一节点上所设置的数据监视 (假定数据设置成功)，而一次成功的 create() 操作则会出发当前节点上所设置的数据监视以及父节点的子节点监视。一次成功的 delete() 操作将会触发当前节点的数据监视和子节点监视事件，同时也会触发该节点父节点的 child watch。
    

Zookeeper 中的监视是轻量级的，因此容易设置、维护和分发。当客户端与 Zookeeper 服务器端失去联系时，客户端并不会收到监视事件的通知，只有当客户端重新连接后，若在必要的情况下，以前注册的监视会重新被注册并触发，对于开发人员来说 这通常是透明的。只有一种情况会导致监视事件的丢失，即：通过 exists() 设置了某个 znode 节点的监视，但是如果某个客户端在此 znode 节点被创建和删除的时间间隔内与 zookeeper 服务器失去了联系，该客户端即使稍后重新连接 zookeeper 服务器后也得不到事件通知。

### Zookeeper C API 常量与部分结构 (struct) 介绍

#### 与 ACL 相关的结构与常量：

struct Id 结构为：

struct Id {     char * scheme;     char * id; };

struct ACL 结构为：

struct ACL {     int32_t perms;     struct Id id; };

struct ACL_vector 结构为：

struct ACL_vector {     int32_t count;     struct ACL *data; };

与 znode 访问权限有关的常量

-   const int ZOO_PERM_READ; // 允许客户端读取 znode 节点的值以及子节点列表。
    
-   const int ZOO_PERM_WRITE;// 允许客户端设置 znode 节点的值。
    
-   const int ZOO_PERM_CREATE; // 允许客户端在该 znode 节点下创建子节点。
    
-   const int ZOO_PERM_DELETE;// 允许客户端删除子节点。
    
-   const int ZOO_PERM_ADMIN; // 允许客户端执行 set_acl()。
    
-   const int ZOO_PERM_ALL;// 允许客户端执行所有操作，等价与上述所有标志的或 (OR) 。
    

与 ACL IDs 相关的常量

-   struct Id ZOO_ANYONE_ID_UNSAFE; //(‘world’,’anyone’)
    
-   struct Id ZOO_AUTH_IDS;// (‘auth’,’’)
    

三种标准的 ACL

-   struct ACL_vector ZOO_OPEN_ACL_UNSAFE; //(ZOO_PERM_ALL,ZOO_ANYONE_ID_UNSAFE)
    
-   struct ACL_vector ZOO_READ_ACL_UNSAFE;// (ZOO_PERM_READ, ZOO_ANYONE_ID_UNSAFE)
    
-   struct ACL_vector ZOO_CREATOR_ALL_ACL; //(ZOO_PERM_ALL,ZOO_AUTH_IDS)
    

#### 与 Interest 相关的常量：ZOOKEEPER_WRITE, ZOOKEEPER_READ

这 两个常量用于标识感兴趣的事件并通知 zookeeper 发生了哪些事件。Interest 常量可以进行组合或（OR）来标识多种兴趣 (multiple interests: write, read)，这两个常量一般用于 zookeeper_interest() 和 zookeeper_process() 两个函数中。

#### 与节点创建相关的常量：ZOO_EPHEMERAL, ZOO_SEQUENCE

zoo_create 函数标志，ZOO_EPHEMERAL 用来标识创建临时节点，ZOO_SEQUENCE 用来标识节点命名具有递增的后缀序号 (一般是节点名称后填充 10 位字符的序号，如 /xyz0000000000, /xyz0000000001, /xyz0000000002, ...)，同样地，ZOO_EPHEMERAL, ZOO_SEQUENCE 可以组合。

#### 与连接状态 Stat 相关的常量

以下常量均与 Zookeeper 连接状态有关，他们通常用作监视器回调函数的参数。

ZOOAPI const int

ZOO_EXPIRED_SESSION_STATE

ZOOAPI const int

ZOO_AUTH_FAILED_STATE

ZOOAPI const int

ZOO_CONNECTING_STATE

ZOOAPI const int

ZOO_ASSOCIATING_STATE

ZOOAPI const int

ZOO_CONNECTED_STATE

#### 与监视类型 (Watch Types) 相关的常量

以下常量标识监视事件的类型，他们通常用作监视器回调函数的第一个参数。

-   ZOO_CREATED_EVENT; // 节点被创建 (此前该节点不存在)，通过 zoo_exists() 设置监视。
    
-   ZOO_DELETED_EVENT; // 节点被删除，通过 zoo_exists() 和 zoo_get() 设置监视。
    
-   ZOO_CHANGED_EVENT; // 节点发生变化，通过 zoo_exists() 和 zoo_get() 设置监视。
    
-   ZOO_CHILD_EVENT; // 子节点事件，通过 zoo_get_children() 和 zoo_get_children2() 设置监视。
    
-   ZOO_SESSION_EVENT; // 会话丢失
    
-   ZOO_NOTWATCHING_EVENT; // 监视被移除。
    

### Zookeeper C API 错误码介绍 ZOO_ERRORS

ZOK

正常返回

ZSYSTEMERROR

系统或服务器端错误 (System and server-side errors)，服务器不会抛出该错误，该错误也只是用来标识错误范围的，即大于该错误值，且小于 ZAPIERROR 都是系统错误。

ZRUNTIMEINCONSISTENCY

运行时非一致性错误。

ZDATAINCONSISTENCY

数据非一致性错误。

ZCONNECTIONLOSS

Zookeeper 客户端与服务器端失去连接

ZMARSHALLINGERROR

在 marshalling 和 unmarshalling 数据时出现错误 (Error while marshalling or unmarshalling data)

ZUNIMPLEMENTED

该操作未实现 (Operation is unimplemented)

ZOPERATIONTIMEOUT

该操作超时 (Operation timeout)

ZBADARGUMENTS

非法参数错误 (Invalid arguments)

ZINVALIDSTATE

非法句柄状态 (Invliad zhandle state)

ZAPIERROR

API 错误 (API errors)，服务器不会抛出该错误，该错误也只是用来标识错误范围的，错误值大于该值的标识 API 错误，而小于该值的标识 ZSYSTEMERROR。

ZNONODE

节点不存在 (Node does not exist)

ZNOAUTH

没有经过授权 (Not authenticated)

ZBADVERSION

版本冲突 (Version conflict)

ZNOCHILDRENFOREPHEMERALS

临时节点不能拥有子节点 (Ephemeral nodes may not have children)

ZNODEEXISTS

节点已经存在 (The node already exists)

ZNOTEMPTY

该节点具有自身的子节点 (The node has children)

ZSESSIONEXPIRED

会话过期 (The session has been expired by the server)

ZINVALIDCALLBACK

非法的回调函数 (Invalid callback specified)

ZINVALIDACL

非法的 ACL(Invalid ACL specified)

ZAUTHFAILED

客户端授权失败 (Client authentication failed)

ZCLOSING

Zookeeper 连接关闭 (ZooKeeper is closing)

ZNOTHING

并非错误，客户端不需要处理服务器的响应 (not error, no server responses to process)

ZSESSIONMOVED

会话转移至其他服务器，所以操作被忽略 (session moved to another server, so operation is ignored)

##### Watch 事件类型：

ZOO_CREATED_EVENT：节点创建事件，需要 watch 一个不存在的节点，当节点被创建时触发，此 watch 通过 zoo_exists() 设置  
ZOO_DELETED_EVENT：节点删除事件，此 watch 通过 zoo_exists() 或 zoo_get() 设置  
ZOO_CHANGED_EVENT：节点数据改变事件，此 watch 通过 zoo_exists() 或 zoo_get() 设置  
ZOO_CHILD_EVENT：子节点列表改变事件，此 watch 通过 zoo_get_children() 或 zoo_get_children2() 设置  
ZOO_SESSION_EVENT：会话失效事件，客户端与服务端断开或重连时触发  
ZOO_NOTWATCHING_EVENT：watch 移除事件，服务端出于某些原因不再为客户端 watch 节点时触发

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTU0MDk4MjE1M119
-->