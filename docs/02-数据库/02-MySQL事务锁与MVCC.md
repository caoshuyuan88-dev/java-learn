# MySQL事务锁与MVCC

> **定位**：理解事务隔离、MVCC、InnoDB 加锁，以及这些机制在复制和备份中的作用。
> **重要度**：★★★★★
> **建议复习时长**：6 小时
> **关联文档**：[MySQL体系结构与索引](./01-MySQL体系结构与索引.md)、[MySQL调优主从与分库分表](./03-MySQL调优主从与分库分表.md)、[分布式事务](../05-分布式与架构/02-分布式事务.md)

本文以 **MySQL 8.0、InnoDB、默认 REPEATABLE READ（RR）** 为主要讨论对象，特别说明时才涉及 5.7 或其他隔离级别。具体加锁范围、复制参数和命令会随版本、配置和执行计划变化，应以实际环境和对应版本手册为准。

> **实验约定**：SQL 分别在两个独立连接 A、B 中执行；按表格或注释顺序执行，不能把两个会话的语句当成一个脚本运行。实验结束后提交或回滚，避免留下锁。所有示例请在测试库运行。

## 0. 知识地图

```text
事务 ACID
├── 原子性：undo 支持回滚
├── 持久性：redo + 合适的刷盘策略支持崩溃恢复
├── 隔离性
│   ├── 一致性读：ReadView + undo 版本链（MVCC）
│   └── 加锁读/写：索引记录锁、间隙锁等
└── 一致性：约束 + 正确的事务与业务逻辑

事务提交 → binlog → 复制到从库 → 从库应用 → 读写分离
             └── 配合一致性备份 → 时间点恢复
```

| 章节 | 需要回答的问题 |
| --- | --- |
| 1 | ACID 是什么？隔离级别允许哪些并发异常？ |
| 2～3 | 旧版本从哪里来？RC 和 RR 为什么读到不同结果？ |
| 4～5 | 事务锁住了什么？为什么一个 DDL 或 INSERT 会等待？ |
| 6 | 如何区分、排查和处理死锁、锁等待与长事务？ |
| 7～9 | binlog 如何复制？为什么从库可能读不到刚写入的数据？ |
| 10 | 怎样获得一致的备份，并利用 binlog 恢复？ |

## 1. ACID 与隔离级别

### 1.1 事务是什么

事务把多条操作组成一个逻辑单元：成功时 `COMMIT`，需要放弃时 `ROLLBACK`。例如转账需要同时扣减 A 和增加 B；只完成一半不符合业务要求。

```sql
START TRANSACTION;
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- 发生错误时应由应用执行 ROLLBACK
```

这只是演示事务边界：实际转账还需校验账户存在、余额足够、更新行数及请求幂等。MySQL 默认 `autocommit=1`，没有显式开启事务时，一条成功的 DML 通常就是一个独立事务；有些 DDL 会隐式提交，不能简单地和普通 DML 一起回滚。

### 1.2 ACID 与实现

| 特性 | 含义 | InnoDB 的相关机制与边界 |
| --- | --- | --- |
| Atomicity 原子性 | 一个事务的修改整体提交或整体撤销 | undo 记录支持回滚；崩溃恢复也依赖日志和事务状态，不是只靠应用执行反向 SQL |
| Consistency 一致性 | 事务执行前后，数据满足业务与数据库约束 | 唯一键、外键、CHECK、正确的业务逻辑与其他 ACID 特性共同作用；数据库不能自动发现所有业务规则 |
| Isolation 隔离性 | 并发事务之间按约定的隔离级别观察数据 | 一致性读使用 MVCC；加锁读和写操作使用锁；具体行为受隔离级别影响 |
| Durability 持久性 | 已确认提交的数据在故障后仍可恢复 | redo 和 WAL（先写日志再刷数据页）用于崩溃恢复；实际保障还依赖刷盘配置、存储和故障范围 |

`innodb_flush_log_at_trx_commit=1` 通常表示每次提交将 redo 写入并刷到持久存储；其他值可以提高吞吐，但故障时可能丢失已确认事务。启用 binlog 时也要考虑 `sync_binlog`、存储可靠性及备份，不能把 redo 和 binlog 混为一谈：前者服务于 InnoDB 崩溃恢复，后者用于复制和基于日志的恢复。

### 1.3 并发异常与隔离级别

- **脏读**：事务 A 读到了事务 B 尚未提交的数据，B 随后可能回滚。
- **不可重复读**：A 两次读同一行，B 在两次读取之间修改并提交，A 读到不同值。
- **幻读**：A 两次按同一条件查询，B 在中间插入或删除符合条件的行，结果集发生变化。它侧重的是**符合条件的记录集合**；与不可重复读在广义描述上可能重叠。

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | InnoDB 中的典型行为 |
| --- | --- | --- | --- | --- |
| READ UNCOMMITTED（RU） | 可能 | 可能 | 可能 | 可以读到其他事务尚未提交的修改 |
| READ COMMITTED（RC） | 不允许 | 可能 | 可能 | 每次一致性读创建新视图；通常不使用间隙锁 |
| REPEATABLE READ（RR） | 不允许 | 不允许 | SQL 标准允许 | 普通一致性读复用视图；加锁范围查询通常使用临键锁 |
| SERIALIZABLE | 不允许 | 不允许 | 不允许 | 使用更强隔离；显式事务中的普通 SELECT 通常转为共享加锁读 |

