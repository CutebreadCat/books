# Redis Key 与数据库表设计

> Redis Key 设计和数据库表设计是分布式系统的"地基"——地基不稳，上层架构再华丽也会崩塌。面试中这往往作为系统设计的开场问题，答不好直接影响后续追问。本篇给出可落地的工程规范。

---

## 目录

- [1. 为什么要重视 Key/表设计](#1-为什么要重视-key表设计)
- [2. Redis Key 设计规范](#2-redis-key-设计规范)
- [3. Redis Key 过期策略](#3-redis-key-过期策略)
- [4. 大 Key 与热 Key 治理](#4-大-key-与热-key-治理)
- [5. 数据库表设计规范](#5-数据库表设计规范)
- [6. 分库分表后的表设计](#6-分库分表后的表设计)
- [7. 三类表的设计差异](#7-三类表的设计差异)
- [8. 踩坑与反模式](#8-踩坑与反模式)
- [9. 面试追问应对](#9-面试追问应对)

---

## 1. 为什么要重视 Key/表设计

```
面试开场常见对话：

面试官：设计一个用户签到系统。
候选人：用 Redis Bitmap，key 是 sign:user_id:2024，按日期 offset。
面试官：user_id 是什么格式？有重复风险吗？
候选人：就是用户 ID...
面试官：如果用户量 1 亿，key 命名空间怎么管理？过期怎么清？
候选人：...

设计问题：
  1. Key 命名随意 → 线上运维找不到数据、误删
  2. 没有过期策略 → Redis 内存无限增长，最终 OOM
  3. 表字段类型乱选 → 空间浪费、索引失效、迁移困难
  4. 分库分表后表名不规范 → 脚本遍历报错、运维混乱
  5. 没有通用字段 → 数据审计、问题排查无从查起

结论：
  Key/表设计不是"规范文档"，是工程习惯。
  好的设计让系统可维护、可扩展、可排查。
```

---

## 2. Redis Key 设计规范

### 2.1 命名规范

```
核心原则：可读、可定位、可统计、可隔离

规范模板：
  {业务}:{模块}:{对象}:{ID}[:{子对象}]

示例：
  user:profile:10001          → 用户 10001 的资料
  user:session:10001:web      → 用户 10001 的 Web 端会话
  order:detail:O20240001      → 订单 O20240001 的详情
  stock:sku:1001:warehouse:1  → 仓库 1 中 SKU 1001 的库存
  config:app:banner           → APP  Banner 配置

分隔符选择：
  - 推荐冒号 :（Redis 官方统计工具 redis-cli --bigkeys 默认按冒号分组）
  - 不推荐下划线 _（容易和业务字段混淆）
  - 不推荐空格（命令行难处理）

反例：
  userinfo10001         → 看不出业务/模块，无法按前缀扫描
  a:b:c:d:e:f           → 层级太深，维护困难
  temp_data_123         → 临时 Key 没有 TTL，成为垃圾
```

### 2.2 环境隔离

```
多环境必须隔离，防止测试数据污染生产：

方案对比：

  方案1：DB 号隔离（推荐）
    生产：Redis DB 0
    测试：Redis DB 1
    预发：Redis DB 2
    - 优点：简单，Key 不变
    - 缺点：SELECT DB 有开销；Cluster 不支持多 DB（只能选 0）

  方案2：Key 前缀隔离（Cluster 必用）
    生产：prod:user:profile:10001
    测试：test:user:profile:10001
    - 优点：Cluster 兼容，逻辑清晰
    - 缺点：Key 变长，内存稍增

  方案3：独立实例隔离
    生产：redis-prod.company.com
    测试：redis-test.company.com
    - 优点：完全隔离，最安全
    - 缺点：成本高

推荐组合：
  单机/哨兵：DB 号隔离 + 环境前缀兜底
  Cluster：Key 前缀隔离（必须） + 独立实例（有条件）
```

### 2.3 Key 命名空间管理

```
问题：线上 Redis 有 10 万个 Key，怎么知道每个 Key 是干什么的？

管理方案：

  1. 命名注册表（文档化）
     维护一个内部文档/配置中心：
       user:profile:{user_id}    → 用户资料，TTL 1h，String
       user:session:{user_id}    → 用户会话，TTL 2h，Hash
       order:detail:{order_id}   → 订单详情，TTL 30min，String
       stock:sku:{sku_id}        → SKU 库存，无 TTL，String

  2. Key 前缀监控
     用 redis-cli --bigkeys 或 rdb-tools 分析 Key 分布：
       user:*      → 占 30%，约 30000 个
       order:*     → 占 50%，约 50000 个
       stock:*     → 占 20%，约 20000 个

  3. Key 命名审查
     代码 Review 时检查 Key 是否符合规范
     不合规的 Key 禁止合入
```

### 2.4 ID 嵌入 Key 的注意事项

```
问题：user_id 是数字还是字符串？有前导零吗？长度固定吗？

规范：
  1. ID 统一用字符串（避免 64 位整数溢出、前导零丢失）
  2. 如果 ID 是数字，不补零（节省空间）
     - 正确：user:profile:10001
     - 错误：user:profile:000010001（浪费 4 字节）
  3. 如果 ID 可能重复（如多租户），加租户标识
     - 正确：user:profile:t1:10001（租户 1 的用户 10001）

复合 ID 的处理：
  场景：关注关系，需要 user_id + follow_user_id
  Key：follow:{user_id}:{follow_user_id}
  问题：两个 ID 都是变长的，怎么区分？
  解决：固定分隔符，如 follow:10001:20002
       或者把复合 ID 哈希后作为 Key：follow:hash(10001_20002)
```

---

## 3. Redis Key 过期策略

### 3.1 TTL 设计原则

```
所有业务 Key 必须有 TTL，不允许永久 Key（除非明确配置表）。

TTL 设计矩阵：

| 数据类型 | 推荐 TTL | 说明 |
|----------|----------|------|
| 用户会话 | 2h ~ 24h | 登录态，根据安全要求 |
| 验证码 | 5min | 短信/邮件验证码 |
| 限流计数 | 1min ~ 1h | 滑动窗口周期 |
| 缓存数据 | 10min ~ 1h | 业务数据缓存 |
| 热点配置 | 1h ~ 24h | 变化少的配置 |
| 库存数据 | 活动期间不设 TTL，结束后 1h | 活动库存需持久化兜底 |
| 分布式锁 | 10s ~ 30s | 必须短，配合看门狗 |
| 排行榜 | 1h ~ 1d | 定期重建 |
| 延迟队列 | 到期望时间 | ZSet score 即过期时间 |

兜底策略：
  1. 业务 Key 默认 TTL（如代码框架层统一设置 24h）
  2. 永久 Key 白名单：只有注册到白名单的 Key 才允许无 TTL
  3. 定期扫描：用 SCAN 找出无 TTL 的 Key，告警并清理
```

### 3.2 过期清理策略

```
Redis 过期清理机制：

  1. 惰性删除（Lazy Expiration）
     - 访问 Key 时，检查是否过期，过期则删除
     - 优点：不消耗额外 CPU
     - 缺点：如果 Key 一直不被访问，会一直占内存

  2. 定期删除（Active Expiration）
     - 每 100ms 随机抽查一批 Key，删除过期的
     - 每次抽查不超过 25ms，避免阻塞
     - 优点：平衡了 CPU 和内存
     - 缺点：如果过期 Key 太多，抽查不过来，内存泄漏

  3. 内存淘汰（Eviction）
     - 当内存达到 maxmemory，按策略淘汰 Key
     - 策略：allkeys-lru / volatile-lru / allkeys-lfu 等

线上问题：
  "Redis 内存突然涨了，但 Key 数量没涨" → 大量 Key 同时过期，惰性删除来不及

解决方案：
  1. TTL 加随机偏移，避免集中过期
     SET key value EX 3600
     → SET key value EX (3600 + random(0, 300))  // 分散 5 分钟

  2. 对于大批量 Key（如活动结束），主动删除
     - 不要用 KEYS *（阻塞）
     - 用 SCAN 分批删除
     - 或者设置统一前缀，用 UNLINK（异步删除）

  3. 监控过期 Key 比例
     - info stats 中的 expired_keys
     - 如果 expired_keys 增长但内存不下降，说明清理跟不上
```

---

## 4. 大 Key 与热 Key 治理

### 4.1 大 Key 定义与识别

```
大 Key 定义：
  - String：value > 10KB（注意：100KB 已经很大了）
  - Hash/Set/ZSet/List：元素数 > 5000 或总大小 > 1MB

识别方法：
  1. redis-cli --bigkeys
     - 抽样扫描，找出每种类型最大的 Key
     - 缺点：抽样，可能漏；线上慎用（会消耗 CPU）

  2. rdb-tools
     - 分析 RDB 文件，离线统计 Key 大小
     - 优点：不干扰线上
     - 缺点：数据不是实时的

  3. 内置 MEMORY USAGE 命令
     - MEMORY USAGE key_name
     - 精确计算 Key 占用内存
     - 适合排查单个 Key

  4. 监控指标
     - 监控单个 Key 的命令耗时
     - 如果 HGETALL/ LRANGE 经常慢，可能有大 Key

危害：
  1. 阻塞 Redis（单线程，大 Key 操作耗时）
  2. 网络传输慢（大 Key 序列化/反序列化耗时）
  3. 主从同步延迟（大 Key 同步耗时）
  4. 内存碎片（大 Key 删除后，内存不连续）
```

### 4.2 大 Key 处理方案

```
String 大 Key：
  原因：存储了过大的 JSON / 图片 Base64 / 日志
  方案：
    1. 拆分：把 JSON 拆成多个 Hash 字段
    2. 压缩：存 gzip 压缩后的数据
    3. 迁移：大 value 存文件系统（OSS/S3），Redis 存 URL

Hash 大 Key：
  原因：Hash 存了太多字段（如用户所有订单）
  方案：
    1. 按 ID 分片：user:orders:10001 拆成 user:orders:10001:1, :2, :3
    2. 只存热点字段：不存全量，只存最近 100 条
    3. 用 Hash Tag 分散到多个 slot（Redis Cluster）

List 大 Key：
  原因：消息队列积压（如 LIST 当 MQ 用）
  方案：
    1. 限制长度：LPUSH 后检查长度，超过则 RPOP/LPOP
    2. 分队列：按时间/用户分多个 List
    3. 换结构：List 改为 Stream 或专业 MQ

ZSet 大 Key：
  原因：排行榜无限增长
  方案：
    1. 只保留 Top N：定期 ZREMRANGEBYRANK 删除低分
    2. 分周期：按日/周/月建多个 ZSet
    3. 分层：粗排用 ZSet，精排用本地排序
```

### 4.3 热 Key 定义与处理

```
热 Key 定义：
  - 单个 Key 的 QPS > 1000（视集群规模而定）
  - 或者该 Key 的 QPS 占所在节点总 QPS 的 10% 以上

识别方法：
  1. Redis 监控：MONITOR 命令（生产环境慎用，性能开销大）
  2. 客户端埋点：在代码里统计每个 Key 的访问频率
  3. Redis 4.0+：MEMORY STATS 中的 hotkeys 统计
  4. Proxy 层统计：如 Twemproxy、Codis、Redis Cluster Proxy

处理方案：

  方案1：本地缓存（最常用）
    应用层加 Caffeine/Guava Cache：
      - 热 Key 在本地缓存一份，减少 Redis 访问
      - TTL 设短（如 5s），容忍短暂不一致
      - 数据变更时，通过 MQ 广播清理本地缓存

  方案2：热 Key 分片
    把热 Key 复制到多个 Redis 节点：
      stock:sku:1001:shard:1  → 节点 1
      stock:sku:1001:shard:2  → 节点 2
      stock:sku:1001:shard:3  → 节点 3
    读时随机选一个，写时全写或写主
    适合：读多写少，且可以容忍最终一致

  方案3：读写分离
    热 Key 的读请求分散到多个从节点
    写请求仍走主节点
    适合：主从架构，且读远大于写

  方案4：自动发现 + 本地缓存（智能化）
    1. 监控发现热 Key（如 1 分钟内访问超过 1000 次）
    2. 自动触发本地缓存
    3. 数据变更时，自动清理
    4. 热 Key 冷却后，自动取消本地缓存
```

---

## 5. 数据库表设计规范

### 5.1 命名规范

```
表名：
  规范：{业务}_{模块}_{实体}
  示例：
    order_main          → 订单主表
    order_item          → 订单明细表
    user_profile        → 用户资料表
    product_sku         → 商品 SKU 表
    config_banner       → Banner 配置表

  规则：
    - 全小写 + 下划线分隔
    - 单数形式（order 不是 orders）
    - 不超过 32 个字符（某些数据库有限制）
    - 避免关键字（order、user、group 等加前缀）

字段名：
  规范：全小写 + 下划线分隔
  示例：user_id, create_time, is_deleted

  规则：
    - ID 字段：表名_id 或 id（主键）
    - 状态字段：status（枚举值）或 is_xxx（布尔）
    - 时间字段：create_time, update_time
    - 避免缩写（除了通用的 id, url, ip）
```

### 5.2 通用字段（每张表必须）

```sql
-- 所有业务表必须包含的字段
CREATE TABLE example_table (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    
    -- 业务字段在这里
    
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    
    -- 二选一（根据业务）
    is_deleted TINYINT NOT NULL DEFAULT 0 COMMENT '是否删除：0-否，1-是',
    -- 或者
    delete_time DATETIME DEFAULT NULL COMMENT '删除时间，NULL表示未删除',
    
    -- 可选：数据版本控制（乐观锁）
    version INT UNSIGNED NOT NULL DEFAULT 1 COMMENT '版本号',
    
    -- 可选：操作人追踪
    create_by VARCHAR(64) DEFAULT NULL COMMENT '创建人',
    update_by VARCHAR(64) DEFAULT NULL COMMENT '更新人',
    
    PRIMARY KEY (id),
    -- 软删除索引
    INDEX idx_is_deleted (is_deleted)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='示例表';

说明：
  1. id：BIGINT，预留百亿级数据量
  2. create_time / update_time：自动维护，审计必备
  3. is_deleted / delete_time：软删除，便于数据恢复和审计
  4. version：乐观锁，防止并发更新覆盖
```

### 5.3 字段类型选择

```
常用字段类型指南：

| 数据类型 | 推荐场景 | 不推荐 |
|----------|----------|--------|
| BIGINT | 主键 ID、用户 ID | INT（容易溢出） |
| VARCHAR(64) | 订单号、手机号、邮箱 | CHAR（变长浪费） |
| VARCHAR(512) | URL、地址、描述 | TEXT（索引受限） |
| TEXT | 长文本（文章、评论） | VARCHAR（超长度） |
| DECIMAL(18,2) | 金额（精确计算） | FLOAT/DOUBLE（精度丢失） |
| TINYINT | 状态码（0/1/2） | VARCHAR（占空间） |
| DATETIME | 时间戳（带时区业务用 TIMESTAMP） | VARCHAR |
| JSON | 灵活结构（配置、扩展字段） | TEXT（无法索引） |

金额字段特别注意：
  - 单位：统一用"分"存储（DECIMAL(18,0)），或 DECIMAL(18,2) 存元
  - 绝不用 FLOAT/DOUBLE
  - 计算时在应用层用 Decimal 类型

状态字段：
  - 用 TINYINT + 枚举注释
  - 示例：status TINYINT COMMENT '状态：0-待支付，1-已支付，2-已发货，3-已完成'
  - 不要用 VARCHAR 存状态（占空间、易出错）
```

### 5.4 索引设计原则

```
核心原则：
  1. 每张表必须有主键（聚簇索引）
  2. 查询 WHERE 条件的字段必须有索引（或组合索引）
  3. 索引列的顺序：等值查询在前，范围查询在后
  4. 避免冗余索引（如 (a,b) 和 (a) 同时存在，(a) 是冗余的）
  5. 单表索引数不超过 5 个（索引过多影响写入）
  6. 索引字段选择性要高（如性别字段不适合建索引）

常见索引模式：

  按用户查询：
    INDEX idx_user_id (user_id)
    INDEX idx_user_status (user_id, status)  -- 如果经常按用户+状态查

  按时间范围查询：
    INDEX idx_create_time (create_time)
    INDEX idx_user_time (user_id, create_time)  -- 按用户+时间查

  唯一约束：
    UNIQUE KEY uk_order_no (order_no)
    UNIQUE KEY uk_user_phone (user_id, phone)  -- 联合唯一

  前缀索引（长文本）：
    INDEX idx_email (email(20))  -- 只索引前 20 个字符

索引命名规范：
  - 主键：PRIMARY
  - 唯一索引：uk_{字段名}
  - 普通索引：idx_{字段名}
  - 组合索引：idx_{字段1}_{字段2}
```

---

## 6. 分库分表后的表设计

### 6.1 分片键设计

```
分片键（Sharding Key）选择原则：
  1. 高频查询条件：大部分查询都带这个字段
  2. 数据均匀：避免热点（如按时间分片导致最新片热点）
  3. 不可变：分片键变更等于数据迁移

常见分片键：
  - 用户类表：user_id
  - 订单类表：order_id（或 buyer_id，看查询场景）
  - 日志类表：create_time（按时间范围分片）

分片策略：
  1. Hash 分片：user_id % 16
     - 优点：数据均匀
     - 缺点：范围查询要查所有片
  2. 范围分片：user_id 0~1000万 → 库0，1000~2000万 → 库1
     - 优点：范围查询快
     - 缺点：数据可能不均匀（新用户集中在最新片）
  3. 混合分片（Hash + 范围）：先按 Hash 分到库，再按时间分到表
```

### 6.2 分库分表后的字段调整

```sql
-- 分库分表后必须增加的字段
CREATE TABLE order_main_0 (
    order_id VARCHAR(64) NOT NULL COMMENT '订单号（全局唯一）',
    user_id BIGINT NOT NULL COMMENT '买家ID',
    
    -- 分片辅助字段（用于双写迁移、数据校验）
    db_no TINYINT NOT NULL COMMENT '数据库编号',
    table_no TINYINT NOT NULL COMMENT '表编号',
    
    -- 业务字段...
    
    create_time DATETIME NOT NULL,
    
    PRIMARY KEY (order_id),
    -- 买家维度查询索引
    INDEX idx_user_id (user_id, create_time)
) COMMENT='订单主表-片0';

说明：
  1. 全局唯一 ID：分库分表后自增 ID 不唯一，用 Snowflake/Leaf
  2. db_no / table_no：记录数据所在的分片，便于后续迁移和对账
  3. 分片键必须是索引前缀：如按 user_id 分片，(user_id, create_time) 索引
  4. 跨片查询：
     - 能避免的尽量避免（如订单详情按 order_id 查，直接路由）
     - 必须跨片的，用异步汇总或 ES
```

### 6.3 分库分表后的关联查询

```
问题：分库分表后，JOIN 操作怎么办？

方案：
  1. 避免跨片 JOIN
     - 把需要 JOIN 的数据冗余到同一张表
     - 或把相关数据分到同一分片（如按 user_id 分片，用户资料+订单都在同一库）

  2. 应用层组装
     - 先查主表，再按 ID 分批查关联表
     - 用 IN 查询批量获取（单表内 IN 是高效的）

  3. 宽表冗余
     - 订单表冗余用户昵称、商品名称等
     - 用空间换时间，避免 JOIN
     - 注意：冗余字段变更时，需要同步更新

  4. 全局表（广播表）
     - 配置表、字典表在每个分片都存一份
     - 适合数据量小、变更少的表
```

---

## 7. 三类表的设计差异

```
业务表 vs 配置表 vs 日志表的设计差异：

┌────────────┬────────────────────┬────────────────────┬────────────────────┐
│ 维度       │ 业务表             │ 配置表             │ 日志表             │
├────────────┼────────────────────┼────────────────────┼────────────────────┤
│ 数据量     │ 百万 ~ 亿级        │ 百 ~ 万级          │ 亿 ~ 千亿级        │
│ 写入频率   │ 中                 │ 低                 │ 极高               │
│ 查询模式   │ 按 ID/用户查       │ 按 Key 查          │ 按时间范围查       │
│ 主键设计   │ 全局唯一 ID        │ 配置 Key           │ 时间+序列号        │
│ 索引策略   │ 多个二级索引       │ 唯一索引           │ 时间索引+少量索引  │
│ 归档策略   │ 冷热分离           │ 不归档             │ 定期归档/删除      │
│ 存储引擎   │ InnoDB             │ InnoDB             │ MyRocks/归档存储   │
│ 字段要求   │ 完整通用字段       │ version 字段防并发 │ 精简字段，大表     │
└────────────┴────────────────────┴────────────────────┴────────────────────┘

配置表设计要点：
  - 必须有 version 字段（乐观锁，防止并发修改配置）
  - 缓存前置：配置表数据量小，全量加载到 Redis/本地缓存
  - 变更通知：配置修改后，广播通知应用刷新缓存

日志表设计要点：
  - 不写 update，只 insert
  - 不写 is_deleted（日志不删）
  - 按时间分表（如按月）：order_log_202401, order_log_202402
  - 过期归档：3 个月前的日志迁移到冷存储
  - 不建太多索引（写入性能优先，查询走 ES/数仓）
```

---

## 8. 踩坑与反模式

### 坑 1：Redis Key 没有 TTL
```
问题：所有 Key 永不过期，Redis 内存持续增长直到 OOM
解决：业务 Key 必须有 TTL，永久 Key 需注册白名单并定期 Review
```

### 坑 2：Key 命名混乱导致误删
```
问题：线上执行 KEYS order* 准备清理测试数据，结果删了生产数据
解决：
  1. 环境隔离（Key 前缀 + 独立实例）
  2. 生产环境禁用 KEYS 命令（rename-command KEYS ""）
  3. 清理用 SCAN 分批 + UNLINK 异步删除
```

### 坑 3：数据库表没有 update_time
```
问题：数据不一致，排查时发现不知道哪条记录被改过、什么时候改的
解决：所有业务表必须有 create_time + update_time，update_time 自动更新
```

### 坑 4：软删除没有索引
```
问题：SELECT * FROM order WHERE is_deleted = 0 走了全表扫描
解决：软删除字段必须建索引，或者分区（已删除的分到历史分区）
```

### 坑 5：分库分表后主键用自增 ID
```
问题：两个分片都产生了 id=100 的记录，数据冲突
解决：分库分表后必须用全局唯一 ID（Snowflake/Leaf/UUID）
```

### 坑 6：大表加字段导致锁表
```
问题：ALTER TABLE ADD COLUMN 在百万级表上执行，锁表 30 分钟
解决：
  1. MySQL 8.0+ Instant ADD COLUMN（部分场景支持）
  2. pt-online-schema-change 在线改表
  3. 预先设计扩展字段（如 JSON 类型 extra）
```

### 坑 7：金额用 FLOAT/DOUBLE
```
问题：0.1 + 0.2 != 0.3，对账差一分钱
解决：金额字段用 DECIMAL(18,2) 或整数分
```

---

## 9. 面试追问应对

**Q1：Redis Key 怎么设计？**
> 我遵循 `{业务}:{模块}:{对象}:{ID}` 的命名规范，比如 `user:profile:10001` 表示用户 10001 的资料，`order:detail:O20240001` 表示订单详情。分隔符用冒号，方便按前缀统计。所有业务 Key 必须有 TTL，不允许永久 Key。多环境用 Key 前缀隔离，比如 `prod:user:profile:10001` 和 `test:user:profile:10001`，防止测试数据污染生产。

**Q2：怎么发现和治理大 Key？**
> 发现方法：用 redis-cli --bigkeys 抽样扫描，或者用 rdb-tools 离线分析 RDB 文件，也可以在代码里对 HGETALL、LRANGE 等命令的耗时做监控，如果经常慢查询，可能有大 Key。治理方案：String 大 Key 拆成 Hash 字段或存文件系统；Hash 大 Key 按 ID 分片或只存热点数据；List 大 Key 限制长度或改用 Stream；ZSet 大 Key 定期清理低分数据。核心思路是拆分、压缩、限制、迁移四选一。

**Q3：热 Key 怎么处理？**
> 热 Key 是指单个 Key 的 QPS 过高，打爆单个 Redis 节点。处理方案：1. 本地缓存（Caffeine），热 Key 在应用层缓存 5~10 秒，减少 Redis 访问；2. 热 Key 分片，把热 Key 复制到多个 Redis 节点，读请求随机路由；3. 读写分离，读分散到多个从节点；4. 智能化方案，监控自动发现热 Key，自动触发本地缓存，冷却后自动释放。

**Q4：数据库表设计的通用字段有哪些？**
> 所有业务表必须有：id（BIGINT 主键）、create_time（创建时间，自动填充）、update_time（更新时间，自动更新）、is_deleted（软删除标记）或 delete_time（删除时间）。可选的有 version（乐观锁版本号）、create_by/update_by（操作人）。这些字段是数据审计和问题排查的基础，绝对不能省。

**Q5：分库分表后表设计要注意什么？**
> 1. 全局唯一 ID：用 Snowflake 或 Leaf，绝不用自增 ID；2. 分片键选择：高频查询条件 + 数据均匀 + 不可变，如订单表按 user_id 或 order_id；3. 增加 db_no/table_no 字段，记录数据所在分片，方便迁移和对账；4. 避免跨片 JOIN，需要 JOIN 的数据冗余到同表或同分片；5. 索引以分片键为前缀，保证查询能路由到单分片。配置表用全局表（每个分片广播一份），日志表按时间分表，只 insert 不 update。

**Q6：软删除怎么设计？**
> 用 is_deleted TINYINT DEFAULT 0，查询时默认加 WHERE is_deleted = 0。is_deleted 必须建索引，否则查询会全表扫描。如果要彻底删除（如 GDPR 合规），可以定期把 is_deleted=1 的数据归档到历史表，然后物理删除。不要用 DELETE_TIME 作为唯一删除标记而不设 is_deleted，因为查询时 IS NULL 判断走不了索引。

---

## 关联阅读

- [00-Redis 业务场景实战](../01-基础存储与中间件/00-Redis业务场景实战.md) — Redis 命令和场景
- [01-MySQL 业务场景实战](../01-基础存储与中间件/01-MySQL业务场景实战.md) — MySQL 索引和分库分表
- [02-MySQL InnoDB 页结构详解](../01-基础存储与中间件/07-InnoDB页结构详解.md) — InnoDB 存储引擎原理

---

> **一句话总结**：Redis Key 设计 = 命名规范 + 环境隔离 + TTL 兜底 + 大/热 Key 治理；数据库表设计 = 通用字段 + 合理类型 + 索引前缀 + 分片键规划。这些不是文档规范，是工程习惯，直接影响系统的可维护性和面试的第一印象。
