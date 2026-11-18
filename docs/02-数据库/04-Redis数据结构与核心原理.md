# Redis数据结构与核心原理

> **定位**：从单线程模型与九种数据类型的底层编码，到持久化、复制、Cluster、内存淘汰与缓存一致性，覆盖 Redis 面试的全部主干。
> **重要度**：★★★★★（面试出现频率）
> **建议复习时长**：6 小时
> **关联文档**：[./05-Redis高可用与缓存治理.md](./05-Redis高可用与缓存治理.md)、[./01-MySQL体系结构与索引.md](./01-MySQL体系结构与索引.md)、[../06-计算机基础/02-操作系统.md](../06-计算机基础/02-操作系统.md)

## 0. 知识地图

### 零基础阅读方式

先把 Redis 想成一个“用 key 找 value 的内存服务”：

```text
key（名字） → value（数据）
```

例如 `customer:1001:name → "Ada"`。Redis 除了普通字符串，还提供 Hash、Set、ZSet 等更适合不同操作的数据类型。**key 是查找入口，数据类型决定能做哪些操作，底层编码决定 Redis 实际怎样摆放数据。**

读后面的内容时，每个名词都问三个问题：

1. 它解决什么问题？
2. 数据在内存里大致长什么样？
3. 它的优点和代价分别是什么？

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| 1 定位、单线程模型与为什么快 | ★★★★★ | 八股原理 / 源码追问 |
| 2 九种数据类型与底层编码 | ★★★★★ | 八股原理 / 源码追问 |
| 3 渐进式 rehash、过期删除与内存淘汰 | ★★★★★ | 八股原理 / 调优排查 |
| 4 持久化：RDB、AOF 与混合持久化 | ★★★★★ | 八股原理 / 故障复盘 |
| 5 主从复制与哨兵 | ★★★★★ | 八股原理 / 故障复盘 |
| 6 Cluster 分片与重定向 | ★★★★★ | 八股原理 / 场景题 |
| 7 事务、Lua 脚本与 Pipeline | ★★★★☆ | 八股原理 / 场景题 |
| 8 发布订阅与 Stream | ★★★☆☆ | 八股原理 / 场景题 |
| 9 慢查询、bigkey/hotkey 与内存优化 | ★★★★★ | 调优排查 / 故障复盘 |
| 10 缓存与数据库一致性 | ★★★★★ | 场景题 / 故障复盘 |

## 1. 定位、单线程模型与为什么快

### 1.1 核心原理

#### 先用人话理解：Redis 如何接待很多客户端

客户端使用 TCP 连接 Redis，并把命令（例如 `GET customer:1001:name`）发给它。Redis 不是“一个连接配一个线程”，而是让一个事件循环观察很多连接：哪个连接有数据就读哪个，哪个命令执行完就把结果写回哪个连接。

```text
客户端 A ─┐
客户端 B ─┼─ TCP 连接 ─→ IO 多路复用等待就绪
客户端 C ─┘                    ↓
                    读取命令、解析协议
                              ↓
                     主线程执行命令
                              ↓
                    写回对应客户端结果
```

**IO 多路复用**可以先理解为“一个服务员同时看着很多桌，哪桌有需求就去处理哪桌”，而不是让每桌都配一个服务员。在 Linux 上常见机制是 epoll。它负责发现连接是否可读/可写，不负责执行 Redis 命令。

Redis 所说的“单线程”，主要是指普通命令在主线程中按顺序执行。它不是说 Redis 只能接受一个连接，也不是所有后台工作都只有一个线程。由于命令执行时不会有另一个命令同时改同一份数据，很多命令不需要额外加锁；加上数据主要在内存里，简单读写通常很快。

但单线程也有代价：一条命令没结束，后续命令就要等待。CRM 客户详情缓存的 `GET` 通常很短；如果执行 `KEYS customer:*` 扫描几十万个 key，主线程需要遍历大量 key，期间其他租户的请求也会排队。IO 多路复用只能高效管理连接，不能让一条慢命令变快。

Redis 是以内存读写为主的 key-value 数据库。它也能把数据写到磁盘、复制给其他节点、拆分到集群，并提供脚本和多种数据结构，所以常用于缓存、计数、排行榜、限流、分布式锁和轻量消息流。它不是“只能当缓存”，但若把 Redis 作为唯一数据来源，就必须认真设计持久化、备份和故障恢复。

**Redis 快的原因要分开看：**

1. 数据主要在内存中，不必每次都等待磁盘随机读写。
2. IO 多路复用让一个事件循环照看大量连接，不需要为每个连接创建线程。
3. 命令执行串行，减少多个线程争抢同一数据时的锁和上下文切换。
4. 数据结构按常见操作设计，例如用哈希表快速按 key 查找，用跳表维护有序集合。

这不意味着任何 Redis 操作都快。大集合全量读取、复杂脚本、大 key 删除会做很多工作，依然可能占住主线程。

**关于“Redis 是不是单线程”**：RDB 保存和 AOF 重写可以 fork 子进程；异步释放内存可以由后台线程完成；AOF 刷盘也有后台工作。Redis 6.0 还增加了可选的多线程网络 IO，主要分担 socket 读写和协议处理，普通命令仍由主线程执行。因而“Redis 完全单线程”不准确，“Redis 命令执行主要串行”更准确。

### 1.2 面试问答

- **Q1：Redis 单线程为什么还这么快？** 内存操作 + epoll 多路复用 + 单线程免锁免切换 + 高效数据结构；瓶颈通常在网络或内存，只有真跑到 CPU 瓶颈时多线程 IO 才有意义。
- **Q2：单线程有什么缺点？** 单个慢命令（`KEYS *`、大集合 `HGETALL`、大 key `DEL`）会阻塞全部请求，且无法利用多核。对策：禁用 `KEYS` 改用 `SCAN`、`UNLINK` 异步删除、大 key 拆分、6.0 开多线程 IO。
- **Q3：6.0 的多线程 IO 提升了什么？** 把 socket 读写、协议解析与回写分摊到多线程，主线程专注执行命令；优化的是**高并发小命令**的网络吞吐，对 CPU 密集的单条大命令无效；`io-threads` 官方建议不超过 8（以官方文档为准）。
- **Q4：Redis 与 Memcached 的区别？** 数据类型丰富（Memcached 只有 KV）、支持持久化与复制、支持 Lua/事务/发布订阅/Stream、单线程（Memcached 多线程）；Memcached 在纯 KV 多核场景仍有内存利用率优势，但生态上 Redis 已全面胜出。

### 1.3 高频追问与陷阱

- 陷阱：说"Redis 完全单线程"。准确说法是"命令执行单线程"，另有后台线程/子进程与 6.0 的多线程 IO；也不要以为调大 `io-threads` 就能万能提速，线程数超过核数反而退化。

### 1.4 代码/配置/命令示例

```bash
redis-cli INFO server | grep -E "redis_version|io_threads"
redis-cli CONFIG GET io-threads io-threads-do-reads
redis-cli --latency          # 判断是否被慢命令拖累
```

### 1.5 一句话速记

**命令执行单线程 + 内存 + epoll + 免锁 = 快；慢命令阻塞是最大风险，6.0 多线程只优化网络 IO。**

## 2. 九种数据类型与底层编码

### 2.1 核心原理

#### 先用人话理解：Redis 类型是“操作能力”，编码是“内部怎么存”

例如业务把客户资料当作 Hash 使用，但 Redis 不一定真的用哈希表保存它：少量字段时可用紧凑的 listpack，字段增多或值变长后再转成 hashtable。也就是说：

```text
业务看到的数据类型（Hash）
          ↓
Redis 根据大小和内容选择内部编码
          ↓
执行 HGET / HSET 等类型对应的操作
```

选类型时先看需要的操作：需要按字段取客户属性用 Hash；需要去重用 Set；要按分数排序用 ZSet；需要高效计数用 String 的 `INCR`；需要低内存签到状态可用 Bitmap。不要只看“底层是什么”而忽略业务访问方式。

| 类型 | 常见底层编码/结构 | 典型用途 |
| --- | --- | --- |
| String | `int` / `embstr` / `raw`（SDS） | 缓存、计数器、分布式锁、位图底座 |
| List | `quicklist`（由多个 listpack 节点组成） | 双端队列、简单时间线 |
| Hash | 小对象 `listpack`，大对象 `hashtable` | 对象字段、购物车 |
| Set | `intset`、小整数集合可用 `listpack`（7.2+）、`hashtable` | 去重、交并差、共同成员 |
| ZSet | 小对象 `listpack`，大对象 `skiplist` + `dict` | 排行榜、延迟任务 |
| Bitmap | String 的位操作 | 签到、开关标记、低成本状态记录 |
| HyperLogLog | String 中的稀疏/稠密寄存器表示 | 大规模 UV 基数估算 |
| GEO | ZSet（member + GeoHash 编码的 score） | 附近的人、地理范围搜索 |
| Stream | rax 基数树 + listpack 节点 | 可回溯、带消费组的消息流 |

先认识一层共同的外壳：Redis 的键空间字典保存 `key → redisObject`。`redisObject` 记录值的类型、内部编码、引用计数以及 LRU/LFU 等淘汰信息，实际数据由 `ptr` 指向对应结构。**Redis 数据类型是对外提供的操作语义，encoding 是内部实现**；同一个 Hash key 可能从 listpack 转成 hashtable，但客户端仍然使用 `HGET`、`HSET`。

```text
键空间 dict
└── key（SDS） → redisObject（type + encoding + ptr）
                            └── 具体值：SDS / listpack / dict / quicklist / skiplist ...
```

这里的 **dict** 是 Redis 自己用来存 key 的哈希表；**redisObject** 是 value 的“包装信息”，里面记着类型和编码；真正的数据放在 `ptr` 指向的结构里。后面说“Hash 编码为 listpack”，意思是 Hash 的 value 被包装成 Hash 类型，但内容当前以 listpack 这种字节布局保存。

**先弄清 listpack 到底长什么样。**它不是链表，也不是一个个独立对象，而是一段连续的字节数组。开头有总字节数和元素数量，随后紧挨着保存一个个 entry，末尾有结束标记：

```text
低地址
┌─────────────────────┬──────────────────────────┬────────┐
│ 总字节数（4 bytes） │ 元素数量（2 bytes）      │        │
├─────────────────────┴──────────────────────────┤        │
│ entry 1 │ entry 2 │ entry 3 │ ... │ EOF 标记   │
└────────────────────────────────────────────────┴────────┘
高地址

每个 entry 大致是：
┌────────────────┬───────────────┬──────────────────┐
│ 编码/类型信息  │ 实际数据      │ 当前 entry 的长度 │
└────────────────┴───────────────┴──────────────────┘
```

例如小 Hash `{name: Ada, status: active}` 可以依次存成：

```text
header | name | Ada | status | active | EOF
          field value  field   value
```

读取时从前往后解析 entry；需要从后往前遍历时，entry 尾部的长度信息帮助 Redis 找到前一项。整数可以用紧凑编码保存，短字符串也不需要额外的指针和哈希桶。**好处**是连续、紧凑、分配次数少，特别适合小对象；**代价**是按值查找通常要从头扫描，插入/删除可能要移动后续字节并重新分配，因此不适合无限增长的大对象。Redis 超过阈值会转成更适合查找的大结构。

这也解释了为什么 Redis List 通常不是一个巨大的 listpack：List 用 quicklist 把数据分成多个小 listpack 节点，再用双向链表连接。节点内部连续紧凑，节点之间又能从两端快速增删，兼顾空间和操作效率。

下面逐个看每种类型的内部结构，以及这些结构为什么适合对应的操作。

