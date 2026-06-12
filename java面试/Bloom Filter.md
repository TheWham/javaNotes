# RedissonBloomFilter 复习笔记

## 1. 布隆过滤器是什么？

布隆过滤器是一种用于判断元素是否存在的概率型数据结构。

它的特点是：

```text
判断不存在：一定不存在
判断存在：可能存在
```

也就是说，布隆过滤器可能出现误判，但不会误判不存在。

例如：

```text
BloomFilter 判断 videoId = 1001 不存在
=> 1001 一定不存在

BloomFilter 判断 videoId = 1001 存在
=> 1001 可能存在，但也可能是误判
```

---

## 2. 底层原理

布隆过滤器底层主要由两部分组成：

```text
bitmap + 多个 hash 函数
```

### 2.1 bitmap

bitmap 本质是一个很长的二进制位数组。

初始状态：

```text
0 0 0 0 0 0 0 0 0 0
```

每个 bit 位只有两种状态：

```text
0：没有被标记
1：已经被标记
```

### 2.2 添加元素

假设添加：

```text
videoId = 1001
```

经过多个 hash 函数计算：

```text
hash1(1001) % size = 2
hash2(1001) % size = 5
hash3(1001) % size = 8
```

然后把 bitmap 中对应的位置设置为 `1`：

```text
0 0 1 0 0 1 0 0 1 0
```

### 2.3 查询元素

查询 `videoId = 1001` 时，也计算同样的 hash 位置：

```text
2、5、8
```

如果这几个位置全是 `1`：

```text
可能存在
```

如果有任意一个位置是 `0`：

```text
一定不存在
```

---

## 3. 为什么会有误判？

因为不同元素经过 hash 之后，可能会映射到相同的 bit 位。

例如：

```text
videoId = 1001 -> 2、5、8
videoId = 2002 -> 2、5、8
```

或者多个不同元素组合起来，刚好把某个不存在元素的所有 hash 位都置为了 `1`。

所以布隆过滤器查询时：

```text
所有 bit 都是 1
```

只能说明：

```text
这个元素可能存在
```

不能说明：

```text
这个元素一定存在
```

但是如果有一个 bit 是 `0`，说明这个元素一定没有被添加过。

---

## 4. RedissonBloomFilter 是什么？

RedissonBloomFilter，也就是 Redisson 提供的 `RBloomFilter`，本质上还是传统布隆过滤器。

它没有改变布隆过滤器的核心原理，仍然是：

```text
bitmap + hash 计算 + 概率判断
```

区别在于：

```text
传统本地 BloomFilter：
bitmap 通常放在 JVM 内存中，比如 BitSet

Redisson RBloomFilter：
bitmap 和配置数据放在 Redis 中，由 Redisson 封装操作
```

---

## 5. RedissonBloomFilter 和传统布隆过滤器的区别

|对比点|传统本地 BloomFilter|Redisson RBloomFilter|
|---|---|---|
|底层原理|bitmap + 多个 hash|一样|
|存储位置|JVM 内存|Redis|
|多实例共享|不方便|方便|
|服务重启|本地数据丢失，需要重建|Redis 数据不丢就还能用|
|初始化参数|自己计算 bitmap 大小和 hash 次数|Redisson 自动计算|
|并发问题|需要自己处理|Redisson 封装 Redis 操作|
|网络开销|无网络开销|每次 add / contains 都要访问 Redis|
|适合场景|单体服务、本地快速判断|微服务、多实例共享判断|

---

## 6. RedissonBloomFilter 底层是 128bit 吗？

不是。

RedissonBloomFilter 底层不是固定 128bit。

准确理解：

```text
bitmap 的大小不是固定的
bitmap 大小由 expectedInsertions 和 falseProbability 决定
```

例如：

```java
bloomFilter.tryInit(1_000_000L, 0.01);
```

表示：

```text
预计插入 100 万个元素
允许 1% 的误判率
```

Redisson 会根据这两个参数计算：

```text
需要多大的 bitmap
需要多少次 hash
```

所以 bitmap 可能是几百万 bit、几千万 bit，甚至更大，不是固定 128bit。

所谓的 `128bit`，通常指的是 hash 算法的输出位宽，不是 bitmap 的总大小。

不要把下面两个概念混淆：

```text
hash 输出位宽：可能是 128bit
bitmap 容量：由容量和误判率计算出来，通常远大于 128bit
```

---

## 7. RedissonBloomFilter 常用 API

### 7.1 获取 BloomFilter

```java
RBloomFilter<Long> bloomFilter = redissonClient.getBloomFilter("video:bloom");
```

这里的 `"video:bloom"` 是 Redis 中布隆过滤器的名称。

---

### 7.2 初始化

```java
bloomFilter.tryInit(1_000_000L, 0.01);
```