这里“允许/不允许”描述隔离保证，并不意味着每次并发都会发生异常。SERIALIZABLE 并不是所有事务逐个排队运行：兼容的读仍可并发；是否将普通 SELECT 转成加锁读还与事务及 `autocommit` 状态有关。RR 的快照读和加锁读混用有特殊现象，见第 3 节。

```sql
SELECT VERSION(), @@transaction_isolation, @@autocommit;
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- 仅影响之后启动的事务；已有事务不应依赖这个设置即时改变隔离级别
```

MySQL 5.7 的变量名与小版本有关，部分环境使用 `tx_isolation`；请用 `SHOW VARIABLES LIKE '%isolation%';` 核对。选择 RC 可能减少间隙锁冲突，但不能因此忽略业务对重复读、范围保护和复制格式的要求。

## 2. MVCC：隐藏列、版本链与 ReadView

### 2.1 版本链解决什么问题

MVCC（多版本并发控制）让**一致性读**根据事务视图读取合适的行版本：读者一般不必等待写者提交，也不阻止写者更新。它不消除写写冲突；更新同一行仍需要互相等待。

InnoDB 聚簇索引记录包含 `DB_TRX_ID`（6 字节，最近修改该记录的事务 ID）和 `DB_ROLL_PTR`（7 字节，指向相关 undo 记录）。没有合适的显式主键或非空唯一键时，InnoDB 会生成隐藏的 `DB_ROW_ID`。不要把 undo 简化成一组一定会被执行的反向 SQL：它记录的是供回滚及构造旧版本使用的信息。

例如一行从 100 被先后更新为 120、150，示意如下：

```text
聚簇索引中的当前版本
  balance=150, trx_id=30
       │ DB_ROLL_PTR
       ▼
  undo：可重建 balance=120, trx_id=20
       │
       ▼
  undo：可重建 balance=100, trx_id=10
```

查询先判断当前版本对自己的 ReadView 是否可见；不可见则沿 undo 重建并判断更早的版本，直到找到可见版本或确认记录对该视图不存在。删除也需要保留供旧视图使用的信息；不应把它理解为立即物理删除所有版本。

**例子：两个事务如何通过版本链读取同一行**

假设余额最初为 100，最后修改该行的事务 ID 为 10；事务 A 的 ID 为 20。以下 ID 仅用于说明：

1. A 将余额改为 200，但尚未提交。当前行标记为事务 20，旧版本可通过 undo 重建：

   ```text
   当前行：余额 200，事务 ID = 20
                  │ DB_ROLL_PTR
                  ▼
   undo：可重建余额 100，事务 ID = 10
   ```

2. 事务 B 第一次执行普通 `SELECT`，创建 ReadView。因为事务 20 此时仍活跃，当前版本对 B 不可见；B 沿 undo 找到事务 10 的旧版本，返回 **100**。
3. A 提交后，B 再次执行普通 `SELECT`：在 RR 下，B 复用旧 ReadView，仍返回 **100**；在 RC 下，B 创建新 ReadView，此时 A 已提交，返回 **200**。

简言之：**事务 ID 标记版本由谁修改，ReadView 判断版本是否可见，undo 帮助重建不可见时需要的旧版本**。若 B 自己执行 `UPDATE`，则要处理当前记录并获取锁，而不是只修改快照中的旧版本。

### 2.2 ReadView 如何判断可见性

以下字段名称是常见的源码/教学表述，不是 SQL 可查询的公共接口；不同版本细节可能不同：

| 字段 | 含义 |
| --- | --- |
| `m_ids` | 创建视图时活跃的读写事务 ID 集合 |
| `min_trx_id` | 活跃事务 ID 的下界；集合非空时是最小活跃 ID |
| `max_trx_id` | 创建视图时下一待分配事务 ID 的边界，并非最大的活跃 ID |
| `creator_trx_id` | 创建该视图的事务 ID，用来识别自己的修改 |

对候选版本的 `trx_id`，按以下顺序判断：

1. 是创建者自己的修改：可见。
2. 小于 `min_trx_id`：可见。
3. 大于或等于 `max_trx_id`：不可见。
4. 位于两者之间：在 `m_ids` 中则不可见，否则可见。

例如视图创建时 `m_ids=[20, 30]`、`min=20`、`max=40`：事务 10 的版本可见；事务 20 的版本不可见；事务 25 的版本不在活跃集合中，可见；事务 40 的版本不可见。即使事务 20 **之后**提交，对这个已经创建的视图仍不可见。事务 ID 何时分配有实现细节，不应等同于业务调用 `BEGIN` 的时间。

### 2.3 旧版本何时清理

purge 清理不再被任何相关视图需要的历史版本。长时间持有旧 ReadView 会延迟清理，可能增加 undo 空间和查询旧版本的成本。`History list length` 反映待清理的历史工作量趋势，但不是 undo 字节数或长事务数量；需结合负载、持续时间及事务信息分析。

