# Redis数据结构与核心原理

> **定位**：从单线程模型与九种数据类型的底层编码，到持久化、复制、Cluster、内存淘汰与缓存一致性，覆盖 Redis 面试的全部主干。
> **重要度**：★★★★★（面试出现频率）
> **建议复习时长**：6 小时
> **关联文档**：[./05-Redis高可用与缓存治理.md](./05-Redis高可用与缓存治理.md)、[./01-MySQL体系结构与索引.md](./01-MySQL体系结构与索引.md)、[../06-计算机基础/02-操作系统.md](../06-计算机基础/02-操作系统.md)

## 0. 知识地图

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

Redis 是**内存型键值数据库**，同时提供持久化、复制、集群、Lua、事务、发布订阅与 Stream，因此常被当作缓存、分布式锁、计数器、排行榜、消息队列与限流器使用。

**为什么快**：① 数据在**内存**中，读写不碰磁盘；② **IO 多路复用**（Linux 下 epoll）用单线程管理海量连接，无需每连接一线程；③ **单线程执行命令**，免锁、免上下文切换、免竞态（Lua 与事务也因此天然原子）；④ 底层数据结构高度优化（SDS、listpack、跳表）。

**单线程的准确范围**：单线程指**命令执行**这一条路径。另有后台线程/子进程：`bgsave`/`bgrewriteaof` 的 fork 子进程、`lazyfree` 异步删除（4.0+ `UNLINK`）、AOF 刷盘、关闭大文件。**Redis 6.0 引入多线程 IO**（`io-threads` 默认 1 即关闭，`io-threads-do-reads` 默认 no），把网络读写与协议解析分摊到多线程，**命令执行仍单线程**——它解决的是网络瓶颈而非 CPU 瓶颈。

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

| 类型 | 底层编码 | 典型用途 |
| --- | --- | --- |
| String | `int` / `embstr`（≤44 字节）/ `raw`（SDS） | 缓存、计数器、分布式锁 |
| List | `listpack`（小）/ `quicklist`（3.2+） | 消息队列、时间线 |
| Hash | `listpack` / `hashtable` | 对象缓存、购物车 |
| Set | `intset` / `listpack`（7.2+）/ `hashtable` | 去重、共同好友 |
| ZSet | `listpack` / `skiplist` + `dict` | 排行榜、延迟队列 |
| Bitmap / HyperLogLog / GEO | String 位操作 / 稀疏稠密表示 / ZSet（GeoHash 编码为 score） | 签到与 UV（误差约 0.81%）、附近的人 |
| Stream | `listpack` + rax 树 | 带消费组与 ACK 的消息队列 |

**SDS（简单动态字符串）**：含 `len`（已用）、`alloc`（已分配）、`flags`、`buf[]`。优点：① O(1) 取长度；② **二进制安全**（可存 `\0`）；③ 预分配减少重分配（长度 < 1MB 按 2 倍扩容，≥ 1MB 每次多分配 1MB）；④ 缩短时惰性释放。`embstr` 把 `redisObject` 与 SDS 放在同一块内存（一次分配、只读，修改转 `raw`）。

**ziplist → listpack**：`ziplist` 每个 entry 记录**前一个 entry 的长度**，修改大 entry 会让后续 `prevlen` 扩张引发**连锁更新（cascade update）**，最坏 O(n²)；`listpack`（5.0 引入）改为记录**自身长度**，从根上消除该问题。**Redis 7.0 起 Hash/ZSet 的小对象编码由 ziplist 改为 listpack**，List 的 quicklist 内节点也换成 listpack，旧配置 `*-ziplist-*` 相应变为 `*-listpack-*`。

**为什么 ZSet 用跳表**：① 实现远简单于平衡树/B+ 树；② 底层是有序链表 + 多级索引，范围查询与 rank 定位天然友好；③ 增删查平均 O(logN)，与平衡树同阶，且无旋转重平衡逻辑。ZSet 同时用 `dict`（member→score，O(1) 查分）与 `skiplist`（按 score 排序）。

**编码转换阈值**（Redis 7.0；5.0/6.0 对应参数名为 `hash-max-ziplist-entries` 等）：`hash-max-listpack-entries` 默认 **128**、`hash-max-listpack-value` 默认 **64**；`zset-max-listpack-entries` 默认 **128**、`zset-max-listpack-value` 默认 **64**；`list-max-listpack-size` 默认 **128**；`set-max-intset-entries` 默认 **512**；`set-max-listpack-entries/value` 默认 128/64（7.2+）。**转换是单向的**，转成大编码后即使删元素也不会退回。

