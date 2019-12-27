# Feign

## 简介

Feign 是一个声明式,模板化的HTTP 客户端,Feign 可以更加便捷优雅的调用HTTP API.

Feign 支持继承.使用继承可以将一些公共操作分组到父接口中.

## maven 依赖

```xml
<!-- openfeign中已经包含了org.springframework:spring-web 的依赖-->
<dependency>    
    <groupId>org.springframework.cloud</groupId>    
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

##  使用FeignClient

```java
@SpringBootApplication
//启用Feign
@EnableFeignClients
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

```java
//指定注册中心中的服务名,会基于此创建一个Ribbon 的负载均衡
//Ribbon 会 将 服务名 解析成 eureka 服务列表中的真实地址
@FeignClient("${service.name}")
public interface StoreClient {
    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    List<Store> getStores();
    
	@RequestMapping(method = RequestMethod.GET, value = "/stores/byName")
    List<Store> getStoresByName(@RequestParam("name") String name);
    
    @RequestMapping(method = RequestMethod.POST, value = "/stores/bymap")
    List<Store> getStoresByMap(@RequestParam Map<String,Object> paramMap);
    
    /**
    * 可以使用 @SpringQueryMap 用于在 GET 请求下,解析对象或map
    */
    @RequestMapping(method = RequestMethod.POST, value = "/stores/byVO")
    List<Store> getStoresByQueryVO(@RequestBody StoreQueryVO queryVO);
    
    @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")
    Store update(@PathVariable("storeId") Long storeId, Store store);
}
```



## Feign配置

Feign 默认配置 是FeignClientsConfiguration  定义了Feign 默认使用的编译器,解码器,契约等.

@FeignClient 的 configuration 属性 可以自定义Feign 的配置,自定义配置会覆盖FeignClientConfiguration配置

Spring Cloud Netflix 默认Feign 配置如下:

| 组件          | 默认实现                                        |
| ------------- | ----------------------------------------------- |
| Decoder       | ResponseEntityDecoder( wraps a `SpringDecoder`) |
| Encoder       | SpringEncoder                                   |
| Logger        | Sl4jLogger                                      |
| Contract      | SpringMvcContract 使用哪些注解来创建Feign 接口  |
| Feign.Builder | HystrixFeign.Builder                            |
| Client        | 启用Ribbon 时LoadBalancerFeignClient,           |

### 特定配置

```yaml
feign:
  client:
    config:
      feignName:#此处是feignName @FeignClient 指定的name
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: full
        errorDecoder: com.example.SimpleErrorDecoder
        retryer: com.example.SimpleRetryer
        #需要实现 RequestInterceptor 接口
        requestInterceptors:
          - com.example.FooRequestInterceptor
          - com.example.BarRequestInterceptor
        decode404: false
        encoder: com.example.SimpleEncoder
        decoder: com.example.SimpleDecoder
        contract: com.example.SimpleContract
```

### 通用配置

```yaml
feign:
  client:
    config:
      default:#default 代表设置所有FeignClient
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: basic
         #需要实现 RequestInterceptor 接口
        requestInterceptors:
          - com.example.FooRequestInterceptor
          - com.example.BarRequestInterceptor
```

### 支持压缩

```properties
feign.compression.request.enabled=true #开启请求压缩
feign.compression.response.enabled=true #开启响应压缩
feign.compression.request.mime-types=text/xml,application/xml,application/json
feign.compression.request.min-request-size=2048 #请求压缩的最小值
```

### 输出log

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: NONE # 决定Feign 输出多少log
# 指定接口日志级别为DEBUG
logging:
  level:
    com.crazy.smn.user.remote.UserRemoteService: DEBUG
```

feign.client.config.default.loggerLevel 可选参数:

- `NONE`  不记录日志 默认
- `BASIC` 仅记录请求方法,url,响应状态码,执行时间
- `HEADERS` 记录`BASIC`内容,并增加请求头 与 响应头
- `FULL` 记录请求 与 响应 的 headers,body,元数据

## Feign Hystrix 支持

