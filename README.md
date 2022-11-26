## [深度剖析zookeeper核心原理](https://s.juejin.cn/ds/rVnRsQr/)

## 1. 什么是zookeeper？

**分布式的协调中心**，也叫**注册中心**。它负责**维护集群内元数据**以及**主节点选举**等功能，它能很好地支持集群部署，并且具有很好的分布式协调能力。

zookeeper采用的是CAP理论的**CP模型**，**用服务的可用性来换取数据的相对强一致性**，也就是牺牲一部分可用性来提高数据的一致性。

> 比如一个集群中有5个节点，按照过半原则来说，3个节点及以上算半数。若集群内挂了3台，只保留2台服务，那么此时集群是不能做读写请求的，这是CP的特征之一。
当然，这导致了集群的可用性较低，因为仍然活着两个节点，却不能提供服务。不过zookeeper的选主效率也比较高，根据官方压测结果，它不会超过200ms。

### 1.1 zookeeper的特点

- **支持分布式集群部署**
- **顺序一致性**: 客户端发送的每一个请求到zookeeper都是有序的，在整个集群内都是有序的
- **原子性**: 一个请求在分布式集群中具备原子性，要么所有节点都处理了这个请求，要么都不处理
- **可靠性**: 某台服务器宕机，保证数据的完整性
- **实时性**: 发生数据变更，及时通知客户端，采用的事`Watcher`机制

### 1.2 zookeeper的角色

- **Leader**: **集群之首**，主节点，提供**读写服务**，且写请求只能由Leader来完成，也负责将写请求的数据同步给各个节点。
（每个写请求都会生成一个zxid，将此请求同步给各个Follower和Observer时，是zxid决定了顺序一致性）
- **Follower**: 从节点，仅提供**读能力**，且**能参加选举**
- **Observer**: 也是从节点，仅**提供读能力**，但是**不能参加选举**。它只会接收Leader的同步数据，合理利用它能够提高集群的**读并发能力**

![img.png](img.png)

### 1.3 zookeeper的节点类型
znode，可以理解为zookeeper内**存储数据的数据结构**，有持久节点和临时节点两种类型

- **持久节点**: 具备持久化功能，客户端断开链接了也会一直存在，用作元数据存储
- **临时节点**: 基于内存，客户端断开链接了临时节点会自动消失，大多分布式协调工作都是用临时节点来做的。
如果客户端采用zookeeper的 `watch` 机制来监听临时节点，一旦该节点消失，客户端就会收到临时节点被删除的通知，以此来做一些自己的业务逻辑

> **顺序节点**: 顺序节点不过是顺序持久节点或顺序临时节点，如在 `/test` 下创建临时顺序节点，那么会自动为这些子节点编号，且编号自增。
> 临时顺序节点一般用于分布式锁的场景，在加锁时创建一个临时顺序节点，比如 `lock0000000000`，当其他客户端再想获取锁时会自增编号，
> 如 `lock0000000001`，并且会注册 `Watcher` 监听上一个临时顺序节点，当持有锁的客户端断开链接时，那么下一个编号的客户端会收到通知，并尝试获取锁

### 1.4 什么是节点
znode，像Linux的文件系统一样，每个节点都是用一个**路径作为唯一标识**，**每个节点都能保存数据**，**也可以多层级**。

![img_1.png](img_1.png)

如上图，`/，/app1，/app2`属于三个根节点，`/app1/p_1`属于`/app1`节点的子节点

### 1.5 什么是Session
![img_3.png](img_3.png)

客户端与zookeeper服务端**通过session**建立连接，每个与服务器建立链接的客户端都会被分配一个sessionID，且全局唯一。
而且session也能设置超时时间，如果在规定时间内客户端没能与服务器进行心跳或者通信，则链接失效。

## 2. Leader选举
- 我们在搭建集群的时候，都会添加如下配置，其中server`.1 .2 .3`这个编号，在选举中会用到，暂且称它为`myid`
>server.1= ZooKeeper 1:2888:3888 </p>
server.2= ZooKeeper 2:2888:3888 </p>
server.3= ZooKeeper 3:2888:3888