```sql
SHOW ENGINE INNODB STATUS\G
SELECT trx_id, trx_mysql_thread_id, trx_state, trx_started,
       trx_rows_locked, trx_rows_modified
FROM information_schema.INNODB_TRX
ORDER BY trx_started;
```

## 3. RC 与 RR、快照读和当前读

### 3.1 两种读取方式

| | 一致性读（通常称快照读） | 加锁读/写（通常称当前读） |
| --- | --- | --- |
| 常见语句 | RC/RR 下普通 `SELECT` | `SELECT ... FOR UPDATE`、`FOR SHARE`、`UPDATE`、`DELETE` |
| 数据来源 | ReadView 可见版本，并能看到本事务自己的修改 | 处理当前记录；遇到未提交冲突可能等待，等待后再处理 |
| 数据行锁 | 普通一致性读不加行锁，仍可能持有 MDL | 根据索引扫描和隔离级别加锁 |

`INSERT` 是写操作，会参与唯一性检查并获取相应锁，但不宜简单说它是一个 SELECT 式的“读取”。MySQL 5.7 的共享加锁语法为 `LOCK IN SHARE MODE`，8.0 也支持 `FOR SHARE`。加锁读应放在显式事务中，才能让锁跨语句保护后续操作。

RC 的**每次一致性读**建立新视图；RR 通常在事务中**首次一致性读**建立视图并复用。RR 下 `START TRANSACTION WITH CONSISTENT SNAPSHOT` 可提前建立一致性快照；普通 `BEGIN` 本身不一定建立快照。当前读不使用该旧视图。

### 3.2 双会话实验：不可重复读

准备数据：

```sql
CREATE TABLE mvcc_demo (
    id INT PRIMARY KEY,
    balance INT NOT NULL
) ENGINE=InnoDB;
INSERT INTO mvcc_demo VALUES (1, 100);
```

按顺序执行，下表每行在前一行完成后执行：

| 步骤 | 会话 A：RR | 会话 B |
| --- | --- | --- |
| 1 | `SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;` | |
| 2 | `START TRANSACTION;` | |
| 3 | `SELECT balance FROM mvcc_demo WHERE id=1;` → 100，建立视图 | |
| 4 | | `UPDATE mvcc_demo SET balance=200 WHERE id=1;`（自动提交） |
| 5 | `SELECT balance FROM mvcc_demo WHERE id=1;` → 100 | |
| 6 | `SELECT balance FROM mvcc_demo WHERE id=1 FOR UPDATE;` → 200，并加锁 | |
| 7 | `COMMIT;` | |

将初始值恢复为 100，再把 A 的隔离级别改为 RC 重做：第 5 步返回 200。上表默认 B 为自动提交，且没有其他并发操作。

### 3.3 RR 下幻读和“看到自己的修改”

RR 中，反复执行普通一致性读，其他事务之后插入的行不会出现在旧视图中；对**实际覆盖范围**执行加锁读后，临键锁通常能阻止其他事务向受保护间隙插入。两者混用时不能认为所有语句看到的数据都相同：

| 步骤 | 会话 A：RR | 会话 B：自动提交 |
| --- | --- | --- |
| 1 | `START TRANSACTION;` | |
| 2 | `SELECT * FROM mvcc_demo WHERE id=5;` → 空，建立视图 | |
| 3 | | `INSERT INTO mvcc_demo VALUES (5, 500);` |
| 4 | `SELECT * FROM mvcc_demo WHERE id=5;` → 仍为空 | |
| 5 | `UPDATE mvcc_demo SET balance=501 WHERE id=5;` → 更新 1 行 | |
| 6 | `SELECT * FROM mvcc_demo WHERE id=5;` → 看到 501 | |
| 7 | `ROLLBACK;` | |

第 5 步是写操作，能处理 B 已提交的当前记录；第 6 步看到的是 **A 自己修改过的版本**，不是“仍然为空”。因此，不要把 RR 描述成所有语句都只看同一时刻的完整数据库状态。要保护“先检查再写”，应在事务中设计合适的约束及加锁策略，并验证实际锁范围；不能仅靠先做普通 SELECT。

## 4. 锁体系：全局锁、表锁、MDL、意向锁与自增锁

### 4.1 按作用范围理解

| 类型 | 作用与特点 |
| --- | --- |
| 全局读锁 | `FLUSH TABLES WITH READ LOCK`（FTWRL）用于某些备份/切换场景，会阻塞写入；锁由持有它的连接释放或断开时释放 |
| 只读配置 | `read_only`、`super_read_only` 是服务器只读策略，**不是全局锁**；对特权用户及复制线程的行为不同 |
| 显式表锁 | `LOCK TABLES ... READ/WRITE` 限制表的并发访问，有隐式提交等语义，使用前需核对事务边界 |
| MDL 元数据锁 | 保护表定义，访问表时自动获取；防止使用中的表结构被不兼容 DDL 修改 |
| 意向锁 IS/IX | InnoDB 的表级锁，声明事务打算在表中获取行级 S/X 锁，便于快速判断表锁是否兼容 |
| AUTO-INC 锁 | 自增值分配的特殊**表级**锁；是否使用及持有方式取决于插入类型和 `innodb_autoinc_lock_mode` |
| 行锁 | 实际锁定索引记录和/或间隙，详见第 5 节 |

