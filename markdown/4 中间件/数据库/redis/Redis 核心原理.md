# Redis 核心原理

## 1 Redis 的单线程 和 高性能

数据存储在内存中,所有的运算都是内存级别的运算(纳秒级别),

多线程是复杂的,会涉及同步等问题.同时单线程避免了多线程的上下文切换等性能损耗.

注:因为redis 是单线程的,避免使用耗时指令 `keys *`

## 2 Redis 单线程如何处理那么多的客户端连接

Redis 依靠IO 多路复用来处理大量客户端连接.

Redis 利用 epoll 来实现IO 多路复用,将连接信息和事件放到队列中,依次分发给事件分派器,事件分派器分发给事件处理器

注: 典型的C10K 问题,Nginx 也采用 IO多路复用原理来解决C10K

##  3 Redis 持久化

### 3.1 RDB

默认情况下,Redis 将内存数据库快照保存在名字为dump.rdb 的二进制文件中.

可以通过save 命令设置 在N秒内数据集至少有M个改动时 保存一次数据集.

redis.conf 中默认有如下配置

```
#900 秒 1个改动
save 900 1
#300 秒 10个改动
save 300 10
#60  秒 10000个改动
save 60  10000
```



### 3.2 AOF

将修改的每一条指令记录进文件

通过 `appendonly yes` 来打开AOF 功能

开启AOF 后,每当Redis 执行一个修改数据命令时,这个命令就会被追加到AOF 文件的末尾.这样当Redis 重启时,变可以通过AOF 文件重放Redis 内存数据.

通过`fsync` 来控制多久将数据刷到磁盘.

- `no` 		表示不执行fsync,由操作系统保证数据同步到磁盘,速度最快
- `always`表示每次写入都执行fsync 保证数据同步到磁盘
- `everysec` 表示每秒执行一次fsync,兼顾速度与数据安全.但可能会丢失1s 的数据

### 3.3 混合持久化

重启Redis 时,一般不使用rdb 来恢复内存状态,因为会丢失大量数据.通常使用AOF 日志重放.但重放AOF 日志 相对 RDB 恢复快照来说慢的多.这样在redis 实例很大的情况下.启动需要花费很长时间.

在Redis 4.0版本上,可以通过混合持久化来解决这个问题.

AOF 在重写时,将重写这一刻之前的内存rdb 快照文件的内容和增量的AOF 修改内存数据的命令日志文件存在一起,都写入新的aof 文件.新的文件一开始不叫 `appendonly.aof`,等到冲写完新的AOF 文件才会进行改名,覆盖原有的AOF 文件,完成新旧文件的替换;

aof 根据配置规则在后台自动重写,也可以认为执行`bgrewriteaof` 重写 AOF.在redis 重启时,可以先加载rdb 内容,在重放增量AOF 日志,替代之前的AOF 全量文件重放,提升效率.

## 4 Redis 缓存淘汰策略

### 4.1 缓存淘汰策略

当Redis 内存超出物理内存限制时(通过 `maxmemory`限制),内存的数据会开始和磁盘产生频繁的交换(swap).交换会让redis 的性能急剧下降,对于访问量比较频繁的Redis 来说,这样的龟速存取效率基本上等同于Redis 不可用.

当实际内存超出maxmemory 限制时,Redis 提供了几种缓存淘汰策略让用户决定如何腾出新空间来继续提供读写服务.

- noeviction

  ​	默认淘汰策略,丢弃写请求操作,读请求可以继续执行.保证不会丢失数据.但会中断正常业务

- volatile-lru

  ​	尝试淘汰设置了过期时间的key ,最少使用的key 优先淘汰.没有设置过期时间的key 不会被淘汰,这样可以保证需要持久化的数据不会突然丢失.

- volatile-ttl

  ​	移除即将过期的key

- volatile-random

  ​	随机移除设置过期时间的key.

- allkeys-lru

  ​	利用LRU 移除任何key

