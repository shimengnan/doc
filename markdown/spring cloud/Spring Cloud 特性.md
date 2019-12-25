# Spring Cloud 特性

- Distributed/versioned configuration 分布式配置
- Service registration and discovery    服务注册发现
- Routing 路由
- Service-to-service calls 服务调用
- Load balancing 负载均衡
- Circuit Breakers 熔断保护
- Distributed messaging 分布式消息

Spring cloud 的许多特性来自于Spring boot,除此之外,其余特性被划分成两个类库:

Spring Cloud Context 与 Spring Cloud Commons

## Spring Cloud Context

提供了一些常用工具类 以及基于`ApplicationContext`  的特性服务,例如

>  bootstrap context 引导上下文
>
> encryption 加密
>
> refresh scope 刷新作用域
>
> environment endpoints 环境支持

## Spring Cloud Commons

一系列的抽象与通用类的不同实现,例如

> Spring Cloud Netflix
>
> Spring Cloud Consul

