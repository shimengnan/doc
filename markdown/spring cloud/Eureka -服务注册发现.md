# Eureka -服务注册发现

## 版本与依赖

<spring-cloud-release> Greenwich.SR4

<spring-boot-dependencies> 2.1.10.RELEASE

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
    <dependency>
        <groupId>com.google.code.gson</groupId>
        <artifactId>gson</artifactId>
    </dependency>
</dependencies>
```

## 构建 单机 eureka server

配置

```yaml
spring:
  application:
    name: eurekaserver
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka

```

## 构建高可用 eureka server

```yaml
spring:
  application:
    name: eurekaserver
server:
  port: 8761
eureka:
  client:
    #是否将自己注册到eureka server
#    registerWithEureka: false
    #是否从eureka server 获取注册信息
#    fetchRegistry: false
    serviceUrl:
      defaultZone: http://192.168.140.129:8761/eureka/,http://192.168.140.130:8761/eureka/,http://192.168.140.133:8761/eureka/,http://192.168.140.132:8761/eureka/
  instance:
  	#当前ip
    hostname:  ${spring.cloud.client.ip-address}
```

## Eureka 自我保护模式

默认情况下,如果eureka server 在一定时间内没有接收到某个微服务实例的心跳,eureka server 将会注销该实例(默认90s).但如果是网络故障,服务本应是健康的.那么此时不应注销这个服务.

eureka 自我保护模式,当eureka server 节点短时间内丢失过多客户端时(可能发生网络分区故障),那么这个节点就会进入自我保护模式.一旦进入该模式,eureka 将会保护服务注册表中的信息,不在删除服务注册表中的数据.当网络故障恢复后.该eureka server 节点会自动退出自我保护模式.
```yaml
eureka:
  server:
    enable-self-preservation: false #禁用自我保护模式
```

##  多网卡下的IP选择

对于多网卡的服务器,可能会针对特定某一块网卡提供服务

```yaml
spring:
  cloud:
    inetutils:
      #忽略指定的网卡 docker0 以及以veth 开头的网卡
      ignored-interfaces:
        - docker0
        - veth.*
eureka:
  instance:
    prefer-ip-address: true
```

## Eureka 的健康检查

Eureka 中 服务的status 状态 有:

- UP 							应用状态正常-- 只有此状态能提供服务
- DOWN
- OUT_OF_SERVICE
- UNKNOWN

默认情况下 eureka server 与 eureka client 的心跳保持正常,应用程序就会始终保持UP 状态.但是上述机制并不能完全反应服务状态.(应用数据库连接异常等)

这时可以通过Spring Boot Actuator 提供的 /health 节点.来判断服务的健康信息.

可以通过如下配置,开启eureka server 中的 健康检查

```yaml
eureka:
  client:
  	#注意,此配置只能配置在application.yml中,如果配置到bootstrap.yml 中可能会导致状态异常
    healthcheck:
      enabled: true
```



可以通过实现 `com.netflix.appinfo.HealthCheckHandler`中的getStatus方法 来细粒度的控制健康检查