- **zxid**: 事务ID，也是选举中会被用到的一个值。每次请求都会使zxid加一，这个zxid设计的很巧妙。
它是64位long类型，前32为代表选举更迭次数（朝代，改朝换代次数）我们称之为`epoch`，后32位记录请求次数，也是事务次数。

初始时是这个样子

`00000000000000000000000000000000 00000000000000000000000000000000`

客户端发起3次请求后，后32位结果为3，如下

`00000000000000000000000000000000 00000000000000000000000000000011`

Leader宕机，选举完成后，更新前32位，加一，并且后32位事务次数归0

`00000000000000000000000000000001 00000000000000000000000000000000`

如果这个服务很坚挺，写请求又很频繁把后32位写满了

`00000000000000000000000000000001 11111111111111111111111111111111`

那么再请求一次，则会将前32位加一

`00000000000000000000000000000010 00000000000000000000000000000000`

zookeeper节点包含的选举状态
- **LOOKING**: 竞选状态，此状态下还没有 Leader 诞生，需要进行 Leader 选举
- **FOLLOWING**: 对应Follower角色，并且它知道Leader是谁
- **OBSERVING**: 对应Observer角色，知道Leader是谁
- **LEADING**: 对应Leader角色

### 2.1 选举流程原理
#### 2.1.1 选举结果依据比较流程图
![img_4.png](img_4.png)

选举结果比较依据如上图所示，会先比较 `zxid` 的 `epoch`，有大选大，相同的话继续比较事务次数，有大选大，如果还相同的话，那么只能选 `myid` 比较大的节点了

以上比较过程暗含的逻辑: **哪个Follower上的消息最新就让哪个Follower被选为Leader**

#### 2.1.2 选举过程
熟悉了选举的比较依据，那么选举过程是什么样的呢？先来熟悉下选举必要的参数，如下

- **self_myid**: 当前服务器的myid
- **self_zxid**: 当前服务器的zxid
- **vote_id**: 被推荐服务器的myid
- **vote_zxid**: 被推荐服务器的zxid
- **logicClock**: 逻辑时钟，表示的是**发起投票的第几轮**，保存在**内存**中，每轮投票后自增一

以下是选举过程图例，**三台服务器都在LOOKING状态下**，指定服务器1的myid为1，zxid为10，logicClock为1；
服务器2的myid为2，zxid为20，logicClock为2；服务器3的myid为3，zxid为30，logicClock为2

第一轮服务器互相不知道 `myid` 和 `zxid` ，所以都把票投给自己，并把投票信息广播给各个参选的服务器，
投票信息中包含 **vote_id**, **vote_zxid**, **logicClock** 信息

![](选举流程图-第%201%20页.jpg)

收到各个服务器第一轮的投票信息后，开始处理，过程如下
- 服务器1的 `logicClock` 为1，收到服务器2和3的投票信息，发现自己的 `logicClock` 小，此时会将自己的 `logicClock` 改成2，之后清空自己的票箱，
并比较自己的投票和收到的投票的 `zxid` 和 `myid` ，选一个大的，也就是服务器3，通过网络传给服务器2和服务器3
- 服务器2收到的投票信息中 `logicClock` 分别为1和2，它会将服务器1的投票信息忽略，继续处理服务器3的投票信息，
服务器3的投票信息中 `vote_zxid` 比自己的大，会将自己的投票信息改成服务器3的投票信息，并通过网络传给服务器1和服务器3
另外将收到的票据和自己刚改的票据放入投票箱中
- 服务器3发现收到的投票信息都比自己的投票信息 **"要小"** ，那么它会把票投给自己，并通过网络把投票信息传给服务器2和服务器3

结果如图所示

![](选举流程图3.jpg)

最终选举结果，如果票数**超过服务器的半数**，那么则选举该节点为Leader节点。
三个节点会结束LOOKING状态，服务器3进入LEADING状态，服务器1和服务器2进入FOLLOWING状态

比较规则判断流程图

![](选举流程图2.jpg)

### 2.2 选举网络通信原理
zookeeper投票机制是异步的，它会将投票信息放入**队列**，**开启线程异步消费消息进行发送**，网络连接和消息通信借助`Socket + IO流`实现。

流程图如下，右侧 `QuorumCnxManager` 是网络通信 **"组件"**，负责将投票信息在节点间发送和接收，注意各个节点只和比自己myid小的建立连接，避免连接重复；
左侧 `FastLeaderElection` 则负责将投票信息提供给网络通信组件

