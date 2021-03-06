# CAP 和 BASE 理论

## CAP

C:一致性

​	所有节点访问都是同一份最新的数据副本

A:可用性

​	每次请求都能够获取正确的响应- 不保证数据是最新的

P:分区容错性

​	对实际而言,分区相当于对通信的时限要求.如果不能在限时内达成数据一致性,意味着发生了分区的情况.需要在一致性 与 可用性中选择

### CA without P

​	一般不会选择此模型,放弃P 意味着放弃了系统扩展性,违背了分布式系统设计的初衷.

### CP without A

​	Zookeeper,ETCD

​	不要求可用性,保持强一致性.而P(分区) 会导致同步时间无限延长(也就是等待数据同步完才能正常访问服务)

### AP without C

​	Eureka,Consul

​	高可用允许异常分区,放弃一致性.

## BASE

BA-Basically Available

​	基本可用

S-Soft State

​	软状态

E-Eventual Consistency

​	最终一致

BASE 是对CAP 中一致性与可用性 CA 权衡的结果,核心思想是无法做到强一致性,但可以根据业务自身特点,采用适当的方式来使系统达到最终一致.