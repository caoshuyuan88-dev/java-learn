# Elasticsearch搜索与调优
> **定位**：讲清 ES 的倒排索引与写入原理、Mapping/DSL/聚合与深分页的正确用法，以及集群设计、写入查询调优与线上故障排查。
> **重要度**：★★★★☆
> **建议复习时长**：4.5 小时
> **关联文档**：[01-消息队列与RocketMQ](./01-消息队列与RocketMQ.md)、[02-Kafka深度剖析](./02-Kafka深度剖析.md)、[03-MySQL调优主从与分库分表](../02-数据库/03-MySQL调优主从与分库分表.md)、[03-数据结构与算法](../06-计算机基础/03-数据结构与算法.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| ES 定位、核心概念与适用场景 | ★★★★☆ | 场景题 |
| 倒排索引、正排索引与分词器 | ★★★★★ | 八股原理 |
| 写入流程与近实时搜索 | ★★★★★ | 八股原理 |
| Mapping 与字段类型设计 | ★★★★★ | 配置实践 |
| Query DSL 与 query/filter 上下文 | ★★★★★ | 配置实践 |
| 聚合与相关性打分 | ★★★★☆ | 源码追问 |
| 分页与深度分页三方案 | ★★★★★ | 场景题 |
| 集群、分片设计与并发控制 | ★★★★★ | 架构设计 |
| 性能优化与 MySQL 数据同步 | ★★★★★ | 架构设计 |
| 常见线上问题（red/yellow/OOM/拒绝） | ★★★★★ | 故障复盘 |

## 1. ES 定位、核心概念与适用场景
### 1.1 核心原理
ES 是分布式搜索与分析引擎，基于 Lucene。核心概念：**Index**（索引，类似库）、**Type**（7.0 起废弃、8.0 完全移除，统一 `_doc`）、**Document**（JSON 文档，类似行）、**Field**（字段，需 Mapping 定义类型）、**Mapping**（字段类型与索引方式定义，类似 schema）、**Shard**（分片，索引的水平切分单位，一个分片就是一个 Lucene 索引，**数量创建后不可变**）、**Replica**（副本分片，提供高可用与读扩展，可动态调整）、**Node**（节点，按角色分为 master-eligible/data/ingest/coordinating）、**Cluster**（集群）。适用：全文检索（商品、文章搜索）、日志与指标分析（ELK）、聚合统计与多维分析、地理位置检索；不适合：强事务（无 ACID）、频繁更新的场景（更新 = 删除 + 新建）、作为唯一数据源（需与 MySQL 等权威存储配合）、复杂多表 Join（需宽表或 nested）。
### 1.2 面试问答
**Q1：为什么 ES 不能替代 MySQL？**
A：① 无事务（无法满足扣款、库存等强一致需求）；② 近实时而非实时（默认 1s 才可搜到）；③ 更新代价高（文档是不可变的，更新等于标记删除 + 写入新文档）；④ 关联能力弱（无真正的 Join）；⑤ 运维与成本高。结论——ES 是"检索与分析的从库"，权威数据仍在 MySQL。

**Q2：为什么分片数不可变？**
A：分片的路由公式是 `shard = hash(_routing) % number_of_primary_shards`，分片数变化会改变路由结果，导致已有数据"找不到"。要改只能 `_reindex` 重建、`_shrink`（缩小）、`_split`（扩大），或使用别名 + 双写切换。

**Q3：主分片与副本分片的关系？**
A：写入先到主分片，主分片完成后再同步到副本分片（默认 `wait_for_active_shards=1`）；读请求可被主分片或副本分片处理，因此副本能提升读吞吐；主分片所在节点故障时，副本会被提升为主分片。

**Q4：ES 常见的使用误区有哪些？**
A：① 把 ES 当数据库用（写多、更新频繁、需要事务）；② 分片数拍脑袋定（如单索引 1000 分片）；③ 用 text 字段做聚合排序（应使用 keyword 子字段）；④ 深分页用 from+size；⑤ 同步双写不做补偿，导致数据不一致。

**Q5：ES 7.x 与 8.x 的主要差异？**
A：7.0 废弃 Type（默认 `_doc`）、7.0 起默认自动选主（Zen2，不再需要 `minimum_master_nodes`）、7.x 默认开启安全（7.1+ 基础版免费）；8.0 彻底移除 Type、默认开启安全并强制 HTTPS、部分默认分片数从 5 改为 1（`index.number_of_shards` 默认 1）、移除 `_type` 相关 API。迁移时重点关注客户端兼容性与安全配置。
### 1.3 高频追问与陷阱
- 陷阱：把"索引"当成"数据库表"设计——ES 里更接近"宽表"，要反范式化。
- 陷阱：以为副本数减少就能省空间——副本是完整数据副本，空间与写入成本都翻倍。
- 追问：ES 为什么用 `_doc` 作为 Type？因为移除 Type 后保留字段以兼容旧客户端，8.0 起彻底不再需要。
### 1.4 配置/命令示例
```bash
curl -X GET "localhost:9200/_cluster/health?pretty"
curl -X GET "localhost:9200/_cat/indices?v&s=store.size:desc"
curl -X GET "localhost:9200/_cat/shards/order_index?v&h=index,shard,prirep,state,docs,store,node"
```
### 1.5 一句话速记
ES 是检索与分析引擎而非数据库；分片数不可变，副本可调，Type 已废弃，权威数据留在 MySQL。

## 2. 倒排索引、正排索引与分词器
### 2.1 核心原理
**倒排索引**由三部分组成：**Term Dictionary**（所有词项按字典序排列，存磁盘）、**Term Index**（词项前缀索引，用 **FST** 压缩后常驻内存，用于快速定位词项在词典中的位置，避免读整个词典）、**Posting List**（该词项命中的文档 ID 列表，含词频/位置，delta 编码压缩存储，多个词项求交集时用**跳表（skip list）**加速）。**正排索引**：**doc_values**（默认对非 text 字段开启的列式存储，磁盘，用于排序、聚合、脚本，替代了堆内存中的 fielddata）、**fielddata**（text 字段聚合时才在堆内存中构建的正排结构，容易 OOM，应改用 keyword 子字段）。**分词器**由 Character Filter + Tokenizer + Token Filter 组成，中文常用 IK：`ik_max_word`（细粒度，适合写入）与 `ik_smart`（粗粒度，适合查询），支持自定义词典。
### 2.2 面试问答
**Q1：为什么倒排索引要用 FST + 跳表？**
A：结论——用少量内存换磁盘访问次数与求交效率。FST 把词典的前缀结构压缩成有限状态机，常驻内存，可快速定位词项在 Term Dictionary 中的大致位置（减少磁盘 seek）；Posting List 用 delta 编码 + 跳表，多词求交时可跳过大段不匹配的文档 ID。

**Q2：doc_values 和 fielddata 的区别？**
A：doc_values 是磁盘上的列式正排结构，构建在写入时完成，用于排序/聚合，几乎不占堆内存；fielddata 是为 text 字段在**堆内存**中临时构建的正排结构，代价高、易触发熔断与 OOM。结论——需要聚合排序的字段应使用 keyword（默认有 doc_values），不要用 text + fielddata。

**Q3：text 和 keyword 该怎么选？**
A：text 会分词、用于全文检索，不能直接用于排序和聚合（除非开 fielddata）；keyword 不分词、保留原值，用于精确匹配、聚合、排序、过滤。最佳实践：一个字段同时提供两种能力用多字段映射 `"name": {"type":"text","fields":{"keyword":{"type":"keyword","ignore_above":256}}}`。

**Q4：分词对查询有什么影响？**
A：检索时查询词会经过与字段相同的分析器（除 term 查询）。结论——若写入用 `ik_max_word`、查询用 `ik_smart`，会出现"写入拆得更细、查询拆得较粗"的差异，导致某些词召不回。常见做法：写入用 `ik_max_word` 提升召回，查询也用 `ik_max_word` 或按业务调整 search_analyzer。

**Q5：自定义词典怎么配？**
A：在 IK 配置目录修改 `IKAnalyzer.cfg.xml`，新增 `ext_dict`/`ext_stopwords`（可远程 URL），重启或热加载后生效。场景：新品牌名、行业术语、内部编码（如 "鸿蒙OS"）被拆散导致召回率下降。注意：词典膨胀会增大词典体积与内存占用。

**Q6：一个查询同时命中多个词项时怎么算分与求交？**
A：`bool` 的 `must`/`should` 会把各子查询转成 Lucene 的 BooleanQuery，底层对 Posting List 做求交/求并（跳表加速），再按各子句的 boost 与打分模型合并得分；`filter` 子句只求交不打分，且结果可被缓存。
### 2.3 高频追问与陷阱
- 陷阱：对 text 字段做 `terms` 聚合触发 fielddata 加载，造成堆内存暴涨。
- 陷阱：以为 keyword 没有长度限制——`ignore_above`（默认 256）超长的值不会被索引（但仍存在 _source 中），导致查不到。
- 追问：norms 是什么？存储字段长度归一化因子，用于打分；只用于过滤的字段可 `"norms": false` 省空间。
### 2.4 配置/命令示例
```json
PUT /order_idx
{ "mappings": { "properties": {
  "title": { "type": "text", "analyzer": "ik_max_word", "search_analyzer": "ik_smart" },
  "brand": { "type": "keyword", "ignore_above": 256 },
  "created_at": { "type": "date" },
  "desc": { "type": "text", "index": false }
}}}
```
### 2.5 一句话速记
FST 在内存定位词项、Posting List 在磁盘靠跳表求交；排序聚合用 doc_values，text 聚合是 OOM 之源；中文分词写入用 ik_max_word。

## 3. 写入流程与近实时搜索
### 3.1 核心原理
写入链路：文档先写入**内存缓冲区（memory buffer）**并追加到 **translog**（预写日志，保证宕机不丢）→ **refresh**（默认每 `refresh_interval=1s`）把缓冲区数据生成一个新的 **segment** 写入文件系统缓存，此时文档才**可被搜索**（近实时）→ **flush**（默认每 30 分钟或 translog 达到 512MB）把 segment **fsync 落盘**并清空 translog → **merge**（后台合并小 segment 为大 segment，同时物理删除被标记删除的文档）。因此 ES 是"近实时（NRT）"而非实时：写入后需等 refresh 才能搜到。
### 3.2 面试问答
**Q1：为什么 ES 是近实时的？**
A：结论——为了用"延迟 1 秒可见"换取极高写入吞吐。原因：每个 segment 都是一个完整索引结构，每次写入立刻新建 segment 代价过大，因此按 refresh 周期批量生成并借助 OS 文件系统缓存让 segment 可搜（不需要 fsync 即可搜索）。

**Q2：translog 的作用是什么，能做到不丢数据吗？**
A：translog 是 WAL，默认 `index.translog.durability=request`（每个请求 fsync，最安全），也可设 `async` 并按 `sync_interval`（默认 5s）批量 fsync（更快但可能丢最多 5s 数据）。结论——默认配置下已 ack 的写入不会丢；改成 async 就是用少量数据风险换吞吐。

**Q3：更新和删除文档是怎么实现的？**
A：ES 文档不可变：更新 = 标记旧文档删除 + 写入新文档（`_version`/`_seq_no` 递增）；删除只是标记（.del 文件）。真正的物理删除发生在 segment merge 时。因此频繁更新会导致大量垃圾数据与 merge 压力。

**Q4：merge 有什么代价，能调吗？**
A：merge 是 CPU 和磁盘 IO 大户，会与查询争抢资源（ES 用 `indices.store.throttle.max_bytes_per_sec` 限速，7.x 默认已较合理）。运维上应避免写入高峰做 force merge；对只读索引（如按天日志索引）可在写入停止后 `_forcemerge?max_num_segments=1` 合并以提升查询效率。

**Q5：refresh_interval 怎么设置？**
A：结论——写入密集的批量导入场景可设 `-1`（关闭自动 refresh）或 `30s`，显著提升写入吞吐，导入完成后手动 `_refresh`；但实时搜索场景保持 1s。注意 refresh 太频繁会生成大量小 segment，反而加重 merge 负担。
### 3.3 高频追问与陷阱
- 陷阱：以为写入返回成功就能搜到——需等 refresh（`?refresh=wait_for` 或 `true` 可强制，但会降低吞吐）。
- 陷阱：把 flush 当成"刷新搜索"——flush 是落盘并清空 translog，与搜索可见性无关。
- 追问：`refresh=wait_for` 与 `refresh=true` 的区别？前者等下一个 refresh 完成才返回（批量友好），后者立即强制 refresh（开销大）。
### 3.4 配置/命令示例
```json
PUT /order_idx/_settings
{ "index.refresh_interval": "30s", "index.translog.durability": "async", "index.translog.sync_interval": "5s" }
```

```bash
curl -X POST "localhost:9200/order_idx/_forcemerge?max_num_segments=1"   # 只读索引可做
```
### 3.5 一句话速记
buffer →（1s refresh）segment 可搜 →（30min/flush）落盘清 translog →（merge）物理删除；近实时的本质是"用 1 秒延迟换吞吐"。

## 4. Mapping 与字段类型设计
### 4.1 核心原理
常用类型：`text`（分词）、`keyword`（不分词，精确匹配/聚合/排序）、数值（`long`/`integer`/`double`；`scaled_float` 适合价格）、`date`（支持多种格式与 `format`）、`boolean`、`ip`、`nested`（对象数组独立索引，解决 object 数组的跨对象匹配问题）、`geo_point`。关键选项：`dynamic`（`true` 默认自动映射、`false` 忽略新字段但不索引、`strict` 直接报错）、`index`（是否建索引，false 则只能从 _source 取）、`doc_values`（是否列式存储）、`copy_to`（复制到目标字段，可模拟 `_all`）、多字段（`fields`）、`ignore_above`、`null_value`、`norms`。生产建议：**关闭 dynamic（用 strict）**，显式定义 Mapping，避免字段类型被推断错误（如把数字 ID 推断成 text 或被数字字符串污染）。
### 4.2 面试问答
**Q1：dynamic 三个取值怎么选？**
A：结论——生产用 `strict` 或 `false`。原因：默认 `true` 会让首个文档决定字段类型（如 `"1001"` 可能被推断为 text 或 long），一旦类型错误后续写入直接失败或数据不可查；`strict` 能在写入时立刻暴露问题。

**Q2：text 与 keyword 混用时怎么设计？**
A：用多字段：主字段 text 做全文检索，子字段 keyword 做精确匹配/聚合/排序（配 `ignore_above` 防长文本占用）。查询时注意：`term` 查 keyword 子字段，`match` 查 text 主字段，用错会"查不到或全不匹配"。

**Q3：object 数组为什么会有跨对象匹配问题？**
A：ES 会把对象数组"扁平化"存储，丢失对象边界，导致 `{"a":1,"b":2}` 与 `{"a":2,"b":1}` 无法区分（查询 a=1 AND b=2 会误命中）。解法：使用 `nested` 类型（配 nested 查询），代价是查询更慢、内存开销更大。

**Q4：copy_to 有什么用？**
A：把多个字段的值复制到一个目标字段，用一个查询同时检索多字段（类似旧版 `_all`），如把 title、desc、brand 复制到 `all_text`。注意：copy_to 不改变原字段，目标字段不能被写入到 _source，且会增加索引体积。

**Q5：Mapping 能改吗？**
A：已存在字段的类型**不能修改**（除少数兼容变更，如新增 `fields`、`ignore_above`、`null_value`）。要改类型只能新建索引 + `_reindex` + 别名切换，所以 Mapping 设计要一次到位。
### 4.3 高频追问与陷阱
- 陷阱：用 `integer` 存可能超过 21 亿的业务 ID，写入溢出报错。
- 陷阱：把 URL、日志正文等超长内容设为 keyword，索引体积暴涨。
- 追问：`index: false` 与 `enabled: false` 的区别？前者不建索引但保留可检索结构外的一切（_source 可取），后者连解析都不做（整个对象不索引、不存储，但仍保留在 _source 中）。
### 4.4 配置/命令示例
```json
PUT /sku_idx
{ "mappings": {
  "dynamic": "strict",
  "properties": {
    "sku_id":   { "type": "keyword" },
    "price":    { "type": "scaled_float", "scaling_factor": 100 },
    "tags":     { "type": "keyword" },
    "comments": { "type": "nested", "properties": { "user": {"type":"keyword"}, "score": {"type":"byte"} } }
}}}
```
### 4.5 一句话速记
生产用 dynamic=strict，text 配 keyword 子字段，数组对象用 nested，类型一旦定下不能改只能 reindex。

## 5. Query DSL 与 query/filter 上下文
### 5.1 核心原理
常用查询：`term`/`terms`（不分词精确匹配，针对 keyword）、`match`（分词后查询）、`match_phrase`（短语，`slop` 允许间隔）、`multi_match`（多字段）、`range`（范围）、`bool`（`must`/`should`/`must_not`/`filter`）、`wildcard`（通配，慎用）、`exists`（字段存在）、`ids`。**两种上下文**：**query context**（参与打分，`_score` 有意义，`must`/`should` 属于它）、**filter context**（不打分、结果可被**filter cache** 缓存，适合状态、时间范围、分类等精确筛选）。`constant_score` 可把 filter 包成打分恒定的查询，`boost` 调整权重。
### 5.2 面试问答
**Q1：为什么 filter 比 query 快？**
A：结论——filter 不打分且结果可缓存（bitset 缓存）。原因：打分需要遍历 Posting List 计算 TF/IDF 等统计量；filter 只需判断"是否命中"，且相同过滤条件的 bitset 可跨查询复用。因此"能用 filter 就别用 must"是 ES 性能优化的第一条。

**Q2：term 和 match 怎么选？**
A：`term` 对输入**不做分词**，直接在倒排表里找完全相同的词项，因此只适用于 keyword（或确定已分词的场景）；`match` 会先对输入分词再匹配，适用于 text。常见事故：用 `term` 查 text 字段导致只命中"恰好等于整个词"的文档。

**Q3：bool 的 should 什么时候必须满足？**
A：`should` 默认"满足更好但不强制"（会贡献分数）；若 bool 中只有 should 子句（没有 must/filter），则至少满足其一（等价 min_should_match=1）；可用 `minimum_should_match` 显式控制。

**Q4：wildcard 为什么危险？**
A：通配符（尤其是前缀 `*` 开头）无法利用词典定位，需要遍历大量词项，CPU 与响应时间不可控，容易引发慢查询与集群抖动。替代：前缀查询用 `prefix`/`edge_ngram`、模糊匹配用 `match` + 分词、需要搜索建议用 completion suggester。

**Q5：怎么只返回需要的字段？**
A：用 `_source` 过滤（`"_source": ["sku_id","price"]`）或 `stored_fields`；聚合查询可设 `"size": 0` 不返回文档只返回聚合结果，显著减少网络与序列化开销。
### 5.3 高频追问与陷阱
- 陷阱：把 `must` 用于日期范围或状态过滤，白白消耗打分与缓存能力。
- 陷阱：`must_not` 在 filter 中不能缓存（negative 查询需要取补集），大量使用会变慢。
- 追问：`_source` 与 `doc_values` 的读取路径？排序聚合走 doc_values，返回原始内容走 _source。
### 5.4 配置/命令示例
```json
GET /sku_idx/_search
{ "size": 20,
  "_source": ["sku_id", "price", "brand"],
  "query": { "bool": {
    "filter": [
      { "term":  { "status": "ON_SALE" } },
      { "range": { "created_at": { "gte": "now-7d/d" } } }
    ],
    "must": [ { "match": { "title": { "query": "蓝牙耳机", "boost": 2 } } } ],
    "should": [ { "term": { "tags": "hot" } } ]
  }}}
```
### 5.5 一句话速记
精确筛选用 filter（可缓存不打分），全文检索用 must/match；term 不分词查 keyword，match 分词查 text；wildcard 是慢查询杀手。

## 6. 聚合与相关性打分
### 6.1 核心原理
**聚合**分三类：**metric**（`avg`/`sum`/`min`/`max`/`stats`/`cardinality`/`percentiles`）、**bucket**（`terms`/`date_histogram`/`histogram`/`range`/`filters`/`nested`）、**pipeline**（对聚合结果再计算，如 `derivative`、`cumulative_sum`）。`terms` 聚合默认只返回 top 10 个桶（`size` 控制），全量分桶统计用 `composite` 聚合配合 `after` 游标。`cardinality`（去重计数）底层是 **HyperLogLog++** 近似算法：`precision_threshold`（默认 3000）以内的基数精确，超过则为估算值（误差约 1%~5%）。**打分**：ES 5.0 起默认 **BM25**（替代 TF-IDF），核心是词频饱和（参数 k1）与文档长度归一（参数 b）；可调手段：字段级 `boost`、`function_score`（`field_value_factor`、`gauss`/`linear` 衰减、`random_score`、`script_score`）、`rescore` 二次打分。
### 6.2 面试问答
**Q1：terms 聚合为什么只返回 10 个桶？怎么取全量？**
A：默认 `size=10`，返回按 doc_count 排序的前 10 个；要全量或分页统计用 `composite` 聚合（支持 `after` 游标、可按多字段组合分页），`partition` 参数还能并行拉取。

**Q2：cardinality 准确吗？**
A：不精确，是 HyperLogLog++ 的概率估算；当实际基数小于 `precision_threshold`（默认 3000）时结果准确，超过后约 1%~5% 误差。结论——UV/去重计数类指标可用，但不要用于财务口径的精确计数（那要用 MySQL 或精确去重方案）。

**Q3：BM25 相比 TF-IDF 的改进？**
A：① 词频饱和——同一个词出现 100 次并不会比出现 10 次重要 10 倍（k1 控制）；② 文档长度归一——长文档不会因为词多而占优（b 控制，默认 0.75）；③ 更贴近实际相关性，且无需复杂的字段长度归一化预处理。

**Q4：怎么把某个字段的权重调高？**
A：三种方式：① 查询时 `"boost": 2`；② 索引时 `boost`（不推荐，改动需 reindex）；③ `function_score` 按业务规则加权（销量、好评率、时效性衰减），这是电商搜索排序最常用的做法。

**Q5：聚合会 OOM 吗？**
A：会。常见原因：① text 字段聚合触发 fielddata 加载堆内存；② 高基数 terms 聚合产生海量桶（`size` 很大或桶嵌套）；③ 深分页 + 大 `size`。防护：使用 doc_values 的 keyword、限制桶数量、拆分查询、调大堆并配置断路器（`indices.breaker.fielddata.limit` 默认 40% 堆，`request` 60%，`total` 70%）。

**Q6：percentiles 聚合适合什么场景？**
A：统计 P95/P99 等分位值（如接口耗时分布）。注意它同样是近似算法（TDigest），且对数据分布有假设；跨分片时由协调节点合并各分片的近似结果，误差会叠加。
### 6.3 高频追问与陷阱
- 陷阱：用 `terms` 聚合统计"全量品牌数"却忽略 size 限制，得出错误结论。
- 陷阱：在 text 字段上做聚合/排序，触发 fielddata 熔断（`CircuitBreakingException`）。
- 追问：`doc_count_error_upper_bound` 是什么？terms 聚合在分片级取 top N 再合并时可能漏掉真实的低频词，该字段表示 doc_count 的误差上界，非 0 说明结果不精确。
### 6.4 配置/命令示例
```json
GET /order_idx/_search
{ "size": 0,
  "aggs": {
    "by_brand": { "terms": { "field": "brand", "size": 50 },
      "aggs": { "avg_price": { "avg": { "field": "price" } } } },
    "uv": { "cardinality": { "field": "user_id", "precision_threshold": 10000 } },
    "per_day": { "date_histogram": { "field": "created_at", "calendar_interval": "1d" } }
  },
  "query": { "range": { "created_at": { "gte": "now-30d/d" } } }
}
```
### 6.5 一句话速记
metric 算值、bucket 分组、pipeline 再算；terms 默认 top 10、全量用 composite；cardinality 是近似值，BM25 是默认打分模型。

## 7. 分页与深度分页三方案
### 7.1 核心原理
`from + size` 的执行方式是"每个分片都取 from+size 条，协调节点归并后取前 from+size 条再截取"，因此分页越深，协调节点内存与网络开销越大；受 `index.max_result_window`（默认 **10000**）限制，超出直接报错。**scroll**（滚动）为查询创建快照并保持上下文，适合大批量导出，但占用资源且数据不是实时的；**search_after + PIT**（Point In Time，7.10+ 的 PIT API 替代旧版 `scroll` 的一些场景）通过上一页最后一条的排序值继续往后取，状态轻、可并行、适合深分页与稳定遍历。

| 方案 | 适用场景 | 优点 | 缺点 |
| --- | --- | --- | --- |
| from + size | 前几页（如 ≤ 1000 条） | 简单、实时 | 深分页开销大，受 max_result_window 限制（默认 10000） |
| scroll | 全量导出、离线批处理 | 遍历稳定、吞吐高 | 快照非实时、上下文占用资源、不适合线上交互 |
| search_after + PIT | 深分页、无限下拉、线上查询 | 无状态、开销稳定、可并行 | 需唯一排序键，不能跳页 |
### 7.2 面试问答
**Q1：为什么深分页会拖垮集群？**
A：`from=10000&size=10` 时每个分片都要排序并返回 10010 条候选，协调节点还要归并 N×10010 条数据，分片越多放大越明显；同时大 `size` 会占用大量堆内存，可能触发熔断。

**Q2：线上要做"跳页到第 100 页"怎么办？**
A：结论——ES 不适合跳页。可选方案：① 限制最大可跳页数（如前 100 页）并在 UI 上引导改用筛选条件；② 用 search_after 做"下一页"语义；③ 在应用层做二级索引（把 ID 结果缓存到 Redis/MySQL 再回表）；④ 调整 `max_result_window`（不推荐，只是把你的限制放宽到内存里）。

**Q3：scroll 和 search_after 怎么选？**
A：需要"一次拉全量、允许非实时、可以长时间持有上下文"用 scroll；需要"实时分页、深度未知、多用户并发"用 search_after + PIT。scroll 用完必须显式 `_clear_scroll`（默认保留 5 分钟），否则会一直占用资源。

**Q4：search_after 有什么使用约束？**
A：① 必须有**唯一且稳定**的排序（通常是业务 ID 或 `_id`，避免文档更新导致顺序抖动）；② 不能跳页，只能顺序翻；③ 需要配合 PIT 保证分片快照一致（否则分片变动会导致漏数据/重复）。

**Q5：为什么 PIT 是必要的？**
A：深分页期间如果发生 refresh 或分片迁移，排序与分片状态会变化，导致翻页重复或漏数据；PIT 在指定时刻冻结索引视图，保证遍历的一致性。
### 7.3 高频追问与陷阱
- 陷阱：为了支持深分页把 `max_result_window` 改成 100 万，结果每次查询都堆内存爆掉。
- 陷阱：scroll 上下文忘记清理，导致堆中堆积大量搜索上下文（`_nodes/stats/indices/search` 可看 open_contexts）。
- 追问：`track_total_hits` 有什么用？默认超过 10000 后 total 只返回 10000（7.x 起默认 `10000`），需要精确总数时设 `true`（开销大），只需"是否超过阈值"可设具体数字。
### 7.4 配置/命令示例
```json
GET /order_idx/_search
{ "size": 100, "track_total_hits": false,
  "pit": { "id": "PIT_ID", "keep_alive": "1m" },
  "sort": [ { "created_at": "desc" }, { "_id": "asc" } ],
  "search_after": [ 1714000000000, "ord_10086" ] }
```

```bash
curl -X DELETE "localhost:9200/_search/scroll" -H 'Content-Type: application/json' -d '{"scroll_id":["ID"]}'
```
### 7.5 一句话速记
from+size 只看前几页，导出用 scroll（记得清理），深分页用 search_after + PIT（需唯一排序键）。

## 8. 集群、分片设计与并发控制
### 8.1 核心原理
**节点角色**（7.9+ 用 `node.roles`）：`master`（管理集群元数据、索引创建、分片分配，需奇数个 eligible 节点保持 quorum）、`data`（存数据，可细分为 data_hot/warm/cold/frozen）、`ingest`（写入前预处理）、`coordinating`（只做请求分发与结果归并，默认每个节点都具备）。**分片设计**：单分片大小经验值 10~50GB（官方偏保守，常用 20~40GB），分片数 = 预估数据量 / 单分片大小，同时满足"分片数 ≥ 数据节点数"以保证并行；**分片数不可变**。**路由**：默认 `hash(_routing) % 主分片数`，可自定义 `_routing` 把同一用户/租户数据路由到同一分片（减少分片数、提升聚合效率，但要注意倾斜）。**副本**：可动态调整，提供故障转移与读扩展。**脑裂**：7.0 起改用 Zen2 自动选主（`discovery.seed_hosts` + `cluster.initial_master_nodes`），不再需要手工设置 `minimum_master_nodes`，但正式集群仍必须**至少 3 个 master-eligible 节点**。**并发控制**：`_version`（乐观锁）与 7.x 推荐的 `if_seq_no` + `if_primary_term`。
### 8.2 面试问答
**Q1：分片数怎么定？**
A：步骤：① 估算总数据量与增长（含副本与索引膨胀系数）；② 按单分片 20~40GB 算主分片数；③ 检查是否 ≥ 数据节点数 ×1（并行度）；④ 评估分片总数上限（单节点分片数过多会消耗大量文件句柄与堆内存，经验上每 GB 堆对应 20~30 个分片以内）；⑤ 写入量大的索引可适当多分片以提升写入并行。结论——宁可少而大，也不要多而碎。

**Q2：副本数怎么定？**
A：结论——生产至少 1 个副本（保证节点故障时数据与服务可用），高读场景可加到 2；写入密集的批量导入可临时设为 0 再改回 1（省去副本写入开销）。注意副本会成倍占用磁盘与写入放大。

**Q3：`_routing` 自定义路由的利弊？**
A：利：同一租户/用户的数据落到同一分片，查询只打一个分片（减少 scatter-gather），聚合更快。弊：容易数据倾斜（大客户占满一个分片），且改路由规则等于迁移数据。适合"按租户/用户维度查询"明确的场景。

**Q4：集群变 yellow 和 red 分别意味着什么？**
A：yellow = 所有主分片已分配、部分**副本**未分配（常见于单节点集群）；red = 存在**主分片**未分配（该索引部分数据不可用）。排查顺序：`_cluster/allocation/explain` 看未分配原因（节点离线、磁盘水位、分配规则、分片数超过节点限制）、检查磁盘水位（85%/90%/95% 三档）、检查节点是否被 exclude。

**Q5：7.x 还需要配 minimum_master_nodes 吗？**
A：不需要。7.0 起由 Zen2 自动管理选举与 quorum，配置项已废弃；但必须正确设置 `discovery.seed_hosts` 与首次启动的 `cluster.initial_master_nodes`，否则可能形成"多个独立集群"。集群规模仍建议 master-eligible 节点数为奇数（3 或 5）。

**Q6：怎么安全地更新一条文档？**
A：用乐观锁：① 读取文档拿到 `_seq_no` 与 `_primary_term`；② 更新时带上 `if_seq_no`/`if_primary_term`；③ 版本不匹配返回 409，由业务重试或放弃。旧版用 `_version` 参数（`version_type=external` 常用于外部数据同步）。
### 8.3 高频追问与陷阱
- 陷阱：把单索引分片数设成 1000+，导致集群元数据膨胀、查询归并开销大、恢复慢。
- 陷阱：只部署 1 个 master-eligible 节点，节点重启期间元数据操作全部失败。
- 追问：索引生命周期（ILM）解决什么？按 hot/warm/cold/delete 阶段自动 rollover、迁移节点、shrink、删除，是日志类索引的标准治理手段。
### 8.4 配置/命令示例
```bash
curl -X GET "localhost:9200/_cluster/allocation/explain?pretty"
curl -X PUT "localhost:9200/order_idx/_settings" -H 'Content-Type: application/json' \
  -d '{"index":{"number_of_replicas":1}}'
```

```json
POST /order_idx/_doc/1?if_seq_no=12&if_primary_term=3
{ "status": "PAID" }
```
### 8.5 一句话速记
master 节点要奇数、分片数不可变（20~40GB 一片）、副本可调；yellow 是副本缺失、red 是主分片缺失；7.x 起自动选主，并发控制用 seq_no + primary_term。

## 9. 性能优化与 MySQL 数据同步
### 9.1 核心原理
**写入优化**：`_bulk` 批量（单批 5~15MB，避免过大）、`refresh_interval` 调大或关闭、批量导入时副本数设 0、`translog.durability=async`、避免逐条写入、合理设置分片数、禁止 swap（`bootstrap.memory_lock: true`）、去掉不必要的字段索引。**查询优化**：用 filter 上下文、避免深分页、避免 `script` 与 `wildcard`、控制返回字段（`_source` 过滤 + `size: 0`）、限制聚合桶数量、用 `search_after` 替代深分页、必要时用 `rescore` 只对前 N 条二次打分。**Mapping 优化**：只过滤的字段 `index: true` 但 `doc_values: false`、完全不用检索的字段 `index: false`、不需要排序聚合的可关 `doc_values`、长文本不设 keyword。**MySQL → ES 同步**：同步双写（简单但事务一致性差）、MQ 异步（解耦、最终一致）、Canal/Debezium 订阅 binlog（无侵入、准实时，最主流）、Logstash/Flink CDC（批流一体）。
### 9.2 面试问答
**Q1：bulk 批量多大合适？**
A：结论——按"字节数"而非"条数"控制，单批 5~15MB（官方常见建议），并配合压测调整；过大会导致内存与超时风险（协调节点要缓冲整批请求），过小则请求数过多。批量导入时并发别开太多（4~8 个并发即可），避免 merge 与写入争抢。

**Q2：怎么优化一个慢查询？**
A：排查链路：① 慢查询日志定位（`index.search.slowlog.threshold.query.warn=10s`、`...fetch.warn=1s`）；② `profile: true` 看各阶段耗时（query/fetch 分别计）；③ 常见原因与对策——深分页（改 search_after）、`wildcard`/正则（改分词或 ngram）、脚本计算（改为索引期计算字段）、分片过多（合并或按时间分索引）、聚合基数过高（用 composite 或预聚合）、`must` 换 `filter`。

**Q3：ES 与 MySQL 怎么同步最可靠？**
A：结论——优先 **Canal/Debezium 订阅 binlog**（业务无侵入、可重放、准实时），配合 ES 消费端的幂等（按主键 upsert）与对账任务。同步双写只在"要求极低延迟且业务可容忍不一致"时使用，必须有补偿（本地消息表 + 定时对账）；MQ 异步适合已有 MQ 体系且能接受秒级延迟的场景。

**Q4：怎么保证 ES 与 MySQL 的最终一致？**
A：① 每条同步操作带业务版本号或 binlog 位点，保证乱序消费不会用旧数据覆盖新数据（在 ES 侧用 `_version`/外部版本号 `version_type=external` 或应用层比大小）；② 定时全量/增量对账任务比对（对差异按主键重推）；③ 关键业务在写 MySQL 后立刻触发刷新或提供"强制读主库"的降级路径。

**Q5：ES 更新频繁的场景怎么优化？**
A：① 减少更新频率（合并变更、批量写）；② 用 `_update_by_query` 或 bulk update 而非逐条；③ 对不需要检索的字段设 `index: false` 减少索引更新开销；④ 使用按时间滚动的索引，让"旧索引只读"（可 force merge）；⑤ 评估是否应把高频变化的部分留在 MySQL/Redis，只在 ES 保留检索所需字段。
### 9.3 高频追问与陷阱
- 陷阱：为了"实时"把 `refresh_interval` 设为 1s 却按 `?refresh=true` 每条写入强制刷新，吞吐直接崩掉。
- 陷阱：ES 里直接存大文本原文，导致索引与 _source 体积膨胀，内存压力大。
- 追问：为什么要禁用 swap？JVM 堆被换出会导致 GC 与响应时间急剧恶化；`bootstrap.memory_lock: true` + 系统层面关闭 swap。
### 9.4 配置/命令示例
```yaml
# elasticsearch.yml 关键项
bootstrap.memory_lock: true
discovery.seed_hosts: ["es01", "es02", "es03"]
cluster.initial_master_nodes: ["es01", "es02", "es03"]
node.roles: ["master", "data"]
```

```bash
# 批量导入（每批 10MB 左右，并发 4~8）
split -l 5000 data.json part_ && for f in part_*; do curl -s -H "Content-Type: application/x-ndjson" \
  -XPOST "localhost:9200/_bulk" --data-binary @$f; done
```
### 9.5 一句话速记
写入靠批量 + 大 refresh_interval + 副本 0，查询靠 filter + 控字段 + 避深分页，同步首选 binlog 订阅 + 幂等 upsert + 定时对账。

## 10. 常见线上问题排查
### 10.1 核心原理
五类高频故障：① **分片未分配**（`_cluster/allocation/explain` 定位：节点离线、磁盘水位、分配规则、`index.routing.allocation.enable=none`）；② **集群 red/yellow**（red 查主分片为何未分配，yellow 查副本；单节点集群 yellow 是正常现象，可设副本 0 消除）；③ **慢查询**（慢日志 + profile）；④ **OOM 与熔断**（`CircuitBreakingException` 常见于大聚合、深分页、字段 fielddata；`Data too large` 说明断路器生效）；⑤ **写入拒绝**（`EsRejectedExecutionException`，写线程池队列满，通常由写入速率超过磁盘/merge 能力引起）。配套观测：`_cluster/health`、`_cat/indices`、`_nodes/stats`、`_cat/thread_pool?v&h=node_name,name,active,queue,rejected`。
### 10.2 面试问答
**Q1：集群变 red 怎么处理？**
A：① `_cluster/health?level=indices` 定位具体索引；② `_cat/shards` 找到未分配分片；③ `_cluster/allocation/explain` 看原因；④ 若节点临时故障，修复后分片会自动分配（可用 `_cluster/reroute` 手动重试分配）；⑤ 若数据确实丢失（副本也损坏），只能从上游（MySQL/binlog/快照）重建。原则：**先恢复节点，再考虑手动 reroute，最后才接受数据重建**。

**Q2：磁盘水位告警怎么处理？**
A：ES 有三档水位：`low`（默认 85%，不再向该节点分配新分片）、`high`（90%，尝试把分片迁走）、`flood_stage`（95%，把索引设为只读 `index.blocks.read_only_allow_delete`）。处理：删除或归档旧索引（ILM）、扩容磁盘、临时调高水位（应急，别当常态）、检查是否索引膨胀异常。

**Q3：写入被拒绝（429）怎么办？**
A：原因：写线程池队列（`thread_pool.write.queue_size` 默认 200）满。对策：① 降低客户端并发或加入退避重试；② 检查是否 merge 或磁盘过慢导致积压；③ 评估分片数是否过少（写入并行不足）或过多（merge 压力大）；④ 必要时扩容数据节点。

**Q4：JVM 堆怎么设置？**
A：结论——堆不超过物理内存的 50%，且**最大不超过 31GB**（压缩指针界限，超过后指针变宽反而更耗内存）；剩余内存留给文件系统缓存供 Lucene 使用。监控 GC（7.x 默认 G1，`jvm.gc.collectors.old.collection_time_in_millis`），频繁 Full GC 通常意味着 fielddata 过大或聚合过重。

**Q5：日志类索引怎么治理？**
A：使用 **ILM**：hot（当日索引、写入）→ warm（次日不再写、可 force merge / shrink）→ cold（低频查询、迁移到慢节点）→ delete（按保留期删除），配合索引模板与 rollover 别名，避免"一个索引无限增长导致分片过大、删除困难"。

**Q6：怎么监控 ES 集群？**
A：核心指标：集群状态（green/yellow/red）、未分配分片数、节点 CPU/磁盘水位、写入与查询拒绝数、JVM 堆使用与 GC 时间、查询延迟分位（`indices.search.query_time_in_millis`）、段数量与 merge 时间、open scroll 上下文数。工具：Kibana Stack Monitoring、Prometheus + elasticsearch_exporter，告警应覆盖"red 状态、磁盘 >85%、JVM 堆 >85%、429 拒绝激增"。
### 10.3 高频追问与陷阱
- 陷阱：red 时直接删索引——先确认是不可恢复还是暂态（节点重启中）。
- 陷阱：把 `flood_stage` 只读块当最终方案，忘记解除导致后续写入全失败（需 `index.blocks.read_only_allow_delete: null`）。
- 追问：`_cluster/reroute` 什么时候用？自动分配失败（如临时磁盘满、节点抖动导致分配重试次数耗尽）时可手动 `POST _cluster/reroute?retry_failed=true`。
### 10.4 配置/命令示例
```bash
curl -X GET "localhost:9200/_cat/thread_pool/write?v&h=node_name,active,queue,rejected"
curl -X GET "localhost:9200/_cat/allocation?v"
curl -X POST "localhost:9200/_cluster/reroute?retry_failed=true"
curl -X PUT "localhost:9200/old_idx/_settings" -H 'Content-Type: application/json' \
  -d '{"index":{"blocks.read_only_allow_delete":null}}'
```
### 10.5 一句话速记
red 查主分片、yellow 查副本，allocation/explain 是确诊入口；磁盘 85/90/95 三档水位，429 查写队列，堆 ≤31GB，日志索引上 ILM。

## 自测题（含答案要点）
1. **为什么 ES 是近实时的？** refresh 默认 1s 生成可搜索的 segment，用延迟换写入吞吐。
2. **倒排索引三部分是什么？** Term Dictionary（磁盘）、Term Index（FST，内存）、Posting List（delta 编码 + 跳表求交）。
3. **doc_values 与 fielddata 的区别？** 前者磁盘列式正排（默认对非 text 开启），后者堆内存正排（text 聚合时构建，易 OOM）。
4. **text 与 keyword 怎么选？** text 分词做全文检索，keyword 做精确匹配/聚合/排序，需兼顾时用多字段。
6. **translog 的两种 durability？** request（默认，每请求 fsync，不丢）与 async（按 sync_interval 默认 5s 批量落盘，可能丢几秒）。
8. **dynamic 三个取值？** true（自动映射）、false（忽略新字段）、strict（报错，生产推荐）。
9. **object 数组的问题与解法？** 扁平化丢失对象边界导致跨对象误命中，用 nested 类型 + nested 查询。
10. **filter 上下文为什么快？** 不打分且结果可被 bitset 缓存；能 filter 就不用 must。
12. **terms 聚合只返回 10 个桶怎么办？** 调 size 或改用 composite 聚合（支持 after 游标全量分页）。
13. **cardinality 精确吗？** HyperLogLog++ 近似，precision_threshold（默认 3000）内精确，超出误差 1%~5%。
14. **BM25 相比 TF-IDF 的改进？** 词频饱和（k1）+ 文档长度归一（b，默认 0.75）。
15. **深度分页三种方案及取舍？** from+size 仅前几页、scroll 用于导出（需清理）、search_after+PIT 用于深分页（需唯一排序键）。
17. **yellow 与 red 的含义？** yellow 是副本未分配（单节点正常），red 是主分片未分配（数据不可用）。
19. **并发更新怎么保证不覆盖？** 乐观锁 `if_seq_no` + `if_primary_term`（旧版用 `_version`）。
21. **JVM 堆怎么设？** ≤ 物理内存 50% 且不超过 31GB（压缩指针界限），其余留给文件系统缓存。
22. **写入 429 的原因与对策？** 写线程池队列满（默认 200）；降并发 + 退避重试 + 查 merge/磁盘 + 评估分片与扩容。

## 背诵卡片
1. ES 是近实时检索引擎，不是数据库，权威数据在 MySQL。
3. 主分片数创建后不可变，副本数可动态调整。
5. 倒排三件套：Term Dictionary、Term Index（FST 常驻内存）、Posting List。
6. Posting List 用 delta 编码压缩，多词求交靠跳表加速。
7. doc_values 在磁盘、默认对非 text 字段开启，用于排序聚合。
8. fielddata 在堆内存，text 聚合的 OOM 元凶。
9. text 分词、keyword 不分词；两者兼顾用 fields 多字段。
10. ignore_above 默认 256，超长 keyword 值不索引。
11. 中文分词：写入 ik_max_word、查询 ik_smart（或按需统一）。
12. 写入链路：buffer → refresh(1s) → segment 可搜 → flush(30min/512MB) → merge。
14. translog durability=request 不丢数据，async 可能丢约 5s。
15. 文档不可变：更新 = 删除 + 新建，物理删除发生在 merge。
16. 批量导入：refresh_interval 调大或 -1、副本设 0、用 bulk。
17. Mapping 已有字段类型不能改，只能 reindex。
20. filter 上下文不打分、可缓存；must 会打分。
22. wildcard 前缀通配是慢查询杀手，改用 ngram/suggester。
23. terms 聚合默认返回 top 10，全量统计用 composite。
24. cardinality 是 HyperLogLog++ 近似值（阈值默认 3000）。
25. BM25 默认打分模型，k1 控词频饱和、b 控长度归一（0.75）。
26. 排序权重用 function_score 做业务加权（销量、时效衰减）。
27. from+size 受 max_result_window 限制（默认 10000）。
28. 导出用 scroll（用完清理上下文），深分页用 search_after + PIT。
29. search_after 需要唯一稳定排序键，且不能跳页。
30. 单分片建议 20~40GB，分片数不可变，宁可少而大。
31. _routing 自定义路由可减少查询分片，但易倾斜。
32. yellow = 副本缺失，red = 主分片缺失，用 allocation/explain 定位。
33. 磁盘水位三档：85% low、90% high、95% flood_stage（索引只读）。
34. 7.0 起 Zen2 自动选主，不再需要 minimum_master_nodes。
36. 写入优化：bulk 5~15MB、refresh 调大、副本 0、async translog、禁 swap。
37. 查询优化：filter、控返回字段、size:0、避脚本与深分页。
38. 慢查询定位：慢日志 + profile + 分片数与聚合基数排查。
39. MySQL→ES 首选 Canal/Debezium 订阅 binlog + 幂等 upsert + 定时对账。
40. 堆 ≤ 31GB 且不超过物理内存 50%，日志索引用 ILM 治理。
