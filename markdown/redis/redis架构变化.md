# redis 架构变化

[TOC]

## 1 standalone高可用



![](..\resource\redis-alone.png)

```


			                                -----------
			                    -----------|Redis1:6379|
							   |            -----------
			  ------------	   |
client ------|    VIP     |----|            keepalived
			  ------------     |
			  				   |            -----------
							    -----------|Redis2:6379|
											-----------



```



##  2 Redis 多副本(主从)

### 2.1 Redis 主从概述

采取一主多从,主从之间进行数据同步.当master 宕机后,通过选举算法(Paxos,Raft) 从slave 中选举出新的master 继续对外提供服务.主机恢复后以slave 的身份重新加入.

主从另一个目的是读写分离,这是单机读写过高的解决方式,主机提供写操作,与少量读操作.把多余的读操作通过负载均衡算法分流到单个或多个slave 服务器上.

缺点是主机宕机后,slave 虽然成为了新master ,但对外提供IP服务的地址发生变化了.会影响到客户端.

可以通过keepalived 进行配置.漂移IP 规避.

### 2.2 Redis 主从拓扑

#### 2.2.1 一主一从

用于主节点故障转移从节点.当主节点的写命令并发高且需要持久化,可以只在从节点开启AOF(主节点不需要),这样既保证了数据的安全性 又间接提高了主节点的性能(持久化对主节点的影响)

#### 2.2.2 一主多从

针对"读"较多的场景,"读"由多个从节点来分担,但节点越多,主节点同步到多节点的次数也越多,影响贷款,也加重主节点的负担.

#### 2.2.3 树状主从

一主多从的缺点解决方案(主节点推送次数多压力大) 解决方案. 主节点 只推送一次数据到临近节点,临近节点在向自己的从节点推送数据,减轻主节点数据推送的压力.

#### 2.2.4 Redis 的数据同步方式

- 同步方式

  主机收到客户端请求写操作后,同步方式把数据同步到主机上,当从也成功写入后,主机才返回给客户端成功,也称数据强一致性.显然,这种方式性能会降低不少,当从机很多时,可以不用每台同步,使用树形主从拓扑,由从机在从主机同步后,在同步至其余从机,提高主机性能,分担同步压力.

- 异步方式

  主机接收到写操作后,直接返回成功,然后在后台用异步方式把数据同步到从机上.这种同步方式性能较好,但是无法保证数据的完整性.

## 3 Redis Sentinel (哨兵)

### 3.1 概述

Redis-Sentinel 是Redis 官方推荐的高可用（HA）方案，当Redis 用作Master-Slave的高可用方案时，假如master宕机了，Redis 客户端本身功能都没实现主备切换，而Redis-Sentinel本身也是一个独立运行的程序，能监控多个master-slave集群，发现master宕机后能自动进行切换

Redis-Sentinel 支持集群，避免单点故障。

![](..\resource\redis-sentinel.png)

### 3.2 Sentinel运行机制 

#### Sentinel 的原理

Redis Sentinel 通过三个定时任务来监控 主从节点,来判定是否存在节点故障:

- 每隔10秒每个sentinel会向主节点和从节点发送`info replication` 命令来获取最新的redis 拓扑结构
- 每隔2秒每个sentinel 会向redis 节点的 `_sentinel:hello`频道发送自己对主节点是否故障的判断以及自身的节点信息
- 每隔1秒每个sentinel会向redis 节点,其余sentinel节点 发送`ping` 命令来做心跳检测

#### Sentinel 的仲裁会

当一个master 被sentinel集群监控时，需要为它制定一个参数 `sentinel monitor mymaster 192.168.140.129 6379 2` ，这个参数指定了当需要判决master 不可用，并且进行failover 时，需要的sentinel 的数量。简称投票数。



sentinel 对于不可用有两种情况

1. 一种是主观不可用（SDOWN） SDOWN是sentinel 自己主观上检测到的关于master 失效的状态。

   

2. 一种是客观不可用（ODOWN） ODOWN 需要一定数量的sentinel 达成一致意见，才能认为一个master 客观上已经宕机。各个sentinel 之间通过命令`SENTINEL is_master_down_by_addr`来获得其它sentinel对master 的检测结果



从sentinel来看，如果发送了`PING`心跳后，在一定时间内`sentinel down-after-milliseconds mymaster 1000 ` 没有收到合法的回复，就达到了SDOWN的状态。

当sentinel 发送`PING`后，如下回复都认为是合法的(其它回复或没收到回复都是不合法的)：

```shell
PING replied with +PONG.
PING replied with -LOADING error.
PING replied with -MASTERDOWN.
```



从SDOWN 切换到ODOWN 不需要任何一致性算法，只需要 `gossip`协议：如果一个sentinel 收到了足够多的sentinel发来消息告诉，某个master 已经宕机，SDOWN 状态将会ODOWN 状态。如果之后master 可用了，那么会清理这个状态。



真正进行failover 需要一个授权的过程，但是所有的failover 都开始与一个叫ODOWN 的状态。（ODOWN状态只适用于master，对于不是master 的redis 节点sentinel 间不需要任何协商，slaves 和sentinel 不会有ODOWN 状态 。）

