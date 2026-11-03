# 消息队列与RocketMQ
> **定位**：讲清"为什么用 MQ、RocketMQ 内部怎么运转、消息如何不丢/不重/不乱序/不积压"这条面试主线。
> **重要度**：★★★★★
> **建议复习时长**：4 小时
> **关联文档**：[02-Kafka深度剖析](./02-Kafka深度剖析.md)、[03-RabbitMQ与MQ选型对比](./03-RabbitMQ与MQ选型对比.md)、[02-分布式事务](../05-分布式与架构/02-分布式事务.md)、[03-场景设计与开放题](../08-面试实战/03-场景设计与开放题.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| MQ 使用场景与引入代价 | ★★★★★ | 场景题 |
| RocketMQ 架构与路由发现 | ★★★★☆ | 八股原理 |
| 发送方式、延迟消息与顺序发送 | ★★★★☆ | 配置实践 |
| 存储模型 CommitLog/ConsumeQueue/IndexFile | ★★★★★ | 源码追问 |
| 刷盘与主从复制可靠性四象限 | ★★★★★ | 八股原理 |
| 消费模式、位点管理与 Rebalance | ★★★★★ | 八股原理 |
| 重试队列、死信队列与消息过滤 | ★★★★☆ | 故障复盘 |
| 顺序消息与事务消息最终一致性 | ★★★★★ | 架构设计 |
| 消息幂等与消息丢失防范 | ★★★★★ | 故障复盘 |
| 消息积压、5.0 新特性与监控运维 | ★★★★☆ | 故障复盘 |

## 1. MQ 使用场景与引入代价
### 1.1 核心原理
四大场景：**解耦**（上游只发事件，不关心下游数量）、**异步**（非核心链路异步化，压缩接口 RT）、**削峰**（把瞬时洪峰变为平稳消费速率）、**最终一致性**（本地事务表 + 事务消息）。引入代价：可用性依赖 MQ（需降级方案）、一致性债（重复/丢失/乱序）、运维复杂度（集群、监控、积压告警）、链路变长定位难。
### 1.2 面试问答
**Q1：什么场景该用 MQ，什么场景不该用？**
A：结论——只有"能接受异步/最终一致"且"上下游数量或流量明显不匹配"时才用。适用：下单后发券/加积分/发短信、秒杀写请求削峰、日志埋点采集。不适用：强一致且必须同步返回的支付扣款结果、链路短且 RT 要求极低（多一次网络往返）、无消费能力的小系统（运维成本大于收益）。

**Q2：削峰是怎么起作用的？**
A：结论——MQ 把"峰值流量"转化为"可控消费速率的队列深度"。原因：写入是顺序追加，写入能力远高于业务处理能力；消费者按自身能力拉取。边界：前提是峰值持续时间短于可容忍排队时长；若流量持续超过消费能力，队列只会一直涨，必须扩容消费者或降级。

**Q3：MQ 挂了怎么办？**
A：分两侧答。① 生产侧：本地消息表 + 定时补偿重发，或降级为同步调用；② 消费侧：非核心链路可暂停，核心依赖 MQ 集群自身高可用（多主多从、同步刷盘+同步复制）。加分点：明确"降级开关、本地消息表、幂等"三件套。

**Q4：异步化后怎么保证用户看到正确结果？**
A：结论——接口先返回"受理中"，用轮询、推送或回调告知最终状态。原因：异步只把处理挪走，状态机必须显式建模（待处理/处理中/成功/失败）。边界：必须有兜底超时（如 30 分钟未完成置失败或转人工），否则用户永远卡在处理中。
### 1.3 高频追问与陷阱
- 陷阱：把"解耦"和"异步"混为一谈——解耦关注消费者数量可变，异步关注调用方不阻塞。
- 陷阱：忽略"MQ 本身是新的一致性边界"，引入后必须回答重复消费与补偿问题。
- 追问：MQ 相比线程池异步（@Async）的优势？跨进程、可持久化、可重放、可水平扩展。
### 1.4 配置/命令示例
```java
@Transactional
public void createOrder(Order order) {
    orderMapper.insert(order);
    localMsgMapper.insert(new LocalMsg(order.getId(), MsgStatus.PENDING)); // 本地消息表
}
// 定时任务扫描 PENDING 重发，下游必须幂等
```
### 1.5 一句话速记
解耦、异步、削峰、最终一致——每个收益都对应一份一致性债，用幂等与补偿来还。

## 2. RocketMQ 架构与路由发现
### 2.1 核心原理
四角色：**NameServer**（无状态、节点间不通信，只存 Topic→Broker 路由与 Broker 存活信息）、**Broker**（存储与转发，同名 brokerName + 不同 brokerId 组成主从，brokerId=0 为 Master）、**Producer Group**、**Consumer Group**。**Topic** 是逻辑分类，**MessageQueue** 是物理存储与并行消费单位，一个 Topic 的队列分散在不同 Broker；集群模式下并行消费度上限 = 队列数。
### 2.2 面试问答
**Q1：NameServer 为什么设计成无状态的？**
A：结论——为了简单与可用性优先。原因：节点间不通信、不做选举，挂任意一台都不影响整体；Broker 向所有 NameServer 注册，客户端连任一节点即可拿到路由。代价：NameServer 间路由可能秒级不一致，因此客户端默认每 30s 重新拉取并本地缓存。

**Q2：Producer 发送一条消息的完整流程？**
A：① 从本地路由缓存取该 Topic 队列列表，没有则向 NameServer 拉取；② 按负载均衡策略（默认轮询，可自定义 MessageQueueSelector）选队列；③ 定位队列所在 Broker 的 Master 地址；④ 发送并等待 SendResult。失败默认重试 2 次（retryTimesWhenSendFailed）并换 Broker。

**Q3：Broker 主从有什么用？会自动切换吗？**
A：结论——4.x 主从提供数据冗余与 Slave 读能力（slaveReadEnable），但**默认没有自动选主**，主挂后需要人工或 DLedger（Raft）介入；5.0 引入 Controller 模式才默认具备主从自动切换。原因：4.x 主从复制不是多数派协议，无法自行裁决。

**Q4：为什么不用 ZooKeeper 做注册中心？**
A：结论——路由注册只需要 AP 语义。原因：ZK 保证 CP，选举期间不可用；NameServer 无选举、纯内存、可任意水平扩展。Broker 默认 30s 心跳，超时失效即被剔除路由。
### 2.3 高频追问与陷阱
- 陷阱：认为"从节点能自动升主"——4.x 默认做不到，必须 DLedger 或 5.0 Controller。
- 陷阱：把 Topic 队列数当作越大越好——队列越多，Rebalance、句柄、堆外内存开销越大。
- 追问：客户端路由缓存更新机制？30s 定时更新 + 发送失败时重新拉取。
### 2.4 配置/命令示例
```bash
sh mqadmin clusterList -n 127.0.0.1:9876
sh mqadmin topicRoute -n 127.0.0.1:9876 -t OrderTopic
sh mqadmin updateTopic -n 127.0.0.1:9876 -c DefaultCluster -t OrderTopic -r 8 -w 8
```
### 2.5 一句话速记
NameServer 无状态做路由，Broker 主从存数据，Topic 是逻辑分区，Queue 才决定并行度。

## 3. 消息发送方式、延迟消息与顺序发送
### 3.1 核心原理
发送方式：**同步**（有 SendResult，可靠但 RT 高）、**异步**（回调，RT 低吞吐高）、**单向 oneway**（不等结果，可丢）。批量发送把同 Topic 多条小消息合并，减少网络往返（受单消息 maxMessageSize 与批量总大小限制，具体阈值以版本实现为准）。**延迟消息**：4.x 仅 18 个固定级别；5.x 支持任意秒级时间。
### 3.2 面试问答
**Q1：4.x 的 18 个延迟级别是什么？**
A：1s、5s、10s、30s、1m、2m、3m、4m、5m、6m、7m、8m、9m、10m、20m、30m、1h、2h，共 18 级，level 从 1 开始，0 表示不延迟，用法 `msg.setDelayTimeLevel(3)`。原理：消息先写入 `SCHEDULE_TOPIC_XXXX` 中对应级别的队列，定时任务扫到到期后重新投递到真实 Topic。

**Q2：5.0 的定时消息改进在哪？**
A：结论——从 18 个离散级别升级为任意时间精度（秒级），API 为 `setDeliverTimeMs(System.currentTimeMillis()+delay)`，底层换成时间轮 + 定时消息文件。若仍用 4.x，需要任意延迟只能自研（延迟表 + 定时扫描）。

**Q3：顺序发送怎么用？**
A：结论——用 `MessageQueueSelector` 把同一业务键（如 orderId）的消息发到同一队列。原因：顺序语义只能在同一队列内保证。边界：不要直接用 `hash % 队列数` 后随意扩容队列（会破坏历史消息与新消息的同队列假设）。

**Q4：单向发送能保证不丢吗？**
A：不能。oneway 不等 Broker 响应，网络抖动与 Broker 拒绝都无法感知，只适合日志、监控埋点等允许丢失的场景。

**Q5：异步发送有什么坑？**
A：回调里的异常不会影响主流程，很容易漏掉失败补偿；且回调在 IO 线程执行，业务处理应转到独立线程池，避免阻塞网络线程。
### 3.3 高频追问与陷阱
- 陷阱：把秒数当级别传入 `setDelayTimeLevel(60)`——超出 18 级会异常或行为不符预期。
- 陷阱：批量发送的消息属性/Topic 不一致会导致发送失败；单批不宜过大。
- 追问：批量发送为什么快？减少网络往返与 syscall，Broker 侧一次追加写入多条。
### 3.4 配置/命令示例
```java
producer.send(msg, (mqs, m, arg) -> mqs.get(Math.abs(arg.hashCode()) % mqs.size()), orderId); // 顺序
producer.send(msg, new SendCallback() { public void onSuccess(SendResult r) {} public void onException(Throwable e) {} }); // 异步
msg.setDeliverTimeMs(System.currentTimeMillis() + 30_000L); // 5.0 定时
```
### 3.5 一句话速记
同步图可靠、异步图吞吐、单向图便宜；顺序靠 Selector 落同队列；延迟 4.x 看级别、5.x 看时间戳。

## 4. 存储模型：CommitLog / ConsumeQueue / IndexFile / mmap
### 4.1 核心原理
**CommitLog**：所有 Topic 混写、单文件约 1GB、顺序追加，是唯一真实数据源。**ConsumeQueue**：逻辑消费队列，目录 `topic/queueId/`，每条索引 20B = 8B CommitLog 物理偏移 + 4B 消息长度 + 8B Tag 哈希，单文件约 30 万条（约 5.72MB）。**IndexFile**：哈希索引，支持按 key 或时间区间查询（文件头 40B + 500 万 slot×4B + 500 万 index×20B，约 420MB）。读取先查 ConsumeQueue 得偏移，再回 CommitLog 按 offset+size 读（回源）。三文件均通过 mmap 映射，读写都走页缓存。
### 4.2 面试问答
**Q1：为什么所有消息写同一个 CommitLog？**
A：结论——为了顺序写。原因：顺序写吞吐远高于随机写，按 Topic 分文件会产生大量随机 IO。代价：读消息需"ConsumeQueue + CommitLog"两次寻址，用索引空间换读取效率；ConsumeQueue 本身也是顺序追加的小文件。

**Q2：mmap 和页缓存是怎么用的？**
A：结论——CommitLog 与 ConsumeQueue 通过 mmap 映射，写入实际写页缓存，由 OS 异步刷盘；消费时若页缓存仍命中就是纯内存读。注意：mmap 用的是堆外内存，因此 Broker 堆不必开大，但要给页缓存留足物理内存。

**Q3：IndexFile 有什么用？**
A：结论——支持按 key 与时间区间查消息，是排查事故与消息轨迹的基础（`mqadmin queryMsgByKey`）。原因：ConsumeQueue 只有偏移没有业务 key，无法反查；IndexFile 用哈希槽 + 链式冲突组织索引。

**Q4：文件什么时候被删除？**
A：过期清理线程在 `deleteWhen`（默认每天 04:00）触发，删除"最后修改时间超过 fileReservedTime（默认 72 小时）"的整文件；磁盘使用率超过 diskMaxUsedSpaceRatio（默认 75%）时会强制删除。文件以"预留（预分配）"方式创建，避免运行期扩容产生碎片。

**Q5：CommitLog 的 offset 是全局的还是每队列独立的？**
A：全局单调递增的物理偏移，全局唯一；ConsumeQueue 中的每条记录存的正是这个全局偏移，这也是"回源"能直接定位的原因。
### 4.3 高频追问与陷阱
- 陷阱：以为消息"以 ConsumeQueue 为主"——它只是索引，损坏可由 CommitLog 通过 ReputMessageService 重建。
- 陷阱：磁盘打满就归因于消息太多——更常见是有消费者长期堆积，导致文件无法过期清理。
- 追问：为什么 ConsumeQueue 单文件按"条数"而非字节大小切分？便于按条数快速索引与内存映射预估。
### 4.4 配置/命令示例
```properties
fileReservedTime=72
deleteWhen=04
diskMaxUsedSpaceRatio=75
mapedFileSizeCommitLog=1073741824   # 1GB
mapedFileSizeConsumeQueue=300000    # 每文件条数
```
### 4.5 一句话速记
CommitLog 顺序写是唯一真相，ConsumeQueue 是队列级索引，IndexFile 是 key 级索引，mmap+页缓存撑起全部读写性能。

## 5. 刷盘与主从复制：可靠性四象限
### 5.1 核心原理
两个独立维度：**刷盘** FlushDiskType = SYNC_FLUSH / ASYNC_FLUSH（默认异步）；**复制** BrokerRole = SYNC_MASTER / ASYNC_MASTER（默认异步）。二者防护的是不同故障——刷盘防断电，复制防节点丢失。

| 组合 | 单机断电 | Master 挂且切 Slave | 性能 | 可靠性 |
| --- | --- | --- | --- | --- |
| 异步刷盘 + 异步复制（默认） | 丢未刷盘消息 | 丢未复制消息 | 最高 | 最低 |
| 同步刷盘 + 异步复制 | 不丢 | 丢未复制消息 | 中 | 中 |
| 异步刷盘 + 同步复制 | 丢未刷盘消息 | 不丢 | 中 | 中 |
| 同步刷盘 + 同步复制 | 不丢 | 不丢 | 最低 | 最高 |
### 5.2 面试问答
**Q1：生产环境怎么选？**
A：结论——默认组合适合"允许极小概率丢消息"的绝大多数业务；资金、订单类链路用同步刷盘 + 同步复制。原因：同步刷盘 RT 从毫秒升到数毫秒至十几毫秒（受 fsync 限制），同步复制再叠加一次网络 RTT，吞吐明显下降。折中：异步刷盘 + 同步复制，并在生产端用本地消息表兜底。

**Q2：同步刷盘是不是每条消息都 fsync？**
A：结论——不是，采用**组提交（GroupCommit）**：多条消息的刷盘请求合并成一批，由刷盘线程统一 fsync 后批量唤醒等待者，既保证"返回成功即已落盘"，又摊薄 fsync 次数。

**Q3：同步复制在哪个阶段算成功？**
A：Master 收到 Slave 的 ACK 后才向 Producer 返回成功。注意 4.x 的同步复制只是"主从复制确认"，不是多数派 Raft 提交，因此不解决主挂后的数据分叉——这正是 DLedger/Controller 要解决的问题。

**Q4：异步复制的 Slave 落后太多会怎样？**
A：主挂切换后会丢失大量数据，消费位点大幅回退导致重复消费。运维必须监控主从 offset 差值（Dashboard / `mqadmin`）并设告警。
### 5.3 高频追问与陷阱
- 陷阱：把"同步复制"当成"多数派复制"——4.x 只有主从复制，一主一从或多从，不是 Raft。
- 陷阱：只改 BrokerRole 不改刷盘方式，就宣称"消息不丢"。
- 追问：为什么必须把刷盘与复制分开论证？因为它们防的是两类不同故障，缺一不可。
### 5.4 配置/命令示例
```properties
# 高可靠配置
brokerRole=SYNC_MASTER
flushDiskType=SYNC_FLUSH
# 查看主从状态：sh mqadmin brokerStatus -n 127.0.0.1:9876 -b 192.168.1.10:10911
```
### 5.5 一句话速记
异步刷盘怕断电、异步复制怕主挂，两者都同步最可靠但最慢，组合选择取决于能接受哪种丢法。

## 6. 消费模式、位点管理与 Rebalance
### 6.1 核心原理
**集群模式（Clustering，默认）**：组内一条消息只被一个消费者消费，队列在组内分配，offset 存 Broker（consumerOffset.json，默认每 5s 持久化）。**广播模式（Broadcasting）**：每个消费者收全量消息，offset 存本地 offsets.json。**Push 本质是长轮询 Pull**：Broker 无消息时挂起请求（brokerSuspendMaxTimeMillis 默认 15s），有消息立即返回。**Rebalance**：在队列数与消费者数之间重新分配队列。
### 6.2 面试问答
**Q1：Push 和 Pull 怎么选？**
A：结论——业务一般用 `DefaultMQPushConsumer`，它内部就是长轮询 Pull；需要精确控制 offset 与批量节奏时用 Pull（5.x 推荐 SimpleConsumer）。原因：纯 Push 无法反压，纯即时 Pull 空轮询浪费资源。

**Q2：Rebalance 的触发条件有哪些？**
A：① 消费者实例增减；② 订阅关系变化；③ Topic 队列数变化；④ Broker 上下线。默认分配策略 `AllocateMessageQueueAveragely`（连续区间平均分配），另有环形平均、一致性哈希、机房内分配等策略。

**Q3：Rebalance 有什么危害？**
A：① **消费暂停**：相关队列短暂停止消费；② **重复消费**：新消费者从上次提交的位点重来；③ **可能漏消费**：Pull 模式下手动管理 offset 不当，消息已拉取未处理完就被收走。优化：组内订阅关系必须完全一致、避免频繁上下线、队列数按并行度设计。

**Q4：offset 存在哪、什么时候提交？**
A：集群模式存 Broker、广播模式存本地。并发消费下位点按"队列中最小的已成功位点"推进（保证不漏），因此**部分失败必然带来重复消费**——这是必须做幂等的根本原因。

**Q5：新消费组从哪儿开始消费？**
A：由 `ConsumeFromWhere` 决定：`CONSUME_FROM_LAST_OFFSET`（默认，从队列最大位点）、`CONSUME_FROM_FIRST_OFFSET`（补历史数据）、`CONSUME_FROM_TIMESTAMP`（从指定时间）。注意该配置**只对该消费组首次启动生效**，之后以已持久化 offset 为准。
### 6.3 高频追问与陷阱
- 陷阱：组内机器订阅不同 Tag/Topic → 持续 Rebalance，这是线上最常见的消费事故。
- 陷阱：以为位点"按消息逐条提交"——实际按队列位点推进，被跳过的消息等同于已消费。
- 追问：队列数少于消费者数会怎样？多余实例分配不到队列，只能空转，浪费资源。
### 6.4 配置/命令示例
```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("order-group");
consumer.setMessageModel(MessageModel.CLUSTERING);
consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);
consumer.setConsumeMessageBatchMaxSize(32);
consumer.subscribe("OrderTopic", "PAY || SHIP");
consumer.registerMessageListener((MessageListenerConcurrently) (msgs, ctx) -> ConsumeConcurrentlyStatus.CONSUME_SUCCESS);
```
### 6.5 一句话速记
Push 是长轮询的 Pull，集群模式位点在 Broker，并发消费必然重复所以必须幂等，Rebalance 的根因是分配不稳定。

## 7. 重试队列、死信队列与消息过滤
### 7.1 核心原理
消费失败返回 `RECONSUME_LATER` 后，消息进入 **`%RETRY%{consumerGroup}`** 重试 Topic（消费者启动时自动订阅）。重试间隔复用延迟级别，第 N 次重试使用 `delayTimeLevel = N + 3`（约从 10s 起递增）。并发消费默认最多重试 **16 次**，仍失败进入死信 Topic **`%DLQ%{group}`**，该 Topic 默认权限受限，需人工处理。**消息过滤**：Tag 过滤（ConsumeQueue 中带 Tag 哈希，Broker 初筛）与 SQL92 过滤（基于消息属性，需 `enablePropertyFilter=true`）。
### 7.2 面试问答
**Q1：重试次数和间隔能改吗？**
A：能。消费者可设 `setMaxReconsumeTimes(n)`，业务侧通常设 3~5 次（再多只会加剧积压且大概率仍失败）。顺序消费（MessageListenerOrderly）默认重试次数为 Integer.MAX_VALUE，必须设上限并告警。重试间隔档位由 Broker 的 `messageDelayLevel` 全局控制。

**Q2：死信队列怎么处理？**
A：结论——死信队列不是垃圾桶，必须有人管：① 建运维消费组订阅 `%DLQ%xxx` 并告警；② 定位失败原因（数据问题/依赖故障/代码 bug）；③ 修复后用工具重投回原 Topic。原则：能修复重投的别丢，确认无意义的丢弃要留档。

**Q3：什么情况该不重试直接丢弃？**
A：结论——只有"重试必然失败的确定性错误"才跳过：参数非法、消息格式错误、业务状态已终结（订单已取消）。做法：监听器内捕获这些异常，返回 CONSUME_SUCCESS 并打告警日志；网络超时、DB 抖动等仍返回 RECONSUME_LATER。

**Q4：Tag 与 SQL92 过滤怎么选？**
A：结论——能用 Tag 就用 Tag，哈希在 ConsumeQueue 中可直接判断，开销小；需要属性范围条件（如 amount > 100 AND region = 'hz'）时才用 SQL92，注意 Broker 端解析开销与属性类型限制。生产推荐"Tag 粗筛 + 客户端精筛"。

**Q5：用 SQL92 过滤后"一条都消费不到"是什么原因？**
A：最常见是属性名或类型写错（属性不存在、数值被写成字符串），Broker 会把消息全部滤掉；其次是该 Topic 未开启 `enablePropertyFilter`。
### 7.3 高频追问与陷阱
- 陷阱：所有异常都 RECONSUME_LATER，毒消息会堵死队列并给下游造成 16 倍压力。
- 陷阱：以为重试是立即重试——实际按延迟级别递增，第一次约 10s，最大可到小时级。
- 追问：`sendMessageBack` 与 `RECONSUME_LATER` 的差别？前者可指定延迟级别并支持投到其他 Topic，后者走标准重试链路。
### 7.4 配置/命令示例
```java
consumer.setMaxReconsumeTimes(3);
listener = (msgs, ctx) -> {
    try { handle(msgs); return ConsumeConcurrentlyStatus.CONSUME_SUCCESS; }
    catch (BizFatalException e) { log.error("丢弃毒消息", e); return ConsumeConcurrentlyStatus.CONSUME_SUCCESS; }
    catch (Exception e) { return ConsumeConcurrentlyStatus.RECONSUME_LATER; }
};
```
### 7.5 一句话速记
重试队列 %RETRY% 默认 16 次、间隔按延迟级别递增，耗尽进 %DLQ%，毒消息要主动丢弃而非无脑重试。

## 8. 顺序消息与事务消息
### 8.1 核心原理
**顺序消息**：全局有序 = Topic 单队列 + 消费单线程（吞吐极低，生产不用）；分区有序 = 生产端 Selector 按业务键落同队列 + 消费端 `MessageListenerOrderly` 对队列加锁串行消费（Rebalance 锁 + 本地消费锁）。**事务消息**：半消息先写 `RMQ_SYS_TRANS_HALF_TOPIC`（消费者不可见）→ 执行本地事务 → 发送 Commit/Rollback → 长时间未确认由 Broker **回查**（默认间隔 60s、最多 15 次，超限默认丢弃）。
### 8.2 面试问答
**Q1：怎么保证同一订单的消息顺序？**
A：① 发送端以 orderId 为 key 走 MessageQueueSelector 落同队列；② 消费端用有序消费模式，队列级别串行；③ 下游处理本身幂等（顺序只保证先到先处理，不保证不重复）。

**Q2：有序消费失败会怎样？**
A：MessageListenerOrderly 返回 RECONSUME_LATER 会**原地重试并阻塞该队列**，不会跳过继续消费后面的消息，默认重试次数极大，容易导致整队列积压甚至消费停滞。必须设置合理重试上限、确定性错误直接跳过、对该队列做积压告警。

**Q3：顺序消息怎么提高吞吐？**
A：① 增加队列数（并行度 = 队列数）；② 增加消费者实例（不超过队列数）；③ 单消费者内每队列独立加锁，天然并行；④ 5.0 的 POP 消费避免队列独占，提升资源利用率。

**Q4：事务消息解决什么问题，不解决什么？**
A：解决"本地事务与消息发送的原子性"：本地成功则消息必发、本地失败则消息必不发。不解决消费端是否处理成功（靠幂等 + 重试）。典型用于跨服务最终一致：下单扣库存、支付后加积分。

**Q5：为什么需要回查，不能只靠 Commit？**
A：因为第三步 Commit 本身可能失败（网络抖动、Producer 崩溃）。没有回查，Broker 无法判断半消息该提交还是回滚，只能永久挂起；回查把决策权交回最清楚本地事务结果的 Producer。

**Q6：完整落地链路怎么设计？**
A：① 业务库建本地事务表（业务 ID、消息内容、状态）；② 本地事务内同时写业务数据与事务表；③ 发事务消息，`executeLocalTransaction` 返回 COMMIT_MESSAGE；④ 消费端幂等处理；⑤ 定时任务扫长期 PENDING 记录做重发与对账；⑥ `checkLocalTransaction` 只查本地表（回查有超时，默认约 6s）。
### 8.3 高频追问与陷阱
- 陷阱：本地事务写在 send 之前，形成"消息已发出但本地事务失败"的窗口。
- 陷阱：以为事务消息能保证强一致与隔离——中间态对下游可见（库存已扣、积分未加），必须容忍短暂不一致并做对账。
- 追问：事务消息会重复吗？会，回查期间可能重发、Commit 可能重复投递，消费端幂等是必选项。
### 8.4 配置/命令示例
```java
producer.setTransactionListener(new TransactionListener() {
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        return doLocalTx(arg) ? LocalTransactionState.COMMIT_MESSAGE : LocalTransactionState.ROLLBACK_MESSAGE;
    }
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        return txTableMapper.exists(msg.getKeys()) ? LocalTransactionState.COMMIT_MESSAGE : LocalTransactionState.ROLLBACK_MESSAGE;
    }
});
producer.sendMessageInTransaction(msg, orderId);
```
### 8.5 一句话速记
分区有序 = Selector 落同队列 + Orderly 串行消费；事务消息 = 半消息 + 本地事务 + Commit/Rollback + 回查（60s×15 次）。

## 9. 消息幂等与消息丢失防范
### 9.1 核心原理
重复的根因：① 生产端超时重试（实际已成功）；② 消费位点未推进就 Rebalance；③ 消费失败重试；④ 事务消息回查重复 Commit。因此"不重复"无法由 MQ 单独保证，只能消费端幂等：唯一索引（DB 兜底最强）、去重表、Redis `SET NX EX`、状态机（只允许合法迁移）、乐观锁版本号。

| 丢失环节 | 原因 | 防范 |
| --- | --- | --- |
| 生产 | 异步/单向失败未处理、重试耗尽、Broker 拒绝 | 同步发送或异步回调校验 SendStatus、开启重试、本地消息表兜底并告警 |
| 存储 | 异步刷盘未落盘断电、异步复制主挂丢数、Slave 落后、磁盘满、消息过期被删 | 关键链路 SYNC_FLUSH + SYNC_MASTER、监控主从 offset 差、磁盘水位告警、合理设置 fileReservedTime |
| 消费 | 提前 ack、并发消费部分失败仍推进位点、线程池拒绝、下游事务回滚但位点已提交 | 处理完成后才返回消费结果、异常不外抛、消费端幂等、失败进重试队列并监控 DLQ |
### 9.2 面试问答
**Q1：为什么 MQ 做不到只投递一次？**
A：结论——分布式系统的必然：网络超时后无法区分"消息没到"与"响应没回来"，要保证不丢就只能重发，重发意味着可能重复。Exactly-Once 在端到端场景退化为"至少一次投递 + 消费端去重"。

**Q2：用 Redis 去重要注意什么？**
A：① 用 `SET key value NX EX` 原子写入；② 过期时间必须大于业务最大重复窗口（含全部重试间隔），否则过期后仍会重复；③ Redis 主从切换可能丢数据，必须有 DB 唯一索引兜底。

**Q3：唯一索引方案怎么做？**
A：以"业务键 + 消息类型"建唯一索引，插入冲突即视为已处理并直接返回成功。优点：绝对可靠、实现简单；缺点：DB 压力大、DDL 受限、不适合超高 QPS。

**Q4：怎么排查"用户说没收到短信"？**
A：倒查四步：① 生产侧日志有无 SendResult 及状态；② Broker 是否有该消息（`mqadmin queryMsgByKey`）；③ 消费组 offset 是否推进、有无堆积；④ 消费端业务日志与死信队列。四步能锁定 90% 的丢失环节。

**Q5：哪种写法最容易丢消息？**
A：收到消息后立即丢给线程池并返回 CONSUME_SUCCESS，等价于"提前 ack"——进程重启、线程池拒绝、任务异常都会静默丢消息。正确做法是处理完成后再返回消费结果。

**Q6：幂等与防重是一回事吗？**
A：不是。防重是阻止重复请求进来（前端置灰、分布式锁），幂等是重复请求进来结果一致。高并发下防重不可靠，**幂等是最后防线**。
### 9.3 高频追问与陷阱
- 陷阱：幂等只做在最外层 HTTP 接口，漏掉 MQ 消费入口。
- 陷阱：幂等键选 messageId——同一业务动作重发时 messageId 会变，等于没去重；应选业务键。
- 追问：能不能做到绝对不丢？单机最可靠是同步刷盘 + 同步复制 + 同步发送，仍防不住机房级灾难，工程上用"分层保证 + 对账兜底"。
### 9.4 配置/命令示例
```java
Boolean first = redis.opsForValue().setIfAbsent("mq:dedup:" + msg.getKey(), "1", 24, TimeUnit.HOURS);
if (Boolean.FALSE.equals(first)) return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
try { orderService.handleWithUniqueIndex(payload); }
catch (DuplicateKeyException e) { log.warn("已处理过，忽略"); }
```
### 9.5 一句话速记
MQ 只保证至少一次，幂等只能自己做：唯一索引最可靠、Redis 最快、状态机最优雅；生产看 SendStatus、存储看刷盘与副本、消费看"处理完成才 ack"。

## 10. 消息积压、5.0 新特性与监控运维
### 10.1 核心原理
积压 = 生产速率长期大于消费速率，或短期流量尖峰。排查顺序：消费组是否掉线/Rebalance 不停 → 是否有毒消息卡死单队列 → 下游依赖是否变慢 → 队列数是否小于消费者实例数 → 观察 diff 与消费 TPS 曲线。**5.0 关键变化**：无状态 **Proxy** 接入层（gRPC、多语言、统一治理）、**POP 消费**（消费后即标记删除，突破队列独占）、定时消息秒级精度、**Controller 模式主从自动切换**（不依赖 DLedger）、分级存储与可观测增强。**监控四指标**：堆积量、消费 TPS、消费 RT、DLQ 数量。
### 10.2 面试问答
**Q1：积压千万条怎么快速消掉？**
A：分级处理：① 提并行度（实例数 ≤ 队列数，必要时临时扩队列）；② 批量消费（`setConsumeMessageBatchMaxSize` 提到 32/64 并批量写 DB）；③ 跳过非关键消息（直接返回 SUCCESS）；④ 临时转发——写个消费者把消息转到队列更多的新 Topic，再用大量消费者并肩消费（务必提交原 Topic 位点）；⑤ 关闭重试避免重试消息反复占位。

**Q2：积压时为什么不能盲目重启消费者？**
A：重启触发 Rebalance，期间消费暂停并会重复消费一批消息；若积压根因是"处理慢"，重启后依旧积压，反而叠加重复处理压力，必须先定位根因。

**Q3：怎么预防积压？**
A：① 队列数按峰值并行度设计（如 16/32）；② 消费者各起两实例避免单点；③ 消费逻辑轻量化、重活异步化但不提前 ack；④ 下游降级预案（缓存、限流）；⑤ 监控堆积量与消费 RT 并设告警阈值。

**Q4：5.0 的 POP 消费解决了什么？**
A：结论——解决队列独占导致的负载不均与顺序消息并行度受限。原因：传统模式下队列被消费者独占、扩容受队列数限制；POP 以"在途租约"方式分配消息，单队列可被多个消费者并行消费，失败超时自动重投递，更适合云原生弹性伸缩。

**Q5：5.0 的 Proxy 有什么好处？**
A：① 客户端与 Broker 解耦，Broker 专注存储；② 支持 gRPC，多语言客户端统一；③ 无状态可水平扩容，适配容器/K8s；④ 便于鉴权、限流、灰度等治理。代价：多一跳网络，需要关注 Proxy 自身容量与延迟。

**Q6：Spring Boot 集成要点？**
A：用 `rocketmq-spring-boot-starter`：`RocketMQTemplate` 负责发送（syncSend/asyncSend/sendOneWay/syncSendOrderly/sendMessageInTransaction），`@RocketMQMessageListener(topic, consumerGroup, selectorExpression, consumeMode, maxReconsumeTimes)` 声明消费者，`@RocketMQTransactionListener` 实现本地事务与回查。注意 selectorExpression 是 Tag 表达式；Orderly 与并发不能混用；**生产必须关闭 autoCreateTopicEnable** 并显式建 Topic 与队列数。
### 10.3 高频追问与陷阱
- 陷阱：以为"加消费者就能扩容"——队列数才是并行度上限。
- 陷阱：生产环境开启 `autoCreateTopicEnable`，会生成大量只有 1 个队列的 Topic 且路由混乱。
- 追问：需要多少队列？队列数 ≈ 峰值 TPS × 单条平均处理耗时(秒) × 安全系数(1.5~2)，并保证 ≥ 消费者实例数。
### 10.4 配置/命令示例
```yaml
rocketmq:
  name-server: 127.0.0.1:9876
  producer: { group: order-producer-group, send-message-timeout: 3000, retry-times-when-send-failed: 2 }
```

```bash
sh mqadmin consumerProgress -n 127.0.0.1:9876 -g order-consumer-group -s   # 堆积与消费进度
sh mqadmin queryMsgByKey -n 127.0.0.1:9876 -t OrderTopic -k ORD20240501001  # 消息轨迹
```
### 10.5 一句话速记
并行度受队列数限制、批量消费最见效、转发要提交原位点；监控盯堆积量、TPS、RT、DLQ 四项，5.0 看 Proxy/POP/秒级定时/自动切换。

## 自测题（含答案要点）
1. **MQ 四大使用场景？** 解耦、异步、削峰、最终一致性，每个都附带一致性与运维成本。
2. **RocketMQ 的四类角色？** NameServer（无状态路由）、Broker（存储转发，主从）、Producer Group、Consumer Group。
3. **NameServer 为什么取代 ZK？** 路由注册只需 AP；ZK 是 CP，选举期不可用；NameServer 节点间不通信、可任意扩展。
4. **Topic 与 Queue 的关系？** Topic 逻辑分类，Queue 是物理存储与并行消费单位，分布于不同 Broker。
5. **三种发送方式？** 同步（可靠）、异步（高吞吐，回调需处理失败）、单向（可能丢，适合日志）。
6. **4.x 延迟消息几级？** 18 级（1s~2h），level 从 1 开始，0 表示不延迟；5.0 支持秒级任意定时。
7. **CommitLog 为什么全 Topic 混写？** 顺序追加提升吞吐，用 ConsumeQueue 索引换读取性能。
8. **ConsumeQueue 一条索引多大、含什么？** 20B：8B 物理偏移 + 4B 消息长度 + 8B Tag 哈希。
9. **刷盘+复制四种组合怎么选？** 默认异步+异步最快最弱；同步刷盘+同步复制最可靠但最慢。
10. **Push 与 Pull 的关系？** Push 底层是长轮询 Pull，Broker 最多挂起约 15s。
11. **集群与广播模式 offset 存哪？** 集群存 Broker（每 5s 持久化），广播存本地 offsets.json。
12. **Rebalance 触发条件与危害？** 成员增减、订阅变化、队列数变化、Broker 上下线；危害是暂停消费、重复消费、可能漏消费。
13. **重试与死信队列名？** `%RETRY%{group}`、`%DLQ%{group}`；默认重试 16 次，间隔 delayTimeLevel = N+3。
14. **顺序消息怎么做？** Selector 按业务键落同队列 + Orderly 串行消费；全局有序需单队列单线程。
15. **事务消息三阶段？** 半消息 → 本地事务 → Commit/Rollback；失联由 Broker 回查（默认 60s、最多 15 次）。
16. **为什么必须幂等？** 至少一次投递 + 重试 + Rebalance 决定重复必然发生；方案为唯一索引/去重表/Redis/状态机。
17. **丢消息的三环节与对策？** 生产（同步发送+本地消息表）、存储（同步刷盘+同步复制+磁盘水位）、消费（处理完成才 ack）。
18. **积压怎么处理？** 提并行度（受队列数限制）、批量消费、跳过非关键消息、临时转发到多队列 Topic。
19. **5.0 核心新特性？** Proxy 无状态接入 + gRPC、POP 消费、秒级定时消息、Controller 自动主从切换、分级存储与可观测增强。
20. **要监控哪些指标？** 堆积量、消费 TPS、消费 RT、DLQ 数量，外加 Broker 磁盘水位与主从同步延迟。

## 背诵卡片
1. 解耦、异步、削峰、最终一致——四大场景，每个都要还一份一致性债。
2. MQ 引入的代价：可用性耦合、一致性债、运维复杂度、排查难度上升。
3. NameServer 无状态、节点间不通信、客户端 30s 拉路由。
4. brokerId=0 为 Master，>0 为 Slave，brokerName 相同即主从组。
5. 4.x 主从无自动选主，需 DLedger 或 5.0 Controller。
6. 消费并行度上限 = 队列数（集群模式）。
7. 发送三态：同步可靠、异步高吞吐、单向可能丢。
8. 延迟消息 18 级：1s 5s 10s 30s 1m~10m 20m 30m 1h 2h。
9. 5.0 用 setDeliverTimeMs 支持秒级定时消息。
10. 顺序发送用 MessageQueueSelector 按业务键落同队列。
11. CommitLog 单文件约 1GB，顺序追加，是唯一真实数据源。
12. ConsumeQueue 每条 20B：offset 8 + size 4 + tagHash 8。
13. IndexFile 支持按 key 与时间区间查消息。
14. mmap + PageCache：写入即写内存，由 OS 刷盘。
15. flushDiskType：SYNC_FLUSH / ASYNC_FLUSH（默认异步）。
16. brokerRole：SYNC_MASTER / ASYNC_MASTER（默认异步）。
17. 最高可靠性 = 同步刷盘 + 同步复制，代价是 RT。
18. 同步刷盘实际是组提交（GroupCommit），不是每条 fsync。
19. 文件过期：deleteWhen=04、fileReservedTime=72h、磁盘水位触发删除。
20. Push 本质是长轮询 Pull，Broker 挂起请求约 15s。
21. 集群模式 offset 在 Broker，广播模式在本地 offsets.json。
22. ConsumeFromWhere 只对该消费组首次启动生效。
23. Rebalance 触发：成员变化、订阅变化、队列数变化、Broker 上下线。
24. Rebalance 三危害：消费暂停、重复消费、可能漏消费。
25. 订阅关系不一致是 Rebalance 风暴的头号原因。
26. 重试队列 %RETRY%{group}，默认最多重试 16 次。
27. 重试间隔 delayTimeLevel = retryTimes + 3。
28. 死信队列 %DLQ%{group}，必须有人消费并告警。
29. MessageListenerOrderly 失败会阻塞队列，必须设重试上限。
30. 事务消息：半消息 → 本地事务 → Commit/Rollback → 回查。
31. 回查默认 60s 一次、最多 15 次，超限默认丢弃。
32. 事务消息只保证最终一致，不保证隔离性。
33. 幂等三件套：唯一索引、去重表、Redis SETNX + 状态机。
34. 丢消息三环节：生产、存储、消费；异步消费立即返回 SUCCESS 等价于提前 ack。
35. 积压四板斧：提并行度、批量消费、跳过非关键、临时转发。
36. 消息过滤：Tag 粗筛（ConsumeQueue 哈希）+ 客户端精筛；SQL92 需 enablePropertyFilter。
37. 监控四指标：堆积量、消费 TPS、消费 RT、DLQ 数量。
38. 生产环境关闭 autoCreateTopicEnable，显式建 Topic 并定队列数。
39. Spring Boot：RocketMQTemplate 发送、@RocketMQMessageListener 消费、@RocketMQTransactionListener 事务。
40. 队列数估算：峰值 TPS × 单条耗时(秒) × 1.5~2，且不小于消费者实例数。
