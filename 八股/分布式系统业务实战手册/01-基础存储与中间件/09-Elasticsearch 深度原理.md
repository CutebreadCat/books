# Elasticsearch 深度原理

> Elasticsearch 是分布式的搜索和分析引擎，面试中高频考点包括倒排索引、分片机制、写入流程、查询原理、性能优化。本篇从 Lucene 底层到 ES 集群层面逐层拆解。

---

## 目录

- [1. 整体架构与核心概念](#1-整体架构与核心概念)
- [2. 倒排索引与 Lucene 底层](#2-倒排索引与-lucene-底层)
- [3. 写入流程与数据可靠性](#3-写入流程与数据可靠性)
- [4. 查询流程与搜索机制](#4-查询流程与搜索机制)
- [5. 分词与映射](#5-分词与映射)
- [6. 聚合与排序](#6-聚合与排序)
- [7. 性能优化](#7-性能优化)
- [8. 集群与脑裂](#8-集群与脑裂)
- [9. 面试追问应对](#9-面试追问应对)

---

## 1. 整体架构与核心概念

### 1.1 核心概念

```
Document（文档）：
  ES 的最小数据单元，JSON 格式
  如：{"title": "Java 并发", "author": "张三", "price": 99}

Index（索引）：
  文档的集合，类似数据库的表（但不完全等同）
  一个索引由多个分片（Shard）组成

Type（已废弃）：
  ES 6.x 开始单索引单类型，7.x 彻底移除
  原因：同索引不同类型字段冲突，底层 Lucene 不支持

Shard（分片）：
  索引的数据拆分单元，每个分片是一个独立的 Lucene 索引
  Primary Shard：主分片，负责写入
  Replica Shard：副本分片，负责读取 + 故障转移
  分片数创建后不可改（索引重建才能改）

Node（节点）：
  Master：管理集群元数据（索引创建、节点加入、分片分配）
  Data：存储数据，执行查询/写入
  Coordinating：接收请求，转发到 Data 节点，汇总结果（默认所有节点都兼具）
  Ingest：预处理文档（如 Pipelines）

Cluster（集群）：
  多个节点组成，通过 cluster.name 识别
  通过选举机制选出 Master 节点
```

### 1.2 与关系数据库的对比

```
  MySQL              Elasticsearch
  ─────────────────────────────────────
  Database     →     Index
  Table        →     Index（Type 已废弃）
  Row          →     Document
  Column       →     Field
  Schema       →     Mapping
  SQL          →     DSL（Query DSL）
  Index        →     Inverted Index（倒排索引）

关键差异：
  1. ES 无事务，无 JOIN（可用 nested / parent-child 但性能差）
  2. ES 写入近实时（默认 1s refresh，非实时）
  3. ES 最终一致，非强一致
  4. ES 适合搜索和分析，不适合频繁更新和复杂事务
```

---

## 2. 倒排索引与 Lucene 底层

### 2.1 倒排索引（Inverted Index）

```
正向索引（MySQL）：
  文档 ID → 文档内容
  查询 "Java"：扫描所有文档，判断内容是否包含 → 慢

倒排索引（ES/Lucene）：
  词项 → 文档 ID 列表
  "Java" → [doc_1, doc_3, doc_7]
  "并发" → [doc_1, doc_5]
  查询 "Java 并发"：取交集 [doc_1] → 快

倒排索引结构：
  Term Dictionary（词项字典）：所有词项的有序列表
  Posting List（倒排列表）：每个词项对应的文档 ID 列表 + 位置信息 + 词频
  Term Index（词项索引）：FST（Finite State Transducer）压缩的前缀树，加速词项查找

FST 的作用：
  将 Term Dictionary 加载到内存，实现 O(1) 级别词项查找
  压缩比高，数百万词项只需几 MB 内存

Segment（段）：
  Lucene 的索引子单元，每个 Segment 是独立的倒排索引
  写入时创建新 Segment，不修改旧 Segment（不可变设计）
  查询时合并多个 Segment 结果
  Segment 会定期合并（Merge），减少 Segment 数量
```

### 2.2 存储结构

```
Lucene 索引文件：
  .fdt / .fdx：存储原始文档（Field Data / Field Index）
  .tim / .tip：Term Dictionary / Term Index（FST）
  .doc：Posting List（文档 ID 列表）
  .pos：词项位置（用于 phrase query）
  .pay：payload（额外信息，如词频）
  .dvd / .dvm：Doc Values（列式存储，用于排序和聚合）

为什么 Lucene 不可变？
  1. 不需要加锁，并发性能好
  2. 缓存友好（Segment 不变，缓存不失效）
  3. 增量写入新 Segment，旧 Segment 可被快照复用

代价：
  更新文档 = 删除旧文档 + 写入新文档（标记删除 + 新 Segment）
  删除文档 = 标记删除（段合并时物理删除）
```

---

## 3. 写入流程与数据可靠性

### 3.1 写入路径

```
客户端写入文档 → Coordinating Node 接收
  ↓
路由计算：shard = hash(routing) % primary_shard_count
  routing 默认是 _id，可自定义
  ↓
Primary Shard 写入：
  1. 写入内存缓冲区（In-memory Buffer）
  2. 写入 Translog（WAL，写前日志，保证持久化）
  3. 返回成功给客户端（默认异步，可配置 wait_for_active_shards）
  ↓
Replica Shard 同步（异步）：
  Primary 将操作同步给 Replica
  当所有 Replica 确认后，Primary 返回最终成功（ configurable）
  ↓
Refresh（默认 1s）：
  In-memory Buffer 中的数据写入文件系统缓存（FileSystem Cache）
  生成新的 Segment，此时数据可被搜索（但还没落盘）
  注意：refresh 是轻量操作，不 fsync
  ↓
Flush（默认 30min 或 Translog 满）：
  文件系统缓存中的 Segment 数据 fsync 到磁盘
  清空 Translog
  数据真正持久化
```

### 3.2 数据可靠性机制

```
Translog（事务日志）：
  写入顺序：先写 Translog，再写内存 Buffer
  作用：如果节点崩溃，内存 Buffer 数据丢失，但 Translog 可恢复
  刷盘策略：
    async（默认）：每 5s fsync 一次 Translog（性能高，最多丢 5s 数据）
    request：每次请求都 fsync（性能低，不丢数据）

一致性模型：
  ES 默认是最终一致（async 复制）
  可选参数：
    wait_for_active_shards=1（默认）：Primary 写入成功即返回
    wait_for_active_shards=all：所有 Replica 写入成功才返回

写入冲突：
  ES 使用乐观并发控制（version / seq_no + primary_term）
  更新时携带 if_seq_no 和 if_primary_term，版本冲突则报错
  应用层需处理冲突（重试或合并）
```

### 3.3 Segment Merge

```
为什么需要 Merge：
  写入不断创建新 Segment，Segment 过多导致查询慢（需合并多个 Segment 结果）
  删除操作只是标记删除，Merge 时物理删除

Merge 触发条件：
  1. 自动：后台线程按策略合并（根据 Segment 大小和数量）
  2. 强制：Force Merge API（生产环境慎用，会阻塞写入）

Merge 的影响：
  消耗 CPU 和 I/O
  写入大的 Segment 时磁盘占用翻倍（旧 Segment + 新 Segment 共存）
  注意：不要在高写入时段强制 Merge

段合并策略：
  Tiered Merge Policy：优先合并小 Segment，逐步合并成更大的 Segment
  目标：控制 Segment 总数，减少查询时的合并开销
```

---

## 4. 查询流程与搜索机制

### 4.1 查询阶段（Query-then-Fetch）

```
ES 默认搜索分为两个阶段：

Query Phase（查询阶段）：
  1. Coordinating Node 接收请求，解析 DSL
  2. 将请求广播到所有相关分片（Primary + Replica，按负载选择）
  3. 每个分片在本地执行查询，返回 Top N 的文档 ID + 排序值（_score）
  4. Coordinating Node 汇总所有分片结果，按 _score 排序，取全局 Top N

Fetch Phase（取回阶段）：
  5. Coordinating Node 向包含目标文档的分片发送多 GET 请求
  6. 各分片返回完整文档内容
  7. Coordinating Node 组装结果返回客户端

问题：
  如果数据量大，Query Phase 需要返回大量文档 ID，网络开销大
  深度分页（from=10000, size=10）性能极差（所有分片返回 10010 条）

DFS Query-then-Fetch（可选）：
  先计算全局 IDF（跨所有分片），再执行搜索
  解决分片间词频不一致导致的评分偏差
  代价：多一轮网络交互，性能稍差
  推荐：小数据量或评分敏感场景使用
```

### 4.2 相关性评分（BM25）

```
ES 5.0+ 使用 BM25 替代 TF-IDF

BM25 公式核心：
  Score(Q, D) = Σ IDF(qi) * [f(qi, D) * (k1 + 1)] / [f(qi, D) + k1 * (1 - b + b * |D| / avgdl)]

  IDF(qi)：词项 qi 的逆文档频率（罕见词权重高）
  f(qi, D)：词项在文档中的出现频率（TF）
  |D|：文档长度
  avgdl：平均文档长度
  k1, b：调节参数（k1 控制词频饱和度，b 控制文档长度惩罚）

与 TF-IDF 的区别：
  BM25 词频有上限（非线性增长），避免长文档占优
  TF-IDF 词频线性增长，长文档评分偏高

面试要点：
  "ES 的评分算法" → BM25，基于词频和逆文档频率，词频非线性增长，有文档长度归一化
  "为什么不用 TF-IDF" → TF-IDF 词频线性增长，长文档天然占优势，BM25 更合理
```

### 4.3 深度分页问题

```
问题：from=10000, size=10
  每个分片需要返回 10010 条记录， Coordinating Node 需要合并海量数据
  内存消耗大，性能急剧下降，超过 max_result_window（默认 10000）报错

解决方案：

  1. Scroll API（快照）：
     创建查询快照，保持 Segment 不删除，按批次拉取
     缺点：占用资源，不适合实时场景（数据变更后快照不反映新数据）
     适用：数据导出、全量迁移

  2. Search After（推荐）：
     使用上一页最后一条数据的排序值作为下一页起点
     如：search_after: ["2024-01-01", "doc_123"]
     优点：不限制深度，实时性好
     缺点：只能下一页，不能随机跳页
     适用：瀑布流、时间线等连续翻页

  3. 限制最大页码：
     产品层面限制（如最多翻到 100 页）
     适用：大多数搜索场景用户不会翻太多页

  4. 提高 max_result_window（不推荐）：
     加大内存消耗，只是延迟问题爆发
```

---

## 5. 分词与映射

### 5.1 分词器（Analyzer）

```
Analyzer 组成：
  Character Filters（字符过滤）：预处理文本（如去除 HTML 标签）
  Tokenizer（分词器）：将文本拆成词项（如按空格分词、按中文分词）
  Token Filters（词项过滤）：词项加工（如转小写、去停用词、同义词）

内置分词器：
  standard：按空格和标点分词，保留大小写（默认）
  whitespace：仅按空格分词
  keyword：不分词（整个字段作为一个词项）
  simple：非字母处拆分，转小写
  english：英文处理，去停用词，提取词干

中文分词：
  IK 分词器（最常用）：ik_max_word（最细粒度）、ik_smart（最粗粒度）
  可自定义词典（扩展词典、停用词典）

面试要点：
  "keyword 和 text 类型的区别"：
  keyword 不分词，用于精确匹配（如 ID、状态码、枚举值）
  text 分词，用于全文搜索（如标题、描述）
  一个字段可同时设置两种类型（multi-field）："title": {"type": "text", "fields": {"keyword": {"type": "keyword"}}}
```

### 5.2 Mapping（映射）

```
Mapping：定义字段类型和分词方式

动态映射（Dynamic Mapping）：
  默认开启，新字段自动推断类型
  风险：字段类型推断错误（如数字串被推断为 long，导致后续写入字符串失败）
  生产建议：关闭动态映射（dynamic: strict）或启用动态模板

字段类型：
  text：分词，用于全文搜索
  keyword：不分词，用于聚合、排序、精确匹配
  numeric：long, integer, double, float
  date：日期（支持多种格式）
  boolean：布尔值
  object / nested：嵌套对象（nested 保持数组对象独立性，避免交叉匹配）
  geo_point：地理位置

面试要点：
  "nested 和 object 的区别"：
  object：数组对象被扁平化，失去对象独立性（如 [{"name": "Alice", "age": 20}, {"name": "Bob", "age": 30}] 查询 name=Alice AND age=30 会匹配，因为扁平化后 name=[Alice, Bob], age=[20, 30]）
  nested：每个对象独立索引，查询时保持对象独立性（需用 nested query）
  代价：nested 存储和查询开销更大
```

---

## 6. 聚合与排序

### 6.1 Doc Values 与 Fielddata

```
排序和聚合需要列式存储：

Doc Values（默认）：
  字段的列式存储（Lucene 的 .dvd 文件）
  写入时构建，磁盘存储，查询时加载到内存
  优点：预先构建，查询快，内存友好
  缺点：占用磁盘空间，不支持 text 类型（text 分词后词项多，不适合列式）
  text 字段默认关闭 doc_values，需要排序/聚合时用 keyword 子字段

Fielddata（已废弃）：
  旧版本的列式存储，在内存中构建
  缺点：内存消耗大，可能导致 OOM
  ES 5.0+ 已废弃，被 Doc Values 替代

禁用 Doc Values（节省磁盘）：
  如果字段不需要排序、聚合、脚本计算，可禁用 doc_values
  但无法对禁用字段排序/聚合
```

### 6.2 聚合类型

```
Bucket Aggregations（桶聚合）：
  terms：按字段值分组（如按状态分组统计）
  date_histogram：按日期区间分组（如按天统计）
  range：按数值区间分组

Metric Aggregations（指标聚合）：
  avg / sum / max / min / stats：统计计算
  cardinality：去重计数（近似算法，HyperLogLog++）
  percentiles：百分位统计（用于延迟分析）

Pipeline Aggregations（管道聚合）：
  基于其他聚合结果做二次计算（如 bucket_selector、bucket_sort）

聚合内存限制：
  search.max_buckets：默认 10000，防止聚合结果过多导致 OOM
  复杂聚合需控制桶数量和精度
```

---

## 7. 性能优化

### 7.1 写入优化

```
1. 批量写入（Bulk API）：
   单条写入 HTTP 开销大，Bulk 批量提交（每批 5-15MB 或 1000-5000 条）

2. 减少 Refresh 频率：
   refresh_interval: 30s（日志场景）或 -1（批量导入时关闭，导入后手动 refresh）
   注意：refresh 频率越低，数据可见延迟越高

3. 减少 Replica 数量：
   批量导入时，临时将 replica 设为 0（导入后恢复）
   减少写入时的复制开销

4. 禁用 _source（不推荐）：
   如果不需要返回原始文档，可禁用 _source
   但无法更新、reindex，慎用

5. 使用 Routing：
   自定义 routing，将相关数据路由到同一分片
   减少查询时广播的分片数（如用户数据按 user_id routing）

6. 避免频繁更新：
   更新 = 删除 + 插入（创建新 Segment）
   频繁更新导致 Segment 过多，Merge 压力大
   解决方案：用外部版本控制，减少更新频率；或改用追加模式
```

### 7.2 查询优化

```
1. 使用 Filter 替代 Query：
   Filter 不计算评分，可缓存（bitset），性能更好
   如：bool 查询中 must 用 Query，filter 用 Filter

2. 避免深度分页：
   用 search_after 或 scroll 替代 from/size

3. 控制返回字段：
   _source 过滤，只返回需要的字段
   避免返回大字段（如长文本、二进制）

4. 路由优化：
   查询时指定 routing，减少查询分片数
   如：GET /orders/_search?routing=user_123

5. 预热（Index Warmers）：
   重要查询提前执行，加载缓存

6. 分片数规划：
   避免过多分片（每个分片是独立 Lucene 索引，查询需合并）
   经验：单节点分片数不超过 20，单分片大小 20-50GB
   索引分片数 = 数据节点数 * 1~3（考虑未来扩容）

7. 避免跨索引查询：
   跨索引查询需合并更多分片结果，性能差
   设计时按业务拆分索引，而非全部塞一个索引
```

### 7.3 内存与 GC 优化

```
ES 内存结构：
  JVM Heap：默认不超过 32GB（指针压缩，超过后指针膨胀，性能下降）
  推荐：Heap 不超过 30GB，剩余内存给文件系统缓存（Lucene 使用 OS 缓存）

常见内存问题：
  Fielddata 过大：已废弃，改用 Doc Values
  聚合桶过多：控制 cardinality 和 terms 的 size
  大查询（如 deep paging）：限制 from 大小，用 search_after

GC 优化：
  使用 G1 GC（Java 默认）
  避免 CMS（已废弃）
  监控 GC 频率和停顿时间，长时间 GC 可能是内存不足或查询问题
```

---

## 8. 集群与脑裂

### 8.1 脑裂（Split-Brain）

```
问题：网络分区导致多个节点认为自己是 Master，各管各的，数据不一致

解决：
  discovery.zen.minimum_master_nodes（ES 6.x）：
    必须超过半数 Master 候选节点同意才能选举
    如 3 个 Master 候选节点，minimum_master_nodes=2
    网络分区时，小分区节点数不足，无法选举新 Master

  7.x+ 改进：
    移除 zen，使用全新的集群协调子系统
    投票配置（voting configuration）自动管理，无需手动设置
    基于 Raft 类算法，选举更安全

面试要点：
  "ES 怎么解决脑裂" → 6.x 通过 minimum_master_nodes 要求半数以上同意；7.x+ 使用投票配置和 Raft 类算法，自动管理，无需手动设置
```

### 8.2 集群节点角色规划

```
生产环境节点分离：
  Master 节点（3 个）：
    只负责集群管理（索引创建、节点管理、分片分配）
    不存储数据，不处理查询（防止 GC 导致集群不稳定）
    配置：node.master: true, node.data: false

  Data 节点（N 个）：
    存储数据，执行查询/写入
    配置：node.master: false, node.data: true
    需要大内存、大磁盘、SSD

  Coordinating 节点（2+ 个）：
    只接收请求，转发和聚合结果
    适合大聚合、高并发查询场景
    配置：node.master: false, node.data: false
    需要高内存、高 CPU（用于聚合计算）

  Ingest 节点（可选）：
    预处理文档（如 Pipelines）
    可以放在 Data 节点或独立节点

  冷热分离：
    Hot 节点：SSD，处理新数据写入和查询
    Warm 节点：HDD，存储历史数据，查询较少
    Cold 节点：归档存储，很少查询
    Frozen 节点：极低频数据，可搜索快照
```

### 8.3 数据可靠性策略

```
副本策略：
  关键索引：至少 1 个副本（index.number_of_replicas: 1）
  高可用索引：2 个副本（容忍 2 个节点故障）
  日志/监控：可 0 副本（可丢失，可重建）

快照与恢复：
  使用 Snapshot API 备份到远程存储（S3、HDFS、共享文件系统）
  定期快照（如每天凌晨），保留策略（如保留最近 7 天快照）
  恢复：Restore API 从快照恢复索引

跨集群复制（CCR）：
  ES 6.7+ 支持，主集群写入，从集群跟随复制
  适用：异地灾备、读写分离
```

---

## 9. 面试追问应对

**Q1：ES 的写入流程是怎样的？为什么不是实时的？**
> 写入流程：1）数据写入内存 Buffer 和 Translog；2）默认 1s 后 Refresh，Buffer 数据写入文件系统缓存，生成新 Segment，此时可被搜索；3）默认 30min 或 Translog 满后 Flush，数据 fsync 到磁盘。不是实时是因为 Refresh 是轻量操作，默认 1s 间隔，且写入到 Searchable 需要生成 Segment。如果需要实时，可设置 refresh_interval=1s（已是最小）或调用 Refresh API，但影响性能。

**Q2：ES 的倒排索引是什么？怎么加速查询？**
> 倒排索引是"词项 → 文档 ID 列表"的映射。查询时直接查词项对应的文档列表，无需扫描所有文档。加速机制：1）Term Index（FST 前缀树）将 Term Dictionary 加载到内存，实现 O(1) 词项查找；2）Segment 是独立的不可变索引，缓存友好；3）Filter 查询结果缓存为 bitset，复用。FST 是核心加速结构，压缩比高，百万级词项只需几 MB。

**Q3：ES 的分片为什么不能随意改？**
> 分片数是创建索引时决定的，后续修改需要重建索引（Reindex）。因为数据按 hash(routing) % shard_count 分布，修改分片数会导致数据分布规则变化，所有数据需要重新计算路由并迁移。如果业务需要动态扩容，可以：1）预分配较多分片（但太多影响性能）；2）使用 Index Alias 和 Rollover，按时间创建新索引；3）使用 ILM（Index Lifecycle Management）自动管理索引生命周期。

**Q4：ES 的 Query 和 Filter 有什么区别？**
> Query 计算相关性评分（_score），用于全文搜索，结果按相关性排序。Filter 不计算评分，只判断匹配/不匹配，结果可缓存（bitset），性能更好。Filter 适合精确匹配（如状态、范围、日期）。推荐：全文搜索用 Query，精确过滤用 Filter，且 Filter 放 bool 查询的 filter 子句中，可被缓存复用。

**Q5：深度分页问题怎么解决？**
> 三种方案：1）Scroll API：创建快照，适合批量导出，但不实时；2）Search After：用上页最后数据的排序值作为下页起点，适合瀑布流，不限制深度，实时性好；3）产品限制：限制最大翻页数（如 100 页），大多数搜索场景用户不会翻太深。不推荐提高 max_result_window，只是延迟问题。

**Q6：ES 适合做关系型数据库的替代品吗？**
> 不适合。ES 是搜索引擎，不是数据库。差异：1）无事务，无 JOIN（nested 和 parent-child 性能差）；2）最终一致，不是强一致；3）写入非实时；4）更新代价高（删除+插入）。推荐用法：关系型数据库做事务性写入，ES 做搜索和分析（双写），或用 CDC（Canal/Debezium）同步。

**Q7：ES 的脑裂怎么解决？**
> 脑裂是网络分区导致多个 Master 同时存在。ES 6.x 通过 discovery.zen.minimum_master_nodes 要求半数以上 Master 候选节点同意才能选举。ES 7.x+ 使用全新的集群协调子系统，基于投票配置和类似 Raft 的算法，自动管理，无需手动设置 minimum_master_nodes，更安全。

**Q8：ES 聚合为什么快？**
> 因为 Doc Values（列式存储）。写入时预先构建每个字段的列式数据，排序和聚合时直接读取列式数据，不需要扫描文档。对比 Fielddata（旧版本，内存构建），Doc Values 是磁盘存储，按需加载，内存友好。代价是占用额外磁盘空间。

---

## 关联阅读

- [01-基础存储与中间件/00-Redis业务场景实战](../01-基础存储与中间件/00-Redis业务场景实战.md) — Redis 与 ES 的缓存协同
- [01-基础存储与中间件/02-MySQL索引与事务优化](../01-基础存储与中间件/02-MySQL索引与事务优化.md) — 关系型数据库与 ES 的对比
- [01-基础存储与中间件/05-缓存与搜索协同设计](../01-基础存储与中间件/05-缓存与搜索协同设计.md) — 缓存与搜索的配合

---

> **一句话总结**：ES = Lucene（倒排索引 + FST + Segment）+ 分布式（分片、副本、Translog）。写入近实时（Buffer → Refresh → Flush），查询分 Query + Fetch 两阶段。性能优化：Bulk 写入、控制 Refresh、Filter 缓存、Search After 分页、Routing 减少分片扫描。分片数创建后不可改，需提前规划。