**共享对象与引用计数**：Redis 用 `refcount` 做内存回收，并对小整数使用**共享对象池**省内存，`OBJECT REFCOUNT` 对共享对象返回极大值（2147483647）。开启 LRU/LFU 淘汰时对象还要记录访问信息，共享对象的适用范围会受影响，具体行为以官方文档为准。

### 2.2 面试问答

- **Q1：SDS 相比 C 字符串好在哪？** O(1) 取长度、二进制安全、预分配 + 惰性释放减少重分配与拷贝、兼容部分 C 字符串函数。
- **Q2：ziplist 和 listpack 的区别？** ziplist 存**前一个 entry 长度**会引发连锁更新；listpack 存**自身长度**，无连锁更新且更紧凑。Redis 7.0 起 Hash/ZSet/List 的小对象编码全部换成 listpack。
- **Q3：为什么 ZSet 用跳表而不是红黑树？** 跳表实现简单、范围与 rank 查询天然友好、无旋转重平衡，性能同阶于平衡树；红黑树纯查找略优但实现复杂度与范围查询便利性不如跳表。
- **Q4：什么时候小对象会变成大编码？** 元素个数或单个元素长度超过阈值（如 Hash 超 128 个字段或某字段值超 64 字节）即转换，且**只升不降**；因此要避免把大对象写进同一个 key，长期运行可能出现"元素很少但已是 hashtable"的 key。

### 2.3 高频追问与陷阱

- 陷阱：把 GEO 当独立结构——它底层就是 ZSet（score 是 GeoHash 值），继承 ZSet 的编码阈值；另外别以为编码能双向转换，删除元素不会让它退回紧凑编码。

### 2.4 代码/配置/命令示例

```bash
redis-cli OBJECT ENCODING mykey
redis-cli OBJECT REFCOUNT mykey
redis-cli OBJECT FREQ mykey              # 需 maxmemory-policy 为 LFU
redis-cli MEMORY USAGE mykey
redis-cli CONFIG GET hash-max-listpack-entries zset-max-listpack-entries
```

### 2.5 一句话速记

**String/Hash/Set/ZSet 都是"小对象用紧凑编码、超阈值转 hashtable/skiplist"；Redis 7 用 listpack 取代 ziplist 消除连锁更新；ZSet 选跳表为范围与 rank 查询。**

## 3. 渐进式 rehash、过期删除与内存淘汰

### 3.1 核心原理

**渐进式 rehash**：字典 `dict` 有两个哈希表 `ht[0]`/`ht[1]` 与 `rehashidx`。扩容/缩容时给 `ht[1]` 分配空间，之后**每次增删改查都顺带搬迁一个桶**，而非一次搬完。rehash 期间查找**两张表都查**、新增只写 `ht[1]`、删除两张表都删。这样把"大字典一次搬迁"的长阻塞切成无数次微小开销，代价是期间内存占用翻倍。

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

| 项 | RDB | AOF |
| --- | --- | --- |
| 内容 | 某时刻内存的**二进制快照** | **写命令**日志（Redis 7 为 multi-part） |
| 触发 | `SAVE`（阻塞）/`BGSAVE`（fork 子进程）、`save 900 1` 等策略 | `appendonly yes`，按 `appendfsync` 刷盘 |
| 恢复 / 安全 | 快 / 差（两次快照间会丢） | 慢（重放命令）/ 好（`everysec` 最多丢 1 秒） |

**RDB 的 fork 与写时复制（COW）**：`BGSAVE` 用 `fork()` 创建子进程，二者共享物理内存页；父进程继续处理写请求时，**写到的页会被复制一份**，子进程看到 fork 时刻的一致快照。三个要点：① `fork` 本身**阻塞主线程**，耗时与页表大小正相关（大内存实例可达数百毫秒，即"fork 抖动"）；② COW 期间内存可能接近翻倍；③ `vm.overcommit_memory` 应设为 1，否则 fork 可能失败。

**AOF 三策略（`appendfsync`）**：`always`（每条命令 fsync，最安全最慢）、`everysec`（**默认**，后台线程每秒 fsync，最多丢 1 秒）、`no`（交给 OS，通常 30 秒左右）。`no-appendfsync-on-rewrite yes` 可在重写期间不 fsync，避免 fsync 阻塞主线程（牺牲一点安全性）。

