# RabbitMQ 高级笔记整理版

---

## 目录

1. [发送者可靠性](#一发送者可靠性)
   - [发送者重连](#11-发送者重连)
   - [发送者确认](#12-发送者确认)
2. [MQ 的可靠性](#二mq-的可靠性)
   - [数据持久化](#21-数据持久化)
   - [LazyQueue 惰性队列](#22-lazyqueue-惰性队列)
3. [消费者的可靠性](#三消费者的可靠性)
   - [消费者确认机制](#31-消费者确认机制)
   - [失败重试机制](#32-失败重试机制)
   - [失败处理策略](#33-失败处理策略)
   - [业务幂等性](#34-业务幂等性)
   - [兜底方案](#35-兜底方案)
4. [延迟消息](#四延迟消息)
   - [死信交换机和延迟消息](#41-死信交换机和延迟消息)
   - [DelayExchange 插件](#42-delayexchange-插件)
5. [面试速记版](#五面试速记版)

---

# 一、发送者可靠性

发送者可靠性主要解决的问题是：**生产者发送消息到 RabbitMQ 的过程中，消息会不会因为网络、交换机、路由等问题丢失。**

---

## 1.1 发送者重连

有时由于网络波动，生产者连接 RabbitMQ 可能失败。可以通过 Spring AMQP 配置连接超时和发送重试机制。

### 配置示例

```yml
spring:
  rabbitmq:
    host: 8.136.108.248
    port: 5672
    virtual-host: /
    username: pinkboy
    password: 123456
    connection-timeout: 1s

    template:
      retry:
        enabled: true
        initial-interval: 1000ms
        multiplier: 1
        max-attempts: 3
```

### 参数说明

| 配置项 | 作用 |
|---|---|
| `connection-timeout` | 设置 RabbitMQ 连接超时时间 |
| `template.retry.enabled` | 是否开启发送失败重试 |
| `initial-interval` | 第一次重试的等待时间 |
| `multiplier` | 重试间隔倍数 |
| `max-attempts` | 最大重试次数 |

### 注意点

Spring AMQP 的发送重试是**阻塞式重试**。

也就是说，在多次重试和等待期间，当前发送线程会被阻塞。如果业务对响应速度要求较高，不建议把重试次数和等待时间设置得太大。

### 使用建议

- 普通业务可以不开启，或者只设置较短的重试时间。
- 对可靠性要求较高的业务可以开启，但建议异步发送。
- 不要无限重试，否则可能拖垮业务线程。

---

## 1.2 发送者确认

生产者把消息发给 RabbitMQ 后，理论上还可能出现以下问题：

1. 消息到达 MQ 后，MQ 内部处理异常。
2. 消息发到了 MQ，但找不到对应的 Exchange。
3. 消息发到了 Exchange，但没有匹配的 Queue，路由失败。

为了解决这些问题，RabbitMQ 提供了两类生产者确认机制：

| 机制 | 作用 |
|---|---|
| Publisher Confirm | 判断消息是否成功到达 Exchange |
| Publisher Return | 判断消息从 Exchange 到 Queue 的路由是否失败 |

---

### 1.2.1 Confirm 和 Return 的返回情况

| 场景 | 返回结果 |
|---|---|
| 消息到达 MQ，但路由失败 | 触发 Return，同时 Confirm 返回 ACK |
| 临时消息成功入队 | Confirm 返回 ACK |
| 持久消息成功入队并完成持久化 | Confirm 返回 ACK |
| 交换机不存在或 MQ 内部异常 | Confirm 返回 NACK |

简单理解：

- **ACK**：消息投递成功。
- **NACK**：消息投递失败。
- **Return**：消息到了交换机，但是没有路由到队列。

---

### 1.2.2 开启生产者确认配置

```yml
logging:
  pattern:
    dateformat: MM-dd HH:mm:ss:SSS
  level:
    com.itheima: debug

spring:
  rabbitmq:
    host: 8.136.108.248
    port: 5672
    virtual-host: /
    username: pinkboy
    password: 123456
    connection-timeout: 1s

    template:
      retry:
        enabled: true
        multiplier: 2

    publisher-confirm-type: correlated
    publisher-returns: true
```

`publisher-confirm-type` 有三种模式：

| 模式 | 说明 |
|---|---|
| `none` | 关闭 Confirm 机制 |
| `simple` | 同步阻塞等待 MQ 回执 |
| `correlated` | 异步回调返回回执，推荐使用 |

---

### 1.2.3 配置 ReturnCallback

每个 `RabbitTemplate` 只能配置一个 `ReturnCallback`，所以一般在配置类中统一设置。

```java
package com.itheima.publisher.config;

import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.ReturnedMessage;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.context.annotation.Configuration;

import javax.annotation.PostConstruct;

@Slf4j
@AllArgsConstructor
@Configuration
public class MqConfig {

    private final RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init() {
        rabbitTemplate.setReturnsCallback(returned -> {
            log.error("触发 return callback");
            log.debug("exchange: {}", returned.getExchange());
            log.debug("routingKey: {}", returned.getRoutingKey());
            log.debug("message: {}", returned.getMessage());
            log.debug("replyCode: {}", returned.getReplyCode());
            log.debug("replyText: {}", returned.getReplyText());
        });
    }
}
```

---

### 1.2.4 配置 ConfirmCallback

```java
package publisher;

import com.itheima.publisher.PublisherApplication;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.util.concurrent.ListenableFutureCallback;

@Slf4j
@SpringBootTest(classes = PublisherApplication.class)
public class SpringAmqpTest {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Test
    void testPublisherConfirm() throws InterruptedException {
        CorrelationData cd = new CorrelationData();

        cd.getFuture().addCallback(new ListenableFutureCallback<CorrelationData.Confirm>() {
            @Override
            public void onFailure(Throwable ex) {
                log.error("send message fail", ex);
            }

            @Override
            public void onSuccess(CorrelationData.Confirm result) {
                if (result.isAck()) {
                    log.debug("发送消息成功，收到 ack");
                } else {
                    log.error("发送消息失败，收到 nack，原因：{}", result.getReason());
                }
            }
        });

        rabbitTemplate.convertAndSend("amq.direct", "red", "hello", cd);

        Thread.sleep(2000);
    }
}
```

---

### 1.2.5 CorrelationData 的作用

`CorrelationData` 中有两个核心内容：

| 内容 | 作用 |
|---|---|
| `id` | 消息唯一标识，避免多个消息回执混淆 |
| `Future` | 用来异步接收 MQ 的回执结果 |

---

### 1.2.6 测试现象总结

| 测试情况 | 结果 |
|---|---|
| Exchange 正确，RoutingKey 错误 | 触发 ReturnCallback，同时收到 ACK |
| Exchange 正确，RoutingKey 正确 | 不触发 ReturnCallback，只收到 ACK |
| Exchange 错误 | 收到 NACK |

### 使用建议

开启生产者确认会影响 RabbitMQ 性能，一般不建议所有业务都开启。

适合开启的场景：

- 支付通知
- 订单状态同步
- 库存扣减
- 重要业务事件通知

如果只是普通日志、行为记录、非核心异步任务，可以不一定开启 Confirm。

---

# 二、MQ 的可靠性

消息到达 RabbitMQ 后，如果 MQ 没有及时保存，也可能丢失消息。

RabbitMQ 默认会把消息放在内存中以降低延迟，但这会带来两个问题：

1. MQ 宕机后，内存中的消息可能丢失。
2. 消费者故障或消费过慢时，消息积压会导致 MQ 内存压力过大。

---

## 2.1 数据持久化

为了提升消息可靠性，需要配置三类持久化：

1. 交换机持久化
2. 队列持久化
3. 消息持久化

---

### 2.1.1 交换机持久化

创建 Exchange 时，将 `Durability` 设置为 `Durable`。

也可以通过代码创建持久化交换机：

```java
@Bean
public DirectExchange directExchange() {
    return new DirectExchange("hmall.direct", true, false);
}
```

---

### 2.1.2 队列持久化

创建 Queue 时，将 `Durability` 设置为 `Durable`。

代码示例：

```java
@Bean
public Queue queue() {
    return new Queue("simple.queue", true);
}
```

---

### 2.1.3 消息持久化

消息也要设置为持久化消息。

Spring AMQP 默认发送的消息一般是持久化的，也可以显式指定：

```java
Message message = MessageBuilder
        .withBody("hello".getBytes(StandardCharsets.UTF_8))
        .setDeliveryMode(MessageDeliveryMode.PERSISTENT)
        .build();

rabbitTemplate.convertAndSend("hmall.direct", "red", message);
```

---

### 2.1.4 持久化和 ACK 的关系

如果同时开启：

- 消息持久化
- 生产者 Confirm

那么 MQ 会在消息完成持久化之后才返回 ACK。

这样可靠性更高，但由于持久化涉及磁盘 IO，ACK 会有一定延迟，所以建议 Confirm 使用异步回调方式。

---

## 2.2 LazyQueue 惰性队列

默认情况下，RabbitMQ 会优先把消息放在内存中，以提升收发性能。

但在以下场景中，容易出现消息堆积：

1. 消费者宕机或网络异常。
2. 生产者发送速度突然增加。
3. 消费者业务处理阻塞，消费速度跟不上。

当消息大量堆积时，RabbitMQ 内存占用会升高。达到内存预警后，RabbitMQ 会把内存消息刷盘，这个过程称为 `PageOut`。`PageOut` 可能阻塞队列进程，导致生产者发送请求也被阻塞。

---

### 2.2.1 LazyQueue 的特点

LazyQueue，也就是惰性队列，特点如下：

1. 消息到达后直接写入磁盘，而不是优先放入内存。
2. 消费者真正消费时，再从磁盘读取到内存。
3. 适合处理大量消息积压场景。
4. 支持百万级消息存储。

---

### 2.2.2 使用场景

适合使用 LazyQueue 的场景：

- 消费者处理慢
- 消息可能大量堆积
- 峰值流量明显
- 对延迟没有极致要求
- 更看重消息可靠性和堆积能力

不适合：

- 极低延迟实时消息
- 消息量很少，没有积压风险的业务

---

### 2.2.3 通过 Bean 配置 LazyQueue

```java
@Bean
public Queue lazyQueue() {
    return QueueBuilder
            .durable("lazy.queue")
            .lazy()
            .build();
}
```

---

### 2.2.4 通过注解配置 LazyQueue

```java
@RabbitListener(queuesToDeclare = @Queue(
        name = "lazy.queue",
        durable = "true",
        arguments = @Argument(name = "x-queue-mode", value = "lazy")
))
public void listenLazyQueue(String msg) {
    log.info("接收到 lazy.queue 的消息：{}", msg);
}
```

---

### 2.2.5 更新已有队列为 Lazy 模式

可以使用 RabbitMQ Policy 给已有队列设置 Lazy 模式：

```bash
rabbitmqctl set_policy Lazy "^lazy-queue$" '{"queue-mode":"lazy"}' --apply-to queues
```

命令说明：

| 部分 | 含义 |
|---|---|
| `rabbitmqctl` | RabbitMQ 命令行工具 |
| `set_policy` | 设置策略 |
| `Lazy` | 策略名称 |
| `^lazy-queue$` | 匹配队列名称的正则表达式 |
| `{"queue-mode":"lazy"}` | 设置队列模式为 lazy |
| `--apply-to queues` | 策略作用于队列 |

---

# 三、消费者的可靠性

消费者可靠性主要解决的问题是：

> 消息投递给消费者后，消费者是否真的处理成功？

消息被投递给消费者，并不代表业务一定执行成功，可能出现：

1. 投递过程中网络异常。
2. 消费者收到消息后宕机。
3. 消费者业务代码异常。
4. 消费者处理超时或阻塞。

因此 RabbitMQ 需要知道消费者的处理结果，失败时才能重新投递。

---

## 3.1 消费者确认机制

RabbitMQ 提供 Consumer Acknowledgement，即消费者确认机制。

消费者处理完消息后，要告诉 RabbitMQ 处理结果。

### 三种回执

| 回执 | 含义 |
|---|---|
| `ack` | 成功处理，RabbitMQ 删除消息 |
| `nack` | 处理失败，RabbitMQ 可以重新投递 |
| `reject` | 拒绝消息，RabbitMQ 删除消息 |

一般 `reject` 用得较少，除非消息格式本身有问题，不值得继续重试。

---

### 3.1.1 Spring AMQP 的三种 ACK 模式

| 模式 | 说明 |
|---|---|
| `none` | 不处理。消息投递给消费者后立即 ack，不安全 |
| `manual` | 手动 ack，需要业务代码调用 API，灵活但有侵入 |
| `auto` | 自动 ack，业务正常执行返回 ack，异常时根据异常类型返回 nack 或 reject |

---

### 3.1.2 none 模式配置

```yml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: none
```

如果配置为 `none`，即使消费者业务抛异常，消息也可能已经被 RabbitMQ 删除，所以不建议使用。

---

### 3.1.3 模拟消费者异常

```java
@RabbitListener(queues = "simple.queue")
public void listenSimpleQueueMessage(String msg) {
    log.info("spring 消费者接收到消息：{}", msg);

    if (true) {
        throw new RuntimeException("故意抛出异常");
    }

    log.info("消息处理完成");
}
```

---

### 3.1.4 auto 模式配置

```yml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: auto
```

`auto` 模式下：

- 业务正常执行：自动 ack。
- 业务异常：自动 nack，消息重新入队。
- 消息转换、校验等异常：可能 reject，避免无意义重试。

---

## 3.2 失败重试机制

如果消费者处理失败，默认情况下消息可能不断重新入队，再不断投递给消费者。

如果这条消息一直处理失败，就会出现无限重试，造成 MQ 压力升高。

Spring AMQP 提供了消费者本地重试机制：  
消费者异常时，先在本地重试，而不是立刻让消息重新进入 MQ 队列。

---

### 3.2.1 开启消费者失败重试

```yml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: auto
        retry:
          enabled: true
          initial-interval: 1000ms
          multiplier: 1
          max-attempts: 3
          stateless: true
```

### 参数说明

| 配置项 | 作用 |
|---|---|
| `enabled` | 开启消费者失败重试 |
| `initial-interval` | 第一次失败后的等待时间 |
| `multiplier` | 下一次等待时长倍数 |
| `max-attempts` | 最大重试次数 |
| `stateless` | 是否无状态重试；如果业务中有事务，可考虑设为 false |

---

### 3.2.2 本地重试结论

开启本地重试后：

1. 消费者处理异常时，不会马上把消息 requeue 到 MQ。
2. 消息会在消费者本地进行多次重试。
3. 达到最大重试次数后，默认会抛出 `AmqpRejectAndDontRequeueException`。
4. 默认情况下，Spring AMQP 最终会返回 `reject`，消息被丢弃。

---

## 3.3 失败处理策略

本地重试耗尽后，如果消息直接丢弃，对于重要业务来说不合适。

Spring AMQP 提供 `MessageRecoverer` 接口，用于定义重试耗尽后的处理策略。

### 三种实现

| 实现类 | 作用 |
|---|---|
| `RejectAndDontRequeueRecoverer` | 重试耗尽后 reject，直接丢弃，默认策略 |
| `ImmediateRequeueMessageRecoverer` | 重试耗尽后 nack，重新入队 |
| `RepublishMessageRecoverer` | 重试耗尽后，把失败消息投递到指定交换机 |

推荐使用 `RepublishMessageRecoverer`：  
失败消息进入专门的异常队列，后续可以人工排查和补偿。

---

### 3.3.1 定义异常交换机和异常队列

```java
@Bean
public DirectExchange errorMessageExchange() {
    return new DirectExchange("error.direct");
}

@Bean
public Queue errorQueue() {
    return new Queue("error.queue", true);
}

@Bean
public Binding errorBinding(Queue errorQueue, DirectExchange errorMessageExchange) {
    return BindingBuilder
            .bind(errorQueue)
            .to(errorMessageExchange)
            .with("error");
}
```

---

### 3.3.2 配置 RepublishMessageRecoverer

```java
@Bean
public MessageRecoverer republishMessageRecoverer(RabbitTemplate rabbitTemplate) {
    return new RepublishMessageRecoverer(rabbitTemplate, "error.direct", "error");
}
```

---

### 3.3.3 完整配置类示例

```java
package com.itheima.consumer.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.rabbit.retry.MessageRecoverer;
import org.springframework.amqp.rabbit.retry.RepublishMessageRecoverer;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConditionalOnProperty(
        name = "spring.rabbitmq.listener.simple.retry.enabled",
        havingValue = "true"
)
public class ErrorMessageConfig {

    @Bean
    public DirectExchange errorMessageExchange() {
        return new DirectExchange("error.direct");
    }

    @Bean
    public Queue errorQueue() {
        return new Queue("error.queue", true);
    }

    @Bean
    public Binding errorBinding(
            Queue errorQueue,
            DirectExchange errorMessageExchange
    ) {
        return BindingBuilder
                .bind(errorQueue)
                .to(errorMessageExchange)
                .with("error");
    }

    @Bean
    public MessageRecoverer republishMessageRecoverer(
            RabbitTemplate rabbitTemplate
    ) {
        return new RepublishMessageRecoverer(
                rabbitTemplate,
                "error.direct",
                "error"
        );
    }
}
```

---

## 3.4 业务幂等性

幂等性指的是：

> 同一个业务执行一次和执行多次，对最终业务状态的影响是一样的。

例如：

- 根据 ID 删除数据
- 查询数据

这些通常是幂等的。

但有些业务不是幂等的：

- 取消订单并恢复库存，重复执行会导致库存重复增加。
- 退款业务重复执行，会造成资金损失。
- MQ 消息重复投递，可能导致订单状态被错误覆盖。

---

### 3.4.1 为什么 MQ 消费要考虑幂等？

例如支付成功后，支付服务发送 MQ 消息通知交易服务修改订单状态为已支付。

可能出现这种情况：

1. 用户支付成功，消息发送到交易服务。
2. 交易服务把订单改为已支付。
3. 生产者没有收到确认，之后又重新发送了一条相同消息。
4. 在新消息消费之前，用户发起退款，订单变为已退款。
5. 重复消息又被消费，把订单重新改成已支付。

这样就出现业务错误。

---

### 3.4.2 幂等方案一：唯一消息 ID

思路：

1. 每条消息生成一个唯一 ID。
2. 消费者处理成功后，把消息 ID 保存到数据库。
3. 下次收到相同消息时，先查这个消息 ID 是否已经存在。
4. 如果存在，说明是重复消息，直接放弃处理。

Spring AMQP 的消息转换器可以自动创建 MessageID。

```java
@Bean
public MessageConverter messageConverter() {
    Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter();
    converter.setCreateMessageIds(true);
    return converter;
}
```

缺点：

- 需要额外建表或额外字段保存消息 ID。
- 数据库会多一次查询或写入。
- 对原业务有一定侵入。

---

### 3.4.3 幂等方案二：业务状态判断

更推荐基于业务状态做判断。

例如：订单从“未支付”改为“已支付”。  
那么消费消息时，只允许状态为“未支付”的订单被修改。

---

### 3.4.4 普通写法

```java
@Override
public void markOrderPaySuccess(Long orderId) {
    Order old = getById(orderId);

    if (old == null || old.getStatus() != 1) {
        return;
    }

    Order order = new Order();
    order.setId(orderId);
    order.setStatus(2);
    order.setPayTime(LocalDateTime.now());

    updateById(order);
}
```

问题：

判断和更新是两个步骤，极端并发下可能有线程安全问题。

---

### 3.4.5 推荐写法：条件更新保证原子性

```java
@Override
public void markOrderPaySuccess(Long orderId) {
    lambdaUpdate()
            .set(Order::getStatus, 2)
            .set(Order::getPayTime, LocalDateTime.now())
            .eq(Order::getId, orderId)
            .eq(Order::getStatus, 1)
            .update();
}
```

对应 SQL：

```sql
UPDATE `order`
SET status = ?, pay_time = ?
WHERE id = ? AND status = 1;
```

这样可以保证：

- 只有订单状态为未支付时才更新。
- 已支付、已退款、已取消的订单都不会被重复修改。
- 判断和更新合并在一条 SQL 中，更安全。

---

## 3.5 兜底方案

即使使用了：

- 生产者确认
- MQ 持久化
- 消费者确认
- 消费者重试
- 失败消息队列
- 业务幂等

也很难说消息 100% 不会出问题。

因此还需要业务兜底。

---

### 3.5.1 支付状态同步的兜底思路

如果支付服务发送 MQ 通知失败，交易服务就主动查询支付状态。

思路：

1. 用户支付成功后，支付服务通过 MQ 通知交易服务。
2. 交易服务消费消息，修改订单状态。
3. 如果 MQ 通知失败，交易服务通过定时任务主动查询支付状态。
4. 如果查到订单已经支付，则主动更新订单状态。

---

### 3.5.2 为什么需要定时任务？

交易服务不知道用户什么时候完成支付，所以不能只查一次。

通常做法：

- 每隔一段时间扫描未支付订单。
- 主动调用支付服务查询支付状态。
- 如果已支付，则更新订单状态。
- 如果超时未支付，则取消订单。

例如每隔 20 秒查询一次。

---

### 3.5.3 最终一致性总结

支付服务与交易服务之间的订单状态一致性可以这样保证：

1. 用户支付成功后，支付服务通过 MQ 通知交易服务。
2. 生产者确认、消费者确认、失败重试、异常队列提高消息可靠性。
3. 消费者业务通过状态判断保证幂等。
4. 交易服务通过定时任务主动查询支付状态，作为 MQ 通知失败后的兜底方案。
5. 最终实现订单支付状态的最终一致性。

---

# 四、延迟消息

延迟消息解决的问题是：

> 某个任务不是立刻执行，而是在指定时间之后再执行。

典型场景：

- 下单 30 分钟未支付，自动取消订单。
- 支付超时释放库存。
- 优惠券到期提醒。
- 延迟发送通知。
- 用户操作后延迟统计数据。

---

## 4.1 死信交换机和延迟消息

RabbitMQ 实现延迟消息有两种常见方案：

1. 死信交换机 + TTL
2. 延迟消息插件

---

### 4.1.1 什么是死信？

当队列中的消息满足以下情况之一时，会成为死信：

1. 消费者使用 `basic.reject` 或 `basic.nack` 声明消费失败，并且 `requeue=false`。
2. 消息过期，超时无人消费。
3. 队列满了，无法继续投递。

如果队列通过 `dead-letter-exchange` 指定了一个交换机，那么死信会被投递到这个交换机中，这个交换机就叫死信交换机。

---

### 4.1.2 死信交换机的作用

死信交换机主要有三个作用：

1. 收集处理失败并被拒绝的消息。
2. 收集因队列满而无法投递的消息。
3. 收集 TTL 到期的消息。

---

### 4.1.3 使用死信交换机实现延迟消息

实现思路：

1. 生产者发送消息到一个 TTL 队列。
2. TTL 队列不设置消费者。
3. 消息在 TTL 队列中过期后，成为死信。
4. 死信被投递到死信交换机。
5. 死信交换机再把消息路由到真正的业务队列。
6. 消费者监听业务队列，实现延迟消费。

---

### 4.1.4 死信延迟消息流程

假设：

- `ttl.fanout`：普通交换机
- `ttl.queue`：设置了 TTL 的队列，没有消费者
- `hmall.direct`：死信交换机
- `direct.queue1`：真正消费的业务队列
- `blue`：路由 key

流程：

1. 生产者发送消息到 `ttl.fanout`。
2. 消息进入 `ttl.queue`。
3. 消息设置 TTL，比如 5000 毫秒。
4. 因为 `ttl.queue` 没有消费者，所以消息不会被立即消费。
5. 5 秒后，消息过期，成为死信。
6. 死信被投递到 `hmall.direct`。
7. 死信沿用原来的 RoutingKey：`blue`。
8. `hmall.direct` 根据 `blue` 把消息路由到 `direct.queue1`。
9. 消费者监听 `direct.queue1`，5 秒后收到消息。

---

### 4.1.5 TTL + 死信交换机的缺点

RabbitMQ 的过期消息处理不是严格按时间主动扫描。

消息 TTL 到期后，不一定马上被移除或投递到死信交换机。  
只有当消息处于队首时，才会被检查并处理。

所以：

- 如果队列中消息很多，过期消息可能不能准时被处理。
- 延迟时间不够精准。
- 不适合高精度延迟任务。

---

## 4.2 DelayExchange 插件

RabbitMQ 还可以通过延迟消息插件实现延迟消息。

---

### 4.2.1 安装和启用插件

插件目录一般可以放在：

```bash
/usr/lib/rabbitmq/plugins
```

如果目录不存在，可以手动创建。

查看插件列表：

```bash
rabbitmq-plugins list
```

启用延迟消息插件：

```bash
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

重启 RabbitMQ：

```bash
service rabbitmq-server restart
```

---

### 4.2.2 通过注解声明延迟交换机

```java
@RabbitListener(bindings = @QueueBinding(
        value = @Queue(name = "delay.queue", durable = "true"),
        exchange = @Exchange(name = "delay.direct", delayed = "true"),
        key = "delay"
))
public void listenDelayMessage(String msg) {
    log.info("接收到 delay.queue 的延迟消息：{}", msg);
}
```

---

### 4.2.3 通过 Bean 声明延迟交换机

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Slf4j
@Configuration
public class DelayExchangeConfig {

    @Bean
    public DirectExchange delayExchange() {
        return ExchangeBuilder
                .directExchange("delay.direct")
                .delayed()
                .durable(true)
                .build();
    }

    @Bean
    public Queue delayedQueue() {
        return new Queue("delay.queue");
    }

    @Bean
    public Binding delayQueueBinding() {
        return BindingBuilder
                .bind(delayedQueue())
                .to(delayExchange())
                .with("delay");
    }
}
```

---

### 4.2.4 发送延迟消息

发送延迟消息时，需要设置 `x-delay` 属性。  
在 Spring AMQP 中可以通过 `MessagePostProcessor` 设置延迟时间。

```java
@Test
void testPublisherDelayMessage() {
    String message = "hello, delayed message";

    rabbitTemplate.convertAndSend(
            "delay.direct",
            "delay",
            message,
            msg -> {
                msg.getMessageProperties().setDelay(5000);
                return msg;
            }
    );
}
```

---

### 4.2.5 DelayExchange 插件注意点

延迟消息插件内部会维护本地数据，并使用计时器实现延迟。

如果延迟时间设置太长，或者延迟消息堆积太多，可能出现：

- CPU 开销增加
- 延迟时间存在误差
- 消息堆积压力变大

所以不建议设置非常长时间的延迟消息。

---

# 五、面试速记版

## 5.1 RabbitMQ 如何保证消息可靠性？

可以从三个阶段回答：

### 生产者阶段

- 开启生产者 Confirm，确认消息是否到达 Exchange。
- 开启 Return，处理消息路由失败的情况。
- 对重要消息可以设置发送重试，但要注意重试是阻塞式的。

### MQ 阶段

- Exchange 持久化。
- Queue 持久化。
- Message 持久化。
- 消息堆积场景可以使用 LazyQueue，减少内存压力。

### 消费者阶段

- 开启消费者 ACK。
- 使用 `auto` 或 `manual` 模式，避免消息刚投递就被删除。
- 开启本地重试，避免无限 requeue。
- 重试失败后投递到异常队列，后续人工补偿。
- 消费业务必须保证幂等。

---

## 5.2 消息重复消费怎么解决？

核心是保证消费者业务幂等。

常见方案：

1. 消息唯一 ID  
   消费成功后记录消息 ID，重复消息直接丢弃。

2. 业务状态判断  
   例如订单只有在“未支付”状态下才能改成“已支付”。

3. 数据库唯一约束  
   通过唯一索引防止重复插入。

4. Redis 去重  
   用 `SETNX` 记录消息 ID，但要注意过期时间和一致性。

推荐回答：

> 我更倾向于业务状态判断，因为它不需要额外记录消息 ID，而且和业务语义更贴合。比如支付成功消息，只允许 `status=未支付` 的订单更新为 `已支付`，SQL 中带上 `where status = 1`，判断和更新在一条 SQL 里完成，既幂等又避免并发问题。

---

## 5.3 生产者 Confirm 和 Return 的区别？

| 机制 | 关注点 | 典型失败 |
|---|---|---|
| Confirm | 消息是否到达 Exchange | Exchange 不存在、MQ 内部异常 |
| Return | 消息是否从 Exchange 路由到 Queue | RoutingKey 错误、没有绑定队列 |

一句话：

> Confirm 管消息有没有到交换机，Return 管消息有没有从交换机路由到队列。

---

## 5.4 ACK、NACK、Reject 的区别？

| 回执 | 含义 |
|---|---|
| ACK | 消费成功，RabbitMQ 删除消息 |
| NACK | 消费失败，可以重新入队 |
| Reject | 拒绝消息，一般不重新入队，直接删除或进入死信 |

---

## 5.5 为什么要有异常队列？

消费者本地重试耗尽后，如果消息直接丢弃，重要业务会丢数据。

所以可以使用 `RepublishMessageRecoverer` 把失败消息重新投递到异常交换机，再进入异常队列。

好处：

- 不影响主队列消费。
- 失败消息可以集中排查。
- 支持人工补偿。
- 避免无限重试拖垮系统。

---

## 5.6 LazyQueue 适合什么场景？

适合消息容易积压的场景。

例如：

- 秒杀削峰
- 异步通知
- 日志异步处理
- 消费者处理速度不稳定
- 队列可能堆积大量消息

回答重点：

> LazyQueue 会让消息优先落盘，而不是常驻内存，可以降低消息堆积时的内存压力。但因为要读写磁盘，极低延迟场景不一定适合。

---

## 5.7 死信交换机有什么用？

死信交换机可以接收三类消息：

1. 被消费者拒绝并且不重新入队的消息。
2. TTL 过期消息。
3. 队列满后无法投递的消息。

常见用途：

- 失败消息兜底
- 延迟消息
- 异常消息集中处理

---

## 5.8 RabbitMQ 如何实现延迟消息？

两种方案：

### 方案一：TTL + 死信交换机

优点：

- 不需要额外插件。
- 原生支持。

缺点：

- 延迟时间不够准确。
- 消息只有到达队首才会判断是否过期。

### 方案二：DelayExchange 插件

优点：

- 使用更方便。
- 直接通过 `x-delay` 设置延迟时间。

缺点：

- 需要安装插件。
- 延迟消息堆积过多会带来额外 CPU 开销。
- 不建议设置特别长的延迟时间。

---

## 5.9 支付状态最终一致性怎么保证？

可以这样回答：

> 用户支付成功后，支付服务先通过 MQ 通知交易服务修改订单状态。为了保证消息可靠，会开启生产者 Confirm、消息持久化、消费者 ACK、消费者失败重试和异常队列。同时消费者更新订单时使用状态判断保证幂等，例如只允许 `未支付` 状态改成 `已支付`。最后再用定时任务主动查询支付状态作为兜底，防止 MQ 通知极端情况下失败，从而保证订单状态最终一致。

---

## 5.10 项目里怎么说 RabbitMQ 可靠性？

可以结合项目这样说：

> 在我的项目里，RabbitMQ 主要用于异步削峰和解耦。为了保证消息可靠，我会从生产者、Broker、消费者三层处理。生产者侧可以开启 Confirm 机制，核心消息发送失败时记录日志或补偿；Broker 侧保证交换机、队列、消息持久化，消息堆积场景可使用 LazyQueue；消费者侧开启 ACK 和失败重试，重试失败后进入异常队列，避免消息丢失。同时业务层通过唯一约束或状态判断保证幂等，最后通过定时任务或补偿任务做最终一致性兜底。

---

# 六、整体总结

RabbitMQ 高级可靠性可以归纳为一条链路：

```text
生产者发送
  -> 发送重试
  -> Publisher Confirm
  -> Publisher Return
  -> Exchange 持久化
  -> Queue 持久化
  -> Message 持久化
  -> LazyQueue 应对堆积
  -> Consumer ACK
  -> 本地失败重试
  -> 异常队列
  -> 业务幂等
  -> 定时任务兜底
  -> 最终一致性
```

一句话总结：

> RabbitMQ 不能只靠 MQ 本身保证可靠性，而是要从发送端、Broker、消费端、业务幂等和兜底补偿五个层面一起设计，才能在真实业务里做到尽可能可靠和最终一致。