- allkeys-random

  ​	随机移除任何key

  volatile-xxx 策略只会针对设置了过期时间的key 进行淘汰,allkeys-xxx 会对所有的key 进行淘汰.

  如果只是使用redis 当做缓存,在客户端写缓存时可以不设置过期时间,使用allkeys-xxx

  如果使用redis 的持久化功能,建议使用 volatile-xxx 功能,保证数据安全性

### 4.2 过期key处理

1. 立即删除

   在设置过期时间时,创建回调事件,当过期时间到达,由事件处理器自动执行删除key 的操作

2. 惰性删除

   不关心key 的过期时间,从字典中取key时,检查是否过期,过期删除返回nil,否则返回val

3. 定时删除

   每隔一段时间,对expires字典进行检查,删除里面的过期key

过期key 处理方式优缺点

|          | 优点             | 缺点                                                         |
| -------- | ---------------- | ------------------------------------------------------------ |
| 立即删除 | 及时释放内存空间 | cpu 不友好,影响 正常业务执行(redis 事件处理器对时间的处理方式-无序链表 O(n)) |
| 惰性删除 | cpu 友好         | 浪费内存                                                     |
| 定时删除 | 较好             | 立即与惰性的折中                                             |

Redis 使用 惰性删除 与定时删除两者结合使用.

## 5 Pipeline

### 5.1 简介

Redis 是 的server 与 client 通信模型基于TCP 通信协议.

RTT (Round Trip Time)是指一次redis client 操作 的报文从 client 到 server,在从server 返回到 client的时间

简单说就是,将多次 redis client 操作,通过一次socket 通信发送到 redis server 端,等到操作全部处理完,一起返回结果,减少多次请求造成的 socket 请求开销.

注:当client 使用pipeline 发送命令,redis server 将会强制使用队列用来回复.显然会消耗内存.所以如果你需要用pipeline 发送大量指令,最好一批发送合理数量,如发送 10K commands,读取响应,在发送下一批此的10K 条.批次发送与一起发送所有指令的耗时接近相同.但对于redis server 端来说 批次发送10K comands 消耗的内存 远小于 另一种.

### 5.2 不止降低(时间消耗)RTT

开启Pipeline 不光降低 从客户端到server 端的RTT,它同时提升了 redis server 端的性能

如果不使用pipeline,任意一个指令,访问内存 与 读取响应的操作是 轻量级的.但执行socket IO 时,这是较耗时操作.此时会发生 系统 `read` ` 与 `write` 操作 .将数据从用户内存空间拷贝到内核.上下文切换也是一个巨大的性能损失.

当开启pipeline,多条命令可以使用同一个 系统`read` 或 `write` 指令,鉴于此,通过开启pipeline,redis server 的QPS 几乎成线性增长.开启Pipeline 后 QPS 近乎增长了10倍.

### 5.3 Pipeline 与 Redis Script

Redis Script 是 2.6 版本后引入的. 使用script 的一个很大优势是,以最小的延迟去读写数据.(script 中 后续的命令可以使用先前命令的结果,而在pipeline 中不可以)

## 6 Pub/Sub 发布/订阅

### 6.1 简介

发布 不是 发送消息给特定的 订阅.而是发布消息到 不同的频道,不需要知道什么样的订阅者订阅.订阅对一个或多个频道感兴趣,只接收感兴趣的消息,不需要知道是谁发布的. 发布 与定于之间解耦,带来更大的扩展性.

### 6.2 命令

- subscribe  			

  订阅指定的频道消息

- unsubscribe          

  取消订阅频道消息,若不指定频道,则取消订阅所有频道

- psubscribe            

  订阅一个或多个符合给定模式的频道

- punsubscribe        

  取消订阅一个或多个符合给定模式的频道  一个无参的punsubscribe,将取消所有的psubscribe 订阅 

- publish

  向指定channel 发送消息message

- pubsub

  pubsub 是自省命令,能够检测PUB/SUB 的状态

### 6.3 数据库与作用域

发布/订阅 与当前所在数据库 num 没有关系

## 7 

## 参考

https://www.jianshu.com/p/4e6b7809e10a

https://redis.io/topics/pipelining

https://www.cnblogs.com/xingzc/p/5988080.html