**String：整数、embstr 或 raw。** String 是 Redis 最通用的类型，不只存文本，也可以存二进制数据。整数形式的值可用整数编码，执行 `INCR` 时直接做整数运算；普通字符串用 SDS 保存。短字符串通常使用 `embstr`，`redisObject` 与 SDS 放在一次连续分配的内存中，减少分配次数、提高局部性；较长字符串使用 `raw`，对象头和 SDS 分开分配，便于扩容。Redis 7 常见的 embstr 边界是 44 字节，具体以版本和构建为准；字符串需要修改而无法原地完成时可能转换为 raw。

```text
SET customer:1001:name "Ada"
  → key 作为 SDS 保存在键空间
  → value 对象标记为 String
  → 短 value 常用 embstr，长 value 常用 raw/SDS

INCR tenant:1001:login-count
  → 若值可按整数编码，直接进行整数加一
```

String 的字典定位通常是 O(1)，但读写或传输一个长度为 N 的 value 仍要处理 N 个字节，不能把“大 value”理解成真正恒定时间。拼接、扩容或大字符串传输还要考虑数据长度。不要把一个几 MB 的 JSON 全塞进 String 并频繁整体读写：虽然命令简单，网络传输、复制和序列化成本仍然很高。

**List：quicklist 连接多个紧凑节点。** Redis List 面向有顺序的元素序列，当前常见实现是 quicklist：它由双向链接的 quicklist 节点组成，每个节点内部用 listpack 紧凑保存一批元素。这样既能从两端快速插入/弹出，又避免“每个元素一个链表节点”带来的大量指针和分配开销。

```text
quicklist
  ↔ [listpack: a, b, c]
  ↔ [listpack: d, e]
  ↔ [listpack: f, g, h]
```

`LPUSH`/`RPUSH` 和两端弹出通常适合队列操作；按下标随机访问需要从节点逐步定位，不如 String/Hash 按 key 直接查。节点大小配置体现“节点越大越省结构开销，但局部修改/压缩成本可能越高”的取舍。List 适合小型、简单的待办队列；若需要消费组、ACK、消息回溯，应评估 Stream 或专业 MQ。

**Hash：紧凑键值对数组，或真正的哈希表。** 小 Hash 的 listpack 按 field/value 成对连续存放：

```text
listpack: [field1][value1][field2][value2][field3][value3]
```

这种格式没有每个字段单独的哈希桶和指针，少量字段时很省内存；代价是查找通常要顺序扫描。超过字段数或单个 field/value 长度阈值后，转成 hashtable：通过哈希函数定位桶，平均 O(1) 查字段，但要付出桶数组、哈希节点和指针的内存开销。CRM 的小型用户展示属性可以用 Hash；字段特别多、某个字段很大或经常整体读取时，应重新评估是否拆分 key。

```text
HSET customer:1001 name Ada status active
HGET customer:1001 status
```

**Set：保证唯一性的集合。** 如果成员都是整数且数量较少，Redis 可用 intset 保存有序整数数组，查找可二分，插入时可能需要移动元素；Redis 7.2+ 的小 Set 还可在条件满足时使用 listpack。成员较多、含非整数时通常用 hashtable，以成员作为 key 实现唯一性，`SADD`、`SISMEMBER` 平均 O(1)。因此 Set 的“去重”来自哈希表 key 唯一，整数紧凑编码则以省内存换取插入搬移成本。

```text
intset（例如 3、8、20）：
[编码宽度][元素数量][3][8][20]
元素按整数升序连续排列；查找可二分，插入时可能要挪动后面的元素。
```

```text
SADD customer:1001:tags vip trial
SISMEMBER customer:1001:tags vip
SINTER team:A:members team:B:members
```

Set 适合去重、成员判断和集合运算；若需要成员按分数排序，用 ZSet；若只需要估算总数、不需要知道具体成员，可考虑 HyperLogLog。

**ZSet：小集合用 listpack，大集合用字典 + 跳表。** 每个成员都有唯一 member 和 score。小 ZSet 在 listpack 中按 score/member 排序保存，结构紧凑但查找、插入可能要扫描或移动元素；超过阈值后使用两套结构：

```text
dict：member → score                 平均 O(1) 查分
skiplist：按 (score, member) 有序      平均 O(logN) 插入、查找和排名
```

#### 跳表长什么样：一条完整链表，上面加几层“快速通道”

跳表（skiplist）不是树，而是**按顺序连接起来的多层链表**。最底层包含所有节点，保证可以从小到大逐个遍历；上面的层只包含部分节点，像高速公路的入口，帮助查找时一次跨过许多节点。每个节点代表一个 ZSet 成员，排序依据是 `(score, member)`：先比较分数，分数相同再按 member 字典序比较。

例如有 6 个成员，分数依次为 10、20、40、55、70、90。下面只画一种可能的随机层级；真实跳表中每个节点有几层并不固定：

```text
第 2 层（稀疏的快速通道）：HEAD ─────────→ [40] ─────────────→ [90]
第 1 层（较密的快速通道）：HEAD ─→ [10] ─→ [40] ─→ [70] ───→ [90]
第 0 层（完整有序链表）：  HEAD → [10] → [20] → [40] → [55] → [70] → [90] → NULL
```

每个 `[分数]` 节点里保存 member、score，以及一组按层编号的链接信息：

```text
节点 [40]（示意）
┌─────────────────────────────────────┐
│ member="user:40", score=40           │
│ level[0]: forward → [55], span=1     │  第 0 层链接
│ level[1]: forward → [70], span=2     │  第 1 层链接
│ level[2]: forward → [90], span=2     │  第 2 层链接
│ backward → [20]                      │  指向底层的前一个节点
└─────────────────────────────────────┘
```

`forward` 是该层的下一个节点；`backward` 让底层链表能向后走，支持从高分向低分遍历。节点层数由随机算法决定，Redis 的跳表最多 32 层；高层节点更少、能跳得更远，低层节点更多、定位更精细。**span（跨度）**记录沿某层链接前进时跨过的底层节点数量。把经过的 span 累加起来，就能知道当前节点在有序集合中的位置，因此 Redis 能高效计算排名，而不必从头数到目标成员。

#### 查找过程：从高层试探，遇到“超过目标”就下沉

假设要查分数为 55 的成员。跳表从最高层的 `HEAD` 开始，重复做两件事：看当前层的下一个节点；如果它仍然排在目标之前或正好是目标，就沿链接前进；如果它已经排在目标之后，就沿同一个节点下沉一层，再继续比较。

```text
目标：找到 score=55 的节点

第 2 层：HEAD → [40]，下一节点 [90] 已超过 55；停在 [40]，下沉
第 1 层：从 [40] 看下一节点 [70]，已超过 55；仍停在 [40]，下沉
第 0 层：从 [40] 走到 [55]，找到目标

查找路径：HEAD ──→ [40] ──↓──↓──→ [55]
```

所以查找不是从头把 `[10]、[20]、[40]、[55]` 一个个检查，而是先利用稀疏的高层快速靠近，再到底层完成精确定位。层级足够随机且分布合理时，查找、插入和删除的平均复杂度为 **O(log N)**；最坏情况下仍可能退化到 **O(N)**，所以文档和面试中通常说“平均 O(log N)”，不是绝对保证每次都如此。

#### 为什么它适合 ZSet 的排名和范围查询

- **查 member 的分数**：旁边的 `dict` 保存 `member → score`，平均 O(1)；跳表主要负责有序访问。
- **查排名**：按 `(score, member)` 在跳表中搜索，并沿途累加 span，得到成员在有序序列中的位置。`ZRANK` 的排名从 0 开始；`ZREVRANK` 返回反向排名。
- **查分数范围**：先用跳表定位范围起点，再沿第 0 层的底层链表连续向后读；不必重新对所有成员排序。
- **插入或删除**：搜索时记录每层经过的节点和累计跨度，定位后更新相关层的链接与 span。只需修改路径附近的链接，不像数组插入那样整体搬移后续元素。

例如：

```text
ZADD leaderboard 10 "user:10" 20 "user:20" 40 "user:40" 55 "user:55"
ZADD leaderboard 70 "user:70" 90 "user:90"
ZRANGE leaderboard 0 5 WITHSCORES       # 按分数从低到高遍历
ZRANGE leaderboard 0 2 REV WITHSCORES   # 取分数最高的 3 名
ZRANK leaderboard "user:55"             # 查询从低到高的名次（从 0 开始）
```

跳表的代价是每个节点要保存多个链接和跨度，比单链表多占一些内存；但它避免了平衡树复杂的旋转维护，同时自然支持排名、反向遍历和范围扫描，因此适合 ZSet 排行榜、延迟队列等场景。

ZSet 因此能同时支持“按 member 找分数”和“按分数排序/取排名”。排行榜、延迟队列都很适合；但把百万级成员塞进单个 ZSet、频繁全量取出仍会造成大 key 和主线程压力。

**Bitmap：不是独立对象，而是 String 的位操作。** Bitmap 用 bit offset 表示状态，`SETBIT key offset value` 会设置某一位，`GETBIT` 读取，`BITCOUNT` 统计值为 1 的位数。若用户 ID 是连续、范围可控的整数，空间约为最大 offset / 8 字节，适合签到或已读标记；若直接把稀疏且很大的雪花 ID 当 offset，哪怕只写一个用户，也可能分配到很大的字符串空间。

```text
第 0 位：用户 0 是否签到
第 1 位：用户 1 是否签到
...
第 N 位：用户 N 是否签到
```

Bitmap 特别适合“一个对象对应一大批用户，每个用户只有一个是/否状态”的业务。常见场景：

| 业务场景 | key 与 offset 的一种设计 | 可做什么 | 注意点 |
| --- | --- | --- | --- |
| 月度签到 | `sign:{userId}:202611`，offset 为当月日期 - 1 | `SETBIT` 签到、`GETBIT` 查某天、`BITCOUNT` 算本月签到天数 | 每个用户每月一条 Bitmap；要处理补签规则和时区日期 |
| 活动参与/领取标记 | `campaign:{campaignId}:claimed`，offset 为内部连续用户编号 | 判断是否领过、记录已领取用户、统计领取人数 | offset 必须可控；活动资格、库存和真实发奖仍需业务/数据库校验 |
| 消息已读/任务完成 | `read:{messageId}` 或 `task:{batchId}:done`，offset 为内部成员/任务编号 | 快速判断状态、统计完成量、扫描处理进度 | 需要知道编号映射；单个 key 可能成为热点 |
| 日活/访问过标记 | `active:{date}`，offset 为内部用户编号 | 一天内同一用户重复访问仍只把 bit 设为 1，`BITCOUNT` 得到 UV | 只保存“来过没有”，不能查询访问次数和时间明细 |

签到示例：同一个用户的每个月份使用独立 key，日期 1 对应 offset 0：

```bash
# 用户 1001 在 2026 年 11 月 16 日签到
SETBIT sign:1001:202611 15 1

# 查询 11 月 16 日是否签到
GETBIT sign:1001:202611 15

# 统计本月已签到天数
BITCOUNT sign:1001:202611
```

选择 offset 时要看映射是否稠密。Bitmap 实际基于 String，内存大致随**最高被设置的 offset**增长，而不是只按值为 1 的 bit 个数增长。因此，雪花 ID、手机号、数据库自增 ID 如果范围很大或很稀疏，不应直接作为 offset；可以先映射到连续编号，或改用 Set/Hash。还要考虑 key 的维度：例如“每个用户一条月 Bitmap”便于查某用户整月状态，“每天一条全体用户 Bitmap”便于统计当天 UV，但后者容易形成全局热点。

Bitmap 能记录“某位置是否为 0/1”，不保存用户属性或事件明细；需要查询具体用户列表、保存时间或审计轨迹时，仍需 Set/Hash/数据库等结构。

