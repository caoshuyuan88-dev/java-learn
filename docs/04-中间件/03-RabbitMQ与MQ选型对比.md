# RabbitMQ与MQ选型对比
> **定位**：掌握 AMQP 0-9-1 的交换机/队列模型与 RabbitMQ 的可靠性、死信与延迟队列实践，并形成"M 个 MQ 怎么选"的落地结论。
> **重要度**：★★★★☆
> **建议复习时长**：4 小时
> **关联文档**：[01-消息队列与RocketMQ](./01-消息队列与RocketMQ.md)、[02-Kafka深度剖析](./02-Kafka深度剖析.md)、[05-SpringCloud微服务体系](../03-框架/05-SpringCloud微服务体系.md)、[02-分布式事务](../05-分布式与架构/02-分布式事务.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| AMQP 0-9-1 模型与核心概念 | ★★★★☆ | 八股原理 |
| 四种交换机与路由规则 | ★★★★★ | 八股原理 |
| 队列与交换机声明参数 | ★★★★☆ | 配置实践 |
| 生产端可靠性（confirm/return/mandatory） | ★★★★★ | 故障复盘 |
| 队列高可用：镜像队列 vs Quorum 队列 | ★★★★★ | 架构设计 |
| 消费端可靠性（手动 ack/prefetch） | ★★★★★ | 配置实践 |
| 死信队列与延迟队列实现 | ★★★★★ | 场景题 |
| 流控、告警与集群高可用 | ★★★★☆ | 故障复盘 |
| Spring AMQP 集成与三大 MQ 选型 | ★★★★★ | 架构设计 |
| MQ 通用高频八股 | ★★★★★ | 场景题 |

## 1. AMQP 0-9-1 模型与核心概念
### 1.1 核心原理
核心对象：**Connection**（一条 TCP 连接）、**Channel**（连接内的虚拟通道，多路复用，几乎所有操作都作用于 Channel）、**VHost**（虚拟主机，逻辑隔离单位，各自拥有独立的 Exchange/Queue/权限）、**Exchange**（交换机，只负责按规则路由，不存储消息）、**Queue**（队列，真正存储消息）、**Binding**（Exchange 与 Queue 的绑定关系，可带 RoutingKey 与 arguments）、**RoutingKey**（生产者携带的路由键，由交换机按类型解释）。整体模型是"生产者 → Exchange →（Binding 规则）→ Queue → 消费者"，交换机本身不落盘，消息在无匹配队列时会被丢弃（除非开 mandatory 或配置备用交换机）。
### 1.2 面试问答
**Q1：为什么要有 Channel，直接用一个 Connection 不行吗？**
A：结论——为了在一条 TCP 连接上支撑多路并发。原因：建立 TCP 连接代价高（三次握手 + TLS），AMQP 规定 Channel 是轻量的逻辑通道，应用按业务/线程划分 Channel 即可。边界：Channel 非线程安全，多线程应各自使用独立 Channel（Spring AMQP 用 ConnectionFactory 的缓存池管理）。

**Q2：VHost 有什么用？**
A：做资源与权限隔离，例如 `/order`、`/pay` 两个 VHost 互不可见，同一套 RabbitMQ 集群可以安全地服务多个团队/环境。注意：客户端连接时必须指定 vhost，默认是 `/`，权限（configure/write/read）是按 vhost 授权的。

**Q3：Exchange 会存消息吗？**
A：不会。Exchange 只做路由决策，消息立即被投递到匹配的 Queue。结论——若没有任何队列绑定或 routing key 不匹配，消息直接丢弃（生产者如果不设置 `mandatory` 也收不到任何反馈），这是"消息莫名消失"的常见原因。

**Q4：默认交换机是什么？**
A：名字为空字符串的 direct 交换机（`""`），它会自动把消息路由到"队列名等于 routing key"的队列。所以 `basicPublish("", "orderQueue", ...)` 就是直接投递到名为 orderQueue 的队列，常用于最简单场景。

**Q5：一个队列能被多少个消费者消费？**
A：同一个队列可以被多个消费者共同消费（竞争消费，默认轮询投递）；若要求同一条消息被多方都收到，需要为每个业务建独立队列并都绑定到同一个交换机（fanout/topic）。
### 1.3 高频追问与陷阱
- 陷阱：以为"绑定了交换机消息就一定进队列"——需要 routing key 与绑定键匹配。
- 陷阱：Channel 共享给多线程使用，会出现协议错乱与 `IllegalStateException`。
- 追问：Exchange 与 Queue 的声明是幂等的吗？幂等，但参数不一致会报 `PRECONDITION_FAILED`（406），因此客户端声明是"校验"而非"修改"。
### 1.4 配置/命令示例
```bash
rabbitmqctl list_vhosts
rabbitmqctl list_exchanges name type durable
rabbitmqctl list_bindings
rabbitmqctl list_queues name messages consumers
```
### 1.5 一句话速记
Connection 一条 TCP、Channel 多路复用、VHost 做隔离、Exchange 只路由不存储、Queue 才存消息、Binding 决定路由规则。

## 2. 四种交换机与路由规则
### 2.1 核心原理
**direct**：routing key 与绑定键**完全相等**才投递（最常用，适合按业务键精确路由）。**fanout**：忽略 routing key，投递到所有绑定队列（广播）。**topic**：routing key 按 `.` 分词，绑定键支持通配符 `*`（匹配一个单词）与 `#`（匹配零到多个单词，如 `order.*.created`、`order.#`）。**headers**：忽略 routing key，按消息 headers 匹配，`x-match=all`（全等）或 `any`（任一），性能最差、生产少用。此外还有**备用交换机（Alternate Exchange, AE）**与**死信交换机（DLX）**作为兜底路由。
### 2.2 面试问答
**Q1：direct 和 topic 怎么选？**
A：结论——路由规则简单且固定时用 direct（性能最好、语义清晰）；需要按维度组合订阅（如按业务线 + 事件类型）时用 topic。原因：direct 是哈希表精确匹配，topic 需要按单词解析与通配匹配，开销更高但表达力强。

**Q2：topic 的通配符规则容易踩什么坑？**
A：`*` 只能匹配**一个**单词（`order.*` 不匹配 `order.created.paid`）；`#` 可匹配 0 个或多个单词（`order.#` 能匹配 `order` 本身）；单词以 `.` 分隔，`order..created` 这类空段不合法。绑定键写错不会报错，只是收不到消息。

**Q3：headers 交换机为什么少用？**
A：匹配需遍历并比较消息头，性能明显低于 direct/topic；且 `x-match` 语义容易被误用（`any` 的实际匹配范围常被低估）。只有在"多维度属性匹配且 routing key 无法表达"时才考虑。

**Q4：消息路由不到队列怎么办？**
A：三层兜底：① 生产端开启 `mandatory=true` 并注册 return 回调，收到"无法路由"的返回；② 为交换机配置备用交换机（AE），无法路由的消息转入 AE 再进队列；③ 消费端为"兜底队列"建监控与告警，防止消息静默堆积在没人看的队列里。

**Q5：一个交换机可以绑定多少队列？**
A：理论无硬上限，但绑定越多，topic 匹配与 fanout 扇出的开销越大；fanout 到几百个队列会显著拉长单次投递耗时。生产上建议单个交换机绑定队列控制在合理规模，量大时按业务拆分交换机。
### 2.3 高频追问与陷阱
- 陷阱：把 `#` 用在绑定键中间却期望"匹配一个单词"（那是 `*` 的语义）。
- 陷阱：只用 fanout 做广播却不考虑消费端故障——广播时任一队列积压都不影响其他队列，容易被忽略。
- 追问：交换机类型能改吗？不能，类型是声明时的属性，修改类型需要删除重建（删除会导致绑定关系丢失）。
### 2.4 配置/命令示例
```java
// topic 交换机：order.*.created 匹配 order.hz.created，不匹配 order.hz.pay.created
channel.exchangeDeclare("biz.topic", "topic", true);
channel.queueDeclare("orderQueue", true, false, false, null);
channel.queueBind("orderQueue", "biz.topic", "order.*.created");
channel.basicPublish("biz.topic", "order.hz.created", null, body);
```
### 2.5 一句话速记
direct 精确、fanout 广播、topic 通配（`*` 一词、`#` 多词）、headers 按属性；路由不到就用 mandatory + 备用交换机兜底。

## 3. 队列与交换机的声明参数
### 3.1 核心原理
声明时的关键属性：**durable**（元数据持久化，重启后队列/交换机仍存在；注意"队列持久化 ≠ 消息持久化"）、**autoDelete**（最后一个消费者断开后自动删除队列；交换机在最后一个绑定解除后删除）、**exclusive**（排他队列，仅声明它的连接可用，连接断开即删除）、**arguments**：`x-message-ttl`（消息存活时间）、`x-expires`（队列闲置过期删除）、`x-max-length` / `x-max-length-bytes`（长度上限）、`x-overflow`（`drop-head` / `reject-publish` / `reject-publish-dlx`）、`x-dead-letter-exchange` 与 `x-dead-letter-routing-key`、`x-max-priority`（优先级队列）、`x-single-active-consumer`、`x-queue-type`（classic / quorum / stream）。
### 3.2 面试问答
**Q1：队列声明为 durable 就保证消息不丢吗？**
A：不能。结论——需要三件齐备：队列 durable + 消息 `delivery_mode=2`（persistent）+ 消息已落到可信存储（Quorum 队列由多数派确认；classic 队列在镜像/仲裁队列下才有副本）。若消息非持久化，Broker 重启即丢。

**Q2：exclusive 和 autoDelete 的区别？**
A：exclusive 的判定维度是"连接"（同一连接的其他 Channel 可用，其它连接不可用，连接关闭即删除）；autoDelete 的判定维度是"消费者"（最后一个消费者取消订阅后删除）。临时队列常用二者组合，或用 `queueDeclare()` 的自动生成名字（服务端随机名）。

**Q3：TTL 过期和长度超限的消息去哪了？**
A：默认直接丢弃；若队列配置了 `x-dead-letter-exchange`，这些消息会被投递到死信交换机（丢弃原因包括：被 reject/nack 且 requeue=false、TTL 过期、超过 `x-max-length`、Quorum 队列超过 delivery-limit）。这是延迟队列与"溢出兜底"的基础机制。

**Q4：x-overflow 三个值的差别？**
A：`drop-head`（默认，丢队头最老的消息）、`reject-publish`（拒绝新消息，生产者若开启 confirm 会收到 nack）、`reject-publish-dlx`（拒绝新消息并把它送到死信交换机）。业务更在意"不丢新消息"时应选 reject-publish 系列。

**Q5：优先级队列有什么限制？**
A：`x-max-priority` 取值 1~255（建议 10 以内），且 classic 队列下优先级需要内存排序，队列很长时性能下降明显；Quorum 队列不支持优先级。优先级只在"同队列且有积压"时才有意义。
### 3.3 高频追问与陷阱
- 陷阱：先声明了非持久化队列，后来改参数重新声明会报 406 PRECONDITION_FAILED（参数是幂等校验，不能修改）。
- 陷阱：给 Quorum 队列设置不支持的特性（优先级、exclusive、非持久化）会声明失败。
- 追问：`x-expires` 与 `x-message-ttl` 的区别？前者删"队列"，后者删"消息"；临时 RPC 回调队列常用 x-expires 防止泄漏。
### 3.4 配置/命令示例
```java
Map<String, Object> args = new HashMap<>();
args.put("x-message-ttl", 60000);                     // 消息 60s 过期
args.put("x-dead-letter-exchange", "dlx.exchange");   // 过期后进死信
args.put("x-max-length", 100000);
args.put("x-overflow", "reject-publish");
channel.queueDeclare("orderQueue", true, false, false, args);
AMQP.BasicProperties props = new AMQP.BasicProperties.Builder().deliveryMode(2).build(); // 消息持久化
```
### 3.5 一句话速记
durable 管元数据、delivery_mode=2 管消息、Quorum 管副本；TTL 过期与超长消息默认丢弃，配了 DLX 才有第二次机会。

## 4. 生产端可靠性：confirm、return 与持久化
### 4.1 核心原理
生产者可靠投递三步：① 开启 **publisher confirm**（`channel.confirmSelect()`），Broker 收到并持久化（如需）后异步回 `basic.ack`，失败回 `basic.nack`；② 设置 **mandatory=true** 并注册 **ReturnListener/ReturnsCallback**，处理"交换机有但队列无"的不可路由消息；③ 消息与队列都持久化，并使用 Quorum 队列（多数派落盘确认）。此外可用**事务**（`txSelect`/`txCommit`）但性能极差，官方不推荐，仅作了解。
### 4.2 面试问答
**Q1：confirm 和 return 的区别？**
A：confirm 处理"消息有没有到 Broker/交换机"，是 Broker 对生产者的投递确认；return 处理"消息到了交换机但路由不到任何队列"，是 Broker 主动退回。结论——两者解决不同环节，必须同时开启才能覆盖"丢在哪"的全链路。

**Q2：confirm 是同步还是异步？**
A：两种都支持：`waitForConfirms()`（同步阻塞，简单但吞吐低）、`ConfirmListener`（异步回调，生产推荐）。批量场景可累积多条后统一等待，或用带序号（`getNextPublishSeqNo`）维护未确认集合，超时重发。

**Q3：为什么不用 RabbitMQ 事务？**
A：事务会显著降低吞吐（官方数据量级上差一到两个数量级），且 confirm 已能提供"至少一次"语义，配合幂等与重发更实用。事务只在极少数"必须原子提交多条发布"的批处理场景才考虑。

**Q4：消息持久化了就一定不丢吗？**
A：不一定。结论——classic 镜像队列在故障切换时可能丢（异步镜像），因此 3.8+ 推荐 **Quorum 队列**：基于 Raft，多数派落盘后才 ack，能保证已确认消息不丢。另外磁盘写满（disk_free_limit 告警）会导致阻塞或拒绝写入。

**Q5：confirm 丢失（超时无应答）怎么处理？**
A：结论——按"可能已成功"处理：不能简单判失败重发（会重复），应重发并依赖消费端幂等，或先查业务状态再决定。这也是 MQ 无法避免重复的根本原因。
### 4.3 高频追问与陷阱
- 陷阱：只开 confirm 不开 mandatory，路由失败的消息被静默丢弃且生产者毫不知情。
- 陷阱：在 confirm 回调里做耗时业务（回调在 IO 线程），会拖垮发布性能。
- 追问：`basic.nack` 代表什么？通常是内部错误（如队列不可用、磁盘告警），应记录并重试或转本地补偿。
### 4.4 配置/命令示例
```java
channel.confirmSelect();
channel.addConfirmListener((seq, multiple) -> log.info("ack {}", seq), (seq, multiple) -> log.error("nack {}", seq));
channel.addReturnListener((replyCode, replyText, exchange, routingKey, props, body) -> log.error("unroutable {}", routingKey));
channel.basicPublish("biz.topic", "order.hz.created", true, props, body); // mandatory=true
```
### 4.5 一句话速记
confirm 管"到没到交换机"、return/mandatory 管"有没有队列收"、持久化与 Quorum 管"落没落盘"，三者叠加才是生产端不丢。

## 5. 队列高可用：镜像队列 vs Quorum 队列
### 5.1 核心原理
RabbitMQ 集群默认只复制**元数据**（交换机、队列定义、绑定、用户、权限），消息体默认只存在队列所属节点。为使消息有副本，历史方案是**镜像队列**（ha-mode=all/exactly/nodes，主从异步同步，`ha-sync-mode=automatic` 支持同步镜像）；RabbitMQ 3.8 引入**Quorum 队列**（基于 Raft，多数派落盘确认，天然支持自动故障转移与 leader 选举），3.9 起官方明确弃用镜像队列，4.0 已移除，**新项目应直接使用 Quorum 队列**。
### 5.2 面试问答
**Q1：Quorum 队列与镜像队列的核心差异？**
A：① 一致性：Quorum 用 Raft 多数派确认（更接近强一致），镜像队列是异步复制（切换可能丢消息）；② 故障恢复：Quorum 自动选主、语义清晰；镜像队列依赖 master 提升与镜像同步，同步期间可能长时间阻塞；③ 功能：镜像队列支持优先级、exclusive 等特性，Quorum 不支持（但支持 TTL、DLX、delivery-limit）；④ 性能：Quorum 写入需多数派落盘，吞吐低于单节点 classic，但可靠。

**Q2：Quorum 队列部署几个节点合适？**
A：3 节点可容忍 1 个节点故障，5 节点容忍 2 个；**必须奇数**，偶数只会增加开销不增加容错（4 节点同样只容忍 1 个故障）。生产推荐 3 节点起步，跨机房部署要评估 Raft 的延迟（多数派落盘对网络 RTT 敏感）。

**Q3：Quorum 队列的 delivery-limit 是什么？**
A：单条消息被投递（含重新入队）的最大次数，超过后进入死信队列，默认未开启。作用类似 RocketMQ 的重试次数上限，避免"毒消息"无限循环。

**Q4：集群节点全挂了，消息还在吗？**
A：Quorum 队列的数据由多数派持有，只要多数派节点恢复即可恢复服务；若多数派永久丢失（如 3 节点挂 2 个且数据盘损坏），则队列不可用。classic 队列（无镜像）在节点故障时其上的队列数据不可用，需等手段恢复节点或把队列声明为 `x-queue-master-locator` 迁移（迁移需停止客户端）。

**Q5：怎么设计队列分布避免单点热点？**
A：① 使用 `queue-master-locator=min-masters`（3.6+）或 `client-local` 策略，让队列尽量均匀分布；② 关键队列用 Quorum（副本自动分布）；③ 避免所有高流量队列都落在同一节点，监控各节点的队列数与入/出流量。
### 5.3 高频追问与陷阱
- 陷阱：以为"集群 = 队列副本"——默认集群不复制消息体，节点挂掉其上的队列不可用。
- 陷阱：在 3.8+ 新项目继续用镜像队列，导致后续升级到 4.0 无法直接迁移。
- 追问：Quorum 队列能否与 classic 队列混用？可以，同一 vhost 内可同时存在，但同一队列不能改类型；类型在声明时固定。
### 5.4 配置/命令示例
```java
// 声明 Quorum 队列（3.8+）
Map<String, Object> args = new HashMap<>();
args.put("x-queue-type", "quorum");
args.put("x-delivery-limit", 5);
channel.queueDeclare("orderQueue", true, false, false, args);
```

```bash
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'   # 旧镜像队列策略（仅存量集群）
rabbitmqctl list_queues name type messages
```
### 5.5 一句话速记
集群只复制元数据、不复制消息；镜像队列已弃用，3.8+ 新项目直接上 Quorum 队列（Raft 多数派、3/5 节点、奇数）。

## 6. 消费端可靠性：手动 ack、prefetch 与并发消费
### 6.1 核心原理
消费确认模式：**自动 ack**（`autoAck=true`，消息一发出即视为已确认，进程崩溃即丢）、**手动 ack**（业务处理成功后 `basicAck`，失败 `basicNack/basicReject` 并选择 `requeue` 与否）。**prefetch/QoS**（`basicQos(prefetchCount)`）限制每个消费者未确认消息数，是实现公平分发与背压的关键，默认值在 Spring AMQP 中为 250。消费顺序性：同一队列多消费者会破坏顺序，需单消费者或按业务键分流到不同队列。
### 6.2 面试问答
**Q1：手动 ack 的正确姿势？**
A：结论——"处理成功再 ack，处理失败再 nack"。要点：① ack 必须与当前 deliveryTag 对应，且同一 Channel 上按序 ack；② 处理异常且可重试时 `basicNack(tag, false, true)` 重新入队，但要防止立刻死循环（建议先延迟或计数后转死信）；③ 处理超时（超过 `consumer_timeout`，默认 30 分钟）会导致连接被关闭、消息重新入队。

**Q2：prefetch 设大设小各有什么影响？**
A：设太大 → 消息被少量消费者一次性拿走并堆积在内存中，其他消费者空闲，且崩溃时大量消息重新入队；设太小 → 网络往返频繁、吞吐低。经验值：轻量任务 100~300，重任务（耗时秒级）用 1~10；配合消费能力压测确定。

**Q3：nack 的 requeue=true 有什么风险？**
A：消息会被重新放回队头，可能被立刻再次投递给同一消费者，形成"毒消息死循环"并占满 prefetch 配额，导致整个队列消费停滞。规范做法：记录重试次数（header 计数），超过阈值 `requeue=false` 投死信队列，或直接发到延迟队列后再重试。

**Q4：怎么保证同一订单的消息按序处理？**
A：① 同一业务键消息进入同一队列（用 `hash(orderId) % 队列数` 做路由键）；② 该队列只被一个消费者（或 `x-single-active-consumer`）消费；③ 消费端串行处理且不并发提交；④ 失败不要立即 requeue（会打乱），而是走顺序补偿。

**Q5：为什么"消息重复消费"在 RabbitMQ 里同样无法避免？**
A：网络超时或消费者崩溃时，消息可能已被处理但 ack 未送达，Broker 会重新投递（`redelivered=true`）。结论——最终防线仍是消费端幂等：唯一索引、去重表或 Redis 去重。

**Q6：消费者怎么优雅停机？**
A：关闭时先 `basicCancel` 停止接收新消息（Spring 中 `container.stop()`），等待在途消息处理完后最后 ack，再关闭 Channel/Connection；配合 `shutdownTimeout` 与 K8s 的 `preStop` + `terminationGracePeriodSeconds`，避免停机丢消息或大量重复。
### 6.3 高频追问与陷阱
- 陷阱：使用自动 ack 却把处理丢给线程池异步执行——最典型的丢消息写法。
- 陷阱：ack 与业务事务不同步：DB 提交失败但已 ack 会导致丢消息。
- 追问：`x-single-active-consumer` 有什么用？同一队列同时只有一个消费者活跃，其余待命，天然保证顺序并避免空转。
### 6.4 配置/命令示例
```java
channel.basicQos(50);                       // prefetch
DeliverCallback cb = (tag, msg) -> {
    try { handle(msg); channel.basicAck(tag, false); }
    catch (Exception e) { channel.basicNack(tag, false, false); }   // 不重新入队 -> 死信
};
channel.basicConsume("orderQueue", false, cb, t -> {});
```
### 6.5 一句话速记
autoAck 等于提前确认、手动 ack 才可靠；prefetch 是背压与公平分发的旋钮；顺序靠"同键同队列 + 单消费者"，重复靠幂等兜底。

## 7. 死信队列与延迟队列实现
### 7.1 核心原理
**死信交换机（DLX）**：队列声明 `x-dead-letter-exchange`（可加 `x-dead-letter-routing-key`），消息在三种情况成为死信——被 `reject/nack` 且 `requeue=false`、**TTL 过期**、**超过队列最大长度**（Quorum 队列还有 delivery-limit 超限）。**延迟队列两种实现**：① **TTL + DLX**（消息或队列设置 TTL，过期后进 DLX 再由消费者处理；缺点：队列级 TTL 严格 FIFO，队头未过期会阻塞后面消息，即"队头阻塞"）；② **延迟消息插件** `rabbitmq_delayed_message_exchange`（声明 `x-delayed-message` 类型交换机 + `x-delay` 头，精度与体验都好，但需要插件支持）。
### 7.2 面试问答
**Q1：为什么"TTL + DLX"的延迟队列会有队头阻塞？**
A：队列按 FIFO 检查过期：只要队头消息未过期，后面的消息即使已过期也不会被投递到 DLX（RabbitMQ 只保证队列头的 TTL 语义）。解法：① 为不同延迟时长建不同队列（如 5s/30s/1m 各一条），② 或使用延迟插件，③ 或改用 RocketMQ 原生延迟消息。

**Q2：延迟插件和 TTL+DLX 怎么选？**
A：结论——同一个延迟时间集中、量不大时用 TTL+DLX 够了；需要"任意延迟时间、消息量大、精度要求高"时用延迟插件。边界：延迟插件的延迟消息保存在 Mnesia/内存表中，重启或节点故障后依赖持久化恢复，超大延迟（天级）会占用资源，需评估。

**Q3：死信队列怎么治理？**
A：① 每个核心交换机/队列都配 DLX + 兜底队列，不要让死信落回原队列；② 死信队列必须有独立消费者与告警（`messages` 增长即报警）；③ 定位原因（格式错误/依赖故障/超时），可修复的按工具重投；④ 保留消息内容与失败原因便于复盘。

**Q4：延迟任务用 MQ 还是用定时任务？**
A：结论——"精度要求分钟级以上、任务量大且需要解耦"用 MQ 延迟；"任务量小、精度要求秒级且与业务库强相关"用分布式定时任务（如 xxl-job）扫描表更方便。也可以"Redis ZSet 延迟队列"作为轻量替代，但要自己处理可靠性与重试。

**Q5：消息在死信队列里会一直留着吗？**
A：取决于队列配置：默认持久保留直到被消费或达到 `x-max-length` + `x-overflow` 策略；生产建议给死信队列设 `x-message-ttl`（如 7 天）以免无限增长，并显式记录处理结果。
### 7.3 高频追问与陷阱
- 陷阱：给消息单独设 TTL 后混用队列级 TTL 语义，误以为"后面的消息会插队过期"。
- 陷阱：DLX 指向的交换机不存在或未绑定队列，导致死信直接丢失（且感知不到）。
- 追问：DLX 可以再配 DLX 吗？可以形成兜底链，但层级过深会让排查困难，建议只保留一到两级。
### 7.4 配置/命令示例
```java
// 延迟队列：TTL + DLX
Map<String, Object> q = new HashMap<>();
q.put("x-message-ttl", 30000);
q.put("x-dead-letter-exchange", "delay.exchange");
q.put("x-dead-letter-routing-key", "delay.fire");
channel.queueDeclare("delay.30s", true, false, false, q);
// 延迟插件：x-delayed-message 交换机
Map<String, Object> ex = new HashMap<>(); ex.put("x-delayed-type", "direct");
channel.exchangeDeclare("delay.plugin", "x-delayed-message", true, false, ex);
AMQP.BasicProperties p = new AMQP.BasicProperties.Builder().headers(Map.of("x-delay", 60000)).build();
```
### 7.5 一句话速记
死信三来源：拒收、TTL 过期、超长；延迟队列首选插件，TTL+DLX 要注意 FIFO 队头阻塞。

## 8. 流控、告警与集群高可用
### 8.1 核心原理
**资源告警（alarm）**：内存水位 `vm_memory_high_watermark`（默认 0.4，即占用物理内存 40% 触发）与磁盘水位 `disk_free_limit`（默认 50MB）任一触发，RabbitMQ 会**阻塞（block）所有正在发布的生产者连接**（消费与其它操作仍可进行），并给客户端发送 `connection.blocked` 通知；同时把消息分页（paging）到磁盘以释放内存。**集群**：多节点共享元数据、队列落在某一节点（classic）或由 Raft 复制（quorum）；跨集群/跨机房用 **Federation**（交换机或队列联合，松耦合、可容忍高延迟）与 **Shovel**（可靠地把消息从一个 Broker 搬到另一个）。网络分区处理策略：`pause_minority`（少数派暂停，推荐）、`autoheal`、`ignore`。
### 8.2 面试问答
**Q1：收到 connection.blocked 说明什么？**
A：结论——Broker 进入资源告警状态（内存或磁盘），主动阻塞生产者，防止把 Broker 拖垮。原因：为保证自身存活，RabbitMQ 选择"阻塞写入、优先消费与落盘"。处理：① 扩容内存/磁盘；② 调整 `vm_memory_high_watermark`（理解风险）；③ 检查是否有消费者停摆导致消息堆积；④ 生产者侧要处理 blocked 通知并降级。

**Q2：内存水位和磁盘水位哪个更常见？**
A：磁盘水位更常见——通常由"消费者挂掉导致队列堆积、磁盘写满"引起；内存水位多由大量未确认消息（prefetch 过大）或消息堆积导致。运维上应对二者都设更早的监控阈值（如 70%）而不是等默认告警线。

**Q3：为什么推荐 pause_minority 处理网络分区？**
A：结论——它保证"少数派节点暂停服务"，避免脑裂下两边都接受写入造成数据分叉（对 Quorum 队列尤其重要，因为 Raft 本身要求多数派）。代价是少数派节点上的队列不可用，可用性下降。`autoheal` 会在分区恢复后合并，但可能丢数据。

**Q4：Federation 和 Shovel 的区别？**
A：Federation 是"逻辑上的交换机/队列联合"，在目标端建同名的交换机和绑定自动同步，适合多机房同时对外服务、双向复制；Shovel 是"点对点的搬运工具"，只负责从 A 取消息投到 B，配置简单，适合单向汇聚（如分支机房 → 中心机房）。两者都不做集群式复制，属于"松耦合集成"。

**Q5：客户端连接断开后，未确认消息怎么办？**
A：会重新入队（redelivered=true）并可能被其他消费者消费，因此消费端必须幂等。若使用了 `x-single-active-consumer` 或 Quorum 队列，语义不变：未 ack 的消息一律回到队列。
### 8.3 高频追问与陷阱
- 陷阱：把 `vm_memory_high_watermark` 调到 0.8 以上"避免阻塞"，实际只是延后了 OOM 与内核 kill 的时间点。
- 陷阱：以为集群能自动处理网络分区——必须显式配置分区处理策略。
- 追问：paging 是什么？把消息从内存换出到磁盘（默认 `vm_memory_high_watermark_paging_ratio` 0.5 开始换页），换页期间内存紧张会显著影响性能。
### 8.4 配置/命令示例
```bash
rabbitmqctl status
rabbitmqctl set_vm_memory_high_watermark 0.6
rabbitmqctl set_disk_free_limit 5GB
rabbitmqctl set_cluster_name prod-mq
rabbitmqctl set_policy ha-quorum "^order\." '{"ha-mode":"all","queue-master-locator":"min-masters"}' --apply-to queues
rabbitmq-diagnostics check_running
```
### 8.5 一句话速记
内存或磁盘触发告警就阻塞生产者，客户端必须处理 connection.blocked；集群不复制消息体，跨机房靠 Federation/Shovel，分区策略选 pause_minority。

## 9. Spring AMQP 集成与三大 MQ 选型
### 9.1 核心原理
Spring AMQP 核心：`RabbitTemplate`（`convertAndSend`、`setConfirmCallback`、`setReturnsCallback`、`setMandatory`）、`@RabbitListener`（声明队列/交换机/绑定与监听方法）、`SimpleRabbitListenerContainerFactory`（并发数、prefetch、AckMode、重试）、`RabbitAdmin`（自动声明队列与交换机）、`RetryTemplate`（本地重试）与 `RepublishMessageRecoverer`（重试耗尽后重投到 DLQ）、`Jackson2JsonMessageConverter`（JSON 序列化）。**选型对比**（结论式）：

| 维度 | Kafka | RocketMQ | RabbitMQ |
| --- | --- | --- | --- |
| 单机吞吐 | 十万~百万级（最高） | 十万级 | 万级~几万级（最低） |
| 延迟 | 毫秒级（可更低） | 毫秒级 | 微秒~毫秒级（最低） |
| 消息顺序 | 分区内有序 | 分区有序 + 全局可做 | 队列内有序（单消费者） |
| 事务消息 | 支持（内部 EOS） | 原生事务消息 + 回查 | 不支持（靠本地消息表/TCC） |
| 延迟消息 | 无原生（需自研） | 4.x 18 级 / 5.x 秒级 | 插件或 TTL+DLX |
| 消息回溯 | 按 offset/时间重放（最强） | 按时间/offset 重放 | 不支持（消费即删） |
| 主题数规模 | 数百（数万会拖慢） | 数万级（支持好） | 数万级 |
| 运维复杂度 | 高（ZooKeeper/KRaft 等） | 中 | 中低 |
| 社区与生态 | 最活跃、流生态完善 | 活跃、国内生态好 | 活跃、AMQP 标准 |
| 典型场景 | 日志、埋点、CDC、流处理 | 业务消息、订单、削峰、事务 | 复杂路由、通知、任务分发、低延迟 |
### 9.2 面试问答
**Q1：Spring AMQP 里怎么做到"重试 + 死信"？**
A：三层配置：① 容器层面 `setRetryTemplate` 做本地重试；② `RepublishMessageRecoverer` 重试耗尽后把消息重投到死信交换机；③ 队列声明没绑 DLX 时，用 `AcknowledgeMode.MANUAL` + 手动 nack(requeue=false) 触发死信。三者组合起来才能既有重试又不会无限循环。

**Q2：AckMode 有几个值？**
A：`NONE`（无 ack，自动确认）、`MANUAL`（业务显式 ack，最灵活）、`AUTO`（Spring 在监听方法正常返回后 ack，抛异常则 nack/requeue）。生产推荐 `AUTO`（简单）或 `MANUAL`（需要精细控制与转死信时）。

**Q3：三大 MQ 怎么给建议？**
A：结论——① 海量日志/埋点/流处理 → Kafka；② 业务消息、需要事务消息/延迟消息/海量 Topic/可回溯 → RocketMQ；③ 复杂路由（topic 通配、多交换机）、低延迟小消息量、已有 AMQP 生态 → RabbitMQ。附加考虑：团队熟悉度、现有运维体系、国内云厂商托管能力。

**Q4：一个系统能同时用两种 MQ 吗？**
A：可以但代价高（两套运维、两套监控、语义不一致）。适用场景：Kafka 承担数据管道、RocketMQ/RabbitMQ 承担业务消息，且明确边界与责任人。不建议为"某个小功能"引入第二套 MQ。

**Q5：怎么把 MQ 集成进 Spring Boot 测试？**
A：用 Testcontainers 起真实 Broker 做集成测试，或用 `@SpringBootTest` + 嵌入式替代（RabbitMQ 无官方嵌入式，RocketMQ 可用 testcontainers）；单元测试中把 `RabbitTemplate`/`RocketMQTemplate` mock 掉，重点测消费逻辑的幂等与异常分支。
### 9.3 高频追问与陷阱
- 陷阱：`@RabbitListener` 方法抛异常但 AckMode=AUTO 会不断 requeue，导致"死循环 + 日志刷屏"（默认 requeue=true），必须配置重试上限 + DLQ。
- 陷阱：生产者的 confirm 回调与业务事务混用，导致"消息已确认但数据库回滚"。
- 追问：为什么投递到 DLQ 前要加 `x-death` 头信息？记录原始交换机、路由键、失败原因与次数，方便排查与重投。
### 9.4 配置/命令示例
```java
@Bean
public SimpleRabbitListenerContainerFactory factory(ConnectionFactory cf) {
    var f = new SimpleRabbitListenerContainerFactory();
    f.setConnectionFactory(cf);
    f.setPrefetchCount(50);
    f.setAcknowledgeMode(AcknowledgeMode.MANUAL);
    f.setConcurrentConsumers(4);
    f.setMaxConcurrentConsumers(16);
    return f;
}

@RabbitListener(queues = "orderQueue")
public void onMessage(OrderMsg msg, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
    try { orderService.handle(msg); channel.basicAck(tag, false); }
    catch (Exception e) { channel.basicNack(tag, false, false); }  // 进死信
}
```
### 9.5 一句话速记
RabbitTemplate + @RabbitListener + 手动 ack + RetryTemplate/RepublishMessageRecoverer；选型一句话——日志流选 Kafka、业务事务消息选 RocketMQ、复杂路由低延迟选 RabbitMQ。

## 10. MQ 通用高频八股
### 10.1 核心原理
面试官问 MQ，最终都会收口到八个问题：不丢、不重、顺序、积压、削峰、最终一致性、选型、MQ 挂了怎么办。回答的通用框架是"按环节拆解 + 给出方案 + 说明代价与兜底"。
### 10.2 面试问答
**Q1：如何保证消息不丢失？**
A：按三环节答：① 生产端——同步发送或异步回调校验结果、开启重试、本地消息表兜底（RocketMQ 事务消息 / RabbitMQ confirm+mandatory）；② 存储端——同步刷盘 + 同步复制（RocketMQ）或 acks=all + min.insync.replicas≥2 + 禁止 unclean 选举（Kafka）或 Quorum 队列（RabbitMQ），并监控磁盘与副本状态；③ 消费端——处理完成后才 ack，异常进重试队列与死信队列并告警。最后补一句对账兜底。

**Q2：如何保证消息不重复消费？**
A：MQ 只能做到"至少一次"，因此幂等必须自己做：① 唯一索引（最可靠）；② 去重表/消息表（记录 messageId + 状态）；③ Redis `SET NX EX` 快速拦截（注意过期时间要覆盖重试窗口）；④ 状态机幂等（只允许合法状态迁移）；⑤ 乐观锁版本号。补充：幂等键必须用业务键，而非每次变化的 messageId。

**Q3：如何保证顺序？**
A：三步：① 生产端按业务键把同一类消息发到同一分区/队列；② 存储端保证该分区/队列单线程消费或加锁串行；③ 消费端处理失败不要立即 requeue（会乱序），改用顺序补偿或阻塞式重试。全局有序代价极高（单分区单线程），一般只做分区有序。

**Q4：消息积压怎么处理？**
A：短期：提升并行度（Kafka 看分区数、RocketMQ 看队列数，不足则临时扩容）、批量消费、跳过非关键消息、临时转发到多队列新 Topic 后并肩消费。长期：容量规划、消费者实例与分区数匹配、下游降级、积压与 Lag 告警。排查：先确认是"消费者掉线/毒消息卡死"还是"下游变慢"。

**Q5：怎么用 MQ 削峰？**
A：上游只做"校验 + 入队"（如秒杀下单成功即返回排队中），消费者按自身能力拉取并处理，前端轮询结果。要点：队列必须有容量上限与丢弃/降级策略（如超阈值直接拒绝），否则积压会拖垮存储；同时要控制消费者的处理时长以避免队头阻塞。

**Q6：怎么用 MQ 做最终一致性？**
A：两条主流路线：① **RocketMQ 事务消息**——半消息 + 本地事务 + 回查（见 [01-消息队列与RocketMQ](./01-消息队列与RocketMQ.md)）；② **本地消息表**——本地事务写业务表与消息表，再由定时任务/CDC 重发，通用且不绑定具体 MQ。两者都需要消费端幂等 + 定时对账 + 人工兜底。参见 [02-分布式事务](../05-分布式与架构/02-分布式事务.md)。

**Q7：怎么选 MQ？**
A：按四个维度：吞吐与延迟需求、语义需求（事务/延迟/顺序/回溯）、主题与消息规模、团队与运维能力。一句话结论见 9.5。

**Q8：MQ 挂了怎么办？**
A：① 生产端：本地消息表暂存、降级为同步调用、失败快速失败并提示用户重试；② 消费端：暂停非核心链路，核心链路依赖 MQ 集群自身高可用（多节点、Quorum/同步复制）；③ 管理面：故障演练、容量冗余、备份与快速重建预案。面试加分：说明"MQ 不应成为单点，但也不能假设它永不故障"。
### 10.3 高频追问与陷阱
- 陷阱：所有问题都答"加幂等 + 加重试"，却没有具体方案与代价对比。
- 陷阱：声称"用 MQ 事务保证强一致"——它只提供最终一致，中间态可见。
- 追问：如果要求"消息必须恰好一次"？回答：端到端做不到，退化为"至少一次 + 消费端幂等"，这本身也是主流工程结论。
### 10.4 配置/命令示例
```bash
# 三类 MQ 的积压/状态速查
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group g1   # LAG
sh mqadmin consumerProgress -n 127.0.0.1:9876 -g order-group                        # 堆积量
rabbitmqctl list_queues name messages messages_unacknowledged consumers             # 消息与消费者
```
### 10.5 一句话速记
不丢靠三环节加固、不重靠消费端幂等、顺序靠同键同队列、积压靠并行与降级，选型按吞吐/语义/规模/团队四条线。

## 自测题（含答案要点）
1. **Channel 相比 Connection 的意义？** 一条 TCP 上多路复用，轻量且按业务隔离；Channel 非线程安全。
2. **Exchange 会存储消息吗？** 不会，只做路由；无匹配队列时消息被丢弃（除非 mandatory 或备用交换机）。
3. **四种交换机的路由规则？** direct 精确匹配、fanout 广播、topic 通配（`*` 一词、`#` 多词）、headers 按属性匹配。
4. **队列 durable 就保证不丢吗？** 不够，还需消息 delivery_mode=2 与副本（Quorum/镜像）。
6. **TTL 过期消息去哪？** 默认丢弃；配置 x-dead-letter-exchange 后进入死信交换机。
8. **confirm 与 return 分别解决什么？** confirm 确认消息到达 Broker/交换机；return 处理路由不到队列的消息。
9. **为什么不用 RabbitMQ 事务？** 性能下降一到两个数量级，confirm + 幂等更实用。
10. **镜像队列与 Quorum 队列的差别？** 镜像异步复制、功能多但已弃用；Quorum 基于 Raft 多数派、自动选主、4.0 已移除镜像队列。
12. **prefetch 的作用与取值？** 限制未确认消息数以做背压与公平分发；轻任务 100~300，重任务 1~10。
13. **nack requeue=true 的风险？** 毒消息死循环并占满 prefetch，导致队列消费停滞。
14. **怎么保证 RabbitMQ 顺序？** 同键同队列 + 单消费者（或 x-single-active-consumer）+ 串行处理，失败走补偿。
15. **延迟队列两种实现及缺陷？** TTL+DLX 有队头阻塞；延迟插件灵活但需安装且大延迟占资源。
16. **connection.blocked 代表什么？** 内存或磁盘达告警线，Broker 阻塞生产者；客户端需处理并降级。
18. **网络分区策略怎么选？** pause_minority 保一致性（推荐）、autoheal 保可用但可能丢数据。
20. **MQ 通用八股的答题框架？** 按生产/存储/消费三环节拆解，给出方案、代价与兜底（对账与告警）。

## 背诵卡片
1. AMQP 模型：Connection → Channel → VHost → Exchange → Binding → Queue。
2. Channel 非线程安全，多线程各自使用独立 Channel。
3. Exchange 不存消息，只路由；无匹配即丢弃。
4. 默认交换机是空名 direct，按"队列名 = routing key"投递。
5. direct 精确匹配，fanout 广播，topic 支持 `*`/`#`，headers 按属性。
6. `#` 匹配零到多个单词，`*` 只匹配一个单词。
7. 备用交换机（AE）是路由失败的兜底。
8. durable 管元数据，delivery_mode=2 管消息持久化，两者缺一不可。
10. x-message-ttl 删消息，x-expires 删队列。
12. 死信三来源：reject/nack(requeue=false)、TTL 过期、超过最大长度。
13. publisher confirm 处理"到没到"，mandatory + return 处理"有没有队列收"。
14. confirm 可同步 waitForConfirms 或异步 ConfirmListener，生产用异步。
15. RabbitMQ 事务性能差，生产不推荐。
16. 集群默认只复制元数据，不复制消息体。
17. 镜像队列 3.9 起弃用、4.0 移除，新项目用 Quorum 队列。
18. Quorum 队列基于 Raft，多数派落盘后 ack，支持自动选主。
19. Quorum 必须奇数节点：3 容忍 1、5 容忍 2。
20. Quorum 不支持优先级与 exclusive，支持 TTL/DLX/delivery-limit。
21. 手动 ack 才可靠：处理成功 basicAck，失败 basicNack(false)。
22. 自动 ack + 异步处理 = 最典型的丢消息写法。
24. 消费超时 consumer_timeout 默认 30 分钟，超时会关闭连接并重投。
26. redelivered=true 表示消息是重新投递，消费端必须幂等。
27. TTL+DLX 做延迟队列的最大坑是队头阻塞（FIFO）。
28. 延迟插件用 x-delayed-message 交换机 + x-delay 头。
29. 内存水位默认 0.4、磁盘水位默认 50MB，触发即阻塞生产者。
30. connection.blocked 是客户端必须处理的流控通知。
31. 网络分区策略：pause_minority（推荐）、autoheal、ignore。
33. Spring AMQP 三件套：RabbitTemplate、@RabbitListener、SimpleRabbitListenerContainerFactory。
34. 重试 + 死信用 RetryTemplate + RepublishMessageRecoverer + DLX 组合。
36. 不丢三环节：生产 confirm/重试、存储持久化与副本、消费处理完成后 ack。
37. 不重只能靠幂等：唯一索引、去重表、Redis NX、状态机。
38. 顺序三步：同键同分区/队列、单线程消费、失败不立即 requeue。
39. 积压三板斧：提并行度、批量消费、临时转发；配合跳过非关键消息。
40. 最终一致两条路：RocketMQ 事务消息，或本地消息表 + 定时补偿。