![](选举网络通信原理.jpg)

1. zookeeper启动时，会开启两条线程其中 `WorkerSender` 负责将要发送的消息从 `senderqueue` 中拿出来，放到 `queueSendMap` 中，
其中 key为接收者myid(要让Socket知道发给谁), value为要发送的投票信息； 
`WorkerReceiver` 负责将 `QuorumCnxManager` 中接收到的消息从 `recvQueue` 队列里拿出来放入`recvqueue`
2. `QuorumCnxManager` 也开启两条线程（注意这里是每个连接开启两条线程），`SenderWorker` 负责发，`RecvWorker` 负责接收消息
3. 选举开始时，每个节点都为自己投一票，并放入 `senderqueue` 中， `WorkerSender` 和 `SenderWorker` 则不断的轮询处理要发送的消息，
同样 `RecvWorker` 和 `WorkerReceiver` 也是不断地轮询接收投票信息

---

## 服务端核心参数
- **tickTime**: ZooKeeper 服务器与客户端之间的心跳时间间隔。单位是毫秒。也就是每隔多少毫秒发一次心跳
- **initLimit**：Leader 与 Follower 之间建立连接后，能容忍同步数据的最长时间，即 `n * tickTime` 毫秒
- **syncLimit**：Leader 和 Follower 之间能容忍的最大请求响应时间，单位是 `n * tickTime` 毫秒，如果超过该时长，
则这个Follower会被认为已经挂掉了，会被Leader踢出去

**initLimit**和**syncLimit**都是以**tickTime**为基准配置的，前两者只需配置为n，即代表n个基本单位

- **dataDir**：存放 ZooKeeper 里的数据快照文件的目录
- **dataLogDir**：存储事务日志目录，如果不配置则默认存在**dataDir**目录下
- **snapCount**：多少个事务生成一个快照。默认是 10 万个事务生成一次快照，快照文件存储到 **dataDir** 目录下

> server.1= ZooKeeper 01:2888:3888
server.2= ZooKeeper 02:2888:3888
server.3= ZooKeeper 03:2888:3888

其中`ZooKeeper 1`为hostname，之后配置的两个端口号，2888和3888含义如下
- **2888**: 用于Leader和Follower之间进行数据通信
- **3888**: 在集群恢复模式下进行Leader选举投票
- **cnxTimeout**：默认5000毫秒，配置3888端口建立连接的超时时间

- **jute.maxbuffer**：默认是 1mb, 1048575(bytes), 一个节点能保存的最大数据大小
- **maxClientCnxns**：最大与zookeeper服务端的建立的连接数，默认是 60 个


## 基础命令
```shell
# 创建持久节点
create /helloworld
# 创建持久节点v1 并赋值保存值1
create /helloworld/v1 1

# 展示 / 节点下的所有节点
ls /

# 获取/helloworld/v1 的值
get /helloworld/v1
```
- 执行完以上命令，节点如下图所示

![img_2.png](img_2.png)

```shell
# 创建临时节点
create -e /helloworld2

# 创建顺序节点
[zk: localhost:2181(CONNECTED) 1] create -s /helloworld/v3
Created /helloworld/v30000000001
[zk: localhost:2181(CONNECTED) 2] create -s /helloworld/v3
Created /helloworld/v30000000002

# 版本
version

# 为节点设置值
set /helloworld hi

# 删除一个节点
delete /helloworld

# 查看节点状态
stat /helloworld
cZxid：创建znode节点时的事务ID。
ctime：znode节点的创建时间。
mZxid：最后修改znode节点时的事务ID。
mtime：znode节点的最近修改时间。
pZxid：最后一次更改子节点的事物id（修改子节点的数据节点不算）
cversion：对此znode的子节点进行的更改次数。
dataVersion：该znode的数据所做的更改次数。
aclVersion：此znode的ACL进行更改的次数。
ephemeralOwner：如果znode是ephemeral（临时）节点类型，则这是znode所有者的Session ID。 如果znode不是ephemeral节点，则该字段设置为零。
dataLength：znode数据节点的长度。
numChildren：znode子节点的数量。

# 退出客户端
quit
```