参数含义：

```text
expectedInsertions：预计插入多少个元素
falseProbability：允许的误判率
```

例如：

```text
1_000_000L：预计存 100 万个视频 ID
0.01：允许 1% 误判率
```

注意：

```text
expectedInsertions 越大，占用空间越大
falseProbability 越小，占用空间越大，hash 次数也越多
```

---

### 7.3 添加元素

```java
bloomFilter.add(videoId);
```

比如：

```java
bloomFilter.add(1001L);
```

含义是把 `1001` 加入布隆过滤器。

---

### 7.4 判断元素是否存在

```java
boolean mayExist = bloomFilter.contains(videoId);
```

如果返回 `false`：

```text
一定不存在
```

如果返回 `true`：

```text
可能存在
```

---

## 8. 在项目中为什么要用 RedissonBloomFilter？

以视频平台为例，用户请求视频详情：

```text
/video/1001
```

正常流程：

```text
查 Redis 缓存
缓存没有，查数据库
数据库有，写入缓存
数据库没有，返回空
```

如果大量请求访问不存在的视频 ID：

```text
/video/999999999
/video/888888888
/video/777777777
```

这些请求 Redis 查不到，数据库也查不到，会不断打到数据库。

这就是缓存穿透。

使用布隆过滤器后：

```text
请求进来
先判断 videoId 是否在布隆过滤器中
如果不存在，直接返回 null
如果可能存在，再查 Redis / DB
```

流程变成：

```text
请求 videoId
        |
        v
布隆过滤器判断
        |
        |-- 不存在：直接返回 null，不查 Redis，不查 DB
        |
        |-- 可能存在：继续查 Redis
                        |
                        |-- Redis 命中：返回缓存
                        |
                        |-- Redis 未命中：查 DB，重建缓存
```

---

## 9. 为什么不用本地 BloomFilter？

如果是单体服务，本地 BloomFilter 可以用。

但如果是微服务多实例部署：

```text
video-service-1
video-service-2
video-service-3
```

如果每个实例都维护一份本地 BloomFilter，会有几个问题：

```text
1. 每个 JVM 都要占一份内存
2. 服务启动时都要加载全量数据
3. 新增视频后，每个实例都要同步更新
4. 某个实例漏更新，可能导致判断结果不一致
```

使用 RedissonBloomFilter 后：

```text
所有服务实例共享 Redis 中同一份 BloomFilter
```

这样更适合分布式场景。

---

## 10. 布隆过滤器的优点

### 10.1 空间占用小

相比 HashSet、数据库表，布隆过滤器只存 hash 后的 bit 位，不存完整数据。

例如不用存：

```text
videoId = 1001
videoId = 1002
videoId = 1003
```

而是只存这些 ID 映射后的 bit 位。

### 10.2 查询速度快

判断是否存在时，只需要计算几次 hash，然后检查 bitmap 中几个 bit 位。

### 10.3 适合防止缓存穿透

对于不存在的数据，可以提前拦截，避免请求穿透到数据库。

---

## 11. 布隆过滤器的缺点

### 11.1 有误判

布隆过滤器判断存在时，不一定真的存在。

例如：

```text
BloomFilter 判断 videoId = 9999 存在
```

但数据库中可能并没有这个视频。

所以布隆过滤器只能作为前置拦截，不能作为最终数据来源。

最终还是要查 Redis 或数据库。

### 11.2 不适合删除单个元素

普通布隆过滤器一般不支持安全删除单个元素。

因为多个元素可能共享同一个 bit 位。

如果删除某个元素时把 bit 置回 `0`，可能会影响其他元素。

例如：

```text
A 使用 bit 位：1、3、5
B 使用 bit 位：3、5、7
```

如果删除 A，把 3、5 置为 0，就会导致 B 被误判为不存在。

所以普通 BloomFilter 适合：

```text
只增不删
```

如果业务中确实需要删除，可以考虑：

```text
1. 定时全量重建 BloomFilter
2. 使用 Counting BloomFilter
3. 使用 CuckooFilter
```

### 11.3 数据变化后需要同步更新

例如新增视频：

```text
videoId = 2001
```

数据库插入成功后，也要把 `2001` 加入 BloomFilter。

否则会出现：

```text
数据库中有 videoId = 2001
但是 BloomFilter 认为不存在
请求被直接拦截
```

这种情况属于严重问题。

所以新增数据时要保证：

```text
先写数据库成功
再更新 BloomFilter
```

---

## 12. 布隆过滤器和缓存空值的区别

### 12.1 缓存空值

缓存空值是指：

```text
数据库查不到某个 key
也在 Redis 中存一个空值
```

例如：

```text
key = video:9999
value = null
ttl = 5 分钟
```

下次再请求 `video:9999`，Redis 直接返回空，不查数据库。

