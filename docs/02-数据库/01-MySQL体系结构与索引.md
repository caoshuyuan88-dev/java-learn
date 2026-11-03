# MySQL体系结构与索引

> **定位**：从一条 SQL 的生命周期切入 MySQL 架构与 InnoDB 存储引擎，落到 B+ 树原理、联合索引设计与索引失效排查，是数据库所有面试题的底座。
> **重要度**：★★★★★（面试出现频率）
> **建议复习时长**：6 小时
> **关联文档**：[./02-MySQL事务锁与MVCC.md](./02-MySQL事务锁与MVCC.md)、[./03-MySQL调优主从与分库分表.md](./03-MySQL调优主从与分库分表.md)、[../06-计算机基础/03-数据结构与算法.md](../06-计算机基础/03-数据结构与算法.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| 1 逻辑架构、SQL 执行流程与连接管理 | ★★★★★ | 八股原理 / 源码追问 |
| 2 InnoDB 内存结构 | ★★★★★ | 八股原理 / 调优排查 |
| 3 磁盘结构、行格式与 redo/undo/binlog | ★★★★★ | 八股原理 / 故障复盘 |
| 4 B+ 树原理与容量估算 | ★★★★★ | 八股原理 / 场景题 |
| 5 聚集索引、回表与覆盖索引 | ★★★★★ | 八股原理 / SQL 手写 |
| 6 索引类型、最左前缀与联合索引设计 | ★★★★★ | SQL 手写 / 场景题 |
| 7 ICP 与索引失效场景大全 | ★★★★★ | 调优排查 / SQL 手写 |
| 8 EXPLAIN、optimizer trace 与慢查询 | ★★★★★ | 调优排查 / 故障复盘 |
| 9 Join 算法、排序与临时表 | ★★★★☆ | 八股原理 / 调优排查 |
| 10 索引设计规范与三星索引 | ★★★★☆ | 场景题 / SQL 手写 |

## 1. 逻辑架构、SQL 执行流程与连接管理

### 1.1 核心原理

四层：**连接层**（TCP 握手、认证、每连接一线程）→ **服务层**（解析器→预处理器→优化器 CBO→执行器，外加权限、binlog、存储过程、视图、触发器）→ **引擎层**（可插拔，5.5+ 默认 InnoDB）→ **存储层**（`ibdata1`、`*.ibd`、redo、binlog）。

`select name from t where id=10` 路径：认证 → 词法/语法分析出 parse tree → 预处理校验表列存在 → 优化器按统计信息选索引与 join 顺序 → 执行器校验权限后**逐行**调用引擎接口（`handler::index_read`）→ Buffer Pool 命中直接返回、未命中读 16KB 页 → 服务层过滤后逐行返回。

连接管理：`thread_handling=one-thread-per-connection`（默认），`thread_cache_size` 默认 9；`max_connections` 5.7/8.0 默认 **151**，其中 1 个留给 SUPER。单连接内存 ≈ `sort_buffer_size`(256KB) + `join_buffer_size`(256KB) + `read_buffer_size` + `binlog_cache_size`，**连接数 × 单连接内存**是高并发内存雪崩主因。认证插件：8.0 默认 `caching_sha2_password`，5.7 为 `mysql_native_password`，老客户端连 8.0 报 `Authentication plugin 'caching_sha2_password' cannot be loaded`。

### 1.2 面试问答

- **Q1：服务层与引擎层各负责什么？** 服务层管与存储无关的一切（解析、优化、权限、binlog、结果集拼装）；引擎层只提供"按主键/索引取行"接口。换引擎不改 SQL，但事务、锁、索引实现全变；`select 1` 不进引擎层。
- **Q2：优化器为什么选错索引？** CBO 代价来自统计信息（`innodb_stats_persistent=ON` 时取自 `mysql.innodb_index_stats`，采样 `innodb_stats_persistent_sample_pages` 默认 20 页）。大量写入后未 `ANALYZE TABLE`、基数被低估即会误判；二级索引回表代价估算对聚簇度也不敏感。解法：`ANALYZE TABLE`、`FORCE INDEX`、`OPTIMIZER_TRACE` 查代价明细。
- **Q3：一条 update 与 select 的流程差异？** 多了写路径：读页入 Buffer Pool → 写 undo → 改内存页（脏页）→ 写 redo 到 Log Buffer → 提交时按 `innodb_flush_log_at_trx_commit` 刷 redo、同时写 binlog（**两阶段提交**）→ 后台 page cleaner 择机刷脏页到 `.ibd`。
- **Q4：Too many connections 怎么排查？** `Threads_connected`/`Max_used_connections` 看现状与峰值；`information_schema.processlist` 看占用方（`Sleep` 多为连接池泄漏，`Sending data`／`Waiting for table metadata lock` 是慢 SQL 或 DDL 阻塞）；治标调大 `max_connections`、调小 `wait_timeout`；治本查连接池与慢 SQL。
- **Q5：wait_timeout 与连接池的关系？** `wait_timeout` 默认 28800s，服务端会主动断开空闲连接。若它小于池的空闲回收周期，池中会留下服务端已关闭的死连接，首次使用报 `Communications link failure`。做法：HikariCP `maxLifetime` 明显小于 `wait_timeout`（差值 ≥ 30s）并开 keepalive。

### 1.3 高频追问与陷阱

- 陷阱：把 `Handler_read_*` 当索引使用情况。`Handler_read_key` 是走索引定位，`Handler_read_next` 在索引扫描与全表扫描中都会累加。
- 陷阱：只调大 `max_connections` 不同步调应用池，只会把数据库拖垮得更彻底；`processlist` 的 `Time` 在 `Sleep` 状态表示空闲时长，不代表慢。

### 1.4 代码/配置/命令示例

```sql
SET optimizer_trace='enabled=on';
SELECT * FROM t WHERE a=1 AND b>2;
SELECT TRACE FROM information_schema.OPTIMIZER_TRACE\G   -- rows_estimation / chosen
SET optimizer_trace='enabled=off';
ANALYZE TABLE t;
SELECT id,user,host,command,time,state,LEFT(info,80) FROM information_schema.processlist ORDER BY time DESC LIMIT 20;
```

### 1.5 一句话速记

**服务层管逻辑（解析/优化/binlog），引擎层管取行（索引/事务/锁），执行器逐行拉取；连接池必须做背压。**

## 2. InnoDB 内存结构

### 2.1 核心原理

**Buffer Pool**：默认 128MB（`innodb_buffer_pool_size`），生产取物理内存 50%~75%，以 16KB 页缓存数据页与索引页。`innodb_buffer_pool_instances` 默认 8（仅 pool ≥ 1GB 生效），每实例独立 LRU 与锁。

- **改进 LRU**：young(5/8) + old(3/8)，`innodb_old_blocks_pct` 默认 37。新页插 old 头部，须在 old 段停留超过 `innodb_old_blocks_time`（默认 **1000ms**）后再次被访问才升级到 young，防止大范围扫描冲掉热数据。
- **预读**：线性预读 `innodb_read_ahead_threshold` 默认 56 页 + 随机预读。
- **Change Buffer**：缓存**非唯一二级索引**的写（唯一索引必须读盘判重故不适用）；`innodb_change_buffer_max_size` 默认 25%；5.5 起可持久化。
- **Log Buffer**：`innodb_log_buffer_size` 默认 16MB，未提交事务的 redo 先写此。
- **AHI**：`innodb_adaptive_hash_index` 默认 ON，为热点等值查询在内存建哈希；`innodb_adaptive_hash_index_parts` 默认 8。争议：高并发等值查询下 `btr_search_latch` 可能成瓶颈，部分大厂实践中会关闭。

### 2.2 面试问答

- **Q1：为什么不用朴素 LRU？** 朴素 LRU 不区分"只读一次的扫描页"与"真热页"，一次 `select * from big_table` 就会挤掉整池热数据，命中率断崖。改进版加两道闸门：先进 old 段 + 停留 ≥1000ms 才升级。
- **Q2：Change Buffer 为什么只对非唯一二级索引生效？** 唯一索引的插入/更新必须读真实索引页判重，这次读逃不掉，缓存写就失去意义；非唯一索引可先把"在页 P 插入记录 R"记入 change buffer，等该页被读入时再 merge。
- **Q3：怎么判断 Buffer Pool 够不够？** 命中率 `1 - Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests`，OLTP 要求 > 99%；`Innodb_buffer_pool_wait_free > 0` 说明空闲页不足、刷脏跟不上。
- **Q4：脏页什么时候刷？** ① redo 空间不足触发 checkpoint 推进（最凶，会造成卡顿）；② 脏页比例超过 `innodb_max_dirty_pages_pct`（**8.0 默认 90，5.7 为 75**）；③ 后台 page cleaner 空闲刷。
- **Q5：`innodb_buffer_pool_instances` 越大越好吗？** 不是。实例过多会稀释每实例容量、削弱预读与 LRU 的全局效果；官方建议单实例 ≥ 1GB 才拆分，通常 4~8 个。

### 2.3 高频追问与陷阱

- 陷阱：以为 `innodb_buffer_pool_size` 能随时缩小。8.0 支持在线调整，但缩容要分块清理 LRU 并等待，可能造成秒级卡顿，务必低峰执行。
- 陷阱：`innodb_old_blocks_time=0` 等于关掉防污染机制。

### 2.4 代码/配置/命令示例

```sql
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';
SHOW STATUS LIKE 'Innodb_buffer_pool_wait_free';
SELECT POOL_ID,POOL_SIZE,FREE_BUFFERS,DATABASE_PAGES,MODIFIED_DATABASE_PAGES FROM information_schema.INNODB_BUFFER_POOL_STATS;
-- 生产配置示例
-- innodb_buffer_pool_size = 32G
-- innodb_buffer_pool_instances = 8
-- innodb_old_blocks_time = 1000
-- innodb_change_buffer_max_size = 25
-- innodb_log_buffer_size = 64M
```

### 2.5 一句话速记

**Buffer Pool 是内存页缓存，改进 LRU 靠 old 段 + 1000ms 闸门防污染；Change Buffer 只服务非唯一二级索引。**

## 3. 磁盘结构、行格式与 redo/undo/binlog

### 3.1 核心原理

层级：**表空间 → 段 → 区 → 页 → 行**。页默认 16KB；区 1MB = 64 页；一个索引由叶子段 + 非叶子段组成；`innodb_file_per_table` 自 5.6.6 默认 ON。隐藏列：`DB_ROW_ID`（无主键且无唯一非空索引时生成）、`DB_TRX_ID`(6B)、`DB_ROLL_PTR`(7B)。行内上限约 `innodb_page_size/2` ≈ 8126 字节，超出部分放溢出页。

| 行格式 | 大字段处理 | 适用 |
| --- | --- | --- |
| COMPACT（5.6 前默认） | 页内保留 **768 字节前缀** + 20 字节指针 | 兼容老版本 |
| **DYNAMIC（5.6.6+ 默认）** | 只存 **20 字节指针**，其余全在溢出页 | 含 TEXT/BLOB/长 VARCHAR 的表首选 |
| COMPRESSED | 同 DYNAMIC + 页压缩 | 读写比低、CPU 富余 |
| REDUNDANT | 最老，无变长长度列表 | 仅 5.0 前遗留（以官方文档为准） |

| 日志 | 层级 | 内容 | 作用 | 写盘 |
| --- | --- | --- | --- | --- |
| redo log | InnoDB | 物理逻辑日志（页号+偏移+修改） | 崩溃恢复 + WAL | 循环写；5.7 `innodb_log_file_size` 默认 48MB×2；8.0.30+ 用 `innodb_redo_log_capacity` 默认 100MB |
| undo log | InnoDB | 逻辑反向操作 | 事务回滚 + MVCC 版本链 | 落 undo 页，purge 线程清理 |
| binlog | Server 层 | 逻辑日志（statement/row/mixed） | 主从复制 + 归档 + PITR | 提交时写，`sync_binlog` 控 fsync |

**两阶段提交**：① InnoDB 写 redo、状态 `prepare`（带 XID）→ ② Server 层写 binlog（带 XID）→ ③ redo 改 `commit`。**崩溃恢复判定**：redo 已 `commit` 直接提交；redo 为 `prepare` 则用 XID 查 binlog，**binlog 完整就提交，否则回滚**。这是 2PC 的唯一理由：让 redo 与 binlog 对"事务是否生效"给出一致答案。

### 3.2 面试问答

- **Q1：redo log 与 binlog 的区别？** ① 层级：redo 在引擎层、binlog 在 Server 层；② 内容：redo 是页级物理逻辑日志，binlog 是逻辑日志（SQL 或行镜像）；③ 用途：redo 供崩溃恢复且循环覆盖，binlog 供复制归档且追加不覆盖；④ 时机：redo 边执行边写，binlog 提交前一次性写。
- **Q2：为什么必须两阶段提交？** 先 binlog 后 redo：binlog 已发从库但主库回滚 → 从库多数据。先 redo(commit) 后 binlog：主库有数据但 binlog 缺失 → 从库/备份少数据。2PC 让崩溃后可用 binlog 判定并补齐状态。
- **Q3：`innodb_flush_log_at_trx_commit` 与 `sync_binlog` 怎么配？** `1+1` 最安全不丢事务（**8.0 默认 `sync_binlog=1`，5.7 默认 0 需手动改**）；`1+0` redo 安全但 binlog 可能丢、主从不一致；`2+1` 主机崩溃不丢、掉电可能丢；`0+0` 最快但掉电丢最近约 1 秒（`innodb_flush_log_at_timeout` 默认 1s）。金融用 `1+1`，日志类可 `2+0`。
- **Q4：redo 循环写为什么不会覆盖还需要的数据？** checkpoint（`Innodb_checkpoint_lsn`）之前的 redo 对应脏页已刷盘，可被覆盖；写入 LSN 追近 checkpoint 会触发强制刷脏——这也解释了 redo 太小会导致频繁刷脏与性能抖动。
- **Q5：undo 什么时候被删？** purge 线程确认无活跃 ReadView 再需要该版本后清理。**长事务让 ReadView 长期存活、undo 无法回收**，history list length 膨胀，是典型线上隐患（详见 [./02-MySQL事务锁与MVCC.md](./02-MySQL事务锁与MVCC.md)）。

### 3.3 高频追问与陷阱

- 陷阱：把 `innodb_flush_log_at_trx_commit=2` 说成"完全不丢"。准确表述：**进程崩溃不丢，主机掉电会丢**（redo 只在 OS 缓存）。
- 陷阱：`sync_binlog=N` 是"每 N 次事务提交 fsync 一次"，值越大丢得越多，不是越安全；另外"区固定 1MB"也不准确，`innodb_page_size` 非 16KB 时区的页数随之变化，以官方文档为准。

### 3.4 代码/配置/命令示例

```sql
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
SHOW VARIABLES LIKE 'sync_binlog';
SHOW VARIABLES LIKE 'innodb_redo_log_capacity';   -- 8.0.30+
SHOW VARIABLES LIKE 'innodb_log_file_size';       -- 5.7 / 8.0.30 前
SHOW VARIABLES LIKE 'innodb_default_row_format';
ALTER TABLE t ROW_FORMAT=DYNAMIC;
SHOW BINARY LOGS;
-- 解析 binlog（bash）: mysqlbinlog --base64-output=decode-rows -v binlog.000003 | head -50
```

### 3.5 一句话速记

**表空间>段>区(1MB=64页)>页(16KB)>行；DYNAMIC 只留 20 字节溢出指针；redo 保不丢、undo 保回滚与 MVCC、binlog 保复制，2PC 让三者一致。**

## 4. B+ 树原理与容量估算

### 4.1 核心原理

| 结构 | 作为磁盘索引的问题 |
| --- | --- |
| 哈希 | 只支持等值，不支持范围/排序，冲突退化 |
| 红黑树 | 千万行约 24 层，每层一次随机磁盘 IO |
| 跳表 | 内存友好（Redis 采用），但高度随机、磁盘 IO 次数不稳定，页内无法紧凑存储 |
| B 树 | 非叶子也存数据，扇出小 → 树更高；范围查询需反复中序回溯 |
| **B+ 树** | 非叶子只存键+指针（扇出大）；叶子为有序双向链表（范围/排序友好）；层数稳定 |

**容量估算（16KB 页、bigint 主键）**：非叶子节点键 8B + 页号 6B = 14B，每页约 **1170 个指针**。一行 1KB 时叶子每页约 16 行，三层 ≈ `1170 × 1170 × 16 ≈ 2189 万行`；一行 100B 时（每页 160 行）三层 ≈ **2.19 亿行**；两层 ≈ 1.8 万行。**报数字必须说明"每行多少字节"的假设**，否则能差两个数量级。主键等值查询的磁盘 IO 次数 ≈ 树高 − 1（根与中间层常驻 Buffer Pool）。

### 4.2 面试问答

- **Q1：为什么用 B+ 树而不是 B 树？** ① B 树非叶子也存数据，同页扇出更小 → 树更高 → IO 更多；② B+ 树叶子成有序链表，范围查询只需定位起点再顺序遍历，B 树要中序遍历反复上下；③ B+ 树所有查询路径长度一致，性能可预测。
- **Q2：为什么不用哈希索引？** 哈希不支持范围查询、排序、最左前缀，只适合纯等值。InnoDB 的 AHI 只是内存加速层，并非可选索引类型；只有 Memory 引擎有显式 HASH 索引。
- **Q3：三层 B+ 树能存多少数据？** 取决于行宽。bigint 主键、约 1170 指针/非叶子页：1KB/行约 **2200 万行**，100B/行约 **2 亿行**；回答必须带假设。
- **Q4：查询复杂度是 O(log n) 吗？** 更准确的表述是"**磁盘 IO 次数 = 树高**，页内二分查找"。面试官关心 IO 次数的量级（常数 3~4）而非复杂度符号。
- **Q5：为什么主键推荐自增？** 自增保证顺序插入，页分裂只发生在最右侧，避免随机插入带来的页分裂、空间碎片与随机 IO。UUID 主键无序，会让页填充率降到约 50%~70%，且 16 字节主键会复制进每个二级索引，树更高。

### 4.3 高频追问与陷阱

- 陷阱：断言 B+ 树一定三层。层数由行宽与主键宽度决定，宽表/大主键可能到四层。
- 陷阱：忽略页分裂代价。随机主键下每次分裂平均挪动半页数据并留下约 50% 空洞。

### 4.4 代码/配置/命令示例

```sql
SELECT TABLE_NAME,TABLE_ROWS,DATA_LENGTH,INDEX_LENGTH,AVG_ROW_LENGTH
FROM information_schema.TABLES
WHERE TABLE_SCHEMA='mydb' ORDER BY DATA_LENGTH DESC LIMIT 10;
```

### 4.5 一句话速记

**B+ 树 = 大扇出 + 叶子有序链表 + 层数稳定；1KB 行宽三层约 2200 万行，IO 次数 = 树高。**

## 5. 聚集索引、回表与覆盖索引

### 5.1 核心原理

**聚集索引**：表本身即按主键组织的 B+ 树，**叶子存整行**；无显式主键则选第一个唯一非空索引，再没有就生成 6 字节 `DB_ROW_ID`。**二级索引**：叶子存 **索引列 + 主键值**，查非索引列须按主键回聚集索引 → **回表**。**覆盖索引**：所需列全在索引中，`Extra: Using index`，无需回表。

| 方案（`select a,b from t where a=1`） | 步骤 | 代价 |
| --- | --- | --- |
| 仅 `idx_a(a)` | 二级索引取主键 → 逐个回表取 b | N 次随机 IO |
| `idx_a_b(a,b)` | 二级索引直接返回 a、b | 0 次回表 |

注意：主键会隐式附加到二级索引末尾，所以 `(a)` 实际是 `(a, id)`，`select id from t where a=1` 本身就是覆盖索引。

### 5.2 面试问答

- **Q1：什么是回表？怎么优化？** 二级索引不含所需列，必须按主键再查聚集索引。优化：① 建覆盖索引（代价是索引变大、写变慢）；② 少返回列，禁止 `select *`；③ **延迟关联**——先用覆盖索引取主键再 join 回原表，深分页首选，见 [./03-MySQL调优主从与分库分表.md](./03-MySQL调优主从与分库分表.md)。
- **Q2：二级索引为什么存主键而不是行地址？** 行会因页分裂与变长 `update` 移动，行地址不稳定、维护成本极高；主键稳定。代价是回表。推论：**主键不能长**，它会复制进每个二级索引。
- **Q3：`Using index` 和 `Using index condition` 的区别？** `Using index` = 覆盖索引，不需要回表；`Using index condition` = ICP 生效，条件下推到引擎层但**仍然回表**。前者是质变，后者只是量变。
- **Q4：主键用 UUID 还是自增？** 自增或单调有序 ID（雪花）。UUID 三个问题：无序插入导致页分裂与约 50%~70% 填充率；主键过长使所有二级索引膨胀；字符串比较更慢。必须用时存 `binary(16)`，并用 `uuid_to_bin(uuid(),1)` 做时间高位重排（8.0）。

### 5.3 高频追问与陷阱

- 陷阱：以为"建了索引就不回表"，只有覆盖索引才不回表。
- 陷阱：为覆盖索引把十几个列塞进去，结果索引比表还大、写放大严重——按查询频率取舍。

### 5.4 代码/配置/命令示例

```sql
EXPLAIN SELECT a,b FROM t WHERE a=1;                 -- 回表（Extra 无 Using index）
ALTER TABLE t ADD INDEX idx_a_b (a,b);
EXPLAIN SELECT a,b FROM t WHERE a=1;                 -- Extra: Using index
EXPLAIN SELECT * FROM t WHERE a=1 AND b LIKE 'x%';   -- Extra: Using index condition
```

### 5.5 一句话速记

**聚集索引叶子存整行，二级索引叶子存主键（+索引列）；能覆盖就不回表，回表多就延迟关联。**

## 6. 索引类型、最左前缀与联合索引设计

### 6.1 核心原理

| 索引类型 | 语法 | 要点 |
| --- | --- | --- |
| 主键索引 | `PRIMARY KEY` | 聚集索引，唯一且非空 |
| 唯一索引 | `UNIQUE KEY` | 允许多个 NULL（互不冲突）；查询可达 `eq_ref` |
| 联合索引 | `KEY idx(a,b,c)` | 遵循最左前缀 |
| 前缀索引 | `KEY idx(a(10))` | 省空间；**不能覆盖索引、不能用于 order by** |
| 函数索引（8.0.13+） | `KEY idx((LOWER(name)))` | 表达式须加括号；5.7 只能加冗余列 |
| 降序索引（8.0+） | `KEY idx(a, b DESC)` | 5.7 语法接受但被忽略（仍按升序） |
| 全文索引 | `FULLTEXT KEY` | InnoDB 5.6+ 支持；中文需 ngram 解析器（5.7+） |
| 哈希索引 | 引擎相关 | InnoDB 无显式 HASH 索引（仅 AHI）；Memory 支持 |

**最左前缀**：`(a,b,c)` 支持的定位前缀为 `(a)`、`(a,b)`、`(a,b,c)`；中间断裂（`where a=1 and c=3`）时只有 `a` 可定位。**联合索引顺序三步法**：① 等值列在前、范围列在后（范围列之后的列无法缩小区间，仅可用于 ICP）；② 等值条件下区分度意义下降，**以凑出最长等值前缀为第一优先**；③ 多条 SQL 复用时取公共前缀的最大覆盖。

### 6.2 面试问答

- **Q1：`where a=1 and b>2 and c=3` 建 `(a,b,c)` 还是 `(a,c,b)`？** **`(a,c,b)`**。`b` 是范围列，排在其后的 `c` 无法用于索引定位、只能 ICP 过滤；把两个等值列 `a`、`c` 放前面才能形成最长等值前缀。
- **Q2：`(a,b,c)` 上 `where b=1 and c=2` 能用索引吗？** 5.7+ 可能选 `index`（全索引扫描）而非 `ALL`，Extra 出现 `Using where; Using index`，但本质是扫完整棵索引树，不比重建 `(b,c)` 好多少。缺最左列就是无法有效定位。
- **Q3：前缀索引长度怎么算？** `SELECT COUNT(DISTINCT LEFT(col,N))/COUNT(*) AS sel_n, COUNT(DISTINCT col)/COUNT(*) AS sel_full FROM t;` 从 N=4 递增，取 `sel_n` 逼近 `sel_full` 的最小 N（邮箱/手机号经验值 7~10），然后 `ALTER TABLE user ADD INDEX idx_email(email(8));`。
- **Q4：`order by` 必须与索引列顺序完全一致吗？** 不必。等值列顺序无关紧要：`where a=1 and b=2 order by c` 用 `(a,b,c)` 可免排序；但 `where b=1 order by a`（缺最左列 `a` 的等值条件）不行。
- **Q5：怎么发现冗余与无用索引？** `sys.schema_redundant_indexes`（前缀冗余，如 `(a)` 之于 `(a,b)`）与 `sys.schema_unused_indexes`（基于 `performance_schema.table_io_waits_summary_by_index_usage`）。注意使用统计自实例启动累计，**刚重启的实例数据不可信**。

### 6.3 高频追问与陷阱

- 陷阱：为 `order by` 单独再建一条索引。若排序键能作为现有联合索引的后缀（前面全为等值条件），复用即可。
- 陷阱：忘记唯一索引里的 NULL。MySQL 允许唯一索引存在多个 NULL，值为 NULL 的行不受唯一约束。

### 6.4 代码/配置/命令示例

```sql
ALTER TABLE user ADD INDEX idx_email(email(8));              -- 前缀索引
ALTER TABLE user ADD INDEX idx_lower_name ((LOWER(name)));    -- 8.0.13+ 函数索引
SELECT COUNT(DISTINCT city)/COUNT(*) AS city_sel,
       COUNT(DISTINCT order_no)/COUNT(*) AS no_sel FROM orders;
SELECT * FROM sys.schema_redundant_indexes WHERE table_schema='mydb';
SELECT * FROM sys.schema_unused_indexes   WHERE object_schema='mydb';
```

### 6.5 一句话速记

**最左前缀：等值在前、范围在后、区分度高的往左放；前缀索引省空间但换不来覆盖索引与排序。**

## 7. ICP 与索引失效场景大全

### 7.1 核心原理

**索引条件下推 ICP**（5.6+，`index_condition_pushdown` 默认 ON）：原本"只与索引列相关但无法用于区间定位"的条件要在服务层逐行判断、每行都要回表；ICP 把这些条件下推到引擎层，**先在索引上过滤再决定是否回表**，减少回表次数，`Extra: Using index condition`。

**索引失效判定本质**：优化器能否利用索引的有序性做区间定位。

| # | 场景 | 结论 | 说明 |
| --- | --- | --- | --- |
| 1 | `WHERE LEFT(name,3)='abc'`、`WHERE id+1=5` | **失效** | 索引列被函数包裹，有序性丢失 |
| 2 | `WHERE id='5'`（id 为 int） | **失效** | 隐式转换 = 对列做 `CAST`，等价于加函数 |
| 3 | `WHERE name=5`（name 为 varchar） | **不失效** | 转换在常量侧，列仍有序列 |
| 4 | `WHERE name LIKE '%abc'` | **失效** | 前导模糊无法定位；`'abc%'` 可用 |
| 5 | `WHERE a=1 OR b=2`（一支无索引 / 两支均有索引） | **失效** / 可能 index_merge | 一支无索引即整体全表；两支都有索引时 8.0 可做 union 合并但不保证被选中 |
| 6 | `WHERE a IS NULL` / `IS NOT NULL` | 可用 / 常全表 | 二级索引存 NULL 可定位，但区分度差时被弃用；`IS NOT NULL` 大表近似全命中 |
| 7 | `WHERE a!=1` / `a<>1` / `a NOT IN (1,2)` | 通常全表 | **代价弃用**而非"不能用" |
| 8 | `WHERE a=1 AND b>2 ORDER BY c`（索引 `(a,b,c)`） | 部分失效 | `b` 是范围列导致其后无序 → `Using filesort` |
| 9 | `WHERE a IN (1,2,3) ORDER BY b`（索引 `(a,b)`） | 8.0 可用 | `IN` 是多段等值，8.0 支持结果归并；5.7 常出现 filesort |
| 10 | 跳过最左列 `WHERE b=1`（索引 `(a,b)`） | 无法定位 | 退化为索引全扫 |
| 11 | 区分度极低（性别、状态位） | 优化器弃用 | 回表代价 > 全表代价 |
| 12 | 统计信息过期 | 可能误判 | `ANALYZE TABLE` 修复 |
| 13 | join 两侧字符集/排序规则不一致 | **失效** | 如 `utf8` join `utf8mb4`，列被隐式转换 |
| 14 | `WHERE a=1 AND b LIKE '%x%'`（索引 `(a,b)`） | `a` 可用 | `b` 交给 ICP 过滤 |

### 7.2 面试问答

- **Q1：`where id='5'`（int 主键）真的失效吗？** 真的。MySQL 比较时把字符串转数字，等价于 `CAST(id AS SIGNED)=5`，列被函数包裹 → 无法定位 → 全表扫描。这是最典型也最易被忽略的隐式转换事故，常见于把 varchar 参数直接拼进 SQL。
- **Q2：`where name=5`（varchar 列）为什么没事？** 转换方向在常量侧（`name = CAST(5 AS CHAR)`），索引列保持有序。但语义有差异（尾部空格处理），应用层仍应统一类型。
- **Q3：为什么范围列之后的列用不了索引？** 索引按 `(a,b)` 字典序整体排列，`a>1` 命中一个区间，区间内 `b` 在每个 `a` 值上重新从最小开始、全局不单调，无法再缩小区间。修正：把等值列放到范围列之前。
- **Q4：`or` 怎么优化？** 优先改 `union all` 并补去重条件，让两条各走自己的索引：`SELECT * FROM t WHERE a=1 UNION ALL SELECT * FROM t WHERE b=2 AND a<>1;`
- **Q5：怎么区分"索引失效"与"优化器弃用"？** 看 `possible_keys`。它列出候选索引而 `key` 选了别的或为空，属**代价弃用**（`!=`、低区分度）；`possible_keys` 本身为 NULL 才是真正无索引可用。前者可 `FORCE INDEX` 强制，但通常更慢。

### 7.3 高频追问与陷阱

- 陷阱：认为 `LIKE 'abc%'` 一定走索引。若匹配比例很高，优化器可能选全表或索引全扫，以 `EXPLAIN` 为准。
- 陷阱：把 `IS NOT NULL` 当可用索引条件，它在大表上近乎全命中，优化器基本不用索引；同时 ICP 不是"消除回表"，只是"减少回表次数"，`key_len` 也不包含 ICP 用到的列。

### 7.4 代码/配置/命令示例

```sql
-- 反例：函数包裹列
SELECT * FROM orders WHERE DATE(create_time)='2024-01-01';
-- 正例：改区间，可走 idx_create_time
SELECT * FROM orders WHERE create_time>='2024-01-01' AND create_time<'2024-01-02';
-- 反例：隐式转换（phone 为 varchar）
SELECT * FROM user WHERE phone=13800000000;
-- 正例
SELECT * FROM user WHERE phone='13800000000';
```

### 7.5 一句话速记

**失效三大主因：列被函数/隐式转换包裹、破坏最左前缀、范围列之后的列；`possible_keys` 为空才是真失效。**

## 8. EXPLAIN、optimizer trace 与慢查询

### 8.1 核心原理

`EXPLAIN` 输出 12 列：`id, select_type, table, partitions, type, possible_keys, key, key_len, ref, rows, filtered, Extra`。

**type 性能排序（好→坏）**：`system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL`。`const`=主键/唯一索引等值命中一行；`eq_ref`=join 被驱动表按唯一索引等值；`ref`=非唯一索引等值；`range`=索引范围；**`index`=全索引扫描**（仍需警惕）；`ALL`=全表。

**key_len 计算**：tinyint 1 / smallint 2 / int 4 / bigint 8；datetime 5（5.6+）、timestamp 4；`char(N)` = N×字符集字节；`varchar(N)` = N×字符集字节 + 2；**允许 NULL 再 +1**。例：`(a int not null, b varchar(10) not null)`，utf8mb4 下 `a=1 AND b='x'` → 4 + 10×4 + 2 = **46**。

**Extra 关键值**：`Using index`（覆盖索引不回表，好）；`Using where`（服务层过滤）；`Using index condition`（ICP，仍回表）；`Using filesort`（额外排序）；`Using temporary`（临时表，常见于 group by/distinct/union）；`Using join buffer (hash join)`（8.0.18+）；`Using join buffer (Block Nested Loop)`（5.7 / 8.0.20 前）；`Select tables optimized away`；`Using MRR`。

### 8.2 面试问答

- **Q1：EXPLAIN 的 rows 准确吗？** 是估算值，InnoDB 尤其不准（统计信息按 `innodb_stats_persistent_sample_pages` 默认 20 页采样）。要真实数据用 **8.0.18+ 的 `EXPLAIN ANALYZE`**，输出实际耗时与实际行数。
- **Q2：type=index 算好吗？** 不一定。`index` 是全索引扫描，若索引不含所需列则每行都要回表，代价可能接近全表扫描。只有 `index` 配合 `Using index`（覆盖索引全扫）才相对可接受。
- **Q3：怎么知道优化器为什么选了这个索引？** `OPTIMIZER_TRACE` 看 `rows_estimation`（各索引估算行数与代价）、`considered_execution_plans`、`chosen`；8.0 还可用 `EXPLAIN FORMAT=JSON` 看 `cost_info`（`read_cost`/`eval_cost`/`prefix_cost`）。
- **Q4：慢查询怎么定位到问题 SQL？** `slow_query_log=ON` + `long_query_time`（默认 10 秒，**生产建议 0.1~1 秒**）+ `log_queries_not_using_indexes`（慎开，会刷爆日志）；用 `pt-query-digest` 聚合；8.0 更推荐直接查 `performance_schema.events_statements_summary_by_digest`（按 digest 聚合，无需落盘）。
- **Q5：`key_len` 能判断 ICP 用了几列吗？** 不能。ICP 用到的列**不计入 key_len**，只体现在 `Extra: Using index condition`；`key_len` 只反映用于区间定位的列。

### 8.3 高频追问与陷阱

- 陷阱：只看 `EXPLAIN` 不看 `EXPLAIN ANALYZE`，估算值可能差几个数量级。
- 陷阱：忽略 `filtered` 极小的情况——`rows` 大且 `filtered` 极小说明扫描了大量无效行，是加索引的强信号；8.0 的 `EXPLAIN FORMAT=TREE` 可读性比传统表格更好，但很多人不会用。

### 8.4 代码/配置/命令示例

```sql
SET GLOBAL long_query_time=0.5;
EXPLAIN FORMAT=JSON SELECT * FROM t WHERE a=1\G
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id=100 AND status=1;   -- 8.0.18+
SELECT DIGEST_TEXT,COUNT_STAR,AVG_TIMER_WAIT/1e9 AS avg_ms,SUM_ROWS_EXAMINED
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
-- 聚合慢日志（bash）: pt-query-digest /var/log/mysql/slow.log > /tmp/report.txt
--                  : mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
```

### 8.5 一句话速记

**type 看访问方式，key_len 反推用了几列，Extra 看回表/排序/临时表；估算不准就上 EXPLAIN ANALYZE + OPTIMIZER_TRACE。**

## 9. Join 算法、排序与临时表

### 9.1 核心原理

| 算法 | 版本 | 原理 | 要点 |
| --- | --- | --- | --- |
| NLJ | 全版本 | 驱动表每行按索引查被驱动表 | 被驱动表 join 列**必须有索引** |
| BNL | 5.7 / 8.0.20 前 | 驱动表结果集放 `join_buffer_size`，被驱动表整表扫描逐块比较 | **8.0.20 起被移除** |
| Hash Join | **8.0.18+** | 小表建哈希表，大表探测 | 仅支持**等值** join |
| BKA | 5.6+ | 配合 MRR 批量回表 | 需 `optimizer_switch` 打开 `mrr` + `batched_key_access` |

**驱动表选择**：优化器按"过滤后参与 join 的行数"选小表做驱动表，即"小表驱动大表"；`STRAIGHT_JOIN` 可强制顺序（慎用）。**排序**：全字段排序（把 select 所有列放入 `sort_buffer_size`）；单行超过 `max_length_for_sort_data`（默认 1024 字节）退化为 **rowid 排序**（只放排序键+主键，排完回表取其余列）。**临时表**：`group by`、`distinct`、`union`、子查询物化会用到；内存临时表受 `tmp_table_size` 与 `max_heap_table_size` 中**较小值**限制（默认各 16MB），超限转磁盘临时表（8.0 用 InnoDB 临时表）。5.7 的 `group by` 会隐式排序，**8.0 已移除隐式排序**。

### 9.2 面试问答

- **Q1：为什么"小表驱动大表"？** NLJ 代价 = `驱动表行数 × 被驱动表单次索引查找`，驱动表行数是乘数；同时被驱动表反复访问，其索引页常驻 Buffer Pool，缓存友好。
- **Q2：8.0 的 Hash Join 解决什么问题？** 5.7 的 BNL 在无索引等值 join 时每条被驱动行都要与 buffer 全量比较；8.0.18+ 建哈希表后近似 O(1) 探测，并在 8.0.20 完全取代 BNL。但它是兜底方案，**仍应给 join 列建索引**，且**不支持非等值 join**。
- **Q3：为什么出现 Using filesort？** 返回顺序与索引顺序不一致。消除：让排序键作为索引后缀、前面全为等值条件。`where a=1 order by b` 用 `(a,b)` 可免排序；`where a>1 order by b` 不行。
- **Q4：group by 怎么优化？** ① 让分组列走索引（`(a,b)` 上 `group by a` 天然有序）；② 5.7 加 `order by null` 去掉隐式排序；③ 先 `where` 过滤降低参与分组的行数；④ 必要时调大 `tmp_table_size`，但优先从 SQL 层面消除临时表。
- **Q5：`join_buffer_size`、`sort_buffer_size` 能随便调大吗？** 不能。它们是**每连接**分配，N 个并发连接就是 N 倍，调大前先确认并发量，否则容易 OOM。

### 9.3 高频追问与陷阱

- 陷阱：把 `left join` 当 `inner join` 优化。`left join` 语义要求左表为驱动，但若被驱动表上有 `where` 条件过滤成非空，8.0 会自动转 `inner join`（`show warnings` 有提示）。
- 陷阱：小表驱动是"过滤后的行数"而非表总行数，统计信息不准时可能选反。

### 9.4 代码/配置/命令示例

```sql
EXPLAIN SELECT o.id,u.name FROM orders o JOIN user u ON o.user_id=u.id WHERE o.status=1;
EXPLAIN SELECT * FROM t1 JOIN t2 ON t1.a=t2.a;         -- 8.0.18+ 显示 hash join
SET optimizer_switch='mrr=on,batched_key_access=on';   -- 开启 BKA
```

### 9.5 一句话速记

**Join = 小表驱动大表 + 被驱动表 join 列必须有索引；8.0.18+ 用 Hash Join 兜底（仅等值），filesort 靠联合索引消除。**

## 10. 索引设计规范与三星索引

### 10.1 核心原理

**选择度** = `count(distinct col)/count(*)`，越接近 1 越好；小于 0.1（性别、状态位）单独建索引价值低，除非作为联合索引的组成部分。**三星索引**：一星 = `where` 中的等值列都在索引最左连续位置；二星 = `order by`/`group by` 顺序与索引顺序一致（消除 filesort）；三星 = `select` 的列都在索引中（消除回表）。三星是理想目标，实际要用**写放大与空间成本**权衡。

**索引规范清单**：① 主键用 `bigint` 自增或单调有序 ID，禁用 UUID/`char` 主键；② 单表索引数 ≤ 5，单索引列数 ≤ 5；③ 联合索引按"等值列 → 范围列 → 排序列"三段式；④ 禁止 `select *`；⑤ 不在大字段/长文本上直接建索引；⑥ 定期清理无用与冗余索引；⑦ 长字符串用前缀索引并验证选择度；⑧ 尽量 `NOT NULL` + 默认值（省 NULL 标记，优化器判断更准）；⑨ 建索引避开业务高峰，大表 DDL 方案见 [./03-MySQL调优主从与分库分表.md](./03-MySQL调优主从与分库分表.md)。

### 10.2 面试问答

- **Q1：给定一条 SQL，你的索引设计流程？** ① 分离 `where` 的等值列与范围列、`order by` 列、`select` 列；② 等值列组成最长前缀；③ 范围列紧随其后；④ 排序列放在范围列之后（范围列会打断排序可用性，必须实测）；⑤ `select` 列少时追加成覆盖索引；⑥ 用 `EXPLAIN` 验证 `type`/`key_len`/`Extra`；⑦ 评估写放大，能复用已有索引就不新建。
- **Q2：为什么索引不是越多越好？** ① 每个二级索引在写入时都要同步维护（页分裂、change buffer merge），写放大；② 索引占空间，与 Buffer Pool 争内存；③ 候选索引变多，优化器代价估算更易出错；④ 明显拖慢 `insert`/`update`。
- **Q3：怎么发现无用索引？** `sys.schema_unused_indexes`（基于 `performance_schema` 的索引使用统计）与 `sys.schema_redundant_indexes`。注意统计自实例启动累计，重启后要留观察期再判断。
- **Q4：三星索引是硬标准吗？** 不是。等值前缀、排序一致、覆盖三者常互相冲突（为覆盖加很多列会让索引膨胀、写变慢）。按查询频率取舍：高频核心 SQL 追求三星，低频 SQL 只求一星。
- **Q5：为什么建议列 `NOT NULL`？** ① 省 NULL 标记的额外字节与判断；② 语义清晰，`count`/`sum` 行为可预期；③ 优化器对 `NOT NULL` 列的 `IS NULL` 可直接判定恒假，减少无谓代价估算。

### 10.3 高频追问与陷阱

- 陷阱：把选择度绝对化。有等值条件时，低区分度列放进联合索引配合高区分度列，往往比单独建高区分度列更优（能形成更长等值前缀）。
- 陷阱：新建索引立即生效？Buffer Pool 冷、统计信息未更新时优化器可能仍不选，需要 `ANALYZE TABLE`。

### 10.4 代码/配置/命令示例

```sql
-- 目标 SQL: select id,amount,create_time from orders
--           where user_id=? and status=1 order by create_time desc limit 20;
ALTER TABLE orders ADD INDEX idx_uid_st_ct (user_id, status, create_time);
EXPLAIN SELECT id,amount,create_time FROM orders
WHERE user_id=100 AND status=1 ORDER BY create_time DESC LIMIT 20;
ANALYZE TABLE orders;
```

### 10.5 一句话速记

**索引设计 = 等值前缀 + 范围列 + 排序列 + 需要的覆盖列；选择度定顺序，写放大定数量。**

## 自测题（含答案要点）

1. **服务层与引擎层的分工？** —— 服务层解析/优化/权限/binlog；引擎层按索引取行。
2. **update 的写入路径？** —— 读页→写 undo→改 Buffer Pool→写 redo→2PC 刷盘→后台刷脏页。
3. **改进 LRU 的两个关键参数？** —— `innodb_old_blocks_pct`(37)、`innodb_old_blocks_time`(1000ms)。
4. **Change Buffer 为什么只服务非唯一二级索引？** —— 唯一索引必须读盘判重，缓存写无收益。
5. **页与区的大小？** —— 页 16KB，区 1MB = 64 页。
6. **DYNAMIC 与 COMPACT 在大字段上的差异？** —— COMPACT 页内保留 768 字节前缀，DYNAMIC 只留 20 字节指针。
7. **redo/undo/binlog 各自作用？** —— 崩溃恢复 / 回滚+MVCC / 复制归档。
8. **2PC 崩溃恢复的判定规则？** —— redo 为 prepare 且 binlog 完整则提交，否则回滚。
9. **`innodb_flush_log_at_trx_commit=2` 丢什么？** —— 掉电时最近约 1 秒事务（主机崩溃不丢）。
10. **三层 B+ 树能存多少行？** —— 1KB/行约 2200 万；100B/行约 2 亿（必须给假设）。
11. **B+ 树优于 B 树的理由？** —— 非叶子不存数据扇出大、叶子有序链表、层数稳定。
12. **回表与覆盖索引？** —— 二级索引叶子只存索引列+主键；`Using index` 表示无需回表。
13. **`(a,b,c)` 上 `where a=1 and c=3` 能用多少？** —— 只能用 `a` 定位，`c` 不可定位。
14. **`where a=1 and b>2 and c=3` 怎么建索引？** —— `(a,c,b)`，范围列放最后。
15. **`where id='5'`（int）为什么全表？** —— 隐式转换等价于给列加 `CAST` 函数。
16. **type 从好到坏顺序？8.0.18+ 用什么替代 BNL？** —— system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL；BNL 由 Hash Join 取代（仅等值）。
17. **`Using temporary` 常见于哪些语句？** —— group by、distinct、union、子查询物化；ICP 用到的列不计入 `key_len`。

## 背诵卡片

1. 四层架构：连接层 → 服务层（解析/优化/执行）→ 引擎层 → 存储层。
2. 优化器是 CBO，代价来自索引统计信息（采样默认 20 页）；执行器按行调用引擎接口。
3. 默认 one-thread-per-connection，`max_connections` 默认 151，`wait_timeout` 默认 28800s；8.0 默认认证插件 `caching_sha2_password`，5.7 为 `mysql_native_password`。
4. Buffer Pool 默认 128MB，生产设物理内存 50%~75%，`instances` 默认 8。
5. 改进 LRU：young 5/8 + old 3/8，old 停留 1000ms 才升级。
6. Change Buffer 只服务非唯一二级索引，默认上限 25% pool；Log Buffer 默认 16MB，AHI 默认 ON。
7. `innodb_max_dirty_pages_pct` 8.0 默认 90、5.7 默认 75。
8. 页 16KB，区 1MB = 64 页，索引段 = 叶子段 + 非叶子段。
9. `innodb_file_per_table` 自 5.6.6 默认 ON；行格式默认 DYNAMIC。
10. 隐藏列 `DB_ROW_ID` / `DB_TRX_ID`(6B) / `DB_ROLL_PTR`(7B)。
11. COMPACT 大字段页内留 768 字节，DYNAMIC 只留 20 字节指针；行内上限约 8126 字节。
12. redo 循环写；8.0.30+ 用 `innodb_redo_log_capacity` 默认 100MB，5.7 为 48MB × 2。
13. 2PC：redo prepare → binlog → redo commit；崩溃恢复看 binlog 是否完整。
14. `innodb_flush_log_at_trx_commit` 默认 1；`sync_binlog` 8.0 默认 1、5.7 默认 0；WAL 先写 redo 再刷数据页。
15. B+ 树非叶子只存键+指针，叶子是双向有序链表。
16. 非叶子约 1170 指针/页；1KB 行宽三层约 2200 万行。
17. 聚集索引叶子存整行，二级索引叶子存"索引列 + 主键"。
18. `Using index` = 覆盖索引不回表；`Using index condition` = ICP 仍回表。
19. ICP 自 5.6 默认 ON；ICP 用到的列不计入 `key_len`。
20. 最左前缀：等值列在前，范围列在后。
21. 前缀索引用 `COUNT(DISTINCT LEFT(col,N))/COUNT(*)` 定长度，且不能覆盖索引/排序。
22. 8.0.13+ 支持函数索引，8.0 支持降序索引。
23. `key_len` = 各列字节和，varchar 加 2、可空再加 1。
24. type 中 `index` 是全索引扫描，仍可能很慢。
25. `Using filesort` 靠"等值 + 排序"顺序一致的联合索引消除；`Using temporary` 常见于 group by/distinct/union。
26. join 原则：小表驱动大表，被驱动表 join 列建索引；8.0.18+ 引入 Hash Join，8.0.20 起移除 BNL。
27. `max_length_for_sort_data` 默认 1024；内存临时表受 `tmp_table_size` 与 `max_heap_table_size` 较小值限制（默认各 16MB）。
28. 8.0.18+ 用 `EXPLAIN ANALYZE` 拿真实耗时与行数。
29. 三星索引：等值前缀一星、排序一致二星、覆盖三星。
30. `sys.schema_unused_indexes` / `schema_redundant_indexes` 清理索引。
31. 主键禁用 UUID，随机插入会让页填充率降到约 50%~70%；慢查询 `long_query_time` 生产建议 0.1~1 秒，单表索引 ≤ 5 个。
