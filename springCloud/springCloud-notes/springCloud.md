# SpringCloud 学习笔记

## 1.1单体架构

单体架构: 将业务的所有功能集中在同一个项目中开发, 打包一个包部署

**优点**:

* **架构简单**
* **部署成本低**

**缺点**:

* **团队协作成本高**
* **系统发布效率低**
* **系统可用性差**



## 1.2 微服务

微服务架构，首先是服务化，就是将单体架构中的功能模块从单体应用中拆分出来，独立部署为多个服务。同时要满足下面的一些特点：

- **单一职责**：一个微服务负责一部分业务功能，并且其核心数据不依赖于其它模块。
- **团队自治**：每个微服务都有自己独立的开发、测试、发布、运维人员，团队人员规模不超过10人（2张披萨能喂饱）
- **服务自治**：每个微服务都独立打包部署，访问自己独立的数据库。并且要做好服务隔离，避免对其它服务产生影响

<img src="..\springCloud-img\微服务.jpg" alt="img" style="zoom: 33%;" />



## 1.3 SpringCloud

微服务拆分以后碰到的各种问题都有对应的解决方案和微服务组件，而SpringCloud框架可以说是目前Java领域最全面的微服务组件的集合了。

<img src="..\springCloud-img\springcloud包含的组件.jpg" alt="img" style="zoom: 50%;" />

目前SpringCloud最新版本为`2022.0.x`版本，对应的SpringBoot版本为`3.x`版本，但它们全部依赖于JDK17，目前在企业中使用相对较少。推荐使用次新版本：Spring Cloud 2021.0.x以及Spring Boot 2.7.x版本。

以**SpringCloudAlibaba**为例

```xml
<spring-cloud.version>2021.0.3</spring-cloud.version>
<spring-cloud-alibaba.version>2021.0.4.0</spring-cloud-alibaba.version> 
<!--spring cloud-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-dependencies</artifactId>
    <version>${spring-cloud.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
<!--spring cloud alibaba-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>${spring-cloud-alibaba.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```



## 2.微服务拆分

> 一般拆分的微服务, 每个服务连接的自己的服务器

### 2.2.服务拆分原则

服务拆分一定要考虑几个问题：

- 什么时候拆？
- 如何拆？

### 2.2.1.什么时候拆

一般情况下，对于一个初创的项目，首先要做的是验证项目的可行性。因此这一阶段的首要任务是敏捷开发，快速产出生产可用的产品，投入市场做验证。为了达成这一目的，该阶段项目架构往往会比较简单，很多情况下会直接采用单体架构，这样开发成本比较低，可以快速产出结果，一旦发现项目不符合市场，损失较小。

如果这一阶段采用复杂的微服务架构，投入大量的人力和时间成本用于架构设计，最终发现产品不符合市场需求，等于全部做了无用功。

所以，对于**大多数小型项目来说，一般是先采用单体架构**，随着用户规模扩大、业务复杂后**再逐渐拆分为****微服务架构**。这样初期成本会比较低，可以快速试错。但是，这么做的问题就在于后期做服务拆分时，可能会遇到很多代码耦合带来的问题，拆分比较困难（**前易后难**）。

而对于一些大型项目，在立项之初目的就很明确，为了长远考虑，在架构设计时就直接选择微服务架构。虽然前期投入较多，但后期就少了拆分服务的烦恼（**前难后易**）。

### 2.2.2.怎么拆

之前我们说过，微服务拆分时**粒度要小**，这其实是拆分的目标。具体可以从两个角度来分析：

- **高内聚**：每个微服务的职责要尽量单一，包含的业务相互关联度高、完整度高。
- **低****耦合**：每个微服务的功能要相对独立，尽量减少对其它微服务的依赖，或者依赖接口的稳定性要强。

**高内聚**首先是**单一职责，**但不能说一个微服务就一个接口，而是要保证微服务内部业务的完整性为前提。目标是当我们要修改某个业务时，最好就只修改当前微服务，这样变更的成本更低。