共享锁 S 允许多个事务同时持有；排他锁 X 与其他事务对同一记录的 S、X 锁不兼容。`FOR SHARE` 通常请求 S 锁，`FOR UPDATE` 通常请求 X 锁。普通快照读不需要等待这些记录锁，但另一个加锁读或写操作可能等待。

IS 与 IX 彼此兼容；例如表级 S 与 IX 冲突，表级 X 与 IS/IX 冲突。普通 DML 使用的意向锁不意味着所有行都被锁住。

### 4.2 MDL 排队：不一定是死锁

```text
A：BEGIN; SELECT ... FROM t;  ──持有 MDL 读锁，事务未结束
B：ALTER TABLE t ...          ──申请 MDL 写锁，等待 A
C：SELECT ... FROM t          ──可能排在 B 后等待
```

这是典型的**阻塞链**，不一定有循环等待，因此不能直接称为死锁。MDL 写请求通常优先于后来的读请求，具体调度受服务器配置等影响；长期等待的 DDL 可能使后续访问堆积。显式事务访问过的表，其相关 MDL 通常持有到事务结束；自动提交的单条语句则通常在语句结束释放。

排查时先找持有者，再评估终止等待中的 DDL 或长事务的影响；不要直接批量杀连接。`lock_wait_timeout` 控制 MDL 等锁的等待，并非 InnoDB 行锁等待超时。

```sql
SHOW FULL PROCESSLIST;
SELECT * FROM performance_schema.metadata_locks WHERE LOCK_STATUS = 'PENDING';
SELECT * FROM sys.schema_table_lock_waits;  -- 要求相关 instrumentation 已启用
SET SESSION lock_wait_timeout = 30;         -- 在执行 DDL 的会话设置
```

### 4.3 自增锁模式

`innodb_autoinc_lock_mode`：0（traditional）、1（consecutive）、2（interleaved）。0 对插入类语句使用语句级 AUTO-INC 表锁；1 对可预知行数的简单插入使用更轻量的分配机制，对部分批量插入使用表锁；2 允许不同语句的自增值交错。5.7 常见默认值为 1，8.0 默认为 2，以实际变量为准。

自增值不是无间断流水号：回滚、并发、分配预留都可能产生空洞。mode 2 对基于语句的复制可能不安全；ROW 记录行变更，通常适合该模式。

## 5. InnoDB 行锁与加锁规则

### 5.1 三种常见锁

行锁作用于**索引记录及索引间隙**，不是简单地锁住 SQL 文本中的条件。若没有合适索引，InnoDB 仍会使用聚簇索引（必要时为隐藏聚簇索引）扫描。

```text
已有索引值：   5          10          20
间隙：       (-∞,5)     (5,10)      (10,20)      (20,+∞)
临键锁示意：  (-∞,5]     (5,10]      (10,20]      (20,+∞)
```

| 锁 | 锁定内容 | 作用 |
| --- | --- | --- |
| Record Lock | 某条索引记录 | 控制对该记录的修改或加锁读取 |
| Gap Lock | 两条记录之间的开区间 | 抑制其他事务向该间隙插入 |
| Next-Key Lock | 记录锁 + 该记录之前的间隙锁 | 既保护记录，又保护范围内的插入 |

末尾通过 supremum 等内部记录表示尾部间隙，不应将 `+∞` 理解成真实数据。不同事务的普通间隙锁可以共存，间隙锁主要是“禁止插入”，并不是普通意义的互斥锁；插入时需要的**插入意向锁**可能因已有间隙锁而等待。不同位置的插入意向锁通常可以并发，但还可能因唯一键、外键或其他锁发生冲突。

### 5.2 常见加锁场景及边界

以下是 RR、InnoDB、加锁读的经验模型，而不是不看执行计划就能精确计算的通用公式：

| 查询形态 | 常见行为与注意事项 |
| --- | --- |
| 完整唯一索引等值命中 | 对找到的记录通常使用记录锁，不必保护其前面的间隙 |
| 完整唯一索引等值未命中 | 通常保护目标值所在间隙，阻止其他事务插入该位置 |
| 非唯一索引等值或范围查询 | 通常锁扫描到的索引范围，可能包括边界外的间隙或记录；排他锁通过二级索引查找时还会锁相应聚簇索引记录 |
| 无合适索引 | 可能扫描并锁住大量聚簇索引记录及间隙，严重时效果接近锁全表；这不等于获取了同一种显式表锁 |
| `LIMIT` | 可能提前停止扫描，但排序、回表、过滤及执行计划会改变扫描量；不能断言只锁返回的 N 行 |

复合唯一索引只按前缀查询、使用 `NULL`、条件不完整，或优化器根本没有采用该索引时，不应直接套用“唯一等值命中只锁一条”的结论。RC 下通常关闭搜索和扫描使用的间隙锁，但外键检查、重复键检查等仍可能使用间隙锁；不匹配记录的锁释放也取决于语句和执行过程。