优点：

```text
实现简单
对少量不存在 key 很有效
```

缺点：

```text
如果攻击者构造大量不同的不存在 key
Redis 中会缓存大量空值
造成内存浪费
```

### 12.2 布隆过滤器

布隆过滤器是提前存储所有合法 ID。

请求进来先判断：

```text
这个 ID 是否可能存在
```

优点：

```text
可以在 Redis / DB 之前拦截大量非法 ID
空间占用比缓存大量空值更小
```

缺点：

```text
有误判
需要维护过滤器数据
```

### 12.3 实际项目组合

实际项目可以组合使用：

```text
布隆过滤器：拦截明显不存在的 ID
缓存空值：兜底处理布隆过滤器误判
互斥锁 / 逻辑过期：防止缓存击穿
```

---

## 13. expectedInsertions 和 falseProbability 怎么设置？

### 13.1 expectedInsertions

`expectedInsertions` 是预计插入数量。

例如你的系统预计有 100 万个视频：

```java
bloomFilter.tryInit(1_000_000L, 0.01);
```

如果未来数据量可能增长到 500 万，可以预留空间：

```java
bloomFilter.tryInit(5_000_000L, 0.01);
```

如果预计数量设置太小，后续实际插入远超预期，会导致误判率升高。

### 13.2 falseProbability

`falseProbability` 是误判率。

常见设置：

```text
0.1   = 10%
0.01  = 1%
0.001 = 0.1%
```

误判率越低：

```text
需要的 bitmap 越大
需要的 hash 次数越多
内存占用越高
计算开销越大
```

项目里常用：

```java
bloomFilter.tryInit(1_000_000L, 0.01);
```

也就是 1% 左右误判率。

---

## 14. 项目代码示例

### 14.1 初始化布隆过滤器

```java
@PostConstruct
public void initBloomFilter() {
    RBloomFilter<Long> bloomFilter = redissonClient.getBloomFilter("video:bloom");

    // 预计 100 万视频 ID，误判率 1%
    bloomFilter.tryInit(1_000_000L, 0.01);

    // 从数据库加载所有视频 ID
    List<Long> videoIds = videoMapper.selectAllVideoIds();

    for (Long videoId : videoIds) {
        bloomFilter.add(videoId);
    }
}
```

---

### 14.2 查询时使用布隆过滤器

```java
public Video getVideoInfo(Long videoId) {
    RBloomFilter<Long> bloomFilter = redissonClient.getBloomFilter("video:bloom");

    // 布隆过滤器判断不存在，直接返回
    if (!bloomFilter.contains(videoId)) {
        return null;
    }

    // 继续查 Redis 缓存
    Video video = getVideoFromCache(videoId);
    if (video != null) {
        return video;
    }

    // 缓存不存在，再查数据库
    video = videoMapper.selectById(videoId);
    if (video == null) {
        return null;
    }

    // 写入缓存
    setVideoCache(videoId, video);

    return video;
}
```

---

### 14.3 新增视频时更新布隆过滤器

```java
@Transactional
public void publishVideo(Video video) {
    videoMapper.insert(video);

    RBloomFilter<Long> bloomFilter = redissonClient.getBloomFilter("video:bloom");
    bloomFilter.add(video.getId());
}
```

注意：

```text
数据库写入成功后，要把新视频 ID 添加到 BloomFilter
否则新数据可能被误拦截
```

---

## 15. 全量重建 BloomFilter 是什么意思？

全量重建是指：

```text
定时从数据库查询所有有效 ID
重新构建一份新的 BloomFilter
```

为什么需要全量重建？

因为普通布隆过滤器不适合删除单个元素。

例如视频被删除、下架后，BloomFilter 中可能还认为它存在。

这不会造成严重错误，因为：

```text
BloomFilter 判断可能存在
后续还会查 Redis / DB
DB 查不到后返回空
```

但是会增加少量无效查询。

所以可以定时全量重建。

例如每天凌晨：

```text
1. 创建一个新的 BloomFilter：video:bloom:new
2. 查询数据库所有有效 videoId
3. 把所有有效 videoId 加入 video:bloom:new
4. 构建完成后，切换使用新 BloomFilter
5. 删除旧 BloomFilter
```

这样可以清理掉已经删除或失效的数据。

---

## 16. RedissonBloomFilter 在缓存架构中的位置

在缓存保护架构中，布隆过滤器主要解决：

```text
缓存穿透
```

完整缓存保护体系：

```text
1. 布隆过滤器：拦截不存在的数据，防止缓存穿透
2. 缓存空值：兜底处理布隆过滤器误判
3. 分布式锁：防止热点 key 缓存击穿
4. 逻辑过期：热点数据过期后返回旧值，异步重建
5. 随机 TTL：防止大量 key 同时过期造成缓存雪崩
```