一旦微服务做到了高内聚，那么服务之间的**耦合度**自然就降低了。

当然，微服务之间不可避免的会有或多或少的业务交互，比如下单时需要查询商品数据。这个时候我们不能在订单服务直接查询商品数据库，否则就导致了数据耦合。而应该由商品服务对应暴露接口，并且一定要保证微服务对外**接口的稳定性**（即：尽量保证接口外观不变）。虽然出现了服务间调用，但此时无论你如何在商品服务做内部修改，都不会影响到订单微服务，服务间的耦合度就降低了。

明确了拆分目标，接下来就是拆分方式了。我们在做服务拆分时一般有两种方式：

- **纵向**拆分
- **横向**拆分

所谓**纵向拆分**，就是按照项目的功能模块来拆分。例如黑马商城中，就有用户管理功能、订单管理功能、购物车功能、商品管理功能、支付功能等。那么按照功能模块将他们拆分为一个个服务，就属于纵向拆分。这种拆分模式可以尽可能提高服务的内聚性。

而**横向拆分**，是看各个功能模块之间有没有公共的业务部分，如果有将其抽取出来作为通用服务。例如用户登录是需要发送消息通知，记录风控数据，下单时也要发送短信，记录风控数据。因此消息发送、风控数据记录就是通用的业务功能，因此可以将他们分别抽取为公共服务：消息中心服务、风控管理服务。这样可以提高业务的复用性，避免重复开发。同时通用业务一般接口稳定性较强，也不会使服务之间过分耦合。

### 2.3 服务调用

在拆分的时候，我们发现一个问题：就是购物车业务中需要查询商品信息，但商品信息查询的逻辑全部迁移到了`item-service`服务，导致我们无法查询。

最终结果就是查询到的购物车数据不完整，因此要想解决这个问题，我们就必须改造其中的代码，把原本本地方法调用，改造成跨微服务的远程调用（RPC，即**R**emote **P**roduce **C**all）。

> 不同服务不能直接通过注入类调用:
>
> 1.**解耦原则**
>
> 微服务的一个核心理念是“松耦合、高内聚”，每个服务都应独立运行，独立部署。
> 如果直接注入其他服务的类，形成代码上的直接依赖，以下问题将随之而来：
>
> - 服务之间产生强耦合，一个服务的修改可能会影响其他服务。
> - 无法独立部署，多个服务必须共享相同的代码库，失去了微服务架构的独立性。
> - 
>
> 2.**网络通信的分布式架构**
>
> 微服务通常部署在不同的主机或容器中，甚至可能分布在不同的网络节点上。通过注入直接调用其他服务的方法会出现问题：
>
> - 这些服务运行在不同的进程中，不像单体应用那样可以直接通过方法调用。
>
> 3.**服务实例与负载均衡**
>
> 微服务架构中，多个实例（副本）可能会部署在不同的节点上。例如：
>
> - 服务 A 调用服务 B 时，B 服务可能有多个实例。
> - 微服务框架（如 Spring Cloud OpenFeign、LoadBalancer）会自动完成负载均衡，让调用请求分发到可用的实例。 如果直接注入 `Service`，将无法完成实例选择和负载均衡。



```java
// 比如在cart-service中 调用item-service的业务
public void InCartService(){
    itemService.listByids(ids);
    // 跨模块是无法直接调用
}
```

> 无法直接调用但是我们可以想到, 一些接口可以通过网络请求获取数据的.
>
> 所以这就需要springCloud提供的远程调用  **``OpenFeign``**

### 2.4 远程调用

Spring给我们提供了一个RestTemplate的API，可以方便的实现Http请求的发送。