RC 下的 UPDATE 还可能使用**半一致性读**：遇到已被其他事务锁住的行时，先取最近已提交版本供服务器判断 WHERE；如果不匹配，可避免等待，如果匹配则重新读取并尝试获取锁，必要时等待。它不是允许修改未提交数据，也不适用于推断所有 SQL 的锁行为。

### 5.3 双会话实验：唯一键命中与未命中

```sql
CREATE TABLE lock_demo (
    id INT PRIMARY KEY,
    score INT NOT NULL,
    KEY idx_score (score)
) ENGINE=InnoDB;
INSERT INTO lock_demo VALUES (5, 10), (10, 20), (20, 30);
```

两会话均使用 RR；先确保表只有上述数据：

| 步骤 | 会话 A | 会话 B |
| --- | --- | --- |
| 1 | `START TRANSACTION;` | |
| 2 | `SELECT * FROM lock_demo WHERE id=10 FOR UPDATE;` | |
| 3 | | `UPDATE lock_demo SET score=21 WHERE id=10;` → 等待 |
| 4 | `COMMIT;` | B 的更新继续，完成后提交 |

接着在 A 执行 `START TRANSACTION; SELECT * FROM lock_demo WHERE id=12 FOR UPDATE;`，再由 B 执行 `INSERT INTO lock_demo VALUES (12, 40);`：B 通常会等待，因为 12 所在的 `(10,20)` 间隙受到保护。A 执行 `COMMIT` 或 `ROLLBACK` 后 B 才能继续。若要再次实验，先清理新插入的行。

查看计划与锁：

```sql
EXPLAIN SELECT * FROM lock_demo WHERE score=20 FOR UPDATE;
-- 在第三个连接观察，需相应权限；具体记录和模式依版本及执行状态而异
SELECT ENGINE_TRANSACTION_ID, OBJECT_NAME, INDEX_NAME, LOCK_TYPE,
       LOCK_MODE, LOCK_STATUS, LOCK_DATA
FROM performance_schema.data_locks
WHERE OBJECT_NAME = 'lock_demo';
SELECT * FROM performance_schema.data_lock_waits;
```

`EXPLAIN` 只能帮助判断访问路径，不是完整的锁清单。实际冲突还取决于隔离级别、数据分布、唯一性检查和事务时序；结合 `data_locks` 与双会话实验判断。

## 6. 死锁、锁等待与长事务

### 6.1 锁等待和死锁有什么区别

- **锁等待**：A 等 B 持有的锁，不一定形成环；B 提交后 A 通常可继续。
- **死锁**：多个事务形成等待环，彼此无法自行前进。例如 A 持有行 1、等待行 2，同时 B 持有行 2、等待行 1。

```text
A 持有 id=1 ──等待──> id=2（B 持有）
B 持有 id=2 ──等待──> id=1（A 持有）
```

InnoDB 默认启用死锁检测，发现死锁时选择一个受害事务回滚，通常倾向于较小的事务；业务应对整个事务进行**有限次数重试**，配合退避和幂等，不能只重试失败语句。高并发热点场景关闭死锁检测有时可减轻检测开销，但会改为依赖超时，必须结合业务和监控谨慎评估。

行锁等待超时通常报 `Lock wait timeout exceeded`；默认 `innodb_rollback_on_timeout=OFF` 时，InnoDB 一般只回滚当前语句，事务及其此前持有的锁可能仍在。应用通常应显式 `ROLLBACK` 后再决定是否重试。超时本身不能证明不存在死锁，尤其是关闭死锁检测或涉及引擎无法检测的锁时。

例如准备好 `lock_demo` 后，两会话均开启事务，A 更新 `id=5`，B 更新 `id=10`；接着 A 更新 `id=10`，A 开始等待；此时 B 再更新 `id=5`，形成环。通常其中一个事务收到死锁错误并被回滚，另一个得以继续。实验后两个会话都应检查事务状态并结束事务。

### 6.2 如何定位与规避

```sql
SHOW ENGINE INNODB STATUS\G
-- 查 LATEST DETECTED DEADLOCK 中各事务持有/等待的索引与锁
SELECT * FROM sys.innodb_lock_waits;
SELECT * FROM information_schema.INNODB_TRX ORDER BY trx_started;
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';
SHOW VARIABLES LIKE 'innodb_deadlock_detect';
```

`LATEST DETECTED DEADLOCK` 只展示最近一次检测到的死锁；需要持续排查时，可评估开启 `innodb_print_all_deadlocks` 写入错误日志。8.0 可结合 `performance_schema.data_locks` 和 `data_lock_waits`；5.7 使用的部分 `INFORMATION_SCHEMA` 锁表已在 8.0 移除。

优先措施：统一访问记录的顺序、缩小事务与扫描范围、建立合适索引、拆分大批量操作、减少热点。RC 可能减少部分间隙锁冲突，但不会消灭所有死锁，也不能替代业务一致性设计。

### 6.3 长事务的代价

长事务可能延迟 purge、长期持有行锁和 MDL、使回滚耗时增加；大量修改组成的大事务还会增加复制和恢复压力。一个 `Sleep` 连接也可能持有未结束的事务；要结合事务开始时间、连接状态和实际锁，而不是只看当前 SQL。

不要把远程调用或长时间等待放进持锁事务；明确异常路径的回滚与连接归还。终止连接前应确认业务影响：`KILL` 后回滚大事务也可能很慢。Spring 的 `@Transactional` 还要注意代理模式下同类内部调用可能不经过事务代理。

## 7. binlog 格式与提交

### 7.1 STATEMENT、ROW、MIXED

| 格式 | 记录内容 | 主要取舍 |
| --- | --- | --- |
| STATEMENT（SBR） | 执行的 SQL 语句及相关上下文 | 日志通常较小，但部分语句受执行顺序、函数或环境影响，重放可能不安全 |
| ROW（RBR） | 行变更事件及必要的行镜像 | 更便于确定性复制和 CDC；大批量变更可能产生较大日志 |
| MIXED（MBR） | 由服务器根据规则选择语句或行格式 | 折中，但不能保证规避所有应用层问题 |

不应笼统说 `NOW()`、`RAND()` 一定导致 STATEMENT 复制错误：MySQL 对部分函数记录或传递了执行上下文；仍有不安全语句，例如某些不确定执行顺序、`UUID()` 等场景，应查看具体版本的警告和手册。ROW 通常是生产复制与 CDC 的优先选择，但不是绝对保证主从一致：过滤配置、复制错误、从库额外写入等仍可能导致差异。

```sql
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'binlog_row_image';
SHOW GLOBAL STATUS LIKE 'Binlog_cache%';
```

`binlog_row_image=FULL` 包含完整行镜像；`MINIMAL` 可减少部分镜像，具体哪些列记录取决于事件类型和表结构。修改前确认 CDC、审计和恢复工具的要求。ROW 闪回也需要足够的旧值、正确的日志与工具，不能保证所有误操作都能反转。

MySQL 8.0 常见默认格式为 ROW，实际以配置为准。binlog 必须保留足够久，覆盖备份周期、恢复窗口及从库追赶需求；过早清理会使恢复或复制缺少必要日志。日志占用增长也需要监控，必要时归档并演练恢复。

### 7.2 提交与恢复

事务中的 binlog 事件先进入会话的事务缓存，大事务可能使用临时文件；`Binlog_cache_disk_use` 可结合 `Binlog_cache_use` 观察溢出情况，不应仅凭绝对值调大缓存。组提交将多个事务的相关步骤批量处理，常用 flush → sync → commit 描述；`sync_binlog` 控制 binlog 同步策略。

InnoDB 与 binlog 的提交需要协调：简化理解为 redo 准备、写入并按配置同步 binlog、再完成引擎提交；崩溃恢复根据日志和事务状态决定提交或回滚。具体实现和刷盘保证取决于版本及配置。`sync_binlog=1` 与 `innodb_flush_log_at_trx_commit=1` 是常见的强持久性配置，但仍依赖存储正确执行刷盘。

```sql
SHOW MASTER STATUS;  -- MySQL 8.0 常用；不同版本的新命令名称请查对应手册
```

解析 ROW 事件可使用 `mysqlbinlog --base64-output=decode-rows -vv`；输出通常是注释形式的行事件，不是可直接执行的完整恢复 SQL。

## 8. 主从复制、GTID、半同步和并行复制

### 8.1 异步复制流程

```text
主库事务提交
  → binlog
  → 主库 binlog dump 线程发送
  → 从库接收线程（IO）写 relay log
  → 从库应用线程（SQL / workers）执行并提交
  → 从库查询才能看到已应用的数据
```

接收和应用是不同进度：数据到达 relay log 不代表已在从库可查询。8.0 的较新版本使用 source/replica 命名，旧版本常使用 master/slave；具体命令和变量名按版本确认。

### 8.2 GTID 与切换

GTID 为事务提供全局标识，常见形式为 `server_uuid:sequence_number`；`gtid_executed` 是已执行事务集合。启用自动定位后，从库通过比较 GTID 集合请求缺少的事务，降低手工计算文件位点的成本，并避免同一 GTID 被重复应用。

GTID **不会凭空找回已丢失或已清理的 binlog**。故障切换时仍需确认候选节点包含需要的事务、复制状态和数据完整性，并处理旧主隔离；GTID 是工具，不等于零数据丢失。

### 8.3 半同步的保证和局限

半同步正常工作时，主库等待配置数量的从库确认已接收并将事务事件写入、刷新到 relay log，随后才向客户端返回成功；**不等待从库应用完成**。等待超时可能退化为异步，因此不能无条件宣称“不丢数据”。

- `AFTER_SYNC`：主库同步 binlog 后等待 ACK，再提交存储引擎。
- `AFTER_COMMIT`：先提交存储引擎，再等待 ACK；等待期间该事务可能已对主库其他会话可见，主库故障时可能出现已观察到的数据未到达新主。

即便是 `AFTER_SYNC`，也需要半同步当时没有退化、收到 ACK 的副本及其日志仍可用、选主和恢复流程正确，才能利用已收到的事务降低丢失风险。它不保证所有从库都收到，也不提供从库即时读一致。

### 8.4 并行复制

5.6 提供按库并行；5.7 引入 `LOGICAL_CLOCK`，可以按提交依赖在同一库内并行应用。8.0 可使用基于写集合的依赖跟踪，向从库提供更细的依赖信息。**WRITESET 是主库依赖跟踪策略，不是 `slave_parallel_type` 的取值**；从库仍需配置对应的并行应用方式。

不同小版本的默认 worker 数、变量名称和默认策略不同；例如 8.0.27 起默认启用多线程应用及提交顺序保持。增加 worker 也可能受热点冲突、大事务、DDL 和机器资源限制，并非越多越快。

```sql
SHOW REPLICA STATUS\G  -- 8.0.22+；旧版本使用 SHOW SLAVE STATUS\G
SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE '%parallel%workers';
SHOW VARIABLES LIKE '%parallel%type';
SHOW VARIABLES LIKE 'binlog_transaction_dependency_tracking';
```

## 9. 主从延迟与读写分离一致性

### 9.1 延迟从哪里来

| 环节 | 可能原因 | 排查与应对 |
| --- | --- | --- |
| 发送/接收 | 网络、主库压力、日志过大 | 检查接收线程、网络和吞吐 |
| 从库应用 | 大事务、DDL、热点冲突、并行度不足 | 检查 worker 状态和慢事务，优化事务大小及并行配置 |
| 从库资源 | 查询与复制竞争 CPU/IO | 查看资源、慢查询及索引，必要时扩容或分流 |

`Read_Master_Log_Pos` 与 `Exec_Master_Log_Pos` 表示不同进度，**文件相同时**位置差可作为待处理日志量的粗略线索；文件不同时不能直接相减，字节差也不是时间延迟。并行复制时还需要考虑 worker 状态和执行空洞。

`Seconds_Behind_Master` / `Seconds_Behind_Source` 是近似指标，网络或接收线程落后、时钟等情况可能使它失真；`NULL` 可能表示应用线程未运行，或接收线程未运行且 relay log 已处理完，应结合两个线程状态与错误信息判断。心跳表（如 `pt-heartbeat`）可从应用路径测量延迟，但仍要考虑时钟同步和采样间隔。可接受延迟由业务 SLO 决定，没有适用于所有部署的固定毫秒阈值。

### 9.2 写后读一致性怎么保证

```text
客户端写主库成功 → 立即读从库 → 从库可能尚未应用该事务 → 读到旧值
```

| 方案 | 能力与代价 |
| --- | --- |
| 关键读走主库 | 主库上已提交后通常可读到自己的写入；增加主库读压力，也要避免复用更早建立的 RR 快照 |
| GTID 等待 | 获得本次写入的 GTID，在**目标从库**等待其执行后再读；超时或错误要回主库或明确报错 |
| 写后短时间读主 / 延迟感知路由 | 降低读旧概率或按阈值控制风险；固定时间和近似延迟指标不能严格保证 |
| 半同步 | 改善日志接收保障，但不保证从库已应用，单独使用不能保证写后读一致 |
| 最终一致与补偿 | 适合允许短暂不一致的业务；需明确补偿、超时和用户体验 |

GTID 等待示意：

```sql
-- gtid_set 应从本次写入的可靠来源取得；示例 UUID 和序号仅占位
SELECT WAIT_FOR_EXECUTED_GTID_SET('aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee:42', 1);
-- 0 表示已执行，1 表示超时；SQL 错误也必须处理
-- 成功后在同一个目标从库开启新的读事务，再执行查询
```

不要在等待成功后继续使用**等待前已建立的 RR 旧快照**：它可能仍看不到该事务。连接池与路由必须确保等待和后续读取落在同一个目标实例；若失败则按业务约定回主库或报错。

```sql
SHOW REPLICA STATUS\G
-- 关注接收/应用线程状态、最后错误、延迟，以及 Retrieved_Gtid_Set / Executed_Gtid_Set
```

## 10. 一致性备份与锁相关参数

### 10.1 `--single-transaction` 的原理与限制

`mysqldump --single-transaction` 为支持事务的表建立一致性快照，导出期间其他事务通常可继续修改 InnoDB 数据。它不是“整个备份过程完全无锁”：仍有 MDL；配合记录 binlog 位点的 `--source-data` / `--master-data` 时，启动阶段可能短暂获取全局读锁。等待已有写入结束也可能使这个阶段变长。

主要限制：

1. 一致性快照适用于 InnoDB 等支持一致性读的事务表；MyISAM 等非事务表在导出过程中可能变化，**不会因为这个选项自动得到一致备份**。
2. 不要在备份期间对相关表执行 `ALTER`、`DROP`、`RENAME`、`TRUNCATE` 等 DDL；可能阻塞、导致导出失败或破坏一致性，具体表现依语句和时序而定。
3. 这是长时间快照，大库备份可能延迟 purge；逻辑恢复也可能较慢。物理备份工具有各自的准备和恢复步骤。
4. 该选项和锁表选项的组合及权限要求应按客户端版本核对；`--source-data=2` 主要记录 binlog 文件和位点为注释，GTID 信息和导出行为由其他设置共同决定。

### 10.2 从备份恢复到指定时间

PITR（时间点恢复）的流程：恢复经过验证的全量备份 → 从对应的 binlog 文件和位点开始，按顺序回放后续日志 → 在目标操作之前停止 → 校验数据。时间点、时区、跨多个文件的顺序、事务边界和 GTID 均需纳入演练；**不要在生产库直接试回放**。

```powershell
# 示例仅用于隔离测试环境；文件名、位点与时间必须来自真实备份。
# 8.0.26+ 使用 --source-data；较旧客户端使用 --master-data。
mysqldump -u backup_user -p --single-transaction --source-data=2 --routines --triggers --databases mydb --result-file=backup.sql
mysql -u restore_user -p -e "SOURCE backup.sql"
mysqlbinlog --start-position=1234 --stop-datetime="2024-01-01 10:30:00" binlog.000005 --result-file=replay.sql
mysql -u restore_user -p -e "SOURCE replay.sql"
```

示例只覆盖单个 binlog 文件；真实恢复可能需要依次处理多个文件及目标时间前的完整事务。备份包含建库语句但不代表自动包含所有用户和权限。必须验证备份、日志保留、恢复权限及恢复后数据；若误操作已包含在恢复范围内，应停止在误操作之前或依据经过验证的补偿方案处理。

### 10.3 常见参数

| 参数 | 常见默认值 | 用途与注意 |
| --- | --- | --- |
| `innodb_lock_wait_timeout` | 50 秒 | InnoDB 行锁等待超时；不是 MDL 超时 |
| `lock_wait_timeout` | 31536000 秒 | MDL 等等待超时；建议为 DDL 会话设合理上限 |
| `innodb_deadlock_detect` | ON | 是否主动检测 InnoDB 死锁 |
| `innodb_print_all_deadlocks` | OFF | 是否将所有 InnoDB 死锁写入错误日志 |
| `innodb_rollback_on_timeout` | OFF | 行锁超时默认只回滚语句；应用仍需处理事务 |
| `transaction_isolation` | REPEATABLE-READ | 默认事务隔离级别 |

这些是常见 MySQL 8.0 默认值，不同版本或发行版可能不同；生产环境先 `SHOW VARIABLES` 确认，不要直接照搬调参。

## 自测题

1. **ACID 分别解决什么问题？** 原子性是整体提交或撤销；一致性依赖约束和业务逻辑；隔离性控制并发可见性；持久性依赖日志、刷盘和存储。
2. **RC 和 RR 的快照有何区别？** RC 每次一致性读创建新视图；RR 通常首次一致性读创建、后续复用。自己的修改仍可见。
3. **ReadView 如何判断版本？** 先判断是否为自己，再比较下界和上界，中间范围检查活跃事务集合；不可见则沿 undo 重建旧版本。
4. **RR 为什么仍可能看到不同结果？** 普通读使用旧视图，当前读/写操作处理当前记录，且普通读能看到事务自己的修改。
5. **唯一索引未命中为什么可能阻塞插入？** RR 下加锁读可能锁住目标所在间隙；插入意向锁须等待间隙保护解除。
6. **MDL 等待一定是死锁吗？** 不一定；长事务持锁、DDL 等待、后续查询排队可能只是阻塞链。
7. **死锁和锁超时如何处理？** 死锁通常回滚受害事务；锁超时默认通常只回滚当前语句。应用应明确回滚并按幂等策略有限重试。
8. **半同步为什么不能保证从库立即读到写入？** ACK 表示日志已接收并持久化到 relay log，不代表事务已在从库应用；超时还可能退化为异步。
9. **GTID 等待成功后为什么还要注意事务边界？** 已有 RR 快照可能早于目标事务；应在目标从库建立新的读视图。
10. **`--single-transaction` 能一致备份 MyISAM 吗？** 不能；它依赖事务引擎的一致性快照，且备份期间仍需管理 DDL 和可能的短暂全局锁。

## 背诵卡片

1. ACID 需要日志、锁、约束、业务逻辑及正确配置共同实现。
2. RC 每次一致性读创建视图；RR 首次一致性读创建并复用。
3. 一致性读读取可见旧版本，也能看到本事务自己的修改；当前读/写处理当前记录。
4. InnoDB 锁作用于索引记录和间隙；具体范围取决于索引、扫描路径、隔离级别及数据。
5. 间隙锁通常彼此兼容，但可能阻塞插入意向锁；唯一等值命中与未命中要分别分析。
6. MDL、行锁等待和死锁是不同问题；长事务既可能持锁，也可能延迟 purge。
7. ROW 有利于复制和 CDC，但不保证所有故障下主从绝对一致；redo 与 binlog 用途不同。
8. 半同步等待接收，不等待应用；GTID 自动定位也不能恢复已经丢失的日志。
9. 写后读优先考虑读主或目标从库 GTID 等待，并避免复用旧快照。
10. 一致性备份依赖引擎和选项；备份成功不等于恢复成功，必须演练。

## 参考资料

- [MySQL 8.0：事务隔离级别](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
- [MySQL 8.0：一致性非锁定读](https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html)
- [MySQL 8.0：InnoDB 锁及各语句加锁](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)、[加锁规则](https://dev.mysql.com/doc/refman/8.0/en/innodb-locks-set.html)
- [MySQL 8.0：半同步复制](https://dev.mysql.com/doc/refman/8.0/en/replication-semisync.html)
- [MySQL 8.0：mysqldump](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html)