当ODOWN 时，failover 被触发。failover一旦被触发，尝试去进行failover 的sentinel回去获得`大多数sentinel` 的授权（如果票数比大多数还要大的时候，则询问更多的sentinel）

这个区别看起来很微妙。例如，集群中5个sentinel，票数设置为2，当两个sentinel 认为一个master 已经不可用了以后，将会触发failover。但是进行failover 的sentinel 必须获得至少3个sentinel 的授权才可以进行failover。若票数设置为5，要达到ODOWN 状态，必须所有5个sentinel 都主观认为master 不可用，要进行failover 需要获得5个sentinel 的授权。

#### 配置版本号

为什么要先获得大多数的sentinel 的认可时才能真正去执行failover呢。

当一个sentinel 被授权后，它将会获得宕掉的master 的一份最新配置版本号，当failover 执行结束以后，这个版本号将会被用于最新的配置。因为大多数sentinel 都已经知道该版本号已经被要执行failover 的sentinel 拿走了。所以其他的sentinel 都不能再去使用这个版本号。这意味着每次failover 都会有一个唯一的版本号。

sentinel 集群间会遵守这样一个原则：若sentinel A 推荐sentinel B 执行failover，A会等待一段时间后自行再去对同一个master 执行failover，这个等待时间通过 `failover-timeout`配置项配置的。从这个规则可以看出，sentinel 集群中的sentinel不会再同一时刻并发去failover 同一个master。第一个进行failover 的sentinel 如果失败了，另外一个将会在一定时间内进行重新进行failover，以此类推。

redis sentinel集群保证了活跃性：大多数sentinel能够相互通信，最终将会只有一个被授权去进行failover。redis sentinel 也保证了安全性：每个试图去failover 同一个master 的sentinel 都将会得到一个独一无二的版本号。

#### 配置传播

一个sentinel 成功的对一个master 进行failover后，会把关于master 的最新的配置 通过广播的形式通知其余sentinel，其余sentinel 则更新对应的master配置

一个failover 要想被执行成功。执行failover 的sentinel 必须能够向选为master 的slave 发送`SLAVE OF NO ONE`命令，然后能够通过`INFO`命令看到master 的配置信息。

当一个slave 被选举为master 并接收到sentinel 发送的`SLAVE OF NO ONE` ,即使其他的slave 还没针对新master 配置自己，failover 也被认为是成功。然后所有sentinel 将会发布新的配置信息。



每个sentinel只用 `发布/订阅` 方式持续的传播master 的配置版本信息，配置传播的 `发布/订阅`管道是：`_sentinel_:hello`。因为每一个配置都有一个版本号，所以以版本号最大的那个为标准。

```
举个栗子：假设有一个名为mymaster的地址为192.168.1.50:6379。一开始，集群中所有的sentinel都知道这个地址，于是为mymaster的配置打上版本号1。一段时候后mymaster死了，有一个sentinel被授权用版本号2对其进行failover。如果failover成功了，假设地址改为了192.168.1.50:9000，此时配置的版本号为2，进行failover的sentinel会将新配置广播给其他的sentinel，由于其他sentinel维护的版本号为1，发现新配置的版本号为2时，版本号变大了，说明配置更新了，于是就会采用最新的版本号为2的配置。
```

#### Sentinel 之间 和 Slave之间的自动发现机制

虽然sentinel 集群中各个sentinel都互相连接彼此来检查对方的可用性以及互相发送消息。但你不用在任何一个sentinel 配置任何其他的sentinel 的节点。因为sentinel利用了master的`发布/订阅`机制去自动发现其他监控了master 的sentinel 节点。通过向名为`_sentinel_:hello`的管道中发送消息来实现。

同样，你也不需要再sentinel 中配置某个master 的所有slave 地址，sentinel 会通过询问master 来得到这些slave 的地址。

每个sentinel通过向每个master 和slave 的`发布/订阅`管道 `_sentinel_:hello`每秒发送一次消息，来宣布他的存在，每个sentinel 也订阅了每个master 和slave 的管道`_sentinel_:hello`的内容，来发现未知的sentinel，当检测到了新的sentinel ，则将其加入到自身维护的master 监控列表中。

每个sentinel发送的消息中包含了当前维护的最新master 的配置。如果某个sentinel 发现自己的配置版本低于接收到的配置版本，则会用新的配置更新自己的master配置。

在为一个master 添加一个新的sentinel 前，sentinel 总是检查是否已经有sentinel 与 新的sentinel 的进程号或地址是一样。如果是，这个sentinel将被删除，而添加新的sentinel。

#### 网络隔离时的一致性

redis sentinel 集群的配置的一致性模型为最终一致性，集群中每个sentinel 最终会采用最高版本的配置，然而在实际的应用环境中，有三个不同的角色会与sentinel 打交道：

1. Redis 实例
2. Sentinel 实例
3. 客户端



下面有个例子，有三个主机，每个主机分别运行一个redis 和一个sentinel：

