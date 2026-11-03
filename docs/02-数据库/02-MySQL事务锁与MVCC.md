# MySQL事务锁与MVCC

> **定位**：讲透 ACID 的实现机制、MVCC 可见性判断与 InnoDB 加锁规则，回答"为什么幻读、到底锁了什么、为什么死锁、为什么主从延迟"这类必考题。
> **重要度**：★★★★★（面试出现频率）
> **建议复习时长**：6 小时
> **关联文档**：[./01-MySQL体系结构与索引.md](./01-MySQL体系结构与索引.md)、[./03-MySQL调优主从与分库分表.md](./03-MySQL调优主从与分库分表.md)、[../05-分布式与架构/02-分布式事务.md](../05-分布式与架构/02-分布式事务.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| 1 ACID 与隔离级别 | ★★★★★ | 八股原理 |
| 2 MVCC：隐藏列、版本链与 ReadView | ★★★★★ | 八股原理 / 源码追问 |
| 3 RC 与 RR 差异、幻读与快照读/当前读 | ★★★★★ | 八股原理 / 场景题 |
| 4 锁体系总览：全局锁、表锁、MDL、意向锁、AUTO-INC | ★★★★★ | 八股原理 / 故障复盘 |
| 5 行锁三类型与加锁规则实战 | ★★★★★ | 场景题 / 故障复盘 |
| 6 死锁、锁等待与长事务治理 | ★★★★★ | 故障复盘 / 调优排查 |
| 7 binlog 三种格式与选型 | ★★★★☆ | 八股原理 / 场景题 |
| 8 主从复制原理与 GTID、半同步、并行复制 | ★★★★★ | 八股原理 / 场景题 |
| 9 主从延迟与读写分离一致性 | ★★★★★ | 场景题 / 故障复盘 |
| 10 一致性备份与锁相关参数 | ★★★★☆ | 调优排查 / 命令实操 |

## 1. ACID 与隔离级别

### 1.1 核心原理

ACID 与实现机制的对应：**原子性**靠 undo log（记录反向操作，回滚时逆序执行）；**持久性**靠 redo log（WAL，`innodb_flush_log_at_trx_commit` 控制刷盘）；**隔离性**靠 MVCC（读）+ 锁（写），隔离级别决定 ReadView 生成时机与是否加间隙锁；**一致性**是目的而非手段，由 A/I/D 加上业务约束（唯一键、外键、CHECK）共同保证，没有独立实现机制。

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 说明 |
| --- | --- | --- | --- | --- |
| READ UNCOMMITTED | ✗ | ✗ | ✗ | 能读未提交数据，几乎不用 |
| READ COMMITTED | ✓ | ✗ | ✗ | Oracle/PG 默认；每次 SELECT 新建 ReadView |
| **REPEATABLE READ** | ✓ | ✓ | ✓（InnoDB 基本避免） | **MySQL 默认**；快照读 + 间隙锁 |
| SERIALIZABLE | ✓ | ✓ | ✓ | 普通 SELECT 隐式加共享锁，完全串行 |

SQL 标准中 RR **允许**幻读，只有 SERIALIZABLE 才禁止；InnoDB 的 RR 靠"快照读用 MVCC + 当前读用 Next-Key Lock"在绝大多数场景消除幻读。

### 1.2 面试问答

- **Q1：ACID 分别靠什么实现？** 原子性靠 undo log，持久性靠 redo log，隔离性靠 MVCC + 锁；一致性是前三者共同作用的结果加业务约束。
- **Q2：MySQL 默认隔离级别是什么、为什么？** RR（变量 `transaction_isolation`，5.7 中名为 `tx_isolation`）。历史原因是 RR 在 STATEMENT 格式 binlog 下更安全；如今 binlog 默认 ROW，很多互联网公司主动改 RC 以降低间隙锁带来的冲突。
- **Q3：RC 与 RR 的核心差别？** 两点：① ReadView 生成时机（RC 每次 SELECT 新建，RR 事务首次 SELECT 建一次后复用）；② RR 会加间隙锁/临键锁，RC 基本只加记录锁，锁范围更小、并发更高。
- **Q4：不可重复读与幻读的区别？** 不可重复读针对**同一行被 UPDATE**，幻读针对**结果集行数变化（INSERT/DELETE）**，二者的加锁策略（记录锁 vs 间隙锁）也完全不同。

### 1.3 高频追问与陷阱

- 陷阱：认为 RR 完全解决幻读——准确说法是"快照读由 MVCC 避免、当前读由间隙锁避免"，两者混用时仍可能观察到幻读；另外 SERIALIZABLE 下读读不互斥、读写互斥，等价于串行执行。

### 1.4 代码/配置/命令示例

```sql
SELECT @@transaction_isolation;              -- 8.0；5.7 为 SELECT @@tx_isolation;
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET GLOBAL transaction_isolation='REPEATABLE-READ';
SHOW VARIABLES LIKE 'binlog_format';
```

### 1.5 一句话速记

**原子性=undo、持久性=redo、隔离性=MVCC+锁；默认 RR，互联网常改 RC 以减少间隙锁冲突。**

## 2. MVCC：隐藏列、版本链与 ReadView

### 2.1 核心原理

每行有两个隐藏列：`DB_TRX_ID`（6 字节，最后修改该行的事务 ID）、`DB_ROLL_PTR`（7 字节，指向 undo log 中的旧版本）；`DB_ROW_ID`（6 字节）仅在无主键且无唯一非空索引时生成。**版本链**：每次更新把旧值写入 undo log 并用 `DB_ROLL_PTR` 串成链表，链表头为最新版本；purge 线程在确认无 ReadView 需要旧版本后清理。

ReadView 四个字段：`m_ids`（生成瞬间**活跃未提交**事务 ID 列表）、`min_trx_id`（`m_ids` 最小值）、`max_trx_id`（**下一个待分配**的事务 ID）、`creator_trx_id`（创建本 ReadView 的事务 ID）。

**可见性判断算法**（对版本链上某版本的 `trx_id`）：

1. `trx_id == creator_trx_id` → **可见**（自己改的）。
2. `trx_id < min_trx_id` → **可见**（在 ReadView 生成前已提交）。
3. `trx_id >= max_trx_id` → **不可见**（在 ReadView 生成后才开启）。
4. `min_trx_id <= trx_id < max_trx_id`：在 `m_ids` 中 → **不可见**（仍活跃）；否则 → **可见**（已提交）。

不可见则沿 `DB_ROLL_PTR` 取下一个版本重新判断，直到链表尾（全部不可见则本行对该事务"不存在"）。

### 2.2 面试问答

- **Q1：MVCC 是什么、解决什么问题？** 多版本并发控制：读不阻塞写、写不阻塞读，通过 ReadView 判断该读哪个历史版本，实现"无锁读"。它**只作用于快照读**，当前读仍走锁。
- **Q2：ReadView 四个字段分别做什么？** `m_ids` 记录生成瞬间的活跃事务；`min_trx_id`/`max_trx_id` 划定"一定可见"与"一定不可见"的边界；`creator_trx_id` 用于识别自己产生的新版本（自己总能看见自己的修改）。
- **Q3：为什么版本链会越来越长？** 只要有长事务持有旧 ReadView，purge 就不敢清理它还需要的历史版本，undo 页持续增长；监控 `Innodb_history_list_length` 即可发现。
- **Q4：三种 DML 的 undo 分别是什么？RC 下为什么不可重复读？** INSERT 的 undo 是 DELETE，UPDATE 的 undo 是反向 UPDATE（记旧值），DELETE 的 undo 是 INSERT（按主键插回）；RC 下每次 SELECT 都新建 ReadView，第二次查询时前一阶段提交的事务已不在 `m_ids` 里，其新版本随即变为可见。

### 2.3 高频追问与陷阱

- 陷阱：以为 MVCC 等于"不加锁"——当前读（`FOR UPDATE`、`LOCK IN SHARE MODE`、`UPDATE`、`DELETE`）读的永远是最新版本并加锁，MVCC 只覆盖普通 `SELECT`；另外 `max_trx_id` 是**下一个待分配的事务 ID**，不是"最大的活跃事务 ID"。

### 2.4 代码/配置/命令示例

```sql
SHOW ENGINE INNODB STATUS\G                -- 看 History list length
SELECT trx_id,trx_mysql_thread_id,trx_state,trx_started,trx_rows_locked,trx_rows_modified
FROM information_schema.INNODB_TRX ORDER BY trx_started;
SELECT COUNT FROM information_schema.INNODB_METRICS WHERE NAME='trx_rseg_history_len';
```

### 2.5 一句话速记

**隐藏列 DB_TRX_ID/DB_ROLL_PTR 串出 undo 版本链，ReadView 用 m_ids/min/max/creator 四字段判定可见性；MVCC 只管快照读。**

## 3. RC 与 RR 差异、幻读与快照读/当前读

### 3.1 核心原理

| 对比项 | READ COMMITTED | REPEATABLE READ |
| --- | --- | --- |
| ReadView 生成时机 | **每次 SELECT 都新建** | **事务内首次 SELECT 时建一次，之后复用** |
| 能否重复读同一版本 / 间隙锁 | 不能；基本不加间隙锁（仅外键、唯一冲突检查等少数场景） | 能；**加 Gap/Next-Key Lock**，锁冲突与死锁概率更高 |
| binlog 兼容（历史原因） | 需 ROW | STATEMENT 也安全 |

**快照读 vs 当前读**：普通 `SELECT` 是快照读，读 ReadView 对应的历史版本、**不加锁**；`SELECT ... FOR UPDATE`、`LOCK IN SHARE MODE`（8.0 新语法 `FOR SHARE`）、`UPDATE`、`DELETE`、`INSERT` 是当前读，读**最新已提交版本并加锁**。

**RR 下幻读是否被解决，分三种情况**：① 全程快照读 → 不会幻读（同一快照）；② 全程当前读 → 不会幻读（Next-Key Lock 锁住间隙阻止插入）；③ **先快照读、后当前读** → 仍可能幻读：

```sql
BEGIN;
SELECT * FROM t WHERE id=5;            -- 快照读：无此行，不加锁
UPDATE t SET c=1 WHERE id=5;           -- 当前读：若别的事务已插入 id=5，此处会读到并加锁
SELECT * FROM t WHERE id=5;            -- 快照读：仍读旧快照 → 与 UPDATE 的结果不一致
COMMIT;
```

### 3.2 面试问答

- **Q1：为什么 RR 能"可重复读"？** 整个事务复用同一个 ReadView，期间其他事务提交的版本不在该 ReadView 的可见集合里，因此一直读到同一份快照。
- **Q2：RR 到底解决了幻读没有？** **快照读由 MVCC 避免，当前读由 Next-Key Lock 避免，但两者混用时（先快照读后当前读）仍可能观察到幻读**。所以严格说 InnoDB 的 RR 没有 100% 杜绝幻读。
- **Q3：间隙锁在这里起什么作用？** 锁住一个**开区间**，阻止其他事务在该区间内插入新记录，从而阻止当前读场景下"凭空多出几行"。这是 RR 默认开启、RC 基本不用的关键差异。
- **Q4：`FOR SHARE` 与 `FOR UPDATE` 的区别？** 前者加共享锁（S），多个事务可同时持有、彼此不阻塞，但会被 `FOR UPDATE` 的排他锁（X）阻塞；8.0 用 `FOR SHARE` 取代旧的 `LOCK IN SHARE MODE`（旧语法仍可用）。另外未提交事务的修改一定不可见——其 ID 必然在 ReadView 的 `m_ids` 中，于是沿版本链读更旧的版本，这就是避免脏读的机制。

### 3.3 高频追问与陷阱

- 陷阱：认为 RR 事务里所有读都一致。当前读永远读最新版本，"事务内数据不一致"在 RR 下同样可能发生；而 `START TRANSACTION WITH CONSISTENT SNAPSHOT` 可在 `BEGIN` 时立即建立一致性快照（RR 专用），`mysqldump --single-transaction` 的一致性就依赖这个语义。

### 3.4 代码/配置/命令示例

```sql
-- 会话 A                           -- 会话 B
BEGIN;
                                   BEGIN;
SELECT * FROM t WHERE id=5;        -- 空
                                   INSERT INTO t(id,c) VALUES(5,1); COMMIT;
SELECT * FROM t WHERE id=5;        -- RR 下仍为空（快照读）
SELECT * FROM t WHERE id=5 FOR UPDATE;   -- 当前读，能看到 id=5
COMMIT;
```

### 3.5 一句话速记

**RC 每次 SELECT 新建 ReadView、几乎不加间隙锁；RR 事务首次 SELECT 建一次并复用 + Next-Key Lock；快照读走 MVCC、当前读走锁。**

## 4. 锁体系总览：全局锁、表锁、MDL、意向锁、AUTO-INC

### 4.1 核心原理

| 层级 | 锁 | 加锁方式 | 要点 |
| --- | --- | --- | --- |
| 全局 | 全局读锁 | `FLUSH TABLES WITH READ LOCK`（FTWRL） | 全库只读，备份/切换用；InnoDB 备份改用 `--single-transaction` 避免 |
| 全局 | 全局写锁 | `SET GLOBAL read_only=ON` | 普通账号只读，对持有 SUPER 的账号无效 |
| 表级 | 表读/写锁 | `LOCK TABLES t READ/WRITE` | InnoDB 可用但严重限制并发，且会隐式提交当前事务 |
| 表级 | **MDL 元数据锁** | 自动加 | DML 加读锁、DDL 加写锁；**长事务 + DDL 的经典卡库来源** |
| 表级 | **意向锁 IS/IX** | 自动加 | 表级标志位表示"表中已有行锁"，**意向锁之间不冲突**，只与表锁冲突 |
| 行级 | `AUTO-INC` 锁 | 自动加 | 自增列分配用，粒度由 `innodb_autoinc_lock_mode` 决定 |

**MDL 经典死锁场景**：事务 A 持有表 t 的 MDL 读锁（普通 SELECT 后未提交）→ 事务 B 请求 MDL 写锁（`ALTER TABLE`）被阻塞 → 事务 C 请求 MDL 读锁，因**写锁优先级更高**也要排队 → 之后所有访问该表的请求全部堵死，表现为"数据库整体卡住"。规避：DDL 前先清理长事务，并显式调小 `lock_wait_timeout`（默认 31536000 秒 ≈ 1 年）。

**AUTO-INC 锁模式**：0 = traditional（语句级表锁，保证连续）；1 = consecutive（简单插入用轻量互斥量，批量插入仍用表锁，**5.7 默认**）；2 = interleaved（不保证自增值连续、并发最高，**8.0 默认**）。mode=2 在 `binlog_format=ROW` 下安全，若用 STATEMENT 会导致主从自增值不一致。

### 4.2 面试问答

- **Q1：意向锁有什么用？** 它是表级锁，表示"表里存在行锁"。没有它，事务想加表锁就得逐行扫描判断；有 IS/IX 只需检查表上的意向锁。**意向锁之间互不冲突**，IX 与表级 S 冲突、IS/IX 与表级 X 冲突。
- **Q2：MDL 什么时候加、什么时候释放？** 每条 DML 自动加 MDL 读锁、DDL 自动加 MDL 写锁；释放时机是**事务结束**而非语句结束。所以"开了事务没提交 + 另一会话执行 DDL"就是灾难现场。
- **Q3：为什么 `FLUSH TABLES WITH READ LOCK` 危险？** 它阻塞所有写操作。InnoDB 做一致性备份完全可以用 `mysqldump --single-transaction`（靠 MVCC 快照），无需全局锁；用 FTWRL 还容易因客户端断连忘记解锁。
- **Q4：`innodb_autoinc_lock_mode=2` 会导致主从不一致吗？MDL 等待怎么排查？** 只在 `binlog_format=STATEMENT` 时可能（自增值间隔不同）；ROW 格式记录实际行镜像，不依赖自增值连续性，因此安全，这也是 8.0 敢改默认值的前提。MDL 排查：`SHOW PROCESSLIST` 出现 `Waiting for table metadata lock` 即关键信号，8.0 查 `performance_schema.metadata_locks WHERE LOCK_STATUS='PENDING'` 或 `sys.schema_table_lock_waits`。

### 4.3 高频追问与陷阱

- 陷阱：以为 MDL 读锁之间会冲突——多个读锁可共存，问题在于写锁会插队并阻塞后续读锁；另外 `lock_wait_timeout` 默认 31536000 秒，DDL 遇到长事务会长时间挂住而不报错，必须显式调小。

### 4.4 代码/配置/命令示例

```sql
SELECT * FROM performance_schema.metadata_locks WHERE LOCK_STATUS='PENDING';
SELECT * FROM sys.schema_table_lock_waits;
SHOW VARIABLES LIKE 'lock_wait_timeout';
SHOW VARIABLES LIKE 'innodb_autoinc_lock_mode';
SET SESSION lock_wait_timeout = 30;   -- DDL 会话设合理上限
```

### 4.5 一句话速记

**全局锁慎用（InnoDB 备份走 MVCC 快照）；MDL 事务结束才释放、是 DDL 卡库元凶；意向锁是表级标志位不与行锁冲突；AUTO-INC 默认 8.0=2、5.7=1。**

## 5. 行锁三类型与加锁规则实战

### 5.1 核心原理

| 行锁类型 | 锁什么 | 何时使用 |
| --- | --- | --- |
| **Record Lock**（记录锁） | 单个索引记录 | 唯一索引等值**命中** |
| **Gap Lock**（间隙锁） | 开区间 `(a, b)`，不含两端记录 | 唯一索引等值**未命中**、RR 下插入前的间隙保护 |
| **Next-Key Lock**（临键锁） | 记录 + 前面的间隙，即左开右闭 `(a, b]` | 非唯一索引等值、范围查询（RR 默认） |

**插入意向锁**：一种特殊的**间隙锁**，插入前由插入事务设置，表示"我打算在这个间隙的这个位置插入"。**同一间隙的不同位置插入互不阻塞**（与普通间隙锁的关键区别），但要等该间隙上已有的间隙锁释放。

| # | 场景（RR，`FOR UPDATE`） | 锁了什么 |
| --- | --- | --- |
| 1 | 唯一索引等值**命中** `WHERE id=5` / **未命中**（无此行） | 命中：**Record Lock** on id=5（Next-Key 退化为记录锁）；未命中：**Gap Lock** `(上一个值, 下一个值)` |
| 2 | 非唯一索引等值 `WHERE a=5`（a 非唯一）：命中 / 未命中 | 命中：该记录的 **Next-Key Lock** + 对应行在**主键索引**上也加锁；未命中：Gap Lock |
| 3 | 范围查询：唯一索引 `id>5 AND id<=10` / 非唯一索引 `a>5 AND a<=10` | `(5, 10]` 上的 Next-Key Lock；非唯一索引还会多锁到下一个不满足条件的记录 |
| 4 | **无索引列** `WHERE c=5`（c 无索引） | **锁全表所有记录**（每条记录及其间隙都加 Next-Key Lock），等于锁表 |
| 5 | `LIMIT` 语句 `WHERE a>5 LIMIT 2` / `FOR SHARE`（8.0） | LIMIT 只锁扫描到的行；`FOR SHARE` 加共享 Next-Key Lock，**8.0 起对不匹配的行会释放锁**（5.7 不释放） |
| 6 | `INSERT` 撞上已有 Gap Lock | 被阻塞，等间隙锁释放（这是"无索引锁全表"导致插入排队的根因） |

**核心结论：行锁是加在"索引"上的，不是加在"行"上的。** 所以 `WHERE c=5`（无索引）会全表扫描并给每条记录加临键锁——这就是"没索引会锁全表"的根因，也是必须给 WHERE 列建索引的隐藏理由。

### 5.2 面试问答

- **Q1：为什么 `WHERE 无索引列 FOR UPDATE` 会锁全表？** 行锁加在索引记录上，无索引只能全表扫描，扫描路径上每条记录（及其间隙）都被加 Next-Key Lock，效果等同锁表；并发的 INSERT 会因插入意向锁与间隙锁冲突而排队。
- **Q2：唯一索引等值"命中"与"未命中"为什么加的锁不同？** 命中时唯一性保证最多一行，Next-Key Lock 退化为 Record Lock（无需锁间隙）；未命中时没有记录可锁，只能锁住这个"不存在的值"所在的间隙，防止其他事务插进来。
- **Q3：非唯一索引等值为什么要锁主键索引？** 查询可能要回表，为保证回表读到一致的行、也让 UPDATE 正确执行，InnoDB 在命中记录对应的主键索引项上也加锁（此处是 Record Lock）。
- **Q4：间隙锁之间会冲突吗？** **不会**。两个事务可在同一间隙上同时持有间隙锁（意图都是"防插入"），它们只与插入意向锁冲突——这解释了"间隙锁彼此相容但插入仍被阻塞"。RC 下不加间隙锁（少数场景除外），只有 Record Lock，且不满足条件的行上的锁会释放（semi-consistent read），并发度更高。

### 5.3 高频追问与陷阱

- 陷阱：以为"锁加在行上"。同一查询走不同索引，加锁范围完全不同——这是优化锁冲突最容易被忽视的杠杆。
- 陷阱：`FOR UPDATE` 的锁会持有到事务提交；把它放进长事务里必然造成大面积锁等待。

### 5.4 代码/配置/命令示例

```sql
BEGIN;
SELECT * FROM t WHERE c=1 FOR UPDATE;   -- c 无索引 → 全表记录被加临键锁，并发 INSERT/UPDATE 阻塞
SELECT ENGINE_TRANSACTION_ID,OBJECT_NAME,INDEX_NAME,LOCK_TYPE,LOCK_MODE,LOCK_DATA
FROM performance_schema.data_locks;     -- 8.0 观察当前锁
```

### 5.5 一句话速记

**行锁=记录锁/间隙锁/临键锁且加在索引上：唯一等值命中=记录锁、未命中=间隙锁、非唯一等值=临键锁、无索引=锁全表。**

## 6. 死锁、锁等待与长事务治理

### 6.1 核心原理

**死锁成因**：两个及以上事务形成循环等待。典型模式：事务 A 持 `id=1` 的 X 锁、事务 B 持 `id=2` 的 X 锁，随后 A 去锁 `id=2`、B 去锁 `id=1`，双方互等。其他成因：不同索引顺序加锁、唯一键冲突加间隙锁、批量插入顺序不一致、事务粒度过大。

**InnoDB 处理**：`innodb_deadlock_detect` 默认 ON，用**等待图**主动检测环路，发现后回滚**代价最小**（通常修改行数最少）的事务并返回 `Deadlock found ... try restarting transaction`；关闭检测则只能靠 `innodb_lock_wait_timeout`（默认 **50 秒**）超时兜底，生产不建议关闭。注意区分：**锁等待超时是单向等待、无环路**，报错为 `Lock wait timeout exceeded`。

**`SHOW ENGINE INNODB STATUS` 的 `LATEST DETECTED DEADLOCK` 读法**：

| 片段 | 含义 |
| --- | --- |
| `*** (1) TRANSACTION:` | 参与死锁的事务 1 |
| `TRANSACTION ... ACTIVE 5 sec` / `LOCK WAIT 3 lock struct(s)` | 活跃/等待时长（长事务信号）、当前在等锁且持有 3 个锁结构 |
| `MySQL thread id 12, query id 345 ... updating` | 会话线程 ID（可 `kill`） |
| `RECORD LOCKS ... index PRIMARY` / `lock_mode X locks rec but not gap waiting` | **锁在哪个索引上**（定位关键）；`rec but not gap` = 记录锁，含 gap 即临键锁 |
| `HOLDS THE LOCK(S)` / `WAITING FOR THIS LOCK TO BE GRANTED` / `WE ROLL BACK TRANSACTION (2)` | 对照持有与等待的锁即死锁根因；最后一行是 InnoDB 选择回滚的事务 |

**长事务的四重危害**：① purge 不敢清理 undo，`History list length` 持续增长、undo 表空间膨胀；② 行锁与 MDL 长期不释放，阻塞 DML 甚至 DDL（MDL 排队导致整表不可访问）；③ 回滚代价极高（逆序执行大量 undo）；④ 大事务在从库原子重放，造成主从延迟。监控用 `INNODB_TRX` 的 `trx_started`、`trx_rows_locked` 与 `Innodb_history_list_length`；应用侧禁止在事务内做 RPC/HTTP/MQ/循环批处理。

### 6.2 面试问答

- **Q1：死锁怎么产生的？** "加锁顺序不一致 + 持有并等待"即可形成循环等待，与隔离级别关系不大，但 RR 因间隙锁范围大而更易发生。
- **Q2：InnoDB 怎么处理死锁？** 默认开启等待图检测，立刻回滚代价最小的事务；业务侧应**捕获该异常并重试**，而不是让请求失败或崩溃。
- **Q3：怎么定位死锁？** ① 错误日志或 `SHOW ENGINE INNODB STATUS` 的 `LATEST DETECTED DEADLOCK`；② `innodb_print_all_deadlocks=ON` 记录全部（默认只记最近一次）；③ 8.0 实时看 `performance_schema.data_locks`/`data_lock_waits`（5.7 的 `information_schema.innodb_locks` 在 8.0 已移除）。
- **Q4：怎么规避死锁？** 统一加锁顺序（按主键升序批量更新）；缩小事务、尽快提交；建合适索引缩小加锁范围；拆大事务错峰执行；热点行改乐观锁或排队串行化；必要时降级到 RC 减少间隙锁。
- **Q5：长事务怎么发现与治理？** `SELECT trx_id,trx_mysql_thread_id,trx_started,TIMESTAMPDIFF(SECOND,trx_started,NOW()) AS sec,trx_rows_locked FROM information_schema.INNODB_TRX ORDER BY trx_started;`——`sec` 大而 `trx_rows_locked` 小通常是应用忘了提交。治理：统一事务边界、设事务超时、禁止事务内远程调用、必要时 `KILL`。

### 6.3 高频追问与陷阱

- 陷阱：`Sleep` 状态的连接也可能是长事务——它只表示当前没在执行 SQL，判断要看 `INNODB_TRX`；`@Transactional` 的典型陷阱是事务内做远程调用、循环里用 `REQUIRES_NEW`、以及同类内部调用导致注解失效。
- 陷阱：`innodb_rollback_on_timeout` 默认 OFF，锁等待超时只回滚**当前语句**；而 `KILL` 连接后 InnoDB 还要回滚整个事务，大事务回滚可能耗时很久。

### 6.4 代码/配置/命令示例

```sql
SHOW ENGINE INNODB STATUS\G                        -- LATEST DETECTED DEADLOCK / TRANSACTIONS
SET GLOBAL innodb_print_all_deadlocks = ON;
SET GLOBAL innodb_lock_wait_timeout = 10;
SELECT * FROM sys.innodb_lock_waits;               -- 找 blocking_thread 后 KILL
SELECT COUNT FROM information_schema.INNODB_METRICS WHERE NAME='trx_rseg_history_len';
```

### 6.5 一句话速记

**死锁=循环等待，InnoDB 用等待图检测并回滚代价最小的事务；靠统一加锁顺序、缩小事务与锁范围规避；长事务用 INNODB_TRX 与 history list length 监控。**

## 7. binlog 三种格式与选型

### 7.1 核心原理

| 格式 | 记录内容 | 优点 | 缺点 |
| --- | --- | --- | --- |
| STATEMENT（SBR） | SQL 原文 | 日志小、可读性好 | 不确定性函数（`NOW()`/`RAND()`/`UUID()`、无序 `LIMIT`）导致主从不一致；RC 下更危险 |
| ROW（RBR） | 每行的前后镜像 | **最安全**，主从一致；支持 CDC 与行级审计/闪回 | 日志量大（批量 UPDATE 可放大数倍到数百倍） |
| MIXED（MBR） | 默认 SBR，遇到不确定语句切 RBR | 折中 | 判断规则复杂，实践很少使用 |

默认值：MySQL **5.7.7+ 与 8.0 默认 `binlog_format=ROW`**；`binlog_row_image` 默认 `FULL`（可设 `MINIMAL` 减小日志量，但部分 CDC 工具依赖 FULL）。选型结论：**生产一律 ROW**——复制安全 + 支持 CDC（canal/Debezium 订阅 binlog 刷新缓存）+ 支持闪回。代价是日志量大，用 `binlog_expire_logs_seconds`（8.0，默认 2592000 秒 = 30 天）/ `expire_logs_days`（5.7，默认 0 表示不自动删）及时清理并归档到对象存储。

### 7.2 面试问答

- **Q1：为什么 STATEMENT 会主从不一致？** 记录的是 SQL，重放时环境可能不同：`NOW()` 时刻不同、`UUID()`/`RAND()` 随机、无序 `LIMIT` 返回行不确定、`AUTO_INCREMENT` 分配不连续。ROW 记录实际行镜像，与重放环境无关。
- **Q2：ROW 格式日志太大怎么办？** ① `binlog_row_image=MINIMAL` 只记主键与变更列（注意 canal 等工具需 FULL）；② 拆大事务（大事务还导致从库延迟）；③ 合理设置过期时间及时清理；④ 归档到对象存储而非无限保留本地盘。
- **Q3：binlog 的写入链路？** 事务提交时先写 `binlog cache`（`binlog_cache_size` 默认 32KB，大事务溢出到临时文件），再按**组提交**三阶段 flush → sync → commit 落盘；`sync_binlog` 控制 fsync 频率。`Binlog_cache_disk_use` 持续增长说明缓存太小。
- **Q4：误删数据怎么恢复？** ROW 格式下用 `binlog2sql`、`myflash` 等工具把 ROW 事件反转成补偿语句（闪回），或用 `mysqlbinlog` 按时间点回放到误操作前。这也是必须选 ROW 的重要原因。
- **Q5：`binlog_cache_size` 是全局参数吗？** 它是**每个会话/每个事务**的缓存大小，设置过大在几千并发时会显著吃内存；监控 `Binlog_cache_disk_use` 与 `Binlog_cache_use` 的比例来调参。

### 7.3 高频追问与陷阱

- 陷阱：以为 MIXED 能自动规避所有不一致，它只对明确标记"不安全"的语句切 ROW。
- 陷阱：ROW 格式下 `mysqlbinlog` 直接输出不可读，必须加 `--base64-output=decode-rows -v`。

### 7.4 代码/配置/命令示例

```sql
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'binlog_row_image';
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';   -- 8.0；5.7 为 expire_logs_days
SHOW STATUS LIKE 'Binlog_cache_disk_use';
SHOW MASTER STATUS;                                 -- 8.0.22+ 亦可 SHOW BINARY LOG STATUS
-- 解析 binlog（bash）：mysqlbinlog --base64-output=decode-rows -v \
--   --start-datetime="2024-01-01 10:00:00" binlog.000003
```

### 7.5 一句话速记

**生产一律 ROW：复制安全、支持 CDC 与闪回；STATEMENT 有不确定性函数风险；MIXED 实践少用。**

## 8. 主从复制原理与 GTID、半同步、并行复制

### 8.1 核心原理

**异步复制流程**：主库事务提交写 binlog → **binlog dump 线程**（每个从库一个）把 binlog 推给从库 → 从库 **IO 线程**写入本地 **relay log** 并更新 `Read_Master_Log_Pos` → 从库 **SQL/worker 线程**重放 relay log 并更新 `Exec_Master_Log_Pos`。两个进度的差值就是延迟来源。

**GTID（5.6+）**：`gtid_mode=ON` 后每个事务有全局唯一编号 `source_id:transaction_id`，从库用 `gtid_executed` 记录已执行集合。好处：主从切换无需人工算位点（`MASTER_AUTO_POSITION=1` 自动补差异）、不会重复执行、可做会话级一致性等待。

**半同步复制（5.5+ 插件）**：主库提交后至少等一个从库确认收到 binlog 才返回成功。`rpl_semi_sync_master_wait_point` 两个取值：`AFTER_SYNC`（**5.7 默认**）先写 binlog、等 ACK、再提交，主库故障切换**不丢已提交事务**；`AFTER_COMMIT`（5.5/5.6 行为）先提交再等 ACK，主库崩溃可能丢数据。注意 `rpl_semi_sync_master_timeout`（默认 10000ms）超时会**退化为异步**，重新引入丢数据窗口。

**并行复制（从库 `slave_parallel_workers>0` 时启用）**：5.6 按库并行（`DATABASE`，单库热点无效）；**5.7 默认基于组提交**（`LOGICAL_CLOCK`，同组提交内事务可并行，单库也有效）；8.0 增加基于 `WRITESET` 的行级冲突检测，粒度最细。

### 8.2 面试问答

- **Q1：主从复制的完整流程？** 主库 binlog dump 线程读 binlog 推给从库；从库 IO 线程写 relay log；从库 SQL/worker 线程重放 relay log。三个线程（dump、IO、SQL）职责分明。
- **Q2：GTID 相比位点复制好在哪？** ① 切换时不需要人工计算位点；② 事务唯一标识天然避免重复执行；③ 结合 `WAIT_FOR_EXECUTED_GTID_SET` 可做"写后读一致"保证。
- **Q3：为什么推荐 `AFTER_SYNC`？** 它在引擎提交**前**等从库 ACK，因此主库 commit 成功即意味着从库已收到该事务，故障切换不丢数据；`AFTER_COMMIT` 在等待期间主库崩溃就会出现"客户端收到成功但新主没有该数据"。
- **Q4：并行复制的原理？** 5.6 按库并行；5.7 基于组提交的 LOGICAL_CLOCK，同一提交组内事务在主库无锁冲突故可并行重放；8.0 的 WRITESET 进一步按行级是否冲突判断，并行度更高。
- **Q5：半同步能保证强一致吗？主从切换怎么做到不丢数据？** 不能——它只保证"至少一个从库收到 binlog"，**不保证已重放**，主库返回成功时从库可能还没执行完，要读一致性仍需 GTID 等待或强制走主库。切换不丢数据靠：半同步 `AFTER_SYNC` 保证已提交事务至少落在一个从库 + GTID 自动补齐差异 + 切换前确认 `Executed_Gtid_Set` 已追平 + 配合 MHA/Orchestrator 自动化（见 [./03-MySQL调优主从与分库分表.md](./03-MySQL调优主从与分库分表.md)）。

### 8.3 高频追问与陷阱

- 陷阱：`slave_parallel_workers` 调大不等于更快。并行度受主库提交组大小与冲突比例限制，盲目调大可能加剧锁冲突。
- 陷阱：从库 `read_only=ON` 对复制线程无效（复制线程仍有写权限），且持有 SUPER 的账号可绕过。

### 8.4 代码/配置/命令示例

```sql
SHOW MASTER STATUS;                   -- 主库位点；8.0.22+ 亦可 SHOW BINARY LOG STATUS
SHOW SLAVE STATUS\G                   -- 8.0.22+ 亦可 SHOW REPLICA STATUS（见第 9 节三字段）
SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE 'slave_parallel_type';   -- DATABASE / LOGICAL_CLOCK / WRITESET
SHOW VARIABLES LIKE 'slave_parallel_workers';
SELECT WAIT_FOR_EXECUTED_GTID_SET('3f9c...:1-100', 1);   -- 写后读一致，超时 1 秒
```

### 8.5 一句话速记

**主库 dump 线程推 binlog → 从库 IO 线程写 relay log → SQL/worker 线程重放；GTID 免位点、AFTER_SYNC 半同步不丢已提交事务、LOGICAL_CLOCK 并行复制。**

## 9. 主从延迟与读写分离一致性

### 9.1 核心原理

| 延迟原因 | 说明 | 对策 |
| --- | --- | --- |
| 从库单线程重放（5.6 前） | 主库并发写、从库串行重放 | 升级 5.7/8.0 开并行复制 |
| **大事务** | 事务在从库必须整体原子重放，期间进度不动 | 拆小事务（批量更新按主键分批） |
| 从库读压力大 | 读查询抢走 CPU/IO | 读库扩容、加索引、限流 |
| 主库写入突增 | 写入速率持续超过重放速率 | 分库分表、升级从库配置、多从库分流 |
| 网络带宽/延迟 | 大 ROW 日志传输慢 | 提高带宽、`binlog_row_image=MINIMAL` |
| 从库执行大表 DDL | DDL 阻塞重放 | 用 gh-ost/pt-osc 并低峰执行 |

**读写分离一致性方案（强度递增）**：① 关键业务**强制走主库**（下单后查订单）；② **GTID 等待**：写完取 GTID，读前 `WAIT_FOR_EXECUTED_GTID_SET(gtid, timeout)`；③ 半同步 + 读写分离（保证从库收到、不保证重放）；④ **延迟感知路由**（中间件探测从库延迟，超阈值自动回主库）；⑤ 最终一致 + 业务补偿。

### 9.2 面试问答

- **Q1：延迟多少算正常？** 同机房并行复制正常时毫秒到几十毫秒，跨机房几十到几百毫秒；**秒级以上即异常**，需定位大事务、从库压力还是并行度不足。
- **Q2：怎么准确测量主从延迟？** 不要只看 `Seconds_Behind_Master`。用 `pt-heartbeat`（主库周期更新时间戳表，从库读表算差值）得到真实延迟；或结合 `Exec_Master_Log_Pos` 与 `Read_Master_Log_Pos` 差值估算。
- **Q3：`Seconds_Behind_Master` 为什么不准？** 它比较从库 SQL 线程时间与 IO 线程收到事件的时间戳：网络阻塞时会失真、大事务期间进度不动可能显示很大、`NULL` 表示复制线程未运行（不是延迟 0）。用 `pt-heartbeat` 才能拿到真实延迟。
- **Q4：写后立即读怎么保证一致性、大事务为什么特别伤主从？** 关键路径强制走主库、GTID 等待、中间件"写后读主"路由、缓存降低即时读依赖；大事务因为 binlog 事件在从库以事务为单位**原子重放**（并行复制下还会独占 worker），持续几十秒会让 `Exec_Master_Log_Pos` 长时间不动，延迟飙升且主库锁持有时间长。

### 9.3 高频追问与陷阱

- 陷阱：把 `Seconds_Behind_Master=NULL` 理解成"无延迟"，它其实表示 IO 或 SQL 线程已停止，复制中断了；另外为减小日志改 `binlog_row_image=MINIMAL` 前必须确认下游 CDC 工具依赖完整行镜像。

### 9.4 代码/配置/命令示例

```sql
SHOW SLAVE STATUS\G      -- 重点看 Slave_IO_Running / Slave_SQL_Running / Seconds_Behind_Master
                         -- 以及 Retrieved_Gtid_Set / Executed_Gtid_Set 的差值
SET GLOBAL rpl_semi_sync_master_timeout = 1000;
SHOW VARIABLES LIKE 'relay_log_recovery';   -- ON 可从库重启后自动从最新位点恢复
```

### 9.5 一句话速记

**延迟主因=大事务+单线程重放+从库压力；并行复制与拆分大事务治本，GTID 等待与强制走主库保证写后读一致。**

## 10. 一致性备份与锁相关参数

### 10.1 核心原理

**`mysqldump --single-transaction`**：对 InnoDB 表开启一个 RR 一致性快照事务，整个备份读的都是同一快照，**不阻塞写、无需全局锁**。要点：① 只对 **InnoDB** 有效（MyISAM 仍会被锁，且为取一致性位点会有短暂 FTWRL）；② 必须配合 **RR**（全局是 RC 时要在会话内指定）；③ 搭配 `--master-data=2`（5.7）/ `--source-data=2`（**8.0.26+ 改名**）把位点或 GTID 以注释写入备份文件便于 PITR；④ 备份期间禁止 DDL。物理备份用 XtraBackup / MySQL Enterprise Backup，热备且快，恢复需 `--prepare` 应用 redo。8.0 也推荐 `MySQL Shell` 的 `util.dumpSchemas`（并行、一致性快照）。

| 参数 | 默认值 | 用途 |
| --- | --- | --- |
| `innodb_lock_wait_timeout` | 50 秒 | 行锁等待超时 |
| `lock_wait_timeout` | 31536000 秒 | **MDL/表锁等待超时，必须显式调小** |
| `innodb_deadlock_detect` / `innodb_print_all_deadlocks` | ON / OFF | 死锁主动检测 / 记录所有死锁到错误日志 |
| `transaction_isolation`（5.7 为 `tx_isolation`） | REPEATABLE-READ | 隔离级别 |
| `innodb_rollback_on_timeout` | OFF | 超时只回滚当前语句 |

### 10.2 面试问答

- **Q1：`--single-transaction` 为什么能保证一致性？** 它借助 InnoDB 的 MVCC：备份会话的 RR 快照事务看到同一时刻的数据视图，期间其他事务的提交不影响它，因此无需全局读锁即可得到一致的数据集合。
- **Q2：它有什么局限？** ① 只对事务引擎有效；② 备份期间不能执行 DDL（会破坏一致性且 DDL 需要 MDL 写锁会被快照事务阻塞）；③ 大库导出时间可能很长；④ 逻辑备份恢复慢。
- **Q3：怎么用备份 + binlog 做 PITR？** 全量备份（含位点/GTID 注释）恢复到备份时刻 → 从备份位点开始用 `mysqlbinlog` 回放到目标时间（`--stop-datetime`）→ 若为误删则先闪回再回放。
- **Q4：为什么 `lock_wait_timeout` 默认这么离谱？备份期间执行了 DDL 会怎样？** 它是 5.5 引入 MDL 时定的一年（31536000 秒），意味着 DDL 遇到长事务会长时间挂住而不失败，生产必须显式调小（如 30~60 秒）；备份期间执行 DDL 会让备份文件里部分表是新结构、部分是旧结构，恢复后数据与结构对不上，通常直接判定备份不可用。

### 10.3 高频追问与陷阱

- 陷阱：以为 `--single-transaction` 完全不加锁——它仍会短暂 FTWRL 以取得一致性位点，MyISAM 表也会被锁；另外别把 `innodb_lock_wait_timeout`（行锁）与 `lock_wait_timeout`（MDL/表锁）混为一谈。

### 10.4 代码/配置/命令示例

```bash
# 逻辑备份：InnoDB 一致性快照 + 记录位点（8.0.26+ 用 --source-data=2）
mysqldump -uroot -p --single-transaction --source-data=2 --routines --triggers \
  --databases mydb > /backup/mydb_$(date +%F).sql
# 恢复 + 回放到指定时间点
mysql -uroot -p < /backup/mydb_2024-01-01.sql
mysqlbinlog --start-position=1234 --stop-datetime="2024-01-01 10:30:00" binlog.000005 | mysql -uroot -p
```

### 10.5 一句话速记

**`--single-transaction` 靠 MVCC 快照做无锁一致性备份（仅 InnoDB），配 `--source-data=2` 记位点做 PITR；`lock_wait_timeout` 必须调小。**

## 自测题（含答案要点）

1. **ACID 各靠什么实现？** —— undo / 业务约束 / MVCC+锁 / redo。
2. **MySQL 默认隔离级别？** —— REPEATABLE READ（8.0 变量 `transaction_isolation`，5.7 为 `tx_isolation`）。
3. **RC 与 RR 的两点核心差异？** —— ReadView 生成时机、是否加间隙锁。
4. **ReadView 四个字段？** —— `m_ids`、`min_trx_id`、`max_trx_id`、`creator_trx_id`。
5. **可见性判断四条规则？** —— creator 自己可见；小于 min 可见；大于等于 max 不可见；中间看是否在 m_ids。
6. **两个关键隐藏列与长度？** —— `DB_TRX_ID`(6B)、`DB_ROLL_PTR`(7B)。
7. **RR 完全解决幻读了吗？** —— 快照读靠 MVCC、当前读靠 Next-Key Lock，混用时仍可能幻读。
8. **快照读与当前读分别包括哪些语句？** —— 普通 SELECT / `FOR UPDATE`、`FOR SHARE`、UPDATE、DELETE、INSERT。
9. **MDL 锁何时释放？意向锁的作用？** —— 事务结束时（不是语句结束时）；意向锁是表级标志位表示表内已有行锁，避免加表锁时逐行扫描，意向锁之间不冲突。
10. **`innodb_autoinc_lock_mode` 默认值？binlog 默认格式与选型？** —— 8.0 为 2、5.7 为 1；binlog 自 5.7.7/8.0 默认 ROW，生产一律 ROW（复制安全 + CDC + 闪回）。
11. **唯一索引等值未命中加什么锁？无索引列 `FOR UPDATE` 为什么锁全表？** —— 间隙锁；行锁加在索引上，全表扫描导致每条记录及其间隙都被加临键锁。
12. **插入意向锁与间隙锁的关系？** —— 它是一种特殊间隙锁；同间隙不同位置插入可并发，但与已有间隙锁冲突。
13. **死锁如何处理与规避？长事务的四大危害？** —— 等待图检测后回滚代价最小的事务；统一加锁顺序、缩小事务与锁范围。长事务危害：undo 不回收、锁持有久、回滚极慢、主从延迟。
14. **主从复制的三个线程？半同步 `AFTER_SYNC` 与 `AFTER_COMMIT` 的区别？** —— 主库 binlog dump、从库 IO、从库 SQL/worker；前者提交前等 ACK（切换不丢已提交事务），后者先提交再等（可能丢）。
15. **`--single-transaction` 的原理与局限？** —— 靠 InnoDB RR 快照事务无锁备份；仅对事务引擎有效，备份期间禁止 DDL。

## 背诵卡片

1. 原子性=undo，持久性=redo，隔离性=MVCC+锁，一致性是前三者的结果；默认 RR，互联网常用 RC。
2. SQL 标准里 RR 允许幻读，只有 SERIALIZABLE 禁止；不可重复读看同一行被改，幻读看行数变化。
3. 隐藏列 `DB_TRX_ID`(6B) / `DB_ROLL_PTR`(7B)，无主键时才有 `DB_ROW_ID`。
4. ReadView 四字段：`m_ids` / `min_trx_id` / `max_trx_id` / `creator_trx_id`。
5. `max_trx_id` 是"下一个待分配的事务 ID"，不是最大活跃 ID。
6. 可见性：自己可见、小于 min 可见、大于等于 max 不可见、中间看 m_ids。
7. RC 每次 SELECT 新建 ReadView；RR 事务首次 SELECT 建一次并复用。
8. undo 类型：INSERT→DELETE，UPDATE→反向 UPDATE，DELETE→INSERT。
9. MVCC 只作用于快照读；当前读永远读最新并加锁。
10. RR 幻读：快照读避免了，当前读靠 Next-Key Lock 避免，混用仍可能幻读。
11. 间隙锁之间不冲突，只与插入意向锁冲突。
12. MDL 读锁在事务结束才释放，是 DDL 卡库的元凶。
13. `lock_wait_timeout` 默认 31536000 秒，必须显式调小。
14. 意向锁 IS/IX 是表级，彼此不冲突，只与表锁冲突；`FLUSH TABLES WITH READ LOCK` 是全库读锁，InnoDB 备份不需要它。
15. `innodb_autoinc_lock_mode`：8.0 默认 2（interleaved），5.7 默认 1（consecutive）。
16. 行锁加在索引上；无索引 `FOR UPDATE` = 锁全表。
17. 唯一索引等值命中=记录锁，未命中=间隙锁，非唯一索引等值=临键锁。
18. 临键锁 = 记录 + 前面的间隙，左开右闭；插入意向锁让同间隙不同位置的插入可并发。
19. `innodb_deadlock_detect` 默认 ON，回滚代价最小的事务。
20. `innodb_lock_wait_timeout` 默认 50 秒，`innodb_print_all_deadlocks` 默认 OFF。
21. 死锁规避：统一加锁顺序、缩小事务、用索引缩小锁范围、拆大事务。
22. 8.0 用 `performance_schema.data_locks`/`data_lock_waits`，`innodb_locks` 已移除。
23. 长事务监控：`INNODB_TRX` 的 `trx_started` 与 `Innodb_history_list_length`。
24. `innodb_rollback_on_timeout` 默认 OFF，超时只回滚当前语句。
25. binlog 三格式：STATEMENT 小但不确定、ROW 安全但大、MIXED 折中；组提交三阶段 flush → sync → commit。
26. 主从三线程：主库 dump、从库 IO、从库 SQL/worker。
27. GTID 让主从切换无需手算位点，`MASTER_AUTO_POSITION=1` 自动补齐。
28. 半同步 `AFTER_SYNC` 在提交前等 ACK，故障切换不丢已提交事务；并行复制 5.6 按库、5.7 基于组提交、8.0 基于 WRITESET。
29. `Seconds_Behind_Master` 不可靠（`NULL` 表示复制中断），用 `pt-heartbeat` 测真实延迟。
30. 读写分离一致性：强制走主库 / GTID 等待 / 半同步 / 延迟感知路由。
31. `mysqldump --single-transaction` 靠 MVCC 快照免锁（仅 InnoDB）；`--master-data=2`（5.7）在 8.0.26+ 改名 `--source-data=2`。