**HyperLogLog：用少量内存估算不同元素个数。** 它不保存每个用户 ID，而是把多个哈希结果映射到有限个寄存器，根据哈希值中前导零等统计特征估算基数。Redis 使用 16384 个寄存器，标准误差约 0.81%；进入稠密表示后单个 HLL 通常约占 12KB，稀疏时可能更小。它适合每天活跃用户数、活动 UV、不同设备数、独立 IP 数、不同搜索用户数等“只关心去重人数、不关心具体成员”的统计。`PFADD` 添加样本，`PFCOUNT` 估算基数，`PFMERGE` 合并多个统计结果。

```text
存 Set：能回答“这个用户在不在”，但占内存较多
存 HyperLogLog：只能估算“有多少个不同用户”，内存很小
```

例如按活动和日期分 key 统计 UV，便于设置过期时间和合并周期数据：

```bash
# 记录用户访问活动；同一用户重复 PFADD 不会重复计数
PFADD uv:campaign:42:20261116 user:1001 user:1002 user:1001

# 读取当天近似 UV
PFCOUNT uv:campaign:42:20261116

# 合并多日统计，估算这段时间的去重 UV（不是把每天 UV 相加）
PFMERGE uv:campaign:42:period:20261110-13 \
  uv:campaign:42:20261110 uv:campaign:42:20261111 \
  uv:campaign:42:20261112 uv:campaign:42:20261113
PFCOUNT uv:campaign:42:period:20261110-13
```

选择 HLL 前先确认误差能否接受，并给统计 key 设计好日期/活动维度和 TTL。它不能用于判定某个用户是否出现过、列出用户清单，也不能拿来做精确扣费、精确去重或账务统计；误差虽小，但人数越大，可能的绝对误差也会变大。

**GEO：把坐标编码成可排序的分数，复用 ZSet。** 添加经纬度时，Redis 把地理位置编码为 GeoHash 数值并作为 ZSet score，member 是地点标识。因此 GEO 可以复用 ZSet 的有序结构，通过空间网格筛选候选点，再计算距离并返回附近结果。常见命令是 `GEOADD`、`GEOSEARCH`、`GEODIST`。

```text
member = store:1001
score  = 经度/纬度对应的 GeoHash 编码
```

GEO 适合“附近门店/附近销售”等范围查询，不是通用 GIS 引擎；复杂多边形、精确空间拓扑或大规模地理分析应交给专门地理数据库。

**Stream：rax 按消息 ID 排序索引，listpack 批量保存消息。** Stream 的消息 ID 通常是 `毫秒时间-序号`。底层 rax（基数树）按 ID 定位消息块，每个叶子相关节点包含一个 listpack，紧凑保存多条消息及字段值；消费组另有 last-delivered-id、消费者状态和 PEL（待确认消息）等元数据。

```text
rax：消息 ID 范围索引
├── ID 区间 A → listpack（多条消息）
├── ID 区间 B → listpack（多条消息）
└── ID 区间 C → listpack（多条消息）

消费组：已投递位置 + 消费者 + 未 ACK 的 PEL
```

`XADD` 追加消息，`XREAD` 按游标读，`XREADGROUP` 由消费组领取消息，`XACK` 确认完成。未确认消息仍在 PEL 中，可排查并转交；但消费者可能重试处理，所以业务仍需幂等。Stream 会占 Redis 内存，必须规划 `MAXLEN`/`XTRIM`，并按可靠性要求配置持久化和副本。

**SDS（简单动态字符串）**：含 `len`（已用）、`alloc`（已分配）、`flags`、`buf[]`。优点：① O(1) 取长度；② **二进制安全**（可存 `\0`）；③ 预分配减少重分配（长度 < 1MB 按 2 倍扩容，≥ 1MB 每次多分配 1MB）；④ 缩短时惰性释放。`embstr` 把 `redisObject` 与 SDS 放在同一块内存（一次分配，通常按不可变对象处理；需要修改且无法原地完成时转为 `raw`）。

**ziplist → listpack**：`ziplist` 每个 entry 记录**前一个 entry 的长度**，修改大 entry 会让后续 `prevlen` 扩张引发**连锁更新（cascade update）**，最坏 O(n²)；`listpack`（5.0 引入）改为记录**自身长度**，从根上消除该问题。**Redis 7.0 起 Hash/ZSet 的小对象编码由 ziplist 改为 listpack**，List 的 quicklist 内节点也换成 listpack，旧配置 `*-ziplist-*` 相应变为 `*-listpack-*`。

**为什么 ZSet 用跳表**：① 实现远简单于平衡树/B+ 树；② 底层是有序链表 + 多级索引，范围查询与 rank 定位天然友好；③ 增删查平均 O(logN)，与平衡树同阶，且无旋转重平衡逻辑。ZSet 同时用 `dict`（member→score，O(1) 查分）与 `skiplist`（按 score 排序）。

**编码转换阈值**（Redis 7.0；5.0/6.0 对应参数名为 `hash-max-ziplist-entries` 等）：`hash-max-listpack-entries` 默认 **128**、`hash-max-listpack-value` 默认 **64**；`zset-max-listpack-entries` 默认 **128**、`zset-max-listpack-value` 默认 **64**；`list-max-listpack-size` 默认 **128**；`set-max-intset-entries` 默认 **512**；`set-max-listpack-entries/value` 默认 128/64（7.2+）。**转换是单向的**，转成大编码后即使删元素也不会退回。

**共享对象与引用计数**：Redis 用 `refcount` 做内存回收，并对小整数使用**共享对象池**省内存，`OBJECT REFCOUNT` 对共享对象返回极大值（2147483647）。开启 LRU/LFU 淘汰时对象还要记录访问信息，共享对象的适用范围会受影响，具体行为以官方文档为准。

### 2.2 面试问答

- **Q1：SDS 相比 C 字符串好在哪？**

  C 字符串靠遇到 `\0` 才知道结束位置；Redis 的 SDS 额外保存已用长度 `len` 和可用容量 `alloc`，因此取长度是 O(1)，也能安全保存中间包含 `\0` 的二进制数据。追加内容时，如果预留容量足够就不必重新分配和复制整段字符串；缩短字符串后也可以暂时保留容量供后续复用。对 Redis 来说，这些设计能减少频繁拼接协议、命令和 value 时的内存操作。

- **Q2：ziplist 和 listpack 的区别是什么？为什么要替换？**

  两者都是把多个小元素紧凑地连续存放，以减少指针和节点开销。ziplist 的每个 entry 还记录“前一个 entry 的长度”；如果前一个 entry 变长，后一个 entry 的长度字段可能需要扩展，扩展又可能影响再后面的 entry，形成连锁更新。listpack 改为记录当前 entry 自己的长度，避免这种由前项变化引发的级联扩展。

  Redis 7.0 起，Hash/ZSet 的小对象紧凑编码使用 listpack；List 对外仍是 quicklist，只是 quicklist 内部节点使用 listpack。紧凑结构省内存，但元素多时查找和插入可能要扫描或移动数据，因此超过配置阈值会转成更适合大对象的编码。

- **Q3：为什么 ZSet 用跳表而不是红黑树？**

  ZSet 不只要“按 member 找分数”，还要按 score 排序、取排名、查分数范围。跳表的底层是有序链表，上面有多层索引；查找时从高层跳过大段元素，再逐层向下，平均 O(logN)。节点上的 span 可以累计跨越的元素数量，因此排名查询也方便；定位到范围起点后，底层链表又便于顺序扫描。

  Redis 选择跳表并不是说跳表在所有场景都优于红黑树，而是它实现相对直观，不需要树旋转和复杂平衡维护，同时很好地支持范围和排名操作。这个设计适合排行榜、积分排序等 ZSet 的主要使用方式。

- **Q4：什么时候小对象会变成大编码？为什么删除元素后不一定变回去？**

  当紧凑编码中的元素数量或单个元素长度超过配置阈值时，Redis 会把对象转换成更适合查找的大编码。例如 Hash 字段数超过 `hash-max-listpack-entries`，或字段/值超过 `hash-max-listpack-value`，可能转成 hashtable；ZSet 也有类似阈值。阈值与 Redis 版本和配置有关，不能把某个默认值当成所有版本都固定不变。

  转换需要分配新结构并搬移数据，之后即使删到很少元素，通常也不会自动反向转换，因为反复升降会带来额外开销。排查时可用 `OBJECT ENCODING key` 查看当前编码；设计缓存对象时要避免把过多字段或超长字段塞入一个 key。

- **Q5：Redis 的数据类型和底层结构是一回事吗？**

  不是。数据类型规定客户端能对它做什么，例如 Hash 支持 `HGET`/`HSET`，ZSet 支持按分数范围查询；底层编码是 Redis 为当前对象选择的实现方式。比如一个小 Hash 可以用 listpack，数据变大后转为 hashtable，但客户端看到的仍然是 Hash，命令语义不变。

  可以把类型理解成“对外的接口”，把编码理解成“内部的实现”。选择数据类型时先按业务查询和更新方式决定，再理解它可能使用的编码和转换成本。

- **Q6：Bitmap、HyperLogLog、GEO 是完全独立的底层结构吗？分别适合什么业务？**

  它们提供专门的命令接口，但底层复用了 Redis 的通用结构。Bitmap 把 String 当作连续 bit 数组，适合签到、已读、活动是否领取等 0/1 状态；HyperLogLog 以 String 保存统计寄存器，适合近似统计 UV/去重人数，但不能查询成员名单；GEO 复用 ZSet，以 member 保存地点、以 GeoHash 编码作为 score，适合附近门店等范围搜索。

  选择时看结果要求：要精确判断某个用户是否出现，用 Set/Bitmap；只要近似的不同人数，用 HyperLogLog；要根据经纬度查附近位置，用 GEO。不要用 HLL 做精确业务计数，也不要把雪花 ID 直接当 Bitmap offset，否则最高 offset 可能造成很大的字符串空间。

- **Q7：为什么小 Hash 用 listpack，大 Hash 改用 hashtable？**

  小 Hash 的 field/value 数量有限时，listpack 连续存储，不需要为每个字段维护哈希桶、节点和指针，内存利用率高。但查找一个 field 往往要沿着 listpack 顺序扫描，字段越多，查找成本越高。

  hashtable 需要更多内存保存桶和节点，但可以根据 field 的哈希值定位，查找平均 O(1)。所以 Redis 用阈值在“少量数据的空间效率”和“大量数据的查找效率”之间切换。CRM 的几十个展示字段可能适合一个 Hash；字段持续膨胀或单字段很大时，应考虑拆 key 或重新设计缓存模型。

- **Q8：ZSet 为什么同时保存 dict 和 skiplist？这样不是重复占内存吗？**

  是的，确实有额外内存开销，但两套结构各自服务不同查询：dict 保存 `member → score`，让按 member 查询或更新分数更快；skiplist 保存按 `(score, member)` 排序的节点，用于范围查询、升序/降序遍历和排名。

  如果只用 dict，就无法高效按分数排序；如果只用 skiplist，按 member 找到节点通常需要从跳表查找，更新和去重等操作不够直接。Redis 用额外内存换取两类常见操作都高效，因此大 ZSet 要同时考虑排序能力和双结构的内存占用。

- **Q9：Stream 有 PEL 就一定不会丢消息或重复消费吗？**

  PEL 记录已投递但尚未 `XACK` 的消息，消费者崩溃后可以通过 `XPENDING` 检查，并用 `XCLAIM`/`XAUTOCLAIM` 转交给其他消费者。它提供的是“能发现未确认消息并重试”的机制，不等于业务处理 exactly-once。

  消费者若先 `XACK` 再写数据库，进程在两步之间崩溃可能造成业务没落库但消息已确认；若先写数据库再 `XACK`，落库成功后崩溃则消息会重投。因此通常采用“业务处理幂等 + 成功后 ACK”，用消息 ID 或业务唯一键避免重复执行，并配置 Stream 裁剪和持久化策略。

### 2.3 高频追问与陷阱

