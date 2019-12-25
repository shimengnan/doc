# CAP 和 BASE 理论

## CAP

C:一致性

​	所有节点访问都是同一份最新的数据副本

A:可用性

​	每次请求都能够获取正确的响应- 不保证数据是最新的

P:分区容错性

​	对实际而言,分区相当于对通信的时限要求.如果不能在限时内达成数据一致性,意味着发生了分区的情况.需要在一致性 与 可用性中选择

## BASE

BA-Basically Available

​	基本可用

S-Soft State

​	软状态

E-Eventual Consistency

​	最终一致