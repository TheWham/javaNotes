# OpenFeign

> 利用RestTemplate实现了服务的远程调用。但是远程调用的代码太复杂了
>
> 而且这种调用方式，与原本的本地方法调用差异太大，编程时的体验也不统一，一会儿远程调用，一会儿本地调用。
>
> 因此，我们必须想办法改变远程调用的开发模式，让**远程调用像本地方法调用一样简单**。而这就要用到OpenFeign组件了。
>
> 其实远程调用的关键点就在于四个：
>
> - 请求方式
> - 请求路径
> - 请求参数
> - 返回值类型
>
> 所以，OpenFeign就利用SpringMVC的相关注解来声明上述4个参数，然后基于动态代理帮我们生成远程调用的代码，而无需我们手动再编写，非常方便。

## 1. 使用openFeign

### 1.1 引入依赖

```xml
  <!--openFeign-->
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-openfeign</artifactId>
  </dependency>
  <!--负载均衡器-->
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-loadbalancer</artifactId>
  </dependency>
```

### 1.2.启用OpenFeign 

在启动类添加 @EnableFeignClients注解

```java
@EnableFeignClients
@SpringBootApplication
public class application{]
    public static void main(String[] args){
        SpringApplication.run(application.class, args);
    }
}
```

### 1.3.编写OpenFeign客户端

```java
@FeignClient("item-service") // item-service 为服务名称 对应xml配置中的 spring.application.name 配置
public interface ItemClient {

    @GetMapping("/items") // 请求路径
    List<ItemDTO> queryItemByIds(@RequestParam("ids") Collection<Long> ids); //只需定义请求接口即可 底层会通过代理发起http请求
}
```

> 这里只需要声明接口，无需实现方法。接口中的几个关键信息：
>
> - `@FeignClient("item-service")` ：声明服务名称
> - `@GetMapping` ：声明请求方式
> - `@GetMapping("/items")` ：声明请求路径
> - `@RequestParam("ids") Collection<Long> ids` ：声明请求参数
> - `List<ItemDTO>` ：返回值类型

### 1.4.使用FeignClient

```java
@Autowired
private ItemClient itemClient;

public void testMethod(){
    List<Long> testIds;
    List<ItemDTO> items = itemClient.queryItemByIds(ids);
}
```

### 2.连接池

> Feign底层发起http请求，依赖于其它的框架。其底层支持的http客户端实现包括：
>
> - HttpURLConnection：默认实现，不支持连接池
> - Apache HttpClient ：支持连接池
> - OKHttp：支持连接池

#### 2.1 引入依赖

```xml
<!--OK http 的依赖 -->
<dependency>
  <groupId>io.github.openfeign</groupId>
  <artifactId>feign-okhttp</artifactId>
</dependency>
```

#### 2.2.开启连接池

在需要发起调用的服务添加

```yaml
feign:
  okhttp:
    enabled: true # 开启OKHttp功能
```

