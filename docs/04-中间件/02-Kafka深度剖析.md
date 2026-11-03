# Kafka深度剖析
> **定位**：把 Kafka 的架构概念、生产者/消费者机制、高性能原理、可靠性语义与线上排查串成一条可复述的主线，兼顾 2.x 与 3.x/4.x 差异。
> **重要度**：★★★★★
> **建议复习时长**：5 小时
> **关联文档**：[01-消息队列与RocketMQ](./01-消息队列与RocketMQ.md)、[03-RabbitMQ与MQ选型对比](./03-RabbitMQ与MQ选型对比.md)、[04-高并发高可用架构设计](../05-分布式与架构/04-高并发高可用架构设计.md)、[03-CICD与监控告警](../07-工程化与DevOps/03-CICD与监控告警.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| 核心架构与 HW/LEO/ISR 概念 | ★★★★★ | 八股原理 |
| 生产者分区器、拦截器与批量机制 | ★★★★☆ | 配置实践 |
| acks、重试与顺序性 | ★★★★★ | 八股原理 |
| 幂等生产者与事务（Exactly-Once） | ★★★★★ | 架构设计 |
| 消费组与四种分区分配策略 | ★★★★★ | 八股原理 |
| Rebalance、位移提交与消费语义 | ★★★★★ | 故障复盘 |
| 高性能原因与存储结构 | ★★★★★ | 源码追问 |
| 副本同步、ISR 与 Leader 选举 | ★★★★★ | 八股原理 |
| KRaft 模式与元数据演进 | ★★★★☆ | 架构设计 |
| 常见问题排查与参数调优 | ★★★★★ | 故障复盘 |

## 1. 核心架构与关键概念
### 1.1 核心原理
角色与概念：**Broker**（一个 Kafka 进程/节点）、**Topic**（逻辑分类）、**Partition**（物理单位，有序不可变追加日志，是并行与扩展的最小单位）、**Replica**（副本，分为 Leader 与 Follower）、**AR**（所有副本集合）、**ISR**（与 Leader 保持同步的副本集合）、**OSR**（落后副本）、**HW**（高水位，= ISR 中最小的 LEO，消费者只能读到 HW 之前的数据）、**LEO**（日志末端偏移）、**Controller**（集群中一个 Broker 兼任，负责分区 Leader 选举与元数据变更）、**Coordinator**（消费组协调者，管理组成员与位移，位点存在 `__consumer_offsets` 中）。
### 1.2 面试问答
**Q1：HW 和 LEO 有什么区别，为什么要 HW？**
A：LEO 是每个副本自己的日志末端偏移，HW 是 ISR 全体 LEO 的最小值。结论——HW 是"已被足够副本确认、对消费者可见"的边界；消费者只能消费到 HW。原因：如果允许消费未复制的数据，Leader 挂掉后换 Leader 会导致已消费数据消失（数据回退）。

**Q2：ISR 是怎么维护的？**
A：Follower 定期向 Leader 拉取数据，若在 `replica.lag.time.max.ms`（默认 30000ms）内没有追上 Leader 的 LEO，则被移出 ISR；追平后重新加入。注意：新版本只用"时间"判断落后（不再用落后消息条数）。

**Q3：Controller 和 Coordinator 是不是一回事？**
A：不是。Controller 是集群级，负责分区 Leader 选举、Topic 创建删除、副本重分配这类元数据操作，整个集群同时只有一个；Coordinator 是消费组级，负责组成员管理与位移管理，一个消费组只对应一个 Coordinator（由 `__consumer_offsets` 的分区归属决定）。

**Q4：为什么一个分区只能被消费组内一个消费者消费？**
A：结论——这是消费组"分区分配"的定义：一个分区在同一时刻只分配给组内一个消费者，从而实现组内并行且不重复消费。原因：位移是"组 + 主题 + 分区"维度的，若多个消费者共读同一分区，位移会被互相覆盖。

**Q5：Kafka 为什么不做"队列数=消费者数"的强绑定？**
A：分区是"存储 + 并行"的统一单位，再划分一层队列只会增加复杂度。代价是分区数一旦确定不能减少，扩容消费者超过分区数就会出现空闲消费者。
### 1.3 高频追问与陷阱
- 陷阱：把 HW 说成"Leader 的 LEO"——那样 Follower 未同步的数据也会可见。
- 陷阱：混淆 AR 与 ISR，ISR 一定是 AR 的子集，AR = ISR + OSR。
- 追问：Leader Epoch 解决什么问题？作为"任期号"用于副本截断，替代早期单纯依赖 HW 的截断方式，避免 HW 回退导致的数据不一致（0.11+ 引入）。
### 1.4 配置/命令示例
```bash
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic order-topic
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group order-group
kafka-configs.sh --bootstrap-server localhost:9092 --entity-type topics --entity-name order-topic --describe
```
### 1.5 一句话速记
分区是有序的追加日志，HW 是可见边界，ISR 是同步副本集合，Controller 管元数据，Coordinator 管消费者组。

## 2. 生产者：分区器、拦截器与批量机制
### 2.1 核心原理
发送链路：`send()` → **拦截器 ProducerInterceptor** → **序列化器** → **分区器 Partitioner** → **RecordAccumulator**（按分区组织多个批次，`buffer.memory` 默认 32MB）→ **Sender 线程**按 Broker 聚合批量发送。批量参数：`batch.size`（默认 16384 字节）、`linger.ms`（默认 0）、`compression.type`（默认 none）。分区策略：有 key 时 `murmur2(key) % 分区数`；无 key 时 2.4+ 默认**粘性分区（sticky）**——先填满一个批次再换分区，替代早期轮询，以提升批量效率。
### 2.2 面试问答
**Q1：为什么要引入粘性分区策略？**
A：结论——为了在"无 key 消息"场景下凑更大的批次，降低延迟与请求数。原因：早期轮询会把连续消息打散到多个分区，每批次很快被切换打断，批次变小、请求变多。边界：粘性只在"批次满或 linger 到期"后切换，因此单分区会有短暂消息倾斜，但换来整体吞吐提升。

**Q2：buffer.memory 满了会怎样？**
A：结论——`send()` 阻塞最多 `max.block.ms`（默认 60000ms），超时抛 `TimeoutException`。原因：RecordAccumulator 用尽空间后无法再接收新记录。正确做法：捕获异常并落库补偿，或调大 buffer.memory、降低发送速率。

**Q3：batch.size 和 linger.ms 怎么配合？**
A：结论——两者共同决定"批次大小与等待时间"：批次达到 batch.size 立即发送，或等到 linger.ms 到期就发送（先到者为准）。吞吐优先调大两者；延迟优先设 `linger.ms=0` 并把 batch.size 调小。注意 linger.ms 是"最多等"，不是"必须等"。

**Q4：拦截器能做什么？**
A：`ProducerInterceptor` 可在发送前修改记录（加埋点、加追踪 ID）并在回调中统计成功失败；`ConsumerInterceptor` 可在消费前后做加工与埋点。注意拦截器里不要做阻塞 IO，它在发送线程上执行。

**Q5：自定义分区器要注意什么？**
A：① 分区数与公式要稳定（避免改公式导致历史数据与新数据不在同一分区）；② 要保证相同 key 一定落到同一分区，否则有序性失效；③ 生产环境慎用自定义，改分区策略等于改变数据分布。
### 2.3 高频追问与陷阱
- 陷阱：以为 `linger.ms=0` 就是"立刻发"——实际仍是批量发送，只是不额外等待。
- 陷阱：压缩能提升吞吐也会增加 CPU 与端到端延迟，`lz4`/`zstd` 是主流折中。
- 追问：批次在客户端压缩还是服务端压缩？客户端压缩（`compression.type`），Broker 侧也可再压缩，但会重复消耗 CPU。
### 2.4 配置/命令示例
```properties
batch.size=131072
linger.ms=20
compression.type=lz4
buffer.memory=67108864
max.request.size=1048576
```
### 2.5 一句话速记
拦截器 → 序列化 → 分区器 → RecordAccumulator 攒批 → Sender 发送；无 key 走粘性分区，batch.size 与 linger.ms 是吞吐与延迟的两个旋钮。

## 3. 生产者的 acks、重试与顺序性
### 3.1 核心原理
**acks**：`0`（发出即成功，最快最不可靠）、`1`（Leader 写入即成功，Leader 挂则可能丢）、`all/-1`（ISR 全部确认，最可靠）。**重试**：`retries`（2.1+ 默认无限，受 `delivery.timeout.ms` 默认 120000ms 总时限约束）、`retry.backoff.ms`（默认 100ms）。**顺序性**：`max.in.flight.requests.per.connection` 默认 5，未开启幂等时 >1 且发生重试会导致乱序；开启幂等后即使 in-flight >1 也能保序。
### 3.2 面试问答
**Q1：acks=all 就绝对不丢吗？**
A：不是。结论——还要满足 `min.insync.replicas >= 2` 且副本数 ≥3、关闭 `unclean.leader.election.enable`，并保证生产端正确处理发送异常。原因：acks=all 的含义是"ISR 内全部副本确认"，ISR 只剩 Leader 一个时它等价于 acks=1。

**Q2：acks=1 什么时候会丢？**
A：Leader 已写入内存但未刷盘/Follower 未同步时崩溃，新 Leader 没有这条数据，而 Producer 已收到成功响应。

**Q3：怎么保证分区内消息有序？**
A：① 相同 key 走同一分区；② 生产端设 `max.in.flight.requests.per.connection=1`，或开启幂等（`enable.idempotence=true`，此时 in-flight ≤5 仍保序）；③ 消费端单线程按分区顺序处理。注意：跨分区不保证顺序。

**Q4：重试会不会造成重复？**
A：会。Broker 已写入但响应丢失时，Producer 重试会造成重复——这是 Kafka"至少一次"语义的来源。解决：开启幂等生产者（Broker 按 PID + 序列号去重），或消费端幂等。

**Q5：Kafka 3.0 的默认值有什么重要变化？**
A：KIP-679 起，Producer 默认 `acks=all`、`enable.idempotence=true`（因此也要求 `retries>0` 且 `max.in.flight≤5`），可靠性默认值明显变强。2.x 的默认 `acks=1`、幂等关闭，升级时要留意吞吐与延迟的变化。
### 3.3 高频追问与陷阱
- 陷阱：只在 Producer 侧调 `acks=all`，却在 Broker 侧保持 `min.insync.replicas=1`。
- 陷阱：为保序把 in-flight 设为 1，吞吐下降明显却未评估；优先选"幂等 + in-flight 5"。
- 追问：`request.timeout.ms`（默认 30000）与 `delivery.timeout.ms`（默认 120000）的关系？后者是总时限，包含重试与等待，超过即抛异常。
### 3.4 配置/命令示例
```properties
acks=all
enable.idempotence=true
max.in.flight.requests.per.connection=5
retries=2147483647
delivery.timeout.ms=120000
# Broker 侧强约束
min.insync.replicas=2
unclean.leader.election.enable=false
```
### 3.5 一句话速记
acks 决定落几份、重试决定丢不丢、in-flight 决定乱不乱；可靠三板斧是 acks=all + min.insync.replicas≥2 + 幂等。

## 4. 幂等生产者与事务（Exactly-Once）
### 4.1 核心原理
**幂等生产者**：Producer 启动时由 Broker 分配 **PID**，每条消息携带 `<PID, 分区, 序列号>`；Broker 按分区缓存最近序列号并去重，实现"单会话内单分区不重复"。**事务**：配置 `transactional.id`（跨会话标识），通过 `initTransactions → beginTransaction → send → sendOffsetsToTransaction → commitTransaction/abortTransaction` 实现"多分区 + 位移提交"的原子写，事务状态存于 `__transaction_state` 内部主题。**端到端 Exactly-Once** = 幂等 + 事务 + 消费者 `isolation.level=read_committed`。
### 4.2 面试问答
**Q1：幂等生产者能保证什么、不能保证什么？**
A：能保证：同一 Producer 会话内、单个分区上不产生重复（Broker 按 PID+分区+序列号去重）。不能保证：跨会话（Producer 重启后 PID 变化，历史上的重复无法识别）、跨分区原子性、消费端不重复。

**Q2：事务实现 Exactly-Once 的完整条件？**
A：① Producer 设 `transactional.id` 并开启幂等；② 消费端设 `isolation.level=read_committed`（只读已提交，未提交消息被跳过）；③ 处理逻辑用"读-处理-写"模式并把位移提交放进同一事务；④ 失败可 abort，`read_committed` 消费者看不到被中止的数据。

**Q3：read_committed 与 read_uncommitted 的区别？**
A：`read_uncommitted`（默认）会读到未提交甚至会被回滚的消息；`read_committed` 只读到已提交数据，消费进度受 **LSO（Last Stable Offset）** 限制——即"第一条未完成事务的位置"，因此未完成的长事务会阻塞消费进度（这也是事务过大导致消费延迟的原因）。

**Q4：事务有什么代价？**
A：额外两次事务控制请求（begin/commit）、事务超时（`transaction.timeout.ms` 默认 60000）与协调者交互、read_committed 的消费阻塞风险、吞吐下降。**能靠业务幂等解决的问题，不必上事务。**

**Q5：Exactly-Once 真能端到端做到吗？**
A：结论——只在 Kafka 内部（读-处理-写 Kafka）成立，Kafka Streams 就是典型。一旦有外部系统（写 MySQL、调 HTTP），仍需业务幂等或两阶段提交，主流做法是"至少一次 + 幂等去重"。
### 4.3 高频追问与陷阱
- 陷阱：只开 `transactional.id` 却没开幂等（事务要求 `enable.idempotence=true`，未开时 Kafka 会自动开启或报错，以版本文档为准）。
- 陷阱：以为开了事务就不会重复消费——消费位移未在事务内提交时仍会重复。
- 追问：事务超时后会发生什么？事务被协调者中止，Producer 后续操作报错，需要重建事务。
### 4.4 配置/命令示例
```java
producer.initTransactions();
try {
    producer.beginTransaction();
    producer.send(record1);
    producer.send(record2);
    producer.sendOffsetsToTransaction(offsets, groupMetadata);
    producer.commitTransaction();
} catch (Exception e) { producer.abortTransaction(); }
```
### 4.5 一句话速记
幂等管"单会话单分区不重"，事务管"多分区 + 位移原子"，端到端 Exactly-Once 只在 Kafka 内部闭环成立。

## 5. 消费者：消费组与分区分配策略
### 5.1 核心原理
消费者通过 `poll()` 拉取，属于**消费组**（`group.id`）实现组内分区独占、组间广播。四种分配策略：**Range**（按主题逐一分区排序后按消费者均分，默认，容易倾斜）、**RoundRobin**（所有主题分区整体轮询，分布更均匀）、**Sticky**（尽量保留上次分配并均衡，减少迁移）、**CooperativeSticky**（Sticky 的增量协作版，分区先回收再分配，避免全量停止，KIP-429）。关键参数：`session.timeout.ms`（默认 45000）、`heartbeat.interval.ms`（默认 3000）、`max.poll.interval.ms`（默认 300000）、`max.poll.records`（默认 500）、`fetch.min.bytes`、`fetch.max.wait.ms`。
### 5.2 面试问答
**Q1：四种分配策略怎么选？**
A：结论——生产优先 CooperativeSticky（增量、迁移少、暂停短），追求最简单均匀分布可用 RoundRobin，Range 在"多主题、分区数不整除"时容易倾斜（消费者订阅多个主题时尤为明显）。注意：同一组内所有消费者必须使用相同的分配策略与协议，混用会导致组无法稳定。

**Q2：消费者数超过分区数会怎样？**
A：多出来的消费者分配不到分区，只能空转（发心跳但不消费），属于资源浪费。因此并行度上限 = 分区数，扩容消费者前先看分区数。

**Q3：session.timeout.ms 和 max.poll.interval.ms 有什么区别？**
A：前者是"心跳超时"——超过它未收到心跳则判定消费者下线；后者是"两次 poll 之间的最大间隔"——业务处理太慢超过它，消费者会被踢出组。两者都会触发 Rebalance，但排查方向完全不同：前者查网络与 GC，后者查消费逻辑耗时。

**Q4：怎么避免"重启就 Rebalance"？**
A：设置**静态成员** `group.instance.id`（KIP-345），消费者重启后在 `session.timeout.ms` 内回归可沿用原分配，避免大规模 Rebalance。适合滚动发布场景。

**Q5：Kafka 4.0 的消费组协议有什么变化？**
A：KIP-848 引入新的消费组协议（服务端负责分配，消费者侧计算量下降），配合解耦的再均衡机制使 Rebalance 更快、更稳定；3.7 起为预览，4.0 逐步成为默认，ZooKeeper 也被彻底移除。使用前仍需结合具体版本确认行为。
### 5.3 高频追问与陷阱
- 陷阱：组内消费者订阅的主题不一致——会引发持续 Rebalance。
- 陷阱：把 `max.poll.records` 调得很大却处理很慢，导致超过 `max.poll.interval.ms` 被踢出组。
- 追问：消费者数量与线程模型？KafkaConsumer 非线程安全，通常"一消费者一线程"，多消费者用多实例或 `ConcurrentMessageListenerContainer` 的并发度。
### 5.4 配置/命令示例
```properties
group.id=order-group
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
session.timeout.ms=45000
heartbeat.interval.ms=3000
max.poll.interval.ms=300000
max.poll.records=200
enable.auto.commit=false
isolation.level=read_committed
```
### 5.5 一句话速记
组内分区独占、并行度上限等于分区数；CooperativeSticky 最省迁移，静态成员免重启抖动，两个 timeout 对应两类故障。

## 6. Rebalance、位移提交与消费语义
### 6.1 核心原理
**Rebalance 触发**：成员加入/离开、订阅变更、分区数增加、心跳超时、处理超时（`max.poll.interval.ms`）、Coordinator 变更。**Eager 协议**：全部撤销再全量重分配（stop-the-world）；**Cooperative 协议**：仅回收需迁移的分区（增量）。**位移**：保存在内部主题 `__consumer_offsets`（默认 50 个分区，compact 清理），键为 `group + topic + partition`。提交方式：自动提交（`enable.auto.commit=true`，`auto.commit.interval.ms` 默认 5000）、手动同步 `commitSync()`、手动异步 `commitAsync()`；重置策略 `auto.offset.reset=latest|earliest|none`。
### 6.2 面试问答
**Q1：Rebalance 有什么危害？**
A：① 期间消费暂停（Eager 下所有消费者停止，影响面大）；② 分区迁移后从上次提交位移继续，造成重复消费；③ 大组高频 Rebalance 会导致消费 Lag 持续上涨（"Rebalance 风暴"）；④ 位移提交失败时可能重复或漏。

**Q2：自动提交会造成什么语义问题？**
A：`enable.auto.commit=true` 是在 poll 时提交"上一批"的位移，因此：若本批处理完但还没提交就崩溃 → 重复消费（至少一次）；若先提交后处理且处理失败 → 漏消费（至多一次）。结论：默认是"至少一次 + 部分重复"，要精确控制必须手动提交。

**Q3：手动提交的规范写法？**
A：处理完再来提交：`poll → 业务处理 → commitSync(offsets)`。若要提吞吐：多数批次用 `commitAsync` + 回调，最后一批或关闭前用 `commitSync` 兜底，避免"异步提交失败但后续成功提交覆盖了失败位移"。

**Q4：什么情况会重复消费？**
A：① 处理成功但位移未提交；② Rebalance 后新消费者从旧位移开始；③ 生产端重试导致重复消息；④ 事务 abort 后重放。所以消费端幂等是标配。

**Q5：什么情况会漏消费？**
A：① 先提交位移后处理，处理失败（自动提交 + 异步处理最常见）；② `auto.offset.reset=latest` 时新组从末尾开始（历史消息被视为已消费）；③ 位移被错误覆盖（异步提交乱序）。

**Q6：auto.offset.reset 怎么理解？**
A：只在"该组没有已提交位移或位移越界"时生效：`latest` 从末尾（默认）、`earliest` 从头、`none` 直接抛异常。注意：它不是你想要的"每次启动都从头读"，那需要新 group.id 或显式 `seekToBeginning`。
### 6.3 高频追问与陷阱
- 陷阱：用 `latency` 问题去解释 Lag，实际根因是 Rebalance 风暴（消费组不稳定）。
- 陷阱：`max.poll.interval.ms` 小于单批处理时间的极端情况会无限循环"被踢出-重平衡-再被踢出"。
- 追问：`__consumer_offsets` 为什么用 compact？只需保留每个 key 的最新值（最新位移），compact 清理最合适；它默认 50 个分区，与消费组数量、位移写入吞吐有关。
### 6.4 配置/命令示例
```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
    for (ConsumerRecord<String, String> r : records) { handle(r); }
    consumer.commitSync();   // 处理完成后再提交
}
```
### 6.5 一句话速记
Rebalance 触发六件事、危害是暂停与重复；位移存在 __consumer_offsets；"先处理后提交"是至少一次，"先提交后处理"是至多一次。

## 7. 高性能原因与存储结构
### 7.1 核心原理
**高性能六大原因**：① 顺序写磁盘（追加日志）；② 充分利用页缓存（PageCache，读写都走内存）；③ 零拷贝 `sendfile`（数据从页缓存直接到网卡，不经用户态）；④ 批量 + 压缩（减少网络与磁盘 IO）；⑤ 分区并行（横向扩展）；⑥ 稀疏索引（不必为每条消息建索引）。**存储结构**：每个分区一个目录，日志分段 **Segment**（`log.segment.bytes` 默认 1GB，或按 `log.roll.hours` 滚动），包含 `.log`（数据）、`.index`（偏移索引，默认每 4KB 一条，即 `index.interval.bytes`）、`.timeindex`（时间戳索引）、`.snapshot`、`leader-epoch-checkpoint`。**清理策略**：`log.cleanup.policy=delete`（默认，按 `log.retention.hours` 默认 168 小时 / `log.retention.bytes`）或 `compact`（按 key 保留最新值，用于位移与状态主题）。
### 7.2 面试问答
**Q1：零拷贝是怎么实现的，有什么限制？**
A：通过 `FileChannel.transferTo`（底层 `sendfile`）把页缓存数据直接送到 socket，避免用户态拷贝。限制：数据不能被修改（否则需重写）、压缩在客户端完成（Broker 不解压）、开启 TLS 时会退化（需要加密处理），因此 SSL 场景零拷贝收益下降。

**Q2：为什么 Kafka 用页缓存而不是自己管理内存？**
A：结论——避免 JVM 堆内对象开销与 GC 压力，同时让操作系统统一调度冷热数据。原因：堆内缓存大数据量会导致长时间 GC，而页缓存由 OS 按访问规律淘汰，服务重启也不会"缓存立刻清零"（同一台机器上的页缓存仍有效）。

**Q3：稀疏索引怎么定位消息？**
A：先根据目标 offset 在 `.index` 中二分找到"不超过目标的最大索引项"得到物理位置，再从该位置顺序扫描 `.log`（最多 4KB）找到目标消息；按时间查则先用 `.timeindex` 转成 offset 再走上面的流程。

**Q4：delete 和 compact 有什么区别？**
A：`delete` 按时间/大小整段删除，适合普通消息主题；`compact` 不删段，而是按 key 保留每个 key 的最新值（可选 `delete,compact` 组合，先 compact 再按时间删除），适合 `__consumer_offsets`、CDC 变更流、状态快照类主题。

**Q5：日志分段为什么必要？**
A：① 便于按段删除与清理（否则无法回收单条消息）；② 便于索引文件与日志文件对应；③ 活动段（active segment）写满才滚动，只有非活动段才可被清理。
### 7.3 高频追问与陷阱
- 陷阱：以为"零拷贝"对所有场景都生效——压缩+TLS+消息转换会破坏条件。
- 陷阱：把 `log.retention.ms` 与 `log.roll.ms` 混淆：前者删数据，后者切段，滚动与删除是两件事。
- 追问：调大 Segment 有什么影响？段越大，索引建立与删除粒度越粗，且清理不及时会占用更多磁盘。
### 7.4 配置/命令示例
```properties
log.dirs=/data/kafka-logs
log.segment.bytes=1073741824
log.roll.hours=168
log.retention.hours=168
log.retention.bytes=-1
log.cleanup.policy=delete
index.interval.bytes=4096
```
### 7.5 一句话速记
顺序写 + 页缓存 + 零拷贝 + 批量压缩 + 分区并行 = Kafka 的性能公式；段可删、索引可稀疏、compact 按 key 留最新。

## 8. 副本同步、ISR 与 Leader 选举
### 8.1 核心原理
Follower 主动向 Leader 拉取（`replica.fetch.max.bytes` 默认 1MB），落后超过 `replica.lag.time.max.ms`（默认 30000ms）被移出 ISR（收缩），追平后重新加入（扩张）。**可靠性三件套**：`replication.factor`（副本数）、`min.insync.replicas`（最少同步副本，默认 1，生产建议 2）、`unclean.leader.election.enable`（是否允许非 ISR 副本当 Leader，默认 false）。Leader 选举由 Controller 从 ISR 中选（通常选最前面的副本），原 Leader 恢复后可通过 `auto.leader.rebalance.enable` 归还（首选副本优先）。
### 8.2 面试问答
**Q1：ISR 收缩/扩张的判定标准是什么？**
A：按时间：Follower 在 `replica.lag.time.max.ms` 内未追上 Leader 的 LEO 即被剔除；追平后由 Leader 重新加入 ISR 并写入元数据。注意：不是按"落后多少条"，而是按"落后多长时间"。

**Q2：min.insync.replicas 与 acks 的关系？**
A：结论——`acks=all` 的实际含义是"ISR 中所有副本确认"，所以 `min.insync.replicas` 才是"最少要几份"。副本数 3 + min.insync.replicas 2 + acks=all 是主流可靠配置；若 ISR 降到低于 min.insync.replicas，Producer 写入会抛 `NotEnoughReplicas`（可用性换一致性）。

**Q3：unclean.leader.election.enable 设为 true 会怎样？**
A：允许 ISR 之外的落后副本当 Leader，换来可用性——分区能继续读写，但会丢失未同步的数据并造成数据回退（消费者可能看到"消息消失"）。结论：金融/订单类必须保持 false；允许丢数据的高可用优先场景才打开。

**Q4：Leader 选举流程是什么？**
A：① Controller 感知分区 Leader 失效（Broker 掉线由 Controller 通过会话超时发现，或由 Broker 上报）；② Controller 从 ISR 中挑选新 Leader（CRASH 场景优先选副本列表靠前的 ISR 成员）并更新元数据、通知所有 Broker 与客户端；③ 客户端通过元数据刷新获知新 Leader 并重连。若 ISR 为空，则分区 offline，只有 unclean 选举才能恢复。

**Q5：分区处于 offline 怎么处理？**
A：`kafka-topics.sh --describe` 会显示 Leader 为 none。步骤：确认副本所在 Broker 是否全部宕机 → 尝试恢复 Broker（首选，避免丢数据）→ 若确实无可用 ISR 且业务可容忍丢数据，再考虑 unclean 选举；同时检查是否由磁盘故障、副本重分配中断导致。
### 8.3 高频追问与陷阱
- 陷阱：只把副本数设为 3 就以为安全，实际 ISR 常掉到 1（follower 拉取慢、磁盘 IO 高）。
- 陷阱：把 `min.insync.replicas` 设成等于副本数，任一副本落后就不可写——可用性急剧下降。
- 追问：副本拉取的瓶颈在哪？常见是副本跨机房带宽、磁盘写能力、`num.replica.fetchers` 过小。
### 8.4 配置/命令示例
```properties
replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false
replica.lag.time.max.ms=30000
num.replica.fetchers=4
```

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --describe --under-replicated-partitions
kafka-topics.sh --bootstrap-server localhost:9092 --describe --unavailable-partitions
```
### 8.5 一句话速记
ISR 按时间判定落后，acks=all 只保证 ISR 全确认，min.insync.replicas 才是底线，允许 unclean 选举就是用数据换可用。

## 9. KRaft 模式与 Exactly-Once 语义
### 9.1 核心原理
**KRaft（KIP-500）**：用内部主题 `__cluster_metadata`（Raft 复制）替代 ZooKeeper 保存元数据，集群由若干 **Controller** 组成 quorum（3 或 5 个），Broker 向 Controller 注册并获取元数据。收益：① 去掉 ZK 依赖，部署与运维简化；② Controller 故障切换从"分钟级"降到"秒级"；③ 元数据变更走日志复制，支持**百万级分区**规模；④ 支持 combined 模式（Controller 与 Broker 同进程，适合小集群）。**生产就绪**：3.3 起宣布可用，3.5+ 成熟，4.0 起 ZooKeeper 模式被移除。
### 9.2 面试问答
**Q1：为什么要去掉 ZooKeeper？**
A：① ZK 成为元数据与分区规模瓶颈（分区越多，ZK watch 与元数据同步越慢）；② Controller 故障切换期间元数据不可用（分钟级），期间无法建 Topic、无法选举 Leader；③ 两套系统带来部署、监控、调优的额外成本；④ Raft 日志复制比 ZK 的 ZAB + 本地缓存更直接可控。

**Q2：KRaft 下元数据管理有什么变化？**
A：元数据以事件日志形式写入 `__cluster_metadata`，Controller quorum 用 Raft 选举与复制，Broker 通过"元数据快照 + 增量日志"同步；客户端拉取元数据的机制不变，但来源变成 Controller。运维上不再需要维护 ZK 集群，改为一组 Controller。

**Q3：Controller 放在哪、几个合适？**
A：独立部署在 3 台（或 5 台，容忍 1/2 个节点故障）与 Broker 分离；小集群可用 combined 模式但生产不推荐与高负载 Broker 同机，避免资源争抢。Controller quorum 也必须是奇数。

**Q4：迁移到 KRaft 有什么注意点？**
A：① 版本要求（3.3+ 生产可用，4.0 移除 ZK）；② 迁移需要 ZK → KRaft 的元数据快照转换流程，务必先在测试环境演练；③ 检查是否有强依赖 ZK 的工具链（旧版监控、脚本）；④ 4.0 起无法回退到 ZK 模式。

**Q5：Exactly-Once 的完整条件是什么？**
A：生产端幂等 + 事务；消费端 `isolation.level=read_committed`；处理模式为"读 Kafka → 处理 → 写 Kafka"，并把位移提交放进事务；Broker 端需支持事务（`transaction.state.log.replication.factor` 等内部主题配置合理）。跨外部系统时退化为"至少一次 + 幂等"。
### 9.3 高频追问与陷阱
- 陷阱：以为 KRaft 只是"换了个元数据存储"——它还改变了 Controller 切换时间与分区规模上限。
- 陷阱：在 3.3 之前的生产集群（如 2.8/3.0）贸然使用 KRaft 预览特性。
- 追问：事务内部主题的分区数怎么定？`transaction.state.log.num.partitions` 默认 50，事务并发高时要评估。
### 9.4 配置/命令示例
```properties
# KRaft 组合模式（单机演示）
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@localhost:9093
log.dirs=/tmp/kraft-logs
```

```bash
kafka-storage.sh random-uuid
kafka-storage.sh format -t <uuid> -c server.properties
kafka-metadata-quorum.sh --bootstrap-server localhost:9092 describe --status
```
### 9.5 一句话速记
KRaft 用 Raft 元数据 quorum 取代 ZK，切换秒级、支撑百万分区；Exactly-Once = 幂等 + 事务 + read_committed + 位移入事务。

## 10. 常见问题排查、参数调优与应用场景
### 10.1 核心原理
**排查清单**：消息丢失（acks/min.insync/自动提交/异步发送未处理异常）、重复（至少一次语义 + Rebalance + 生产重试）、乱序（in-flight>1 且未开幂等、多分区、分区器变更）、积压（`consumer lag`，看 `kafka-consumer-groups.sh --describe` 的 LAG 列）、Rebalance 风暴（两个 timeout 与订阅一致性）、分区数（只能增不能减；增加会破坏 key 有序并触发 Rebalance，同时增加句柄/内存/选举成本）。**调优模板**：吞吐优先 vs 延迟优先。**典型场景**：日志采集、埋点上报、CDC（Debezium 抓 binlog）、流处理（Flink/Kafka Streams）、事件驱动架构、削峰填谷。
### 10.2 面试问答
**Q1：Kafka 消息会丢在哪几个环节？**
A：① 生产端：`acks=0/1`、异步发送未处理回调、重试耗尽、`buffer.memory` 超时；② Broker：ISR 只有 1 时 acks=all 失效、unclean 选举、磁盘故障、保留策略误删（`log.retention` 太短）；③ 消费端：自动提交后处理失败、异步处理提前 poll、位移被覆盖、`auto.offset.reset=latest` 导致新组跳过历史消息。

**Q2：怎么监控消费 Lag？**
A：① `kafka-consumer-groups.sh --describe --group xxx` 查看每个分区的 LAG 与消费位移；② JMX 指标 `records-lag-max`、`consumer-fetch-manager-metrics`；③ 用 Burrow、Kafka Lag Exporter 或自建告警（按组设阈值，如 LAG > 10 万且持续 5 分钟）。注意区分"短暂尖峰"与"持续上涨"。

**Q3：分区数怎么定，扩容有什么副作用？**
A：估算：分区数 ≈ max(目标吞吐 / 单分区吞吐, 消费者并行度上限, 目标消费者实例数)，单分区吞吐经验值在数 MB/s 量级（依硬件与副本数，以压测为准）。副作用：① 无法减少；② 增加会触发全组 Rebalance 并可能破坏 key 顺序（`hash % n` 变化）；③ 每分区都有句柄、索引、内存开销，跨节点选举与元数据变大；集群总分区数与单 Broker 分区数都要控制。

**Q4：吞吐优先和延迟优先怎么配置？**
A：吞吐优先：`linger.ms=50~200`、`batch.size=256KB~1MB`、`compression.type=lz4/zstd`、`buffer.memory=128MB`、`max.poll.records` 调大、`fetch.min.bytes=1MB`。延迟优先：`linger.ms=0`、`batch.size` 减小（如 16KB）、`compression.type=none`、`acks=1`、`max.poll.records` 调小、`fetch.max.wait.ms` 调小。注意：延迟优先通常牺牲吞吐，且 acks=1 降低可靠性。

**Q5：Kafka 适合和不适合什么场景？**
A：适合：高吞吐日志/埋点/CDC 管道、流式处理、事件驱动与事件溯源、削峰。不适合：需要大量 Topic（几万个 Topic 会显著增加元数据与选举负担，RocketMQ 更合适）、需要任意延迟消息（Kafka 无原生延迟消息，需自研或借助定时 Topic）、需要单条消息级强事务与复杂路由（RabbitMQ 更合适）、需要消息优先级。

**Q6：Kafka、RocketMQ、RabbitMQ 怎么选？**
A：结论——海量日志/流处理选 Kafka；业务消息且要延迟/事务/顺序/海量 Topic 选 RocketMQ；复杂路由、低延迟小消息量、需要 AMQP 生态选 RabbitMQ。详见 [03-RabbitMQ与MQ选型对比](./03-RabbitMQ与MQ选型对比.md)。
### 10.3 高频追问与陷阱
- 陷阱：`log.retention.ms` 设置过短导致消费者还没消费数据就被删（丢消息的常见原因之一）。
- 陷阱：把"消费 Lag 高"直接等同于"Kafka 慢"，实际多是消费端处理慢或 Rebalance 不稳定。
- 追问：`replica.fetch.wait.max.ms`、`num.replica.fetchers` 影响什么？前者是 Follower 拉取请求的最大等待时间，后者决定副本同步的并发度，跨机房同步慢时常需调整。
### 10.4 配置/命令示例
```properties
# 吞吐优先（Producer）
linger.ms=100
batch.size=524288
compression.type=zstd
buffer.memory=134217728
```

```properties
# 延迟优先（Producer）
linger.ms=0
batch.size=16384
compression.type=none
acks=1
```

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group order-group  # 看 LAG
```
### 10.5 一句话速记
丢失查 acks/min.insync/位点提交，重复与乱序查语义与 in-flight，积压查 Lag 与 Rebalance，分区数只增不减且扩容要评估副作用。

## 自测题（含答案要点）
1. **HW、LEO、ISR、AR 的关系？** HW = ISR 中最小 LEO，是消费者可见边界；AR = ISR + OSR。
2. **Controller 与 Coordinator 的职责？** Controller 管集群元数据与 Leader 选举；Coordinator 管消费组成员与位移。
4. **RecordAccumulator 满了怎么办？** 阻塞至 max.block.ms（默认 60000ms）后抛 TimeoutException，应捕获并补偿。
5. **acks=0/1/all 的语义？** 0 不等确认；1 等 Leader；all 等 ISR 全部确认（需 min.insync.replicas 配合）。
6. **Kafka 3.0 生产者默认值变化？** acks 默认 all、enable.idempotence 默认 true（KIP-679）。
7. **怎么保证分区内有序？** 同 key 同分区 + in-flight=1 或开启幂等（in-flight ≤5 仍保序）。
8. **幂等生产者的边界？** 仅单会话、单分区去重；跨会话与跨分区原子性需事务。
9. **Exactly-Once 的三个条件？** 幂等 + 事务 + read_committed，且位移提交在事务内。
10. **read_committed 为什么会阻塞消费？** 受 LSO 限制，未完成事务会卡住其后消息的消费进度。
11. **四种分区分配策略的差别？** Range 易倾斜、RoundRobin 更均匀、Sticky 保迁移少、CooperativeSticky 增量协作不停服。
13. **自动提交的语义风险？** 先提交后处理会漏消费，先处理后提交会重复消费，无法精确控制。
14. **auto.offset.reset 何时生效？** 仅在无已提交位移或位移越界时；latest 为默认。
15. **Kafka 高性能的六个原因？** 顺序写、页缓存、零拷贝、批量压缩、分区并行、稀疏索引。
16. **零拷贝在什么情况下退化？** 数据需修改、开启 TLS、Broker 侧需解压/转换时。
17. **delete 与 compact 的区别？** delete 按时间/大小整段删除；compact 按 key 保留最新值（可组合）。
18. **min.insync.replicas 的作用？** 决定 acks=all 时的最少确认副本数，低于它写入直接失败。
19. **unclean 选举的取舍？** 允许落后副本当 Leader 换可用性，代价是数据丢失与回退。
20. **KRaft 的收益？** 去 ZK、Controller 切换秒级、支撑百万分区，3.3 生产可用、4.0 移除 ZK。
21. **消息丢失的三个环节？** 生产（acks/回调）、Broker（ISR/保留策略）、消费（位移提交时机）。
22. **分区数如何定、扩容副作用？** 按吞吐与并行度估算；只能增不能减，会触发 Rebalance 并可能破坏 key 顺序。

## 背诵卡片
1. 分区是有序的追加日志，是并行与扩展的最小单位。
2. HW = ISR 中最小的 LEO，消费者只能读到 HW。
4. Controller 管元数据与 Leader 选举，全局唯一；Coordinator 管消费组，按组唯一。
5. 消费者只能消费到 HW，防止 Leader 切换导致已消费数据回退。
6. Leader Epoch 用于副本截断，替代单纯依赖 HW。
7. 发送链路：拦截器 → 序列化 → 分区器 → RecordAccumulator → Sender。
8. buffer.memory 默认 32MB，满了阻塞至 max.block.ms（60000ms）。
9. batch.size 默认 16384，linger.ms 默认 0，两者先到者触发发送。
10. 无 key 消息 2.4+ 默认粘性分区，攒大批次提吞吐。
12. min.insync.replicas 默认 1，生产建议 2（副本数 3）。
13. max.in.flight 默认 5；未开幂等时 >1 且重试会乱序。
14. Kafka 3.0 起 acks 默认 all、幂等默认开启（KIP-679）。
15. delivery.timeout.ms 默认 120000，包含全部重试时间。
16. 幂等生产者用 PID + 分区 + 序列号去重，仅限单会话。
17. 事务用 transactional.id，状态存 __transaction_state。
18. read_committed 只读已提交，受 LSO 限制。
19. 端到端 Exactly-Once 只在"读 Kafka → 写 Kafka"闭环成立。
20. 分配策略：Range（默认，易倾斜）、RoundRobin、Sticky、CooperativeSticky。
22. session.timeout.ms 默认 45000，heartbeat.interval.ms 默认 3000。
23. max.poll.interval.ms 默认 300000，处理太慢会被踢出组。
24. group.instance.id 静态成员可避免重启引发的大规模 Rebalance。
25. 位移存 __consumer_offsets（默认 50 分区，compact）。
26. enable.auto.commit 默认 true，间隔 5000ms；精确控制需手动提交。
27. "先处理后提交" = 至少一次，"先提交后处理" = 至多一次。
28. auto.offset.reset 仅在无位移或越界时生效，默认 latest。
30. 零拷贝靠 sendfile；TLS、需改数据、需解压时退化。
31. Segment 默认 1GB，索引每 4096 字节一条（稀疏索引）。
32. log.cleanup.policy：delete（默认）或 compact（按 key 留最新）。
33. 副本数 3 + min.insync.replicas 2 + acks=all 是主流可靠配置。
34. unclean.leader.election.enable 默认 false，开启等于用数据换可用。
35. KRaft 用 Raft 元数据 quorum 取代 ZK，3.3 生产可用、4.0 移除 ZK。
36. 事务内部主题默认 50 分区；Controller quorum 用 3 或 5 个节点。
37. 分区数只增不减；扩容会触发 Rebalance 并可能破坏 key 顺序。
38. 消费 Lag 看 kafka-consumer-groups.sh --describe 或 records-lag-max。
39. 吞吐优先：linger.ms 大 + batch.size 大 + 压缩 + buffer.memory 大。
40. 延迟优先：linger.ms=0 + batch.size 小 + 不压缩 + fetch.max.wait.ms 小。