**AOF 重写与"写时复制问题"**：`BGREWRITEAOF` 也 fork 子进程，按当前内存数据生成最小命令集；期间父进程的新写命令同时写入**原 AOF** 与 **AOF 重写缓冲区（`aof_rewrite_buf`）**，子进程完成后父进程把缓冲追加到新文件并原子改名。隐患：fork 的 COW 内存开销 + 重写期间 fsync 阻塞拖慢主线程。

**混合持久化与 Redis 7 multi-part AOF**：4.0 引入 `aof-use-rdb-preamble`——AOF 前半是 RDB 快照、后半是命令追加，兼顾恢复速度与安全性。**Redis 7.0 把 AOF 重构为 multi-part**：`appendonlydir` 目录下由 **manifest 清单 + base 文件（RDB 或 AOF 格式）+ 多个 incr 增量文件**组成，重写只需替换 base 与 incr，不必重写整个大文件；`aof-use-rdb-preamble` 的语义被 manifest 机制取代（具体配置项以官方文档为准）。

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

**全量同步**：从库发 `PSYNC ? -1` → 主库 `BGSAVE` 生成 RDB 并把期间新写命令写入**复制缓冲区（replication buffer）** → 发送 RDB → 从库清空自身数据并加载 → 主库把缓冲区中的增量命令发过去 → 进入命令传播阶段。开销集中在 **fork + RDB 传输 + 从库清库加载**。

**增量同步**：主从各自维护 `run_id` 与复制偏移量，主库另有环形**复制积压缓冲区 `repl_backlog`**（`repl-backlog-size` 默认 **1MB**，`repl-backlog-ttl` 默认 3600 秒）。从库重连时带上偏移量，若仍在 backlog 内则只发差额（部分重同步），否则退化为全量。**backlog 太小是"网络抖动后频繁全量同步"的典型原因**，应按"平均写入速率 × 预计断连时长"估算。

**复制风暴**：多个从库同时请求全量同步会让主库反复 fork 与传输（`repl-diskless-sync-delay` 默认 5 秒用于合并同一时刻的请求）；级联复制中某一层故障会造成连锁重同步。对策：树形级联复制、限制单主从库数量、调大 backlog、开启 `repl-diskless-sync yes`。

**哨兵（Sentinel）**：① **主观下线 sdown**：单个哨兵在 `down-after-milliseconds`（默认 **30000ms**）内未收到有效 PING 回复；② **客观下线 odown**：认为 sdown 的哨兵数达到 `quorum`；③ **Raft 式选举**：需要 odown 的哨兵向其他哨兵拉票，获得**多数派**（含自己一票）者成为 Leader，每个任期只有一次投票机会；④ **故障转移**：Leader 依次按 `slave-priority`（越小越优先，0 不参与）→ **复制偏移量最大**（数据最新）→ `run_id` 字典序最小 选出新主，执行 `SLAVEOF NO ONE` 并让其他从库跟随，旧主恢复后降级为从库。

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

**16384 个槽（slot）**：`slot = CRC16(key) % 16384`。为什么是 16384：① 心跳包用位图表示节点负责的槽，16384 位 = **2KB**，若 65536 位则 8KB，心跳开销显著上升；② 集群规模一般不过千，16384 的分片粒度足够；③ 该数字是作者在官方文档中的明确设计权衡（权威结论以官方文档为准）。

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

**事务**：`MULTI` 开启 → 命令入队（返回 `QUEUED`）→ `EXEC` 顺序执行 → `DISCARD` 放弃 → `WATCH key` 做乐观锁（被 WATCH 的 key 在 EXEC 前被改则 EXEC 返回 nil）。**Redis 事务不支持回滚**：① **入队阶段的语法/参数错误**会让整个 `EXEC` 被拒绝（一条都不执行）；② **执行阶段的运行时错误**（如对 String 用 `LPUSH`）只影响该条命令，其余**照常执行**。官方设计哲学是"运行时错误属于编程错误，回滚带来的复杂度没有实际收益"。

**Lua 脚本**：`EVAL script numkeys key... arg...`，或 `SCRIPT LOAD` + `EVALSHA`（脚本缓存在服务端，省带宽）。特性：① 脚本执行**原子**（单线程期间不会插入其他命令）；② `redis.call` 出错会中断脚本并返回错误，但**已执行的写命令不回滚**；③ 脚本不可长循环，会阻塞整个实例；④ 集群下必须操作同槽键。典型场景：分布式锁释放、库存扣减、限流计数等"读后写"原子逻辑。