> org.springframework.web.client public class RestTemplate
>
> extends InterceptingHttpAccessor
>
> implements RestOperations
>
> \----------------------------------------------------------------------------------------------------------------
>
> 同步客户端执行HTTP请求，在底层HTTP客户端库(如JDK HttpURLConnection、Apache HttpComponents等)上公开一个简单的模板方法API。RestTemplate通过HTTP方法为常见场景提供了模板，此外还提供了支持不太常见情况的通用交换和执行方法。 RestTemplate通常用作共享组件。然而，它的配置不支持并发修改，因此它的配置通常是在启动时准备的。如果需要，您可以在启动时创建多个不同配置的RestTemplate实例。如果这些实例需要共享HTTP客户端资源，它们可以使用相同的底层ClientHttpRequestFactory。 注意:从5.0开始，这个类处于维护模式，只有对更改和错误的小请求才会被接受。请考虑使用org.springframework.web.react .client. webclient，它有更现代的API，支持同步、异步和流场景。  
>
> \----------------------------------------------------------------------------------------------------------------
>
> 自: 3.0 参见: HttpMessageConverter, RequestCallback, ResponseExtractor, ResponseErrorHandler

其中提供了大量的方法，方便我们发送Http请求，例如： 

<img src="..\springCloud-img\RestTemplate.jpg" alt="img" style="zoom: 50%;" />

可以看到常见的Get、Post、Put、Delete请求都支持，如果请求参数比较复杂，还可以使用exchange方法来构造请求。

#### 2.4.1 使用方法

> 这里只是进行简单的示范 直接使用了RestTemplate 后续有OpenFeign的使用

可以建一个config目录,建立个java文件如下

```java
@Configuration
public class RemoteCallConfig{
    
    // 直接可以通过依赖注入RestTemplate
    @Bean
    public RestTemplate restTemplate(){
        return new RestTempalte();
    }
}
```

接下来，我们修改`cart-service`中的`com.hmall.cart.service.impl.``CartServiceImpl`的`handleCartItems`方法，发送http请求到`item-service`：

```java
// 单体架构的时候 直接调用 itemService.queryByIds(ids);
// 微服务用请求的方式
// 先注入
@Autowired
private RestTemplate restTemplate;

ResponseEntity<List<ItemDto>> response = restTemplate.exchange(
	"http://localhost:8081/items?ids={ids}", // 请求路径  
    // 用{ids} 代替请求参数, 然后后面再填入参数值
    HttpMethod.GET, // 请求方法
    null, // 请求实体 
    new ParameterizedTypeReference<List<ItemDto>>(){}, 
    // 返回值类型,因为类型是字节码的形式 集合没有字节码的形式所以用ParameterizedTypeReference
    CollUtils.join(ids, ",")  //填写请求参数
)
// 处理结果
List<ItemDto> items = null;
if (response.getStatusCode().is2xxSuccessfull()){
    //请求成功
    items = response.getBody();
} 
```



### 2.5.总结

什么时候需要拆分微服务？

- 如果是创业型公司，最好先用单体架构快速迭代开发，验证市场运作模型，快速试错。当业务跑通以后，随着业务规模扩大、人员规模增加，再考虑拆分微服务。
- 如果是大型企业，有充足的资源，可以在项目开始之初就搭建微服务架构。

如何拆分？

- 首先要做到**高内聚、低耦合**
- 从拆分方式来说，有**横向拆分和纵向拆分**两种。**纵向就是按照业务功能模块**，**横向则是拆分通用性业务，提高复用性**

服务拆分之后，不可避免的会出现跨微服务的业务，此时微服务之间就需要进行**远程调用**。微服务之间的远程调用被称为**RPC**，即**远程过程调用。****RPC**的实现方式有很多，比如：

- 基于**Http**协议
- 基于**Dubbo**协议

我们课堂中使用的是**Http**方式，这种方式不关心服务提供者的具体技术实现，只要对外暴露**Http**接口即可，更符合微服务的需要。

Java发送**http**请求可以使用Spring提供的**RestTemplate**，使用的基本步骤如下：

- 注册**RestTemplate**到Spring容器
- 调用**RestTemplate**的API发送请求，常见方法有：
  - **getForObject**：发送Get请求并返回指定类型对象
  - **PostForObject**：发送Post请求并返回指定类型对象
  - **put**：发送PUT请求
  - **delete**：发送Delete请求
  - **exchange**：发送任意类型请求，返回ResponseEntity