# 容错-Hystrix

## 雪崩效应

在微服务架构中,服务调用链条通常会包含多层服务.微服务中间通过网络进行通信,从而支撑整体系统的应用.因此.在服务中 难免存在依赖关系.

任何微服务都不一定是100%可用,网络往往也很脆弱,因此服务间的调用很可能会失败.

我们常把 由 `基础服务故障` 导致 `级联故障`的现象称之为 `雪崩效应`. 雪崩效应描述的是 因为服务提供者的不可用,导致消费者不可用,并逐渐放大失败的过程.



在常见调用场景中,如果服务 提供者响应非常缓慢.那么消费者对提供者的请求就会被强制等待.直到提供者响应内容 或 超时.在高负载场景下.若不做任何处理,此类问题很可能会导致服务消费者的资源耗尽,甚至引发消费者整体系统的崩溃.

当依赖的服务出现问题时,服务自身不应被拖垮.这是我们需要用 Hystrix 解决的问题.

## 容错的手段

为了防止出现雪崩效应,必须有一个强大的容错机制.至少包含以下两点:

### 设置请求超时时间

正常情况下,一个远程调用一般在几十毫秒内就能得到响应.如果依赖的服务不可用或者网络有问题,那么响应时间就变得很长(几十秒). 一个远程调用一般对应一个线程资源.如果响应太慢,这个线程的资源得不到释放.那么随着调用重复.可能会耗尽系统资源.导致服务不可用.

因此必须为每个请求设置超时,超时立即释放资源.

### 使用断路器

类似于保险丝,当电流过大时,断路器打开跳闸,保护电路安全.当电流过大问题(超载)解决后,断路器关闭,恢复正常电路.

同样对于请求来说,如果对某个服务的请求有大量超时(该服务请求可能不可用),继续访问该服务已经没有意义,只会无畏消耗资源.例如,设置了超时时间为1秒,如果短时间内 有大量的请求无法在1秒内得到相应,就不在请求依赖的服务.

断路器可以理解为,对容易导致错误的操作的代理.这种代理能够统一一段时间内调用失败的次数,并决定继续请求,还是直接返回.

断路器快速失败:在一段时间内检测到许多错误(超时),会在之后的一段时间内,强迫对该服务的调用快速失败.

断路器自动诊断:快速失败后,若依赖的服务是否已经恢复正常,那么就会恢复该服务的调用,自我修复.

当下游服务不正常时打开断路器快速失败,防止雪崩,正常时 恢复请求.

断路器状态:

关闭:请求正常调用,统计失败次数

打开:请求失败率达到一定阈值,不在请求直接返回

半开:打开一段时间后,自动进入半开状态,允许一个请求访问依赖的服务,若调用成功关闭断路器.否则继续保持打开

## Hystrix 实现容错

hystrix 是由Netflix 开源的一个延迟和容错库,用于隔离访问远程系统,服务,或者第三方库,防止**级联失败**,提升系统的**可用性**与**容错性**.

Hystrix主要通过以下几点实现延迟 和 容错:

- 包裹请求:使用HystrixCommand(或HystrixObservableCommand)包裹对依赖的调用逻辑,每个命令在独立线程中执行.(命令模式)
- 跳闸机制:某服务错误率在某时间窗口内 达到一定阈值,Hystrix 可以 自动或者手动 打开熔断器.停止请求该服务
- 资源隔离:Hystrix 为每个依赖都维护了一个小型的线程池(或信号量).若该线程池已经满了,发往该依赖的请求就立即拒绝不在排队等候.从而加速失败判定.
- 监控:Hystrix 可以近乎实时地监控运行指标和配置的变化.例如成功,失败,超时,以及被拒绝的请求.
- 回退机制: 请求失败,超时,被拒绝,或断路器打开时的兜底方法.返回缺省值
- 自我修复:断路器打开后,会自动进入**半开**状态.尝试恢复服务.

## Hystrix 使用

Hystrix 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

启动类添加`@EnableCircuitBreaker`

```java
@SpringBootApplication
@EnableCircuitBreaker
public class Applicatiion {
    public static void main(String[] args) {
        SpringApplication.run(Applicatiion.class);
    }
}
```

使用HystrixCommand 注解添加 fallbackMethod

```java
@Slf4j
@Service
public class UserServiceImpl implements UserService {
    @Autowired
    private RestTemplate restTemplate;

    public UserPO getUserByIdDefault(String id){
        UserPO userPO = new UserPO();
        userPO.setSex("boy");
        userPO.setAge(22);
        userPO.setName("smn");
        return userPO;
    }
	/****/
    @Override
    @HystrixCommand(fallbackMethod = "getUserByIdDefault")
    public UserPO getUserById(String id) {
        UserPO obj = restTemplate.getForObject("http://producer-service/user/"+id,UserPO.class);
        return obj;
    }
}
```

此时,如果下游服务出现问题.那么就会调用 fallbackMethod 方法.

注意:进入fallbackMethod 不一定断路器处于打开状态.在时间窗口中没有达到打开阈值.

通过引入 actuator 

​	可以通过 /health endpoint 来直观了解断路器状态.

​	可以通过 /hystrix.stream endpoint 

引入actuator

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

注意:添加了 actuator 依赖后,访问/actuator/health 并未返回 hystrix 状态信息.需要在`application.yml` 中 添加

```yaml
management:
  endpoint:
    health:
      # 展示所有detail 信息包括 hystrix
      show-details: always
      #sensitive: false
  endpoints:
    web:
      # 展示endpoint 有哪些
      exposure:
        include: health,info,hystrix.stream
```

当断路器未打开时,访问 `/actuator/health `返回如下:

```json
{
	"status": "UP",
	"details": {
		"diskSpace": {
			"status": "UP",
			"details": {
				"total": 148671901696,
				"free": 70056185856,
				"threshold": 10485760
			}
		},
		"refreshScope": {
			"status": "UP"
		},
		"discoveryComposite": {
			"status": "UP",
			"details": {
				"discoveryClient": {
					"status": "UP",
					"details": {
						"services": ["producer-service"]
					}
				},
				"eureka": {
					"description": "Remote status from Eureka server",
					"status": "UP",
					"details": {
						"applications": {
							"PRODUCER-SERVICE": 1
						}
					}
				}
			}
		},
		"hystrix": {
			"status": "UP"
		}
	}
}
```

当停掉下游服务,短时间内多次访问待访问接口后, 访问 `/actuator/health` 返回如下:

```json
{
	"status": "UP",
	"details": {
		"diskSpace": {
			"status": "UP",
			"details": {
				"total": 148671901696,
				"free": 70056185856,
				"threshold": 10485760
			}
		},
		"refreshScope": {
			"status": "UP"
		},
		"hystrix": {
			"status": "CIRCUIT_OPEN",
			"details": {
				"openCircuitBreakers": ["UserServiceImpl::getUserById"]
			}
		}
	}
}
```

注意: Hystrix 的自动恢复 不是自发的.是需要请求调用,并重试下游接口若成功进入到半开状态,并关闭断路器.

## Hystrix 线程隔离策略 与 传播上下文

Hystrix 中的隔离策略有两种: 

- THREAD-线程隔离 使用该方式,HystrixCommand 会在单独的线程上执行,并发请求受线程池中的线程数量的限制. 默认设置
- SEMAPHORE-信号量隔离 使用该方式,HystrixCommand  会在 当前线程进行执行,并发请求受信号量数量限制.

一般来说,只有当调用负载非常高时(每个实例每秒调用数百次),需要使用信号量隔离,因为这种场景下使用THREAD 开销会比较高.信号量隔离一般仅适用于非网络调用的隔离.

可以通过 execution.isolation.strategy 来指定隔离策略

```java
@HystrixCommand(fallbackMethod = "stubMyService",
    commandProperties = {
      @HystrixProperty(name="execution.isolation.strategy", value="SEMAPHORE")
    }
)
...
```

如果发现找不到上下文的运行时异常,可以考虑将隔离策略换成`SEMAPHORE`

一般来说只有当调用负载非常高时

## Hystrix 的Feign 支持

使用`@FeignClient` 注解的`fallback`属性.

```java
@FeignClient(value = "${user.service.name}",fallback = UserRemoteServiceFallback.class)
public interface UserRemoteService {
    @RequestMapping(value = "/user/{id}",method = RequestMethod.GET)
    UserPO getUserById(@PathVariable("id") String id);
}
```

注册fallback 实例

```java
@Component
public class UserRemoteServiceFallback implements UserRemoteService {

    public UserPO getUserById(String id){
        UserPO userPO=new UserPO();
        userPO.setName(id);
        userPO.setAge(11);
        userPO.setSex("aaa");
        return userPO;
    }
}
```

使用`@FeignClient` 注解的`fallbackFactory`属性.看创建回退,并输出回退原因

```java
@FeignClient(contextId = "roleRemoteService",value = "${user.service.name}",fallbackFactory = RoleRemoteServiceFallbackFactory.class)
public interface RoleRemoteService {
    @RequestMapping(value = "/role/getRoleByUserId/{id}",method = RequestMethod.GET)
    RolePO getRoleByUserId(String userId);
}
```

注册fallbackFactory实例

```java
@Slf4j
@Component
public class RoleRemoteServiceFallbackFactory implements FallbackFactory<RoleRemoteService> {
    RoleRemoteServiceFallback fallbackService = new RoleRemoteServiceFallback();
    @Override
    public RoleRemoteService create(Throwable throwable) {
        log.error("调用remote 方法失败,原因如下:",throwable);
        return fallbackService;
    }
    private static class RoleRemoteServiceFallback implements RoleRemoteService{
        @Override
        public RolePO getRoleByUserId(String userId) {
            RolePO rolePO= new RolePO();
            rolePO.setRoleName("fallbackfactory role:"+userId);
            return rolePO;
        }
    }
}
```

## Hystrix Dashboard 

Hystrix Dashboard 可视化监控数据

引入hystrix dashboard 依赖

```xml
<!--添加hystrix dashboard 可视化依赖-->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

在启动类上使用 `@EnableHystrixDashboard` 注解

不用添加特定配置.直接访问 http://ip:port/hystrix即可

此方式需要手动填写服务的 `hystrix.stream` endpoint

## Hystrix Turbine

从eureka中获取服务,并聚合服务的 `hystrix.stream`  endpoint 到 `turbine.stream`

引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
</dependency>
```

配置

```yaml
server:
  port: 8080
eureka:
  client:
    service-url:
      defaultZone: http://centos1:8761/eureka,http://centos2:8761/eureka,http://centos3:8761/eureka,http://centos4:8761/eureka
    #是否注册到eureka
    register-with-eureka: true
    #是否从eureka 上获取注册信息
    fetch-registry: true
turbine:
  app-config: producer-service,consumer-service
  cluster-name-expression: "'default'"
```



## 参考

https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html#production-ready-endpoints-enabling-endpoints  actuator endpoints doc

https://github.com/Netflix/Turbine/wiki/Configuration-(1.x) turbine config