---

## 17. 缓存穿透、击穿、雪崩的区别

### 17.1 缓存穿透

请求的数据本身不存在。

```text
Redis 没有
DB 也没有
```

大量这种请求会打穿数据库。

解决方案：

```text
布隆过滤器
缓存空值
参数校验
接口限流
```

### 17.2 缓存击穿

某个热点 key 过期，大量请求同时访问。

```text
Redis 过期
大量请求同时查 DB
```

解决方案：

```text
互斥锁
逻辑过期
热点 key 永不过期
```

### 17.3 缓存雪崩

大量 key 同一时间过期，或者 Redis 整体不可用。

解决方案：

```text
TTL 加随机值
多级缓存
Redis 高可用
限流降级
```

---

## 18. 面试高频问题

### 问题 1：RedissonBloomFilter 底层是什么？

回答：

RedissonBloomFilter 本质上还是布隆过滤器，底层是 bitmap 加多个 hash。添加元素时，会通过 hash 计算出多个 bit 位并置为 1；查询元素时，也计算同样的位置，如果有任意一个 bit 是 0，就说明一定不存在；如果所有 bit 都是 1，就说明可能存在。Redisson 只是把 bitmap 和相关配置放在 Redis 中，方便多个服务实例共享。

---

### 问题 2：RedissonBloomFilter 和传统 BloomFilter 有什么区别？

回答：

核心原理一样，都是 bitmap 加多个 hash。区别主要在工程实现上。传统 BloomFilter 通常在 JVM 本地维护 BitSet，多实例部署时每个实例都要维护一份数据，更新和一致性比较麻烦。RedissonBloomFilter 把过滤器放在 Redis 中，所有服务实例共享同一份过滤器，并且通过 `tryInit(expectedInsertions, falseProbability)` 自动计算位图大小和 hash 次数，更适合微服务分布式场景。

---

### 问题 3：布隆过滤器为什么会误判？

回答：

因为不同元素经过 hash 后，可能映射到相同的 bit 位。如果一个不存在的元素经过 hash 计算出来的多个 bit 位，刚好都被其他元素置为了 1，那么布隆过滤器就会误以为它存在。所以布隆过滤器判断存在只是可能存在，判断不存在才是一定不存在。

---

### 问题 4：布隆过滤器能删除元素吗？

回答：

普通布隆过滤器不适合删除单个元素，因为多个元素可能共享同一个 bit 位。删除某个元素时如果把对应 bit 置回 0，可能会影响其他元素，导致本来存在的数据被误判为不存在。对于删除场景，可以定时全量重建布隆过滤器，或者使用 Counting BloomFilter、CuckooFilter 这类支持删除的结构。

---

### 问题 5：RedissonBloomFilter 是固定 128bit 吗？

回答：

不是。RedissonBloomFilter 的底层 bitmap 大小不是固定 128bit，而是根据预期插入数量和误判率计算出来的。`128bit` 通常说的是 hash 算法输出位宽，不是 BloomFilter 的 bitmap 总大小。真正的 bitmap 可能是几百万甚至几千万 bit。

---

### 问题 6：布隆过滤器如何解决缓存穿透？

回答：

缓存穿透是指请求的数据在 Redis 和数据库中都不存在，如果大量请求访问不存在的数据，会直接打到数据库。布隆过滤器会提前保存所有合法 ID，请求进来后先判断 ID 是否存在。如果布隆过滤器判断不存在，就直接返回，不再访问 Redis 和数据库；如果判断可能存在，再继续走缓存和数据库查询流程。这样可以拦截大量非法请求，保护数据库。

---

## 19. 项目简历表述

可以写成：

```text
使用 Redisson RBloomFilter 构建缓存穿透防护层，将平台有效视频 ID 预加载至布隆过滤器中，在查询视频详情前先进行存在性判断，对非法 videoId 请求直接拦截，减少无效请求穿透 Redis 后访问数据库。同时结合逻辑过期缓存与 Redisson 分布式锁控制热点 key 重建，降低高并发场景下数据库瞬时压力。
```

如果想更突出理解，可以写成：

```text
针对不存在视频 ID 导致的缓存穿透问题，引入 Redisson RBloomFilter 维护有效视频 ID 集合。利用布隆过滤器“判断不存在一定不存在、判断存在可能误判”的特性，在缓存查询前拦截非法请求；对于误判请求再通过 Redis 空值缓存兜底，避免大量无效流量直接访问数据库。
```

---

## 20. 一句话总结

RedissonBloomFilter 本质是分布式版本的布隆过滤器，底层仍然是 bitmap 加多个 hash。它把过滤器数据放在 Redis 中，方便多个服务实例共享，适合在微服务项目中用于缓存穿透防护。它的判断结果是：不存在一定不存在，存在只是可能存在。