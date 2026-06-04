# mybilibili-cloud 项目面试完美解答 (底层源码核实版)

> **⚠️ 核心高危预警（面试必看）**
> 我刚才通过后台 Agent 深度扫描了你的完整代码库，发现**你的简历描述与实际代码实现存在严重偏差！**
> 如果遇到较真的面试官让你细讲底层或者手写伪代码，极易露馅。请务必根据以下真实情况调整你的话术或去补齐代码：
> 1. **简历造假点 1：MinIO。** 你的代码里根本没有用 MinIO，视频分片上传完全是用**本地文件系统 (`java.io.File` + `RandomAccessFile`)** 实现的拼接。
> 2. **简历造假点 2：AOP 消息解耦。** 你的代码里根本没有 `@EventAction` 之类的自定义注解。你的解耦是硬编码在 Service 层的，直接注入了 `UserMessageEventProducer` 等类，通过 `rabbitTemplate.convertAndSend` 同步调用的。
> 3. **简历造假点 3：ThreadLocal Feign 透传。** 你的微服务互相调用时，并没有用拦截器隐式透传 Token/上下文，而是直接在方法签名里显式传了 `userId` 参数（例如 `UserInfoClient.countVideoInfoByUserId(userId)`）。
> 4. **简历造假点 4：死信队列 (DLQ)。** 代码里根本没有配置死信交换机。消费者（如 `VideoDanmuPersistConsumer`）报错时，是靠抛出异常触发 RabbitMQ 容器自带的原生无限重试机制。

基于上述**真实的底层代码**，我为你重新定制了绝对不会穿帮的面试回答。

---

## 一、 高并发互动场景（弹幕/点赞）深度解答

### ❓ 1. 你的弹幕和点赞场景写并发极高，你们是怎么优化写瓶颈的？
**【真实代码话术】**
“为了扛住写并发，我设计了一套**『内存缓冲 + MQ 异步削峰 + 批量落库』**的架构。
比如在点赞场景，前端请求过来后，我通过 Lua 脚本 `LUA_VIDEO_TOGGLE_ACTION` 在 Redis 层面原子性地完成了『判断点赞状态』和『互动计数累加』，随后业务直接返回。计数会暂存在 Redis Hash (`video:action:count:delta`) 中。
而针对弹幕，为了防止高频提交打垮 DB，业务先用 ZSet 缓存弹幕做实时展示，然后注入 `VideoDanmuEventProducer` 发送一条 `video.danmu.post` 事件到 RabbitMQ。消费端 `VideoDanmuPersistConsumer` 设置了批量消费（BatchSize 100），在后台按批次组装后，调用 MyBatis-Plus 的 `insertOrUpdateBatch` 进行高效批量落库，把 DB 的 QPS 峰值降到了极低。”

### ❓ 2. MQ 消息丢失和重复消费是怎么处理的？
**【真实代码话术】**
“关于重复消费（幂等），我的消费者里封装了一个统一的 `MqIdempotentComponent`。收到消息后，会以事件 ID 为 Key 去 Redis 执行 `tryStart`（底层是 `SETNX`）。只有拿到锁的消息才会被加入这批待落库的 List 中处理。对于弹幕，我们在 DB 层面也建了基于 `event_id` 的唯一索引，配合批量 upsert 操作做最后兜底。
至于消费异常，如果批量插入报错，我在 `catch` 块里会调用 `mqIdempotentComponent.release()` 释放幂等锁，并**主动向上抛出 `BusinessException`**。这会触发 Spring AMQP 容器的原生拒绝/重投机制，由 RabbitMQ 负责重新分发。”

### ❓ 3. Redis 和 MySQL 的数据最终一致性怎么保证？如果 Redis 挂了怎么办？
**【真实代码话术】**
“针对高频互动的统计数据，我主要靠『凌晨定时任务刷盘』来保证最终一致。
我在 `DailyUserStatsSyncTask` 类里配了两个定时任务。
第一个任务 `0 0 0 * * *` 负责冻结昨天的数据：它基于 `SNAPSHOT_BATCH_SIZE=200`，分页去查所有的 `userId`，然后把每个人实时的 Hash 统计数据拷贝一份作为昨天的静态快照（Snapshot）存到 Redis 里。
第二个任务 `0 5 0 * * *` 负责实际刷盘：它把这些 Snapshot 拿出来，组装成 Entity，调用 `userStatsMapper.insertOrUpdateBatch` 批量写入数据库。如果 Redis 挂了，系统依然可以降级通过 Feign 直接去查 MySQL 聚合计算（我在 `UserStatsCacheAsyncComponent` 里写了兜底回源逻辑）。”

---

## 二、 视频上传与转码（防穿帮版）

### ❓ 4. 长视频分片上传与流媒体处理具体怎么落地的？
**【真实代码话术】**
“首先在分片上传方面，我在 `FileController` 提供了分片接收接口。前端分块上传时，后端通过临时目录（`project_folder/temp/`）把每一片写入本地磁盘。全部上传完后，触发 `union()` 方法，底层通过 `java.io.RandomAccessFile` 把这些分片按顺序追加拼接成一个完整的 `.mp4` 文件。
合并完成后，事务提交时我会发出一个转码事件到 MQ。
后台的 `VideoTransferConsumer` 收到后，调用我的 `FFmpegUtils`。它通过 `ProcessUtils` 执行本地 FFmpeg 命令行，把 MP4 按 10秒一个 segment 切片，生成 `m3u8` 索引文件和一系列 `.ts` 文件。
这样做让前端可以边下边播，极大提升了首屏秒开率和拖拽响应速度。”

---

## 三、 AI 与 RAG 智能检索落地（亮点，100%按代码写）

### ❓ 5. 简历里的 RAG 智能检索管道是怎么设计的？
**【真实代码话术】**
“这是一个打通了 Java 后台、Python AI Worker 和 Elasticsearch 的跨语言系统。
**数据构建侧：** 视频发布后，Java 会向 Redis 的 `easylive:queue:ai:subtitle-vector` 推送任务。Python 端 `worker.py` 消费队列，先调用 `faster-whisper` 的小模型将视频音轨提取成带时间戳的文本，接着调用本地的 `Ollama (bge-m3:567m)` 跑出文本的高维稠密向量，最后通过 `elasticsearch.helpers.bulk` 批量刷入 ES 的 `easylive_video_subtitle_vector` 索引。
**检索侧 (RAG)：** 用户提问时，先过一遍大模型做『意图识别』（提取搜索词）。如果是问答意图，Java 会用一段 Painless 脚本去 ES 里执行 kNN 查询，核心就是算 `cosineSimilarity(params.queryVector, 'contentVector')`。
**生成与交互侧：** 拿到高度匹配的字幕片段后，通过 Prompt 组装好上下文，扔给 `qwen2.5` 生成回答。为了用户体验，前端交互采用了 SSE (`SseEmitter`)，我在独立线程池 `aiChatExecutor` 中实时捕获大模型的流式字元，持续 `send` 刷给前端，体验非常丝滑。”

---

## 四、 架构权衡反思

### ❓ 6. 这个项目你觉得最大的收获是什么？
**【真实代码话术】**
“最大的收获是**懂得了根据业务场景选择最合适的兜底与降级策略**。
比如，在做多端登录时，我利用 `WEB_TOKEN_KEY` 和 `USER_TOKEN_KEY` 建立了双向映射。当新设备登录时，我不是粗暴地删缓存，而是更新用户维度绑定的 Token，当老设备再次请求被拦截器（`UserLoginAspect`）拦下发现 Token 不匹配时再执行平滑踢出。
再比如，微服务拆分后，我在聚合展示（如用户主页）时，没有盲目追求纯异步和强一致，而是接受了展示数据的弱一致性。调用其他微服务（如查用户的视频获赞总数）时如果报错，我会 `catch` 异常并降级返回 0（不阻塞主流程），等下一次用户刷新或者定时缓存重建时自动修复。这种在可靠性和性能之间的 Trade-off，是我在单体架构里学不到的。”