**Pipeline（管道）**：客户端一次性发出多条命令、一次性读回，**减少 RTT**。要点：① **不保证原子**，中间可能插入其他客户端的命令；② 只省网络往返，不减少服务端执行时间；③ 单次 Pipeline 的命令数量要控制（几千条量级），否则会撑大输出缓冲并可能触发 `client-output-buffer-limit` 被杀连接。

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

**发布订阅（Pub/Sub）**：`SUBSCRIBE`/`PSUBSCRIBE` 订阅、`PUBLISH` 发布。特点是**发后即忘**——消息不持久化，订阅者离线期间的消息永久丢失；Cluster 下默认广播到**所有节点**（7.0 起提供 `SSUBSCRIBE`/`SPUBLISH` 的**分片发布订阅**，按槽路由以降低广播开销）。适合实时通知、配置推送等可丢场景；`PUBSUB CHANNELS`/`NUMSUB`/`NUMPAT` 可观测。

**Stream（5.0 引入）**：真正的消息队列结构，底层是 **rax 树 + listpack**。

- 写入：`XADD mystream * field value`（`*` 自动生成 `毫秒-序号` 形式的单调递增 ID）。
- 读取：`XRANGE`（按 ID 范围）、`XREAD`（可 `BLOCK` 阻塞）、`XREADGROUP`（消费组）。
- 消费组：`XGROUP CREATE` 建组，组内消费者 `XREADGROUP GROUP g c` 拉取，读完需 **`XACK`** 确认；未确认消息进入 **PEL（Pending Entries List）**，用 `XPENDING` 查看、`XAUTOCLAIM`（6.2+）/`XCLAIM` 把超时未确认的消息转移给其他消费者。
- 裁剪：`XADD ... MAXLEN ~ 1000000` 近似裁剪（`~` 表示按节点边界裁剪，性能更好），或 `XTRIM`；`XINFO STREAM/GROUPS` 查看状态。

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

**慢查询日志**：`slowlog-log-slower-than` 默认 **10000 微秒（10ms）**、`slowlog-max-len` 默认 **128**。注意：**它只统计命令执行时间，不包含网络传输与排队时间**，所以慢客户端与阻塞排队不会体现在 slowlog 里。用 `SLOWLOG GET/LEN/RESET` 排查。

**bigkey（大 key）**：单个 key 的 value 过大（String > 10KB，或集合类元素数 > 5000，具体按业务定）。危害：单次操作耗时长阻塞主线程、网络传输放大、删除时阻塞（用 `UNLINK`）、集群下数据倾斜。发现：`redis-cli --bigkeys`（基于 `SCAN`，**对线上有压力，建议从库或低峰执行**）、`MEMORY USAGE key`（4.0+）、`--memkeys`；`DEBUG OBJECT` 生产禁用。治理：按字段哈希拆分、用 `HMGET` 而非 `HGETALL`、`UNLINK` 删除、用 `SSCAN`/`HSCAN`/`ZSCAN` 分页读取。

**hotkey（热 key）**：单个 key 的 QPS 极高（秒杀商品、明星微博），会打满单节点 CPU/网卡，集群下也无法靠加节点分摊。发现：`redis-cli --hotkeys`（**需 `maxmemory-policy` 为 LFU**）、客户端埋点统计、代理层统计；**`MONITOR` 会把每条命令实时推送给所有 monitor 客户端，高 QPS 下会打满网络与输出缓冲，生产慎用甚至禁用**。治理见 [./05-Redis高可用与缓存治理.md](./05-Redis高可用与缓存治理.md)。

**内存优化技巧**：控制 key 内字段数不超过紧凑编码阈值、key 名短而有规则、用 Hash 聚合小对象减少 key 数量、用 `intset` 存整数集合、用 Bitmap/HyperLogLog 替代大 Set、设置 TTL 并配合淘汰策略、定期 `--bigkeys` 体检。

**`SCAN` 家族与禁用 `KEYS`**：`SCAN cursor [MATCH p] [COUNT n] [TYPE t]`（`SSCAN`/`HSCAN`/`ZSCAN` 同族）用游标分批遍历、**不阻塞**主线程。两个保证：① 遍历期间**一直存在的元素一定会被返回**；② 中途新增/删除的元素可能返回 0~多次（**可能重复**，业务需幂等/去重）。`KEYS *` 是 O(N) 全量遍历，生产必须禁用（可用 `rename-command KEYS ""`）。

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

**Cache Aside（旁路缓存）**是主流方案：读时先查缓存，未命中则查 DB 并回填（可回填空值防穿透）；写时**先更新 DB、再删除缓存**。

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