- 陷阱：把 GEO 当独立结构——它底层就是 ZSet（score 是 GeoHash 值），继承 ZSet 的编码阈值；另外别以为编码能双向转换，删除元素不会让它退回紧凑编码。

### 2.4 代码/配置/命令示例

```bash
redis-cli OBJECT ENCODING mykey
redis-cli OBJECT REFCOUNT mykey
redis-cli OBJECT FREQ mykey              # 需 maxmemory-policy 为 LFU
redis-cli MEMORY USAGE mykey
redis-cli CONFIG GET hash-max-listpack-entries zset-max-listpack-entries
# 观察各类型的外部操作语义
redis-cli HSET customer:1001 name Ada status active
redis-cli SADD customer:1001:tags vip trial
redis-cli ZADD leaderboard 120 user:1001
redis-cli SETBIT attendance:202611 1001 1
redis-cli PFADD uv:campaign:1 user:1001 user:1002
redis-cli GEOADD stores 116.4 39.9 store:1001
redis-cli XADD events '*' type customer_created id 1001
# 不同版本/数据大小可能显示不同编码
redis-cli OBJECT ENCODING customer:1001
redis-cli OBJECT ENCODING customer:1001:tags
redis-cli OBJECT ENCODING leaderboard
```

### 2.5 一句话速记

**类型定义“能做什么”，编码定义“内部怎么存”；小集合倾向紧凑编码，大集合换查找更快的结构；Bitmap/GEO 复用 String/ZSet，ZSet 用 dict 查分、skiplist 排序与排名。**

## 3. 渐进式 rehash、过期删除与内存淘汰

### 3.1 核心原理

#### 先用人话理解：扩容、过期、淘汰是三件不同的事

把 Redis 的 key 字典想成一排储物柜：

```text
rehash：柜子数量要变，逐步把旧柜物品搬到新柜
过期：某个 key 到了 TTL，逻辑上不再有效
淘汰：内存达到 maxmemory，需要选 key 腾空间
```

过期时间到了，不代表内存一定在同一瞬间释放；key 可能等访问时被惰性删除，或等后台抽查。内存淘汰则是在内存压力达到限制时发生，只有配置允许淘汰时才会按策略选择 key。**TTL 是数据有效期，maxmemory-policy 是内存压力下的处置方式，两者不能混为一谈。**

**渐进式 rehash（哈希表搬家）**：Redis 的字典可以想成很多个桶，每个 key 经哈希计算后落进一个桶。key 数增加到一定程度，旧桶太挤，就需要扩大桶数组；若一次性搬完所有 key，字典很大时会让主线程停顿。

Redis 搬家时同时保留新旧两张表：

```text
搬家前：ht[0]（旧桶数组）
搬家中：ht[0]（尚未搬完） + ht[1]（新桶数组）
搬完后：ht[1] 成为新的 ht[0]
```

`rehashidx` 记录搬到哪里。搬家期间，每次访问会顺便推进一小段搬迁，而不是一次完成；查询可能先查新表，再查旧表；新写入放进新表，删除则确保两边都删。搬迁完成后释放旧表。这样把一次很长的停顿分散成许多小开销，代价是搬迁期间需要同时保留两份桶数组，并且部分操作稍复杂。

**过期删除：TTL 到点后，Redis 何时清掉 key？** 设置 `EXPIRE key 60` 是给 key 一个有效期。到期后 Redis 会把它视为过期，但不会启动一个线程精准地在每个 key 到点那一毫秒删除，因为全量定时检查所有 key 会很浪费。

Redis 用两种方式配合：

1. **惰性删除**：客户端访问 key 时才检查 TTL；若已过期，就删除并当作不存在返回。好处是没人访问的 key 不必立即处理；坏处是永远没人访问的过期 key 可能暂时占内存。
2. **定期删除**：后台定期从“设置了过期时间的 key”里抽样检查，清理过期项；如果抽样发现过期比例较高，会继续清理一会儿，但有时间限制，避免后台任务占住主线程太久。

因此，“过期”代表逻辑上不可再用，不代表内存一定当场释放。副本节点通常跟随主节点传播的删除，不自行主动删除，以免主从看到不同数据。

**内存淘汰：内存达到上限后，Redis 能不能删除仍未过期的 key？** `maxmemory` 是 Redis 用于数据集的内存上限。达到上限后，Redis 按 `maxmemory-policy` 决定怎么办：

| 策略类别 | 人话解释 | 适合什么情况 |
| --- | --- | --- |
| `noeviction` | 不删数据，新的写入报错 | 数据不能随意丢，且业务能处理写失败 |
| `allkeys-lru/lfu/random` | 所有 key 都可参与淘汰 | 实例里的数据都能重建，典型纯缓存 |
| `volatile-lru/lfu/random/ttl` | 只有设置了 TTL 的 key 可淘汰 | 想把无 TTL 数据排除在淘汰候选外 |

`volatile-*` 有个容易忽略的情况：如果可淘汰的带 TTL key 不够，Redis 即使达到内存上限也可能找不到候选项，写命令仍会失败。

**LRU 是什么？** LRU 是 *Least Recently Used* 的缩写，中文叫“最近最少使用”。它的直觉是：缓存满了要腾位置时，优先删除**最长时间没有被访问**的数据；刚刚被访问过的数据则视为“近期还可能会用”，尽量留下。

例如缓存最多放 3 个 key，依次放入 `A`、`B`、`C`，然后访问一次 `A`。此时最近使用顺序是 `A`、`C`、`B`（从新到旧）。再放入 `D` 导致缓存满时，LRU 会淘汰最久没访问的 `B`：

```text
放入 A、B、C：   最近 ← [C, B, A] → 最久
访问 A 之后：    最近 ← [A, C, B] → 最久
放入 D 需要腾位：删除 B，保留刚访问过的 A
```

一个精确的 LRU 缓存通常需要在每次访问时更新访问顺序，并维护“key 到节点”的查找结构；如果 Redis 对所有 key 都精确维护全局访问链表，每次读写也要承担额外的更新和内存成本。因此 Redis 的 LRU 是**近似 LRU**：达到 `maxmemory` 需要淘汰时，随机抽取一小批候选 key，从中选择最久未访问的淘汰，而不是比较所有 key。`maxmemory-samples` 默认是 5；调高通常能让候选更接近真正的最久未访问 key，但也会增加挑选开销。只有配置 `allkeys-lru` 或 `volatile-lru` 等 LRU 淘汰策略时，这套规则才用于淘汰；其中 `volatile-lru` 只从设置了 TTL 的 key 里挑选。

**LRU 和 LFU 的区别**：LRU 主要看“最近一次什么时候访问”，LFU（Least Frequently Used，最不经常使用）主要看“一段时间内访问了多少次”，并通过衰减逐渐降低旧访问的影响。比如一个 key 长期是热点，但刚好短时间没被访问，LRU 可能淘汰它；LFU 更可能保留它。反过来，突然被访问一次的冷 key 会让 LRU 暂时把它当成新近使用的数据，LFU 通常不会因此立刻把它视为长期热点。两者各有适用场景，不存在对所有业务都最优的策略。

**内存碎片是什么？** Redis 申请和释放不同大小的内存块后，系统实际占用的内存（RSS）可能大于 Redis 当前数据结构统计的内存（`used_memory`），这部分差异可能来自分配器对齐、空闲块和进程内其他内存。碎片率只是线索，不应单独作为故障结论；还要同时看 RSS、数据量趋势、swap 和容器内存限制。`activedefrag` 可以在支持的版本中整理部分碎片，但整理本身也消耗 CPU。

**过期删除 = 惰性删除 + 定期删除**：① 访问 key 时检查过期则删除并返回 nil；② 定期删除：`serverCron` 默认每 100ms（`hz` 默认 10）随机抽查 20 个设置了 TTL 的 key，若过期比例 > 25% 则继续抽查（有执行时间上限）。**从库不主动删过期 key**（避免主从不一致），等主库 `DEL` 同步；Redis 3.2+ 从库读到已过期 key 返回 nil 但不物理删除。

**内存淘汰 8 种策略**（`maxmemory-policy`）：`noeviction`（默认，写满后写命令报错）、`allkeys-lru` / `volatile-lru`、`allkeys-lfu` / `volatile-lfu`（4.0+）、`allkeys-random` / `volatile-random`、`volatile-ttl`（优先淘汰剩余 TTL 最短的）。**近似 LRU**：随机采样 `maxmemory-samples`（默认 **5**）个 key，淘汰其中最久未访问的——精确 LRU 需要维护全量访问链表，成本过高。**LFU**：每个对象用 24 bit（16 bit 计数 + 8 bit 衰减时间），计数**概率性递增**（`lfu-log-factor` 默认 10）并按 `lfu-decay-time`（默认 1 分钟）衰减。

**内存碎片与整理**：`mem_fragmentation_ratio = used_memory_rss / used_memory`，> 1.5 说明碎片严重（可开 `activedefrag yes`，配合 `active-defrag-ignore-bytes` 默认 100MB、`active-defrag-threshold-lower/upper` 默认 10/100），< 1 说明发生了 swap（严重告警）。碎片主因是频繁增删不同大小对象与 jemalloc 的分配策略。

### 3.2 面试问答

- **Q1：为什么需要渐进式 rehash？** 一次性 rehash 大字典会阻塞单线程数秒；渐进式把搬迁摊到每次访问，代价是内存翻倍。注意 rehash 期间查找要查两张表。
- **Q2：key 已过期但内存没释放？** 可能从未被访问（惰性删除未触发）且定期删除未抽到，也可能是从库（不主动删）。用 `INFO memory` 看 `expired_keys` 增速，或 `--bigkeys`/`MEMORY USAGE` 定位。
- **Q3：`volatile-lru` 和 `allkeys-lru` 怎么选？** 纯缓存用 `allkeys-lru`；实例里混有需要持久保存的数据（无 TTL）时用 `volatile-lru` 避免误删。**用 `volatile-*` 必须保证有 TTL 的 key 足够多**，否则内存满了却无可淘汰对象，写命令报 `OOM command not allowed`。
- **Q4：LFU 比 LRU 好在哪里？** LRU 只看最近访问时间，一个只被访问一次的冷数据可能顶掉长期热点（缓存污染）；LFU 看访问频次并带时间衰减，更适合有明显热点的场景，但新 key 需预热、对突发流量响应慢。
- **Q5：`maxmemory` 怎么设？** 要留 20%~30% 余量给复制缓冲区、AOF 缓冲、客户端输出缓冲与 fork 的 COW 页；顶到物理内存上限会触发 OOM killer 或 swap，延迟飙升。

### 3.3 高频追问与陷阱

- 陷阱：`DEL` 大 key 会阻塞主线程，应改用 `UNLINK`（4.0+），并配合 `lazyfree-lazy-expire`/`lazyfree-lazy-eviction`/`lazyfree-lazy-server-del` 让过期与淘汰也走异步删除。

### 3.4 代码/配置/命令示例

```bash
redis-cli INFO memory | grep -E "used_memory:|mem_fragmentation_ratio|maxmemory_policy"
redis-cli CONFIG SET maxmemory-policy allkeys-lru
redis-cli CONFIG GET maxmemory-samples activedefrag
redis-cli UNLINK bigkey            # 异步删除，避免阻塞
```

### 3.5 一句话速记

**渐进式 rehash 把搬迁摊到每次访问；过期靠惰性 + 定期删除、从库不主动删；淘汰 8 种策略，`volatile-*` 无 TTL 时等于不淘汰；碎片率 > 1.5 考虑 activedefrag。**

## 4. 持久化：RDB、AOF 与混合持久化

### 4.1 核心原理

#### 先用人话理解：持久化是在“内存速度”和“故障后恢复”之间取舍

Redis 平时主要在内存中处理数据。若机器断电，单靠内存无法保留状态，因此需要把状态或操作写到磁盘：

```text
RDB：某个时间点把“现在全部数据”拍成快照
AOF：把“发生过哪些写命令”记成日志
混合持久化：用快照做底，再追加后续命令
```

RDB 恢复时读快照，通常较快，但两次快照间的数据可能丢失；AOF 重放写命令，恢复可能更慢，但 `everysec` 通常把丢失窗口控制在约 1 秒（仍取决于操作系统和存储可靠性）。两者都是 Redis 本地持久化，不能替代备份、异地容灾或业务数据的权威数据库。

| 项 | RDB | AOF |
| --- | --- | --- |
| 内容 | 某时刻内存的**二进制快照** | **写命令**日志（Redis 7 为 multi-part） |
| 触发 | `SAVE`（阻塞）/`BGSAVE`（fork 子进程）、`save 900 1` 等策略 | `appendonly yes`，按 `appendfsync` 刷盘 |
| 恢复 / 安全 | 快 / 差（两次快照间会丢） | 慢（重放命令）/ 好（`everysec` 最多丢 1 秒） |

RDB 可以想成某一时刻整个 Redis 数据集的“照片”。恢复时直接装载这张照片，所以通常比从头重放很多命令快；缺点是照片只覆盖到保存时刻，照片之后还没保存的数据可能丢失。

**RDB 怎么在服务继续运行时拍快照？** `BGSAVE` 会 fork 一个子进程。fork 完成后，父子进程先共享同一批内存页；子进程把自己看到的那一刻的数据写成 RDB，父进程继续处理客户端写入。

```text
fork 时：父进程 ─┐
                 ├─ 共享内存页（逻辑上各自看到同一时刻状态）
         子进程 ─┘

父进程后来修改某页
        ↓
操作系统复制该页（Copy On Write，写时复制）
        ├─ 父进程改新页
        └─ 子进程仍读旧页，继续生成一致快照
```

fork 需要复制页表等管理信息，因此数据集越大，fork 阶段可能越久；这段时间主线程会短暂停顿。之后父进程写过的页需要额外复制，写入量大时内存会明显增长，极端情况下接近翻倍。这就是“fork 抖动”和 COW 内存压力。生产要给操作系统、复制缓冲和 COW 留内存空间，不能把机器内存全部分给数据集。

**AOF 可以想成“写操作流水账”**。例如收到 `SET k v`，Redis 把相应写命令追加到 AOF。重启时按顺序重放这些写命令，逐步恢复数据。AOF 不是每次都必然立刻写到物理磁盘：先写入操作系统缓存不等于已落到稳定存储，`appendfsync` 决定何时要求操作系统执行 fsync。

| `appendfsync` | 含义 | 取舍 |
| --- | --- | --- |
| `always` | 每次写后都请求 fsync | 丢失窗口小，但磁盘等待多、写入吞吐可能下降 |
| `everysec` | 后台大约每秒 fsync 一次 | 常用折中；故障时可能丢最近约 1 秒，仍取决于系统和硬件 |
| `no` | 由操作系统决定何时刷盘 | 性能可能较好，但丢失窗口更不可控 |

**AOF 重写不是把旧文件原样压缩。** 假如某 key 被 `INCR` 一万次，AOF 有一万条命令；重写时 Redis 根据当前内存状态生成能够恢复同样结果的较短命令，例如一条 `SET counter 10000`。重写期间新命令仍要被记录，避免重写完成时漏掉新写入。

**AOF 重写与"写时复制问题"**：`BGREWRITEAOF` 也 fork 子进程，按当前内存数据生成最小命令集；期间父进程的新写命令同时写入**原 AOF** 与 **AOF 重写缓冲区（`aof_rewrite_buf`）**，子进程完成后父进程把缓冲追加到新文件并原子改名。隐患：fork 的 COW 内存开销 + 重写期间 fsync 阻塞拖慢主线程。

**混合持久化**把“完整快照”和“后续命令”拼在一起：文件前面是 RDB 格式的基础数据，后面是 AOF 格式的增量写入。启动时先加载基础数据，再重放后续命令，目标是兼顾较快恢复与较小数据丢失窗口。

Redis 7 的 multi-part AOF 将 AOF 管理成一组文件：manifest 说明当前有效文件，base 保存某一时点的数据，incr 文件保存之后的命令。应用和运维不要手工删除其中某个文件；应使用 Redis 配置和受支持的备份流程管理整组文件。

### 4.2 面试问答

- **Q1：RDB 和 AOF 怎么选？** 生产一般**两者都开**（常用混合持久化）：RDB 用于快速恢复与备份传输，AOF 用于缩小数据丢失窗口。只允许一个时：能容忍分钟级丢失选 RDB，要求秒级选 AOF。
- **Q2：为什么 fork 会阻塞？** `fork` 要复制父进程页表（不是数据），页表越大耗时越长，且需要内存分配（`vm.overcommit_memory=1`）。大内存实例 fork 可能阻塞数百毫秒，是"主库周期性抖动"的常见原因。
- **Q3：COW 期间内存为什么会涨？** 父进程每写一个页就复制一份新页，最坏情况（所有页都被写）内存接近翻倍，因此要留余量并避免在高写入期同时执行 RDB 与 AOF 重写。
- **Q4：AOF 越来越大怎么办？** 靠**重写**（`BGREWRITEAOF`，可配 `auto-aof-rewrite-percentage` 与 `auto-aof-rewrite-min-size` 自动触发），本质是"用当前数据状态反推最小命令集"（一个 key `INCR` 一万次，重写后只需一条 `SET`）。
- **Q5：Redis 7 multi-part AOF 解决了什么？** 老版本重写要生成完整新文件再改名，内存与磁盘压力集中；multi-part 拆成 base + incr，重写只生成新 base 并回收旧 incr，降低单次重写开销与失败风险。另外 `BGSAVE` 与 `BGREWRITEAOF` **不能同时执行**，会互相排队。

### 4.3 高频追问与陷阱

- 陷阱：认为 RDB 子进程与父进程完全隔离（实际通过 COW 共享页）；把 `appendfsync always` 当成免费安全保证（每条写命令都等磁盘 fsync，吞吐可能降一个数量级）。

### 4.4 代码/配置/命令示例

```bash
redis-cli INFO persistence | grep -E "rdb_last_bgsave_status|aof_last_bgrewrite_status|rdb_changes_since_last_save"
redis-cli CONFIG GET appendfsync save dir
redis-cli BGSAVE && redis-cli BGREWRITEAOF
# 生产配置要点：appendonly yes / appendfsync everysec /
# auto-aof-rewrite-percentage 100 / auto-aof-rewrite-min-size 64mb / save 900 1
```

### 4.5 一句话速记

**RDB 是快照（fork + COW，恢复快但可能丢数据），AOF 是命令日志（everysec 最多丢 1 秒），生产两者都开；Redis 7 的 AOF 变为 base + incr 的 multi-part 结构。**

## 5. 主从复制与哨兵

### 5.1 核心原理

#### 先用人话理解：复制保留副本，哨兵负责“发现故障并换主”

```text
客户端写入主库
       ↓ 命令传播
   从库 A   从库 B
       ↑
哨兵监控主从状态；主库故障且达成仲裁后，选一个从库提升为新主
```

复制本身是异步的：主库确认写成功时，从库可能还没收到。因此“有副本”不等于“每次写都已在副本持久化”。哨兵可以自动故障转移，但不能凭空补回尚未复制的数据，也不能让网络分区下的所有节点都同时安全可写。

**主从复制到底做什么？** 主库处理写命令，并把写命令流发给从库；从库按顺序执行这些命令，尽量追上主库。它主要用于做副本、提供故障转移基础；是否把读流量也发给从库，要由应用/代理明确配置，不能默认认为从库自动分担读请求。

第一次建立复制关系时，从库没有主库当前完整数据，需要**全量同步**：

```text
从库：请求同步
  ↓
主库：生成一份 RDB 快照；同时把生成快照期间的新写入暂存起来
  ↓
主库 → 从库：发送快照
  ↓
从库：用快照装载完整数据
  ↓
主库 → 从库：补发快照期间暂存的写命令
  ↓
之后持续传播新写命令
```

所以全量同步会消耗主库 CPU、内存、磁盘/网络带宽，也会让从库加载数据。网络抖动后，如果从库落后的命令还保存在主库的复制积压缓冲区里，则只需**部分重同步**：主库按从库的复制偏移量补发缺少部分；如果所需内容已经被覆盖，就只能再次全量同步。

主库和从库各自记录“已经处理到哪里”的复制偏移量，主库用环形 `repl_backlog` 暂存近期复制数据。比如写入速率约 1MB/s、希望容忍 60 秒断连，积压缓冲区至少要覆盖约 60MB 的数据，还要留余量。默认值可能不足以覆盖真实网络抖动，需要按写入量和断连时间估算。

**复制风暴**：多个从库同时请求全量同步会让主库反复 fork 与传输（`repl-diskless-sync-delay` 默认 5 秒用于合并同一时刻的请求）；级联复制中某一层故障会造成连锁重同步。对策：树形级联复制、限制单主从库数量、调大 backlog、开启 `repl-diskless-sync yes`。

**哨兵怎么故障转移？** Sentinel 是独立运行的监控进程，彼此交流对主库状态的判断：

```text
一个哨兵联系不上主库 → 主观下线（sdown）
多个哨兵达到 quorum  → 客观下线（odown）
哨兵之间选出 Leader
        ↓
Leader 挑选合适从库并提升为新主
        ↓
其他从库改为复制新主；客户端发现并连接新主
```

`quorum` 是“多少个哨兵同意主库下线”的门槛，不等于选举 Leader 的票数门槛。Leader 还要获得哨兵多数派支持。新主通常先考虑优先级，再考虑复制进度等因素。旧主恢复后通常会被重新配置为新主的从库。

故障转移期间可能有短暂不可写；异步复制也意味着主库刚确认、但尚未复制到从库的写入可能丢失。网络分区时旧主还可能暂时接受写入，形成脑裂。因此哨兵解决的是“自动发现和切主”，不是强一致共识存储。

**脑裂与 `min-replicas`**：网络分区时旧主仍接受写，分区恢复后旧主被降级，其写入会**丢失**。缓解：`min-replicas-to-write`（默认 **0**）设为 ≥1、`min-replicas-max-lag`（默认 **10 秒**）设为合理值，使主库在可用从库不足时**拒绝写入**；也可用 `WAIT` 要求写操作在 N 个副本确认。

### 5.2 面试问答

- **Q1：全量同步与增量同步的触发条件？** 从库首次连接或偏移量已不在 `repl_backlog` 内走全量；断线重连且偏移量仍在 backlog 内走增量（部分重同步）。
- **Q2：`repl-backlog-size` 怎么定？** 按"主库平均写入速率 × 预计最长断连时长"估算。如写入 1MB/s、预计抖动 60 秒，应设 ≥ 64MB（默认 1MB 远不够，是常见生产隐患）。
- **Q3：哨兵怎么选新主？** 先过滤掉已下线、断连过久、`slave-priority=0` 的从库，再按"优先级 → 复制偏移量最大 → run_id 最小"排序选择，最后 `SLAVEOF NO ONE` 提升并让其他从库跟随。
- **Q4：quorum 和多数派是一回事吗？** 不是。`quorum` 只决定"多少个哨兵认为主库下线才算客观下线"；**选举 Leader 需要哨兵总数的多数派**。所以哨兵数量建议为奇数（3 或 5），quorum 通常设为 `N/2 + 1`。
- **Q5：怎么防脑裂导致的数据丢失？** `min-replicas-to-write` + `min-replicas-max-lag` 让主库在从库不足时拒写；分区恢复后人工检查旧主数据；关键业务保证同一时刻只有一个可写主（客户端仲裁或集群方案）；必要时用 `WAIT`。

### 5.3 高频追问与陷阱

- 陷阱：以为有哨兵就等于高可用——故障转移依赖网络与多数派，分区时可能选不出 Leader，且哨兵本身也要集群部署（单哨兵是单点）；另外从库 `read-only` 默认 yes 但手动写入仍会破坏一致性。

### 5.4 代码/配置/命令示例

```bash
redis-cli INFO replication
# 关注 role / connected_slaves / slaveN:ip,port,offset,lag / master_repl_offset / master_link_status
```

```conf
repl-backlog-size 64mb
repl-diskless-sync yes
min-replicas-to-write 1
min-replicas-max-lag 10
```

### 5.5 一句话速记

**全量同步 = bgsave + 复制缓冲区传输，增量同步靠 repl_backlog（默认仅 1MB，必须调大）；哨兵"主观下线 + quorum 客观下线 + 多数派选 Leader + 优先级/偏移量/run_id 选新主"，防脑裂靠 min-replicas。**

## 6. Cluster 分片与重定向

### 6.1 核心原理

#### 先用人话理解：Cluster 是把 key 分到多个 Redis 分片

```text
key
 ↓ CRC16 计算槽位（0～16383）
slot
 ↓ 找到负责该 slot 的主节点
节点执行命令；必要时客户端按 MOVED / ASK 重试
```

槽位是逻辑分区，节点是实际机器。扩缩容通常迁移槽，而不是让客户端重新计算全部 key 的位置。客户端要缓存槽位映射并处理重定向；跨节点的多 key 操作无法像单机那样天然一起执行，因此相关 key 需通过 hash tag 放进同一槽，或在客户端拆分请求。

客户端通常会缓存“每个槽由哪个节点负责”的路由表。发送命令时先对 key 算槽号，再发给对应节点：

```text
GET customer:{1001}:profile
        ↓ CRC16 + 取模
得到 slot 8234
        ↓ 查客户端槽位表
发给当前负责 slot 8234 的节点
```

Redis Cluster 把 key 映射到 0～16383 共 16384 个槽；节点负责其中一部分槽。增加节点时要把一部分槽及其 key 迁过去，这样集群才真正均衡。槽是逻辑编号，不是一台机器，也不是每个 key 一个槽。

`MOVED` 表示“这个槽已经由另一个节点负责”，客户端应更新槽位表后重试；`ASK` 表示“槽迁移进行中，这个 key 暂时去目标节点取”，客户端只在本次请求前发送 `ASKING`，不能因此永久改路由表。

多个 key 的 `MGET`、事务或 Lua 若需要由 Redis 在一次原子操作中处理，相关 key 必须落在同一槽。可以用 `{...}` hash tag 让 Redis 只对花括号里的部分计算槽号，例如 `customer:{1001}:profile` 和 `customer:{1001}:settings` 会同槽；但如果大量 key 都使用同一个 tag，也会造成单槽热点。

`16384` 的位图可用 2KB 表示节点负责的槽位集合，这是心跳包大小与分片粒度之间的设计折中。

**槽分配与迁移**：把槽标记为 `MIGRATING`（源）/`IMPORTING`（目标），逐个 `MIGRATE` 键，完成后 `CLUSTER SETSLOT <slot> NODE <target>` 正式移交。

**MOVED 与 ASK**：`MOVED` 表示"该槽现在**永久**由另一个节点负责"，客户端应**刷新槽映射缓存**后重试；`ASK` 表示"该槽正在迁移，这个键此刻在那个节点"，客户端**只对本次请求**先发 `ASKING` 再发命令，**不更新缓存**。

**功能限制**：① 多键命令（`MGET`/`MSET`/`SUNION`）要求所有键**同槽**，可用 **hash tag `{}`** 强制同槽（`user:{1001}:name` 与 `user:{1001}:age` 都按 `1001` 计槽）；② `MULTI/EXEC` 事务与 Lua 只能操作同槽键；③ pipeline 跨节点需客户端按节点拆分执行（多数客户端已支持）；④ 只有 db0；⑤ `SCAN`/`KEYS` 需遍历所有节点。

**gossip 通信**：节点间通过**集群总线**（客户端端口 + **10000**）用 gossip 协议传播节点状态、槽归属与故障信息（`PING`/`PONG`/`MEET`/`FAIL`）。故障判定为**主观下线 PFAIL + 多数主节点认定 FAIL**，再由主节点投票提升从节点。`cluster-require-full-coverage` 默认 **yes**，意味着**任一槽不可用则整个集群返回 CLUSTERDOWN**，很多生产环境会设为 `no`（代价是部分数据不可访问）。

**扩缩容**：扩容 = 新节点加入 + 迁出部分槽；缩容 = 迁走待下线节点的全部槽后 `CLUSTER FORGET`；批量操作用 `redis-cli --cluster reshard`/`rebalance`。

### 6.2 面试问答

- **Q1：为什么是 16384 个槽？** 心跳消息携带的槽位图若为 16384 位只占 2KB，每秒多次心跳下开销可接受；65536 会让心跳包膨胀到 8KB。同时 16384 已足够支撑常见集群规模的分片粒度。
- **Q2：MOVED 和 ASK 的本质区别？** MOVED 是**永久性重定向**（槽已归属新节点），客户端要刷新缓存；ASK 是**临时性重定向**（槽正在迁移），客户端只对本次请求加 `ASKING` 后重试、不更新缓存。搞混会导致客户端使用错误的槽映射。
- **Q3：集群下 `MGET` 为什么可能报错？** 不同 key 可能落在不同节点，Redis 无法跨节点执行多键命令。解法是用 hash tag 让相关 key 同槽，或客户端分组后分别 `MGET` 再合并。
- **Q4：gossip 的优缺点？** 优点：去中心化、节点可任意增减、故障信息最终传播到全集群；缺点：传播有秒级延迟、规模大时状态消息量增长、故障瞬间各节点视图可能不一致。
- **Q5：扩容时怎么避免客户端大量报错？** 按槽批量迁移且单次量小一些；客户端实现 MOVED/ASK 自动重试（主流客户端都有）；监控 `cluster_slots_pfail`/`migrating`/`importing`；低峰执行，避免迁移大 key（`MIGRATE` 阻塞源节点）。

### 6.3 高频追问与陷阱

- 陷阱：把 `cluster-require-full-coverage` 保持默认 `yes` 却以为集群"部分可用"——任一槽失联则整体 `CLUSTERDOWN`；另外集群节点数建议为奇数（至少 3 主 3 从），主从不要部署在同一台物理机。

### 6.4 代码/配置/命令示例

```bash
redis-cli -c CLUSTER INFO
redis-cli -c CLUSTER SLOTS | head
redis-cli --cluster check 127.0.0.1:7000
redis-cli --cluster reshard 127.0.0.1:7000 --cluster-from <src> --cluster-to <dst> --cluster-slots 1000 --cluster-yes
redis-cli -c CLUSTER COUNTKEYSINSLOT 100
```

### 6.5 一句话速记

**16384 槽（心跳位图 2KB）+ CRC16 取模；MOVED 永久重定向（刷新缓存）、ASK 临时重定向（带 ASKING、不刷缓存）；多键操作必须同槽（hash tag）；gossip 走端口 +10000；扩容靠槽迁移。**

## 7. 事务、Lua 脚本与 Pipeline

### 7.1 核心原理

#### 先用人话理解：三者分别解决“排队执行、原子判断、减少来回”

```text
MULTI/EXEC：先把命令排成一组，EXEC 时连续执行；有 WATCH 时可检测冲突
Lua：把读取、判断、修改写在一段服务端脚本里，中间不插入别的命令
Pipeline：客户端一次发多条、一次收多条，主要省网络往返
```

例如“库存大于 0 才减 1”不能安全地拆成普通 `GET` 和 `DECR` 两次独立请求，因为两个客户端可能都读到 1。用 Lua 把读取、判断和扣减放进同一段脚本，才可保证这组操作在 Redis 中不被其他命令插入。Pipeline 虽可批量发命令，但不提供这种原子性。

**事务（MULTI/EXEC）如何工作？** Redis 收到 `MULTI` 后，不马上执行后续命令，而是先把它们排进队列；收到 `EXEC` 后，再连续执行队列中的命令，这期间不会插入其他客户端命令。

```text
MULTI
SET order:1 status paid  → QUEUED（先入队）
INCR paid:count          → QUEUED
EXEC                     → 按顺序执行
```

`DISCARD` 可以在 `EXEC` 前丢弃队列。`WATCH key` 是乐观锁：客户端先观察 key，再准备事务；如果 key 在 `EXEC` 前被其他客户端修改，`EXEC` 会取消本次事务，应用可重新读取和重试。

Redis 事务**没有关系型数据库那种自动回滚能力**。命令入队阶段发现语法错误时，整个事务不会执行；但命令执行阶段发生运行时错误（例如对 String 执行 `LPUSH`），错误命令失败，其他命令仍可能继续执行，之前已执行的命令也不会撤销。因此事务中每条命令都要保证类型和参数正确。

**Lua 脚本如何实现原子操作？** Redis 在执行脚本期间不会执行其他客户端命令，所以脚本内“读、判断、写”之间不会被插入另一条命令。库存扣减可以在脚本里先读取库存、判断是否足够，再扣减并返回结果。这解决了多个客户端分开执行 `GET` 和 `DECR` 时的竞争问题。

但是 Lua 原子不等于数据库事务：脚本中途报错时，已经执行的写入不会自动回滚；脚本运行过久会阻塞该 Redis 实例上的其他命令。脚本应短小、限制循环规模，并且 Cluster 下脚本访问的多个 key 要放在同一槽。

**Pipeline（管道）怎么省时间？** 假设应用要执行 100 次 `GET`，普通方式可能是“发一条、等回复、再发一条”，网络来回 100 次；Pipeline 把 100 条命令一次发送，再集中读取回复，大幅减少网络往返。Redis 仍然要逐条执行这些命令，Pipeline 不会让命令本身变快，也不保证这批命令作为一个整体原子执行。

因此选择很简单：需要“减少网络来回”用 Pipeline；需要一组命令连续执行用事务；需要“读取后根据结果决定写什么”用 Lua 或 `WATCH` 事务。

### 7.2 面试问答

- **Q1：Redis 事务为什么不支持回滚？** 回滚需要 undo 日志与补偿逻辑，而 Redis 认为运行时错误属于编程错误、生产不应出现；命令本身多为简单操作，回滚收益远小于成本。官方文档明确说明了这一取舍。
- **Q2：`WATCH` 能做什么？** 乐观锁：`WATCH` 后 key 被改则 `EXEC` 返回 nil，客户端重试。它是 Redis 实现 CAS 的基础，但高并发下重试率可能很高，很多场景改用 Lua 更直接。
- **Q3：Lua 和事务、Pipeline 怎么选？** 需要**读后写**的原子逻辑（"值等于 X 才改成 Y"）用 Lua；只要求"命令不被打断"用 `MULTI/EXEC`（可叠加 `WATCH`）；只要求"减少 RTT"用 Pipeline。实践中 Lua 用得最多，因为它能表达条件逻辑。
- **Q4：Lua 脚本太长会怎样？** 单线程执行期间整个实例阻塞；超过 `lua-time-limit`（默认 5000ms）后 Redis 开始回复 `BUSY`，但**脚本仍继续执行**，只能用 `SCRIPT KILL`（仅限未写过数据的脚本）或 `SHUTDOWN NOSAVE` 强行停止。所以脚本必须短小。
- **Q5：Pipeline 和事务的区别？** Pipeline 解决客户端侧的**网络往返**，不保证原子、不保证不被打断；事务解决服务端侧的**不被打断**，但没有回滚能力。两者可叠加（把事务命令放进 pipeline）。

### 7.3 高频追问与陷阱

- 陷阱：以为 `MULTI` 里出错会整体回滚——只有入队时的语法/参数错误才导致整体不执行；运行时错误不回滚。另外别把 Pipeline 当"原子批量"用来保证并发安全。

### 7.4 代码/配置/命令示例

```bash
# 事务
redis-cli MULTI; redis-cli SET k v; redis-cli INCR counter; redis-cli EXEC
# Lua 原子扣减（脚本要短）
redis-cli EVAL "local s=tonumber(redis.call('GET',KEYS[1])); if s and s>=tonumber(ARGV[1]) then redis.call('DECRBY',KEYS[1],ARGV[1]); return 1 end return 0" 1 stock 1
# EVALSHA 未缓存时会报 NOSCRIPT，客户端需回退 EVAL
redis-cli -r 10000 -P 100 INCR counter      # -P 指定 pipeline 深度
```

### 7.5 一句话速记

**事务只保证"不被打断"、不支持回滚（运行时错误继续执行）；Lua 原子且能写条件逻辑但要短；Pipeline 只省 RTT、不保证原子。**

## 8. 发布订阅与 Stream

### 8.1 核心原理

#### 先用人话理解：Pub/Sub 是广播喇叭，Stream 是可追踪的消息队列

```text
Pub/Sub：发布 → 当前在线订阅者收到；离线期间没有消息记录
Stream：XADD 写入带 ID 的消息 → 消费者读取 → XACK 确认
                                      ↘ 未确认消息留在 PEL，可检查和转交
```

因此，Pub/Sub 适合“不重要、错过也没关系”的实时通知；订单、任务等需要重试和追踪的消息应使用 Stream 或专业 MQ。Stream 有消费组并不等于业务自动 exactly-once：消费者可能收到重投消息，业务处理仍要幂等。

**发布订阅（Pub/Sub）**就像广播喇叭：订阅者在线时，发布消息会推给当前订阅者；Redis 不把消息存起来。因此订阅者断线期间错过的消息无法补读，适合可丢失的即时通知，不适合支付、订单等必须可靠处理的业务。

**Stream（5.0 引入）**：真正的消息队列结构，底层是 **rax 树 + listpack**。

Stream 会保存带 ID 的消息，所以消费者可以从某个位置继续读取，也可以建立消费组。消费组把消息交给消费者处理；消费者处理成功后用 `XACK` 确认。未确认消息留在 PEL，可以检查和交给其他消费者重试。

```text
XADD 写入消息
   ↓
XREADGROUP 把消息交给消费者
   ↓
消费者执行业务
   ├─ 业务成功 → XACK，消息从待确认状态移除
   └─ 消费者崩溃/超时 → PEL 保留，可检查、接管、重试
```

Stream 提供可追踪和重试的基础，但不保证业务只执行一次：例如数据库写成功、`XACK` 前进程崩溃，消息会重新投递，业务必须幂等。Stream 消息会占 Redis 内存，应用需要设置裁剪策略；若消息必须长期保存、大量堆积或需要复杂消费语义，应使用专业消息队列。

### 8.2 面试问答

- **Q1：Redis 能做消息队列吗？** 能做但不专业。List 的 `LPUSH`+`BRPOP` 可实现简单队列，但没有 ACK、没有消费组，弹出后进程崩溃即丢消息；Stream 补上消费组、ACK、PEL 与回溯，接近专业 MQ 雏形；但**持久化与堆积能力受内存限制**，海量堆积仍应选 Kafka/RocketMQ（见 [../04-中间件/01-消息队列与RocketMQ.md](../04-中间件/01-消息队列与RocketMQ.md)）。
- **Q2：Pub/Sub 和 Stream 的核心区别？** Pub/Sub 是"广播 + 即发即忘"，无持久化、无消费组、不支持回溯；Stream 有 ID、有持久化、有消费组与 ACK，支持按 ID 回溯与重放。
- **Q3：Stream 怎么保证消息不丢？** 靠 `XACK` + PEL：消费者读取后未 ACK 的消息留在 PEL，可通过 `XPENDING` 监控、`XAUTOCLAIM` 转移重试。仍要注意 Redis 宕机且未持久化会丢、PEL 无限增长会吃内存（需定期 `XACK` + `XTRIM`）。
- **Q4：消费者崩溃后消息怎么接管？** 用 `XPENDING` 找出空闲超阈值的消息，`XCLAIM` 或 `XAUTOCLAIM` 把所有权转给存活消费者并重置空闲计时，形成"超时重投"机制。
- **Q5：集群下 Pub/Sub 有什么问题？** 默认模式 `PUBLISH` 广播到所有节点，节点多时网络开销大且不保证顺序；7.0 的分片发布订阅按槽路由，只发给负责该槽的节点，显著降低广播成本。

### 8.3 高频追问与陷阱

- 陷阱：用 Pub/Sub 做可靠通知（订单状态变更），订阅者离线就永久丢消息；另外 Stream 只 `XADD` 不 `XTRIM` 会导致内存持续增长。

### 8.4 代码/配置/命令示例

```bash
redis-cli XADD mystream * orderId 1001 status paid
redis-cli XGROUP CREATE mystream g1 0
redis-cli XREADGROUP GROUP g1 c1 COUNT 10 STREAMS mystream '>'
redis-cli XPENDING mystream g1
redis-cli XACK mystream g1 1700000000000-0
redis-cli XAUTOCLAIM mystream g1 c2 60000 0-0     # 接管空闲超 60s 的消息
redis-cli XADD mystream MAXLEN ~ 1000000 * f v      # 近似裁剪
```

### 8.5 一句话速记

**Pub/Sub 即发即忘（无持久化、无消费组），Stream 有 ID/消费组/ACK/PEL 可回溯；Redis 做 MQ 受内存限制，海量堆积仍选专业 MQ。**

## 9. 慢查询、bigkey/hotkey 与内存优化

### 9.1 核心原理

#### 先用人话理解：Redis 慢不一定是“命令本身慢”

一次请求的总耗时可以拆成：

```text
客户端排队/连接池等待
+ 网络往返
+ Redis 命令排队
+ 命令执行
+ 返回数据传输与反序列化
```

`SLOWLOG` 主要记录命令执行时间，所以若应用端耗时很高但 slowlog 很干净，还要检查网络、连接池、客户端线程池、输出数据量和 Redis 前面是否有长命令阻塞。bigkey 主要是单次操作数据量大，hotkey 则是同一个 key 被访问得特别频繁；两者是不同问题，可能同时存在。

**慢查询日志**记录 Redis 执行单条命令花了多久，不包含客户端排队、网络传输和应用反序列化时间。若应用感觉请求很慢，但 slowlog 没有慢命令，就继续检查网络、连接池和上游等待；若 slowlog 有 `HGETALL`、`KEYS` 等耗时命令，则检查是否全量处理了过多元素。常用 `SLOWLOG GET` 查看样本。

**bigkey（大 key）**指一个 key 的 value 特别大或包含很多成员。比如一个 Hash 有几百万字段，`HGETALL` 一次要扫描并返回全部字段，会占用 Redis 主线程、网络和客户端内存。删除它也可能需要释放大量内存，因此应拆成多个 key、按需取字段、用 `HSCAN` 分批遍历，删除时考虑 `UNLINK`。

**hotkey（热 key）**是一个 key 被很多请求反复访问。例如热门活动配置即使只有几百字节，也可能每秒被请求几十万次。Cluster 只能按 key 把它放到一个分片，不能自动把同一个 key 的请求摊到所有节点。可以用本地缓存、多个只读副本或业务侧分片来缓解。bigkey 是“单次处理量大”，hotkey 是“访问次数多”，两者可能同时发生。

**内存优化技巧**：控制 key 内字段数不超过紧凑编码阈值、key 名短而有规则、用 Hash 聚合小对象减少 key 数量、用 `intset` 存整数集合、用 Bitmap/HyperLogLog 替代大 Set、设置 TTL 并配合淘汰策略、定期 `--bigkeys` 体检。

`KEYS pattern` 会一次遍历整个 key 字典并返回所有匹配项，key 越多工作越久；因为 Redis 命令执行串行，生产环境不应在大库上运行 `KEYS *`。`SCAN` 则使用游标分批返回：每次调用处理一小部分，然后把新游标交给下一次调用。它不会像 `KEYS` 一样一次返回全集，但遍历过程中可能重复返回元素；应用要能去重或保证重复执行安全。新增、删除中的元素不保证恰好返回一次。

### 9.2 面试问答

- **Q1：`SCAN` 会漏数据吗？** 不会漏"遍历期间一直存在的元素"，但**可能重复**（rehash 与桶迁移导致），新增或删除的元素不保证；因此业务要能容忍重复。
- **Q2：`--bigkeys` 能在线跑吗？** 它基于 `SCAN`，理论上不阻塞，但会持续占用 CPU 与网络；大实例建议在**从库**或低峰执行并用 `-i` 限速。
- **Q3：`MONITOR` 为什么危险？** 它把每条执行的命令实时推送给所有 monitor 客户端；高 QPS 下监控端成为瓶颈、网络被打满、输出缓冲暴涨甚至触发 `client-output-buffer-limit` 断连。
- **Q4：内存持续增长怎么查？** ① `INFO memory` 看 `used_memory`/`used_memory_rss`/`mem_fragmentation_ratio`；② `--bigkeys` 查大 key；③ 检查 TTL 与淘汰策略（`volatile-*` 但无 TTL 会一直涨）；④ 检查客户端输出缓冲（`CLIENT LIST` 的 `omem`）与复制缓冲；⑤ 检查 `activedefrag` 是否开启。
- **Q5：为什么 `HGETALL` 是隐形坑？** 小 Hash 时它很快，但一旦字段数超过编码阈值（128），hash 变成 `hashtable`，`HGETALL` 就变成阻塞式全量拉取；应改用 `HMGET`/`HSCAN`。

### 9.3 高频追问与陷阱

- 陷阱：slowlog 阈值设得太小会产生海量日志并影响性能，太大则漏掉 10~100ms 的"慢性病"命令；`lazyfree-lazy-user-del yes` 能让 `DEL` 走异步路径，但异步删除期间内存不会立即释放。

### 9.4 代码/配置/命令示例

```bash
redis-cli CONFIG GET slowlog-log-slower-than slowlog-max-len
redis-cli SLOWLOG GET 10
redis-cli --bigkeys -i 0.1
redis-cli --hotkeys                    # 需 maxmemory-policy 为 LFU
redis-cli SCAN 0 MATCH user:* COUNT 100 TYPE hash
redis-cli CLIENT LIST | head
# 禁用危险命令：rename-command KEYS ""  /  rename-command DEBUG ""
```

### 9.5 一句话速记

**slowlog 只看命令执行时间（默认 10ms / 128 条）；bigkey 用 `--bigkeys` + `MEMORY USAGE` 查、拆小并用 `UNLINK` 删；hotkey 用 `--hotkeys`（需 LFU）查；一律用 `SCAN`，生产禁用 `KEYS` 与 `MONITOR`。**

## 10. 缓存与数据库一致性

### 10.1 核心原理

#### 先用人话理解：数据库是账本，缓存是可丢弃的副本

```text
读：先查缓存；没命中时查数据库，再把结果放回缓存
写：先更新数据库；再删缓存；下次读时重新从数据库加载
```

删除缓存而不是直接更新缓存，是因为缓存值常常是多个字段或多张表计算出来的，直接更新容易遗漏；删除后让下一次读取回填，规则更简单。但数据库与 Redis 不是同一个事务，删除失败或并发读写都可能产生短暂不一致，所以要有 TTL、重试/补偿和对账。涉及金额、库存等强正确性判断时，缓存只能辅助展示，最终校验必须落在数据库或具备一致性保证的权威存储上。

**Cache Aside（旁路缓存）**是主流方案：读时先查缓存，未命中则查 DB 并回填（可回填空值防穿透）；写时**先更新 DB、再删除缓存**。

写流程为什么通常是“更新数据库，再删除缓存”？数据库是权威数据源，缓存是可重建副本。先删缓存再更新数据库时，另一个读请求可能在数据库更新前读到旧值，并把旧值重新写进缓存；随后数据库更新完成，缓存却留下旧值。先更新数据库再删除缓存能缩小这类窗口，但仍不是跨数据库和 Redis 的原子事务。

还有一种并发边界：读请求先从数据库读到旧值，写请求更新数据库并删除缓存，之后读请求才把旧值写回缓存。这个窗口通常较短，但确实存在。TTL 只能让旧值最终过期，不能提供强一致；更可靠的最终一致方案是写库成功后可靠地产生缓存失效事件，并通过重试、顺序处理和对账修复。

**为什么是"删缓存"而不是"更新缓存"**：① 并发写下"更新缓存"可能写入由旧值计算出的中间态；② 若缓存值需要复杂计算，写入的内容可能频繁被覆盖而无人读取，浪费；③ 删除让下次读自然回填，逻辑简单。

| 顺序 | 问题 | 结论 |
| --- | --- | --- |
| 先更库、后删缓存 | 删除失败则读到旧数据（可重试/补偿） | **推荐** |
| 先删缓存、后更库 | "删完到更库完成"之间存在窗口，期间读请求会把**旧值**回填，之后长期脏 | 不推荐 |

**延迟双删**：先删缓存 → 更新 DB → 延迟一段时间（大于一次读请求耗时 + 主从延迟）再删一次。它只是**降低概率**，延迟时间难以精确估计，需配 TTL 兜底。

**更可靠的方案：binlog 订阅 + MQ 异步刷新**。用 Canal/Debezium 订阅 MySQL binlog，把变更事件投递到 MQ，由消费者**串行地**删除/刷新缓存。优点：应用只管写 DB（解耦）、可重试可顺序化可回溯、避开"更新成功但删缓存失败"的坑；代价是引入组件与毫秒级最终一致延迟。

**最终一致的兜底**：无论哪种方案都要能回答"不一致时谁兜底"——① 所有缓存**设置 TTL**（不一致也会自动收敛）；② 关键路径**以 DB 为准**（如扣库存前读 DB）；③ 异步对账任务定期扫描修复差异。

### 10.2 面试问答

- **Q1：为什么推荐"先更新 DB 再删缓存"？** 两者在极端并发下都可能不一致，但"先更库后删缓存"的不一致窗口更小（只有"读请求在删缓存前读到旧值并回填"才出问题）；"先删缓存后更库"的窗口正常存在，脏回填几乎必然。
- **Q2：删除缓存失败了怎么办？** ① 把删除投递到 MQ 重试；② 给缓存设短 TTL 让脏数据自然过期；③ 用 binlog 订阅方案从 DB 变更源头驱动删除，绕开"应用侧删除失败"。
- **Q3：延迟双删的延迟时间怎么定？** 要点是"大于一次读请求把旧值回填进缓存所需的时间"，工程上常取 500ms~1s 或基于读 P99 估计，并配 TTL 兜底；它是降低概率而非强一致保证。
- **Q4：强一致能用缓存吗？不一致导致资损怎么防？** 不能——引入缓存就是用一致性换性能；要求强一致的场景（金融扣款、库存扣减）应**直接读 DB 并加锁或用乐观锁**，缓存只服务最终一致的读多写少场景。防资损：关键校验路径不走缓存或加版本号校验、写入用乐观锁（`UPDATE ... WHERE stock >= n`）、用 binlog + MQ 保证删除最终执行、对账 + 熔断（发现不一致立即降级为直读 DB）。

### 10.3 高频追问与陷阱

- 陷阱：把"更新缓存"当优化（并发下会写入过期计算结果）；不设 TTL 只靠删除逻辑，一旦删除失败就**永久脏**；读写分离下"先更主库 → 删缓存 → 从库未同步 → 读从库"会把旧值回填，因此延迟双删必须把**主从延迟**算进去。

### 10.4 代码/配置/命令示例

```java
// Cache Aside 写路径：先更新 DB，再删缓存；删除失败进 MQ 重试，TTL 兜底
public void updateItem(Item item) {
    itemMapper.update(item);                        // 1. 更新 DB（事务内）
    try { redis.del(cacheKey(item.getId())); }      // 2. 删除缓存
    catch (Exception e) { mq.send(new CacheEvictMsg(item.getId())); }  // 3. 失败投递 MQ
}
// 核对：redis-cli GET item:1001 / redis-cli TTL item:1001（确认 TTL 已设置）
```

### 10.5 一句话速记

**Cache Aside + 先更库后删缓存是主流；删缓存失败靠 MQ 重试 + TTL 兜底；彻底可靠的方案是 binlog 订阅 + MQ 异步刷新；强一致场景不要用缓存。**

## 自测题（含答案要点）

1. **Redis 为什么快？单线程的准确范围？** —— 内存 + epoll + 免锁 + 高效结构；单线程指命令执行，另有后台线程与 6.0 多线程 IO。
2. **6.0 多线程 IO 优化了什么？** —— socket 读写与协议解析，命令执行仍单线程。
3. **SDS 的四大优点？** —— O(1) 长度、二进制安全、预分配、惰性释放。
4. **ziplist 与 listpack 的关键区别？** —— 前者存前一个 entry 长度会连锁更新，后者存自身长度；Redis 7 全面改用 listpack。
5. **ZSet 为什么用跳表？编码会降级吗、阈值多少？** —— 实现简单、范围与 rank 查询友好、性能同阶于平衡树；编码只升不降，hash/zset 的 listpack 阈值默认 128 个元素与 64 字节值。
6. **渐进式 rehash 期间查找与新增怎么做？过期删除的机制？** —— 查找两张表都查、新增只写 `ht[1]`；过期靠惰性 + 定期删除，从库不主动删、等主库 DEL 同步。
7. **八种淘汰策略？`volatile-*` 的坑？** —— noeviction + allkeys/volatile × lru/lfu/random + volatile-ttl；无 TTL 时等于不淘汰。
8. **近似 LRU 的实现？LFU 与 LRU 差别？** —— 采样 `maxmemory-samples`（默认 5）；LFU 看频次带衰减，更适合有明显热点的场景。
9. **RDB 的 fork 与 COW 有什么风险？** —— fork 阻塞与页表大小相关、COW 期间内存可能接近翻倍。
10. **AOF 三策略与推荐值？Redis 7 有什么变化？** —— always/everysec/no，默认 everysec；7.0 变为 manifest + base + incr 的 multi-part。
11. **全量同步与增量同步的触发条件？backlog 默认多大？** —— 首次连接或偏移量不在 backlog 内走全量；默认 1MB，必须按写入速率调大。
12. **哨兵如何判定下线与选主？quorum 与多数派的区别？** —— sdown + quorum 达 odown；多数派选 Leader；选新主按优先级 → 偏移量 → run_id。
13. **怎么防脑裂丢数据？** —— `min-replicas-to-write` + `min-replicas-max-lag` 让主库在从库不足时拒写。
14. **为什么是 16384 个槽？MOVED 与 ASK 的区别？** —— 心跳位图只需 2KB，65536 会涨到 8KB；MOVED 永久（刷新槽缓存），ASK 临时（本次带 `ASKING`，不更新缓存）。
15. **集群下 `MGET` 为什么报错？怎么解决？** —— 键可能落在不同槽；用 hash tag `{}` 强制同槽或客户端分组。
16. **Redis 事务为什么不回滚？Pub/Sub 与 Stream 的区别？** —— 运行时错误属编程错误，回滚成本高收益低；Pub/Sub 即发即忘无消费组，Stream 有 ID/持久化/消费组/ACK/PEL 可回溯。
17. **为什么禁用 `KEYS`？`SCAN` 有什么保证？** —— `KEYS` 是 O(N) 阻塞；`SCAN` 保证不丢但可能重复。
18. **bigkey/hotkey 怎么发现？** —— `--bigkeys`/`MEMORY USAGE`；`--hotkeys`（需 LFU）；`MONITOR` 慎用。
19. **缓存一致性为什么推荐"先更库后删缓存"？延迟双删能根治吗？** —— 先删缓存会留下"删完未更库"的窗口，旧值被回填后长期脏；延迟双删只是降低概率，必须配 TTL 兜底。

## 背诵卡片

1. 快的原因：内存 + epoll + 单线程免锁 + 高效数据结构。
2. 单线程指命令执行；6.0 多线程 IO 只优化网络读写与协议解析，`io-threads` 建议不超过 8。
3. 九种类型：String/List/Hash/Set/ZSet/Bitmap/HyperLogLog/GEO/Stream。
4. `embstr` ≤ 44 字节，超过转 `raw`；整数用 `int` 编码。
5. SDS：O(1) 长度、二进制安全、预分配（<1MB 翻倍，≥1MB 加 1MB）、惰性释放。
6. listpack 存自身长度，消除 ziplist 连锁更新；Redis 7 全面替换。
7. `hash-max-listpack-entries` 128、`hash-max-listpack-value` 64。
8. `zset-max-listpack-entries` 128、`zset-max-listpack-value` 64。
9. `set-max-intset-entries` 512；`list-max-listpack-size` 128。
10. 编码只升不降，删元素也不会退回紧凑编码。
11. ZSet 同时用 dict（O(1) 查分）与 skiplist（范围/rank）。
12. 共享对象池用于小整数，`OBJECT REFCOUNT` 返回 2147483647。
13. 渐进式 rehash：查两张表、新增只写 `ht[1]`。
14. 过期删除 = 惰性 + 定期（`hz` 默认 10，每 100ms 抽查 20 个）。
15. 从库不主动删过期 key，等主库 DEL 同步。
16. 淘汰策略 8 种；`volatile-*` 在无 TTL 时等于 `noeviction`。
17. 近似 LRU 采样 `maxmemory-samples` 默认 5；LFU 自 4.0 引入。
18. `mem_fragmentation_ratio > 1.5` 考虑 `activedefrag`；< 1 说明发生 swap。
19. RDB 用 fork + COW；fork 阻塞与页表大小相关。
20. AOF `appendfsync` 默认 everysec，最多丢 1 秒。
21. Redis 7 AOF = manifest + base + incr 的 multi-part 结构。
22. 全量同步走 bgsave + 复制缓冲；增量同步靠 `repl_backlog`（默认 1MB）。
23. 哨兵 sdown 默认 30000ms，odown 需达 quorum；选 Leader 要多数派。
24. 选新主顺序：优先级 → 复制偏移量最大 → run_id 最小。
25. 脑裂防护：`min-replicas-to-write` 默认 0，`min-replicas-max-lag` 默认 10 秒。
26. Cluster 16384 槽 = CRC16(key) % 16384；心跳位图 2KB。
27. MOVED 永久重定向（刷新缓存）、ASK 临时重定向（带 ASKING）。
28. hash tag `{}` 让相关 key 同槽；集群总线端口 = 客户端端口 + 10000。
29. `cluster-require-full-coverage` 默认 yes，任一槽失效则整体 CLUSTERDOWN。
30. 事务不支持回滚；Lua 原子但阻塞，`lua-time-limit` 默认 5000ms。
31. Pipeline 只省 RTT，不保证原子、不保证不被打断。
32. Stream 靠 `XACK` + PEL + `XAUTOCLAIM` 实现可靠消费；7.0 有分片发布订阅。
33. slowlog 默认阈值 10000 微秒、最多 128 条，只统计命令执行时间。
34. `SCAN` 保证不丢但可能重复；生产禁用 `KEYS` 与 `MONITOR`。
35. `UNLINK` 异步删除大 key；`lazyfree-lazy-*` 让过期/淘汰也异步。
36. 缓存一致性：先更库后删缓存，删失败靠 MQ 重试 + TTL 兜底，彻底方案是 binlog 订阅 + MQ。