```
                         +-------------+
                         | Sentinel 1  | <--- Client A
                         | Redis 1 (M) |
                         +-------------+
                     		   |
                     		   |
 +-------------+     		   |           +------------+
 | Sentinel 2  |-----+-- / partition / ----| Sentinel 3 | <--- Client B
 | Redis 2 (S) |                           | Redis 3 (M)|
 +-------------+                           +------------+
```

在这个系统中，初始状态redis3 是master ，redis1和redis2 是slave。之后redis3所在的主机网络不可用了，sentinel1和sentinel2启动了failover 并把redis1 选举为master。

Sentinel 集群的特性保证了sentinel1 和sentinel2 得到了关于master 的最新配置。但是sentinel3 依然保持着旧的配置（网络不可用与外界隔离）

当网络恢复后，我们知道sentinel3将会更新它的配置。但是，如果客户端所连接的master 被网络隔离，那么客户端依然可以向redis3写数据，但是当网络恢复后，redis3将会变为redis 的一个slave，那么隔离期间，客户端向redis3写的数据将丢失。

redis 采用的是异步复制，这种场景下没有办法避免数据的丢失，然后可以通过如下配置来配置redis3和redis1，让数据不丢失。

```
#当是master 节点，如果不能向至少 n 个slave 写数据，将会拒绝客户端的写请求。
min-slaves-to-write 1
#由于是异步复制，master无法向slave写数据意味着slave要么断开连接了，要么未在执行时间发送同步请求
min-slaves-max-lag 10
```

#### Sentinel 状态持久化

sentinel 的状态会被持久化的写入sentinel 的配置文件中。每次当收到一个新的配置时，或者新创建一个配置时，配置会被持久化到硬盘中，并带上配置的版本戳。这意味着，可以安全的停止重启sentinel 进程。

#### Slave 选举 与 优先级

当一个sentinel 准备好了要进行failover，并且受到了其他sentinel 的授权，那么就要选举出一个合适的slave 来作为master。

slave 的选举主要会评估slave 的如下几个方面：

- 与master 断开连接的次数
- slave 的优先级
- 数据复制的下标（用来评估slave 当前拥有多少master 数据）
- 进程Id

如果一个slave 与master 失去联系超过10次，并且每次都超过了配置的最大失联时间（`down-after-milliseconds option`),并且，如果sentinel 在进行failover时发现slave 失联，那么这个slave 就会被认为不适合做新master。

严格定义，如果一个slave 持续断开连接的时间超过

`（down-after-millseconds*10）+milliseconds_since_master_is_in_SDOWN_state` 就会被取消选举资格。

符合上述条件的slave 会根据如下规则排序进行选择：

1. sentinel 首先会根据slave 的优先级来进行排排序，优先级越小排名越靠前。
2. 优先级相同，查看复制的下标，那个从master 复制的数据多，那个靠前
3. 如果优先级与下标都相同，选择进程ID小的那个。

#### Sentinel 和 Redis 身份验证

当一个master配置为需要密码才能连接时，客户端和slave 在连接时都需要提供密码。

master 通过`requirepass`设置自身密码

slave 通过`masterauth`雷设置访问master 的密码

## 4 Redis Cluster

- 优点: 无中心节点;数据按照slot 存储在多个 实例上;平滑扩容/缩容;自动故障转移(节点间通过Gossip 协议交换状态信息,进行投票机制完成Slave 到Master 的提升);降低运维成本,提高了系统的可扩展性和高可用性.
- 缺点:严重依赖外部Redis-Trib;缺乏监控管理;需要依赖 Smart Client(连接维护,缓存路由表,MutiOp 和Pipeline 支持);FailOver 节点的检测过慢,不如 Zookeeper 及时;Gossip 消息的开销;无法统计冷热数据;Slave 资源闲置,不能缓解读压力.

## 5 Redis 集群自研

### 5.1 Codis

https://github.com/CodisLabs/codis

### 5.2 Redisson

https://github.com/redisson/redisson

###  

![](..\resource\redis-cluster.png)

##  6 优缺点对比

|                      | 优点                                                         | 缺点                                                         |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 单体高可用           | 用于可穿透业务场景，后端有db 存储，脱机使用影响不大          | 单点可靠性不高,down 机,穿透对下游服务影响较大,受限于单机容量 |
| Redis 主备           | 用于Redis 主备高可用场景,,keepalived 保证HA                  | 服务正常时闲置备机资源                                       |
| Redis 主从           | 缓解master 读压力,提高主节点性能                             |                                                              |
| Redis Sentinel(哨兵) | Redis 主从高可用,sentinel 用于自主failover 恢复.从节点可以缓解读压力 | 受限于单机容量,failover 时服务不可用                         |
| Redis cluster        | 用于高可用需求场景，可用于大数据量高可用Cache/存储场景.内存/QPS 不限于单机 |                                                              |
| Redis 自研(HA)       | Twemproxy,codis                                              |                                                              |





## 参考资料

https://www.jianshu.com/p/06ab9daf921d

https://www.jianshu.com/p/c2abf726acc7

https://blog.csdn.net/u013473512/article/details/79044683

https://redis.io/topics/cluster-tutorial

https://blog.csdn.net/u013473512/article/details/79044683