# MySQL 业务场景实战

> 本篇聚焦 MySQL 在真实业务场景中的设计与优化，从索引设计到分库分表，从锁机制到事务隔离。

---

## 目录

- [1. 索引设计 —— 业务场景优化](#1-索引设计--业务场景优化)
- [2. 分库分表 —— 海量数据拆分](#2-分库分表--海量数据拆分)
- [3. 读写分离 —— 查询性能提升](#3-读写分离--查询性能提升)
- [4. 幂等设计 —— 防重复处理](#4-幂等设计--防重复处理)
- [5. 乐观锁 vs 悲观锁 —— 业务选型](#5-乐观锁-vs-悲观锁--业务选型)
- [6. 事务隔离级别 —— 业务选型](#6-事务隔离级别--业务选型)
- [7. 死锁排查与预防](#7-死锁排查与预防)
- [8. SQL 优化实战](#8-sql-优化实战)
- [9. MVCC 原理与业务影响](#9-mvcc-原理与业务影响)
- [10. 主从复制原理与延迟优化](#10-主从复制原理与延迟优化)

---

## 1. 索引设计 —— 业务场景优化

### 1.1 紫金法则：索引服务于查询

```
设计步骤：
1. 收集高频查询 SQL（不是凭感觉，要看慢查询日志）
2. 分析查询模式：等值/范围/排序/聚合
3. 选择合适索引类型
4. 验证：EXPLAIN 每条高频 SQL
5. 监控：上线后持续观察索引使用率
```

### 1.2 索引类型与场景

| 类型 | 适用 | 示例 |
|------|------|------|
| 主键索引 | 行唯一标识 | id |
| 唯一索引 | 业务唯一约束 | order_no, email |
| 普通索引 | 高频等值查询 | user_id, status |
| 联合索引 | 多条件查询 | (user_id, create_time) |
| 前缀索引 | 长字符串搜索 | VARCHAR 前 N 字符 |
| 覆盖索引 | 避免回表 | 查询字段都在索引中 |
| 全文索引 | 全文搜索 | 文章内容（一般用 ES） |

### 1.3 联合索引的最左前缀原则

```
索引 idx_abc (a, b, c)
生效查询：
  WHERE a = ?           → 用到 a
  WHERE a = ? AND b = ? → 用到 a, b
  WHERE a = ? AND b = ? AND c = ? → 用到 a, b, c
  WHERE a = ? AND c = ? → 只用到 a（b 缺失，c 无法接续）

范围查询断点：
  WHERE a = ? AND b > ? AND c = ? → 用到 a, b，c 断点失效
  WHERE a = ? AND b IN (...) AND c = ? → IN 算等值，c 可用（MySQL 5.6+）
```

### 1.4 业务实战案例

**电商订单表索引设计**：

```sql
-- 高频查询 1: 用户查自己订单
SELECT * FROM orders WHERE user_id = ? AND status = ?
→ 索引: idx_user_status (user_id, status)

-- 高频查询 2: 查某个状态的订单（运营后台）
SELECT * FROM orders WHERE status = ? ORDER BY create_time DESC LIMIT 20
→ 索引: idx_status_time (status, create_time)  -- 覆盖排序

-- 高频查询 3: 订单详情
SELECT * FROM orders WHERE order_no = ?
→ 索引: uk_order_no (order_no)  -- 唯一索引

-- 高频查询 4: 时间范围查询
SELECT * FROM orders WHERE create_time BETWEEN ? AND ? AND status = ?
→ 索引: idx_time_status (create_time, status)
```

### 1.5 索引下推（ICP）

MySQL 5.6 引入 Index Condition Pushdown：

```sql
-- 索引 idx_name_age (last_name, age)
SELECT * FROM users WHERE last_name LIKE '张%' AND age > 20;

-- 无 ICP：
-- 1. 用 last_name 前缀找到所有"张X"的主键
-- 2. 回表查询完整行
-- 3. 在 Server 层过滤 age > 20

-- 有 ICP：
-- 1. 用 last_name 前缀找到所有"张X"的主键
-- 2. 在索引层直接过滤 age > 20（age 在索引中）
-- 3. 只对满足条件的行回表
-- → 大幅减少回表次数
```

### 1.6 常见踩坑

1. **索引过多** → 写入性能下降，每个索引都增加写入开销
2. **索引选择率低** → status 只有 3 个值，索引过滤效果差 → 考虑联合索引
3. **隐式类型转换** → varchar 列用 int 查询 → 索引失效
4. **函数操作** → `WHERE DATE(create_time) = ?` → 索引失效 → 改为范围查询
5. **ORDER BY 无索引** → Using filesort → 加排序列到联合索引
6. **OR 查询** → `WHERE a = 1 OR b = 2` → 一个条件无索引则全表扫描
7. **NOT IN** → 通常不走索引 → 改用 LEFT JOIN ... WHERE ... IS NULL

### 1.7 索引监控

```sql
-- 查看索引使用情况
SELECT 
    object_schema, object_name, index_name,
    count_read, count_fetch, count_insert, count_update, count_delete
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'your_db'
ORDER BY count_read DESC;

-- 查找未使用的索引（可删除）
SELECT * FROM sys.schema_unused_indexes WHERE object_schema = 'your_db';
```

---

## 2. 分库分表 —— 海量数据拆分

### 2.1 业务痛点

单表数据量过大（>500 万行或 >2GB）导致：
- 查询慢（索引树太深）
- 写入慢（B+树页分裂频繁）
- DDL 困难（ALTER TABLE 锁表时间长）
- 备份恢复慢

### 2.2 拆分策略

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| 只分表不分库 | 同库多表，单表变小 | 单表过大但总量可控 |
| 只分库不分表 | 多库单表，分散 IO/连接 | 单库连接数/IO 瓶颈 |
| 分库 + 分表 | 多库多表 | 数据量和访问量都大 |
| 垂直分库 | 按业务拆库（订单库、用户库） | 业务边界清晰 |
| 水平分表 | 同结构多表存不同数据 | 单表数据量大 |

### 2.3 分片键选择（最关键决策）

```
选择原则：
1. 高频查询条件字段 → 尽量让查询落到单分片
2. 数据分布均匀 → 避免热点分片
3. 业务关联性 → 关联查询尽量同一分片

常见分片键：
- 用户 ID → 用户维度数据（订单、账单、行为）
- 订单 ID → 订单维度数据
- 时间 → 日志、流水类数据（按月分表）
- 地域 → 地域相关数据（城市维度）

避免：
- 自增 ID → 后写入全在最后一个分片，热点
- 性别 → 只有 2 个值，分布极不均匀
```

### 2.4 分片算法

| 算法 | 特点 | 扩容难度 | 适用场景 |
|------|------|----------|----------|
| Hash 取模 | 分布均匀 | 需要数据迁移 | 固定分片数 |
| Range 范围 | 时间友好 | 简单追加 | 日志/流水 |
| 一致性 Hash | 扩容迁移少 | 中等 | 需要动态扩容 |
| 预设分片 | 如 1024 表 | 扩容 = 加库 | 长期规划 |

### 2.5 非分片键查询方案

```
问题：按分片键查询高效，但非分片键查询需要扫所有分片

方案 1：映射表
  维护 user_id → order_id 的映射表（不分片）
  先查映射表获取 order_id → 再按 order_id 分片路由查详情

方案 2：宽表设计
  把高频查询字段冗余到分片表（如用户名冗余到订单表）

方案 3：ES 索引
  MySQL 只存核心数据，ES 做全文检索和复杂查询路由

方案 4：基因法
  把 user_id 的基因（部分 bit）编入 order_id
  → 用 order_id 路由的同时也能提取 user_id 信息

  示例：
  user_id = 12345 (二进制低 8 位: 00111001)
  order_id = 雪花 ID + 00111001
  → 按 order_id 分片时，可以从 order_id 提取 user_id 的基因
  → 再用 user_id 基因查询相关订单
```

### 2.6 全局唯一 ID

分库分表后自增 ID 不再唯一，需要全局 ID 方案。详见 [04-分布式系统核心设计模式 §1](../03-分布式架构与设计/00-分布式系统核心设计模式.md#1-分布式-id-生成)。

### 2.7 常见踩坑

1. **跨分片 JOIN** → 尽量避免，改为应用层组装或宽表冗余
2. **跨分片事务** → 分布式事务复杂，尽量用本地事务 + 补偿
3. **聚合查询** → COUNT/SUM/MAX 需要扫所有分片汇总，性能差 → 放 ES 或预计算
4. **数据迁移** → 分片数变更需要数据重分布，规划时就预留足够分片
5. **分布式事务** → 尽量避免，必须用则选择合适方案（详见 03-数据一致性）

---

## 3. 读写分离 —— 查询性能提升

### 3.1 业务痛点

读多写少的场景（如电商商品浏览），单库承受大量读压力。

### 3.2 架构设计

```
写 → 主库（Master）
读 → 从库（Slave1, Slave2, ...)

中间层：
  - 代码层：框架路由（如 ShardingSphere）
  - 代理层：中间件路由（如 ProxySQL, MyCat）
  - 服务层：独立读写分离服务
```

### 3.3 核心问题：主从延迟

```
主库写入 → Binlog 复制 → 从库应用 → 数据可读
延迟通常：0.1ms ~ 1s（正常情况）
异常延迟：可达秒级甚至分钟级

业务影响：
  用户刚写入的数据，立刻读可能读到旧值
  如：下单后查订单列表 → 从库还没同步 → "我的订单"看不到新订单
```

### 3.4 强制读主库方案

```
方案 1：关键操作读主库
  - 用户自己写入后立即查询 → 路由到主库
  - 其他用户查询 → 路由到从库

方案 2：延迟感知路由
  - 中间件监控主从延迟
  - 延迟 > 阈值（如 100ms）→ 路由到主库
  - 延迟正常 → 路由到从库

方案 3：写后读标记
  - 写操作后设置 Redis 标记：force_master:<user_id> TTL=1s
  - 读请求检查标记 → 有标记则读主库
  - 1 秒后标记过期 → 自动恢复读从库
```

### 3.5 主从延迟优化

```
1. 从库性能优化
   - 使用更快的硬件（SSD）
   - 增大 innodb_buffer_pool_size

2. 网络优化
   - 主从间低延迟网络
   - 主从同机房

3. 复制优化
   - MySQL 5.7+ 多线程复制（基于组提交）
   - MySQL 8.0 写集（WRITESET）并行复制

4. 业务优化
   - 写后读容忍短暂不一致
   - 关键操作读主库
```

---

## 4. 幂等设计 —— 防重复处理

### 4.1 业务痛点

- 网络抖动导致请求重复发送
- MQ 消息重复消费
- 用户重复点击/提交
- 补偿任务重复执行
- 超时重试导致重复

### 4.2 幂等方案对比

| 方案 | 实现 | 适用场景 | 复杂度 |
|------|------|----------|--------|
| 唯一索引 | MySQL UNIQUE KEY | 新增类操作 | 低 |
| 状态机 | 状态字段 + 条件更新 | 有状态流转 | 中 |
| Token 机制 | 先获取 Token → 提交时验证 | 表单提交 | 中 |
| 唯一 ID + 去重表 | 请求 ID 写入去重表 | MQ 消费 | 中 |
| Redis SETNX | 请求 ID SETNX | 简单去重 | 低 |
| 乐观锁 | version 字段 | 更新类操作 | 中 |

### 4.3 各方案详解

**1. 唯一索引（最简单最可靠）**

```sql
-- 订单号唯一索引，重复插入直接报错
CREATE UNIQUE INDEX uk_order_no ON orders(order_no);
-- 应用层捕获 DuplicateKeyException → 返回"已存在"
```

**2. 状态机（订单流转幂等）**

```sql
-- 只有"待支付"才能改为"已支付"
UPDATE orders SET status = 'PAID'
WHERE id = ? AND status = 'PENDING';
-- 影响行数 = 1 成功，= 0 说明状态已变（幂等处理）
```

**3. Token 机制（防重复提交）**

```
1. 页面加载时：GET /api/token → 服务端生成 token 存 Redis(EX 300 秒)
2. 提交时携带 token → 服务端校验并删除 token
3. 重复提交时 token 已删除 → 拒绝

Redis 操作（Lua 原子）:
  if redis.call('GET', KEYS[1]) == ARGV[1] then
      redis.call('DEL', KEYS[1])
      return 1
  else
      return 0
  end
```

**4. 唯一 ID + 去重表（MQ 消费幂等）**

```
1. 生产者为每条消息生成唯一 msgId
2. 消费者处理前：INSERT dedup_table(msgId) → 成功则处理，失败则跳过
3. 去重表定期清理（如保留 7 天）
```

**5. 乐观锁（version 字段）**

```sql
UPDATE product SET stock = stock - 1, version = version + 1
WHERE id = ? AND version = ?;
-- 影响行数 = 0 → 版本不匹配，重试或失败
```

### 4.4 幂等设计原则

1. **入口幂等优先** → 在最外层（如 MQ 消费者、API 网关）做去重
2. **业务幂等兜底** → 即使入口去重失败，业务逻辑也要幂等
3. **天然幂等最好** → 设计数据模型时让操作天然幂等（如唯一索引）
4. **返回值设计** → 幂等请求返回"已处理"而非报错，调用方不需区分

### 4.5 不同场景的幂等方案选择

| 场景 | 推荐方案 | 说明 |
|------|----------|------|
| 用户注册 | 唯一索引（手机号/邮箱） | 数据库层保障 |
| 订单创建 | 唯一索引（订单号）+ Token | 双重保障 |
| 订单状态变更 | 状态机 | 流转控制 |
| 支付回调 | 去重表 + 状态机 | MQ 消费幂等 |
| 库存扣减 | 乐观锁 | 防并发超扣 |
| 评论点赞 | 唯一索引（user_id + target_id） | 防重复点赞 |

---

## 5. 乐观锁 vs 悲观锁 —— 业务选型

### 5.1 对比

| | 乐观锁 | 悲观锁 |
|---|--------|--------|
| 思路 | 先操作，冲突时回滚 | 先锁住，操作完释放 |
| 实现 | version 字段/条件更新 | SELECT ... FOR UPDATE |
| 性能 | 高（无锁等待） | 低（锁等待） |
| 适用 | 读多写少，冲突概率低 | 写多或冲突概率高 |
| 重试 | 需要业务层重试 | 无需重试 |
| 死锁 | 无 | 可能死锁 |
| 典型 | 库存扣减、余额更新 | 账务核心、强一致场景 |

### 5.2 乐观锁实战

```sql
-- 库存扣减（版本号方式）
UPDATE product SET stock = stock - 1, version = version + 1
WHERE id = ? AND version = ?;

-- 库存扣减（条件方式，更常用）
UPDATE product SET stock = stock - 1
WHERE id = ? AND stock > 0;
-- 影响行数 = 0 则库存不足

-- 余额扣减
UPDATE account SET balance = balance - 100
WHERE user_id = ? AND balance >= 100;
```

**乐观锁重试机制**：

```python
def deduct_stock(product_id, quantity):
    for attempt in range(3):
        product = db.query("SELECT stock, version FROM product WHERE id = ?", product_id)
        if product.stock < quantity:
            return False  # 库存不足
        
        affected = db.execute(
            "UPDATE product SET stock = stock - ?, version = version + 1 WHERE id = ? AND version = ?",
            quantity, product_id, product.version
        )
        if affected > 0:
            return True  # 成功
        # 否则重试
    return False  # 重试失败
```

### 5.3 悲观锁实战

```sql
-- 账务转账（必须悲观锁）
BEGIN;
SELECT balance FROM account WHERE user_id = ? FOR UPDATE;  -- 加行锁
-- 检查余额是否足够
UPDATE account SET balance = balance - 100 WHERE user_id = ?;
UPDATE account SET balance = balance + 100 WHERE user_id = ?;
COMMIT;
```

### 5.4 业务决策

| 场景 | 推荐 | 原因 |
|------|------|------|
| 库存扣减 | 乐观锁 | 冲突概率低，性能优先 |
| 账务转账 | 悲观锁 | 钱不能出错，宁可慢 |
| 配额分配 | 乐观锁 | 冲突时重新申请 |
| 座位锁定 | 悲观锁 | 选座不能并发 |
| 配置修改 | 乐观锁 | 修改频率低 |
| 秒杀抢购 | 乐观锁 + Redis 预扣 | Redis 挡大部分，DB 兜底 |

---

## 6. 事务隔离级别 —— 业务选型

### 6.1 四大隔离级别

| 级别 | 脏读 | 不可重复读 | 幻读 | 性能 | InnoDB 实现 |
|------|------|------------|------|------|------------|
| READ UNCOMMITTED | ✅ | ✅ | ✅ | 最高 | 直接读 |
| READ COMMITTED | ❌ | ✅ | ✅ | 高 | MVCC 快照（每语句） |
| REPEATABLE READ | ❌ | ❌ | ⚠️ | 中 | MVCC 快照（首次） + 间隙锁 |
| SERIALIZABLE | ❌ | ❌ | ❌ | 低 | 全加锁 |

### 6.2 业务场景选型

| 业务 | 推荐级别 | 说明 |
|------|----------|------|
| 电商订单 | RC | 允许不可重复读（用户浏览期间价格变了是可接受的） |
| 账务系统 | RR | 转账期间余额不能变 |
| 日志系统 | RC/RU | 日志写入后不需要精确读取 |
| 报表统计 | RR | 统计期间数据必须一致 |
| 配置管理 | RR | 配置查询期间不能被修改 |

### 6.3 RR 级别的幻读问题

```
场景：
  事务 A: SELECT * FROM orders WHERE status = 'PENDING' → 5 条
  事务 B: INSERT 新订单(status='PENDING')
  事务 A: SELECT 同条件 → 还是 5 条（InnoDB MVCC 保证了）
  事务 A: UPDATE status='PENDING' → 发现更新了 6 条！（幻读出现）

解决：
  1. SELECT ... FOR UPDATE → 加间隙锁，阻止新插入
  2. 优化业务逻辑，避免先查再改的模式
```

### 6.4 MVCC 对业务的影响

```
MVCC（多版本并发控制）特点：
  - 读不加锁，不阻塞写
  - 写不加锁，不阻塞读
  - 每个事务看到数据的一个快照

业务影响：
  - 长事务会阻塞 purge（清理旧版本），导致 undo log 膨胀
  - 长事务期间看到的是旧数据，可能业务逻辑错误
  - 建议：事务尽量短，避免长事务
```

---

## 7. 死锁排查与预防

### 7.1 常见死锁模式

```
模式 1：交叉更新
  事务 A: UPDATE t SET ... WHERE id=1 → UPDATE id=2
  事务 B: UPDATE t SET ... WHERE id=2 → UPDATE id=1
  → A 锁 1 等 2，B 锁 2 等 1 → 死锁

模式 2：唯一索引冲突
  事务 A: INSERT(id=1) → INSERT(id=2)
  事务 B: INSERT(id=2) → INSERT(id=1)
  → 唯一索引的锁顺序与 INSERT 顺序不同 → 死锁

模式 3：锁升级
  事务 A: 大范围扫描加行锁 → 试图升级为表锁
  事务 B: 持有表锁 → 死锁

模式 4：间隙锁冲突
  事务 A: SELECT ... WHERE id BETWEEN 1 AND 10 FOR UPDATE → 间隙锁
  事务 B: INSERT id=5 → 等待间隙锁
  事务 A: INSERT id=5 → 等待 B 的插入意图锁 → 死锁
```

### 7.2 死锁排查

```sql
-- 查看最近死锁
SHOW ENGINE INNODB STATUS;
-- LATEST DETECTED DEADLOCK 部分

-- 查看当前锁等待
SELECT * FROM performance_schema.data_lock_waits;
SELECT * FROM performance_schema.data_locks;

-- 查看当前事务
SELECT * FROM information_schema.INNODB_TRX;
```

### 7.3 预防策略

1. **固定加锁顺序** → 按 ID 升序更新，避免交叉更新
2. **缩小事务范围** → 事务尽量短，减少锁持有时间
3. **避免大范围锁** → 用索引精确查询，避免全表扫描
4. **设置锁等待超时** → `innodb_lock_wait_timeout = 5`（秒）
5. **读写分离** → 读操作不加锁（MVCC），减少锁竞争
6. **降低隔离级别** → RC 比 RR 锁范围小（无间隙锁）

### 7.4 死锁处理流程

```
1. 应用层捕获死锁异常（DeadlockLoserDataAccessException）
2. 自动重试（最多 3 次，指数退避）
3. 重试仍失败 → 记录日志 + 告警
4. 人工分析死锁日志 → 优化 SQL/事务设计
```

---

## 8. SQL 优化实战

### 8.1 EXPLAIN 关键字段

| 字段 | 含义 | 关注点 |
|------|------|--------|
| type | 访问类型 | 至少 range，最好 ref/const |
| key | 实际使用的索引 | 是否用了预期索引 |
| rows | 预估扫描行数 | 越少越好 |
| Extra | 额外信息 | Using index 好，Using filesort/temporary 差 |

### 8.2 慢查询优化流程

```
1. 开启慢查询日志
   slow_query_log = ON
   long_query_time = 1

2. 分析慢查询
   mysqldumpslow -s t slow.log | head -20

3. EXPLAIN 分析执行计划

4. 优化方向：
   - 加索引
   - 改写 SQL（避免 SELECT *，避免函数操作）
   - 拆分大查询
   - 分页优化
```

### 8.3 分页优化

```sql
-- 慢：LIMIT 1000000, 20（深度分页）
SELECT * FROM orders ORDER BY id LIMIT 1000000, 20;

-- 优化方案 1：延迟关联
SELECT * FROM orders o
INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 1000000, 20) t
ON o.id = t.id;

-- 优化方案 2：游标分页（记住上一页最后一个 id）
SELECT * FROM orders WHERE id > ? ORDER BY id LIMIT 20;

-- 优化方案 3：覆盖索引
SELECT id FROM orders ORDER BY id LIMIT 1000000, 20;
-- 然后用 id 查详情
```

### 8.4 COUNT 优化

```sql
-- 慢：COUNT(*) 大表
SELECT COUNT(*) FROM orders WHERE status = 'PAID';

-- 优化方案 1：缓存计数
-- 用 Redis 维护计数器，业务变更时同步更新

-- 优化方案 2：汇总表
-- 定时任务统计到汇总表
SELECT count FROM order_stats WHERE status = 'PAID';

-- 优化方案 3：近似值
-- 不需要精确值时用 EXPLAIN 估算
EXPLAIN SELECT * FROM orders WHERE status = 'PAID';
-- rows 字段为估算值
```

---

## 9. MVCC 原理与业务影响

### 9.1 MVCC 核心机制

```
每行数据隐藏字段：
  - DB_TRX_ID：最后修改的事务 ID
  - DB_ROLL_PTR：指向 undo log 的指针
  - DB_ROW_ID：行 ID（无主键时）

ReadView（读视图）：
  - 创建事务时生成
  - 包含：当前活跃事务列表
  - 决定当前事务能看到哪些版本

版本链：
  当前数据 → undo log 旧版本 1 → undo log 旧版本 2 → ...
```

### 9.2 RC vs RR 的快照差异

```
READ COMMITTED：
  - 每条 SELECT 都创建新的 ReadView
  - 能看到其他事务已提交的最新数据
  - 不可重复读

REPEATABLE READ：
  - 事务开始时创建 ReadView，整个事务复用
  - 看到的是事务开始时的快照
  - 可重复读
```

### 9.3 业务影响

1. **长事务危害**
   - 阻塞 purge → undo log 膨胀
   - 占用锁资源
   - 建议：监控长事务，超时自动 kill

2. **快照数据可能过期**
   - RR 级别下，长事务看到的是旧数据
   - 业务依赖最新数据时需注意

3. **并发更新需注意**
   - 即使 MVCC，UPDATE 仍是当前读
   - 并发更新需乐观锁或悲观锁

---

## 10. 主从复制原理与延迟优化

### 10.1 复制流程

```
主库：
  1. 事务提交 → 写 Binlog
  2. Binlog Dump Thread 推送给从库

从库：
  1. IO Thread 接收 Binlog → 写 Relay Log
  2. SQL Thread 执行 Relay Log → 数据更新

复制方式：
  - 异步复制：主库不等待从库确认（默认）
  - 半同步复制：主库等待至少一个从库确认
  - 全同步复制：主库等待所有从库确认（性能差，少用）
```

### 10.2 延迟原因

1. **从库单线程回放**（MySQL 5.6 之前） → 大事务导致延迟
2. **网络延迟** → 主从跨机房
3. **从库性能差** → 硬件不如主库
4. **大事务** → 单事务执行时间长
5. **DDL 操作** → 阻塞复制

### 10.3 优化方案

```
1. MySQL 5.7+ 多线程复制（基于组提交）
   slave_parallel_type = LOGICAL_CLOCK
   slave_parallel_workers = 8

2. MySQL 8.0 写集并行复制
   binlog_transaction_dependency_tracking = WRITESET

3. 半同步复制（减少数据丢失）
   rpl_semi_sync_master_enabled = 1

4. 业务层优化
   - 大事务拆小
   - 批量操作分批
   - 避免长事务
```

---

## 11. 数据库选型与避坑速查

> 本节把数据库选型决策融入业务场景讲，不脱离需求谈技术。选型的本质是"为业务边界选最合适的工具"。

### 11.1 何时该用 MySQL，何时该换

```
该用 MySQL：
  ✓ 核心业务数据（订单、账户、支付）→ 需要事务
  ✓ 强一致、关系复杂的数据 → 关系型天生适合
  ✓ 中小规模（单表 < 5000 万）→ 性能够用

考虑其他方案：
  △ 单表 > 5000 万 → 分库分表 / TiDB
  △ 全文搜索需求 → Elasticsearch 辅助
  △ 灵活 schema（内容管理）→ MongoDB
  △ 时序数据（监控）→ TDengine / InfluxDB
  △ 海量日志分析 → ClickHouse
  △ 图关系（社交）→ Neo4j
```

### 11.2 MySQL vs NewSQL（TiDB / PolarDB）

当单表数据量大到要分库分表时，另一个选择是 NewSQL：

| 维度 | MySQL 分库分表 | TiDB | PolarDB |
|------|----------------|------|---------|
| 一致性 | 最终一致（跨库） | 强一致（Raft） | 强一致 |
| 扩展性 | 需重新分片 | 自动扩缩容 | 存算分离 |
| 运维 | 复杂（中间件） | 中等 | 简单（云） |
| 生态 | 成熟 | 兼容 MySQL | 兼容 MySQL |
| 成本 | 自建低 | 自建高 | 云上按需 |
| 适用 | 已有 MySQL 团队 | 强一致 + 海量 | 云上业务 |

**选型口诀**：
- 已有 MySQL 经验、求稳 → MySQL 分库分表
- 强一致 + 海量 + 自建 → TiDB
- 云上业务 + 弹性 → PolarDB

### 11.3 MySQL 常见踩坑速查

| 坑 | 表现 | 解决 |
|----|------|------|
| 大事务 | 锁表久、undo log 膨胀 | 事务 < 1s、拆分 |
| 深分页 | LIMIT 1000000, 20 慢 | 游标分页 / 延迟关联 |
| 索引失效 | 隐式转换、函数操作 | 用 EXPLAIN 验证 |
| 死锁 | 交叉更新 | 固定加锁顺序 |
| 慢查询 | 全表扫描 | 加索引 / 改 SQL |
| 主从延迟 | 读写不一致 | 强制读主 / 半同步 |
| 大表 DDL | ALTER 锁表久 | pt-online-schema-change |
| 连接数满 | Too many connections | 连接池 / 读写分离 |

### 11.4 索引设计 Checklist

- [ ] 高频查询字段都有索引
- [ ] 联合索引顺序符合最左前缀
- [ ] 避免超过 5 个单表索引
- [ ] EXPLAIN 验证 type >= range
- [ ] 无 Using filesort / Using temporary
- [ ] 无 SELECT *（用覆盖索引）
- [ ] 定期清理未使用索引

---

## 12. 数据生命周期管理

数据不是越多越好。随业务增长，历史数据堆积会拖垮性能。数据生命周期管理是中大型系统的必修课。

### 12.1 冷热分离

```
问题：订单表 5 亿条，查询慢、备份慢、DDL 慢

方案：冷热分离
  热数据（近 3 个月）：MySQL 主库，SSD，高配置
  冷数据（3 个月以上）：归档库/对象存储，HDD，低配置

实现：
  1. 定时任务扫描超期数据 → 迁移到冷库
  2. 主库只保留热数据
  3. 查询路由：近期走主库，历史走冷库
  4. 冷库可选：MySQL（低配）、HBase、对象存储 + 索引
```

### 12.2 数据归档策略

| 策略 | 说明 | 适用 |
|------|------|------|
| 按时间归档 | 按月/季度归档 | 订单、日志、流水 |
| 按状态归档 | 已完成/已关闭的归档 | 工单、任务 |
| 按访问频率归档 | 长期未访问的归档 | 用户行为、操作记录 |
| 全量归档 | 保留摘要，明细归档 | 财务明细 |

### 12.3 归档实现方式

```
方式 1：定时任务迁移
  - 定时任务扫描 → INSERT 归档库 → DELETE 主库
  - 注意：分批 + 事务 + 幂等
  - 详见 [07-分布式定时任务](../05-稳定性与可观测/00-分布式定时任务与调度.md)

方式 2：Binlog 同步 + 删除
  - Canal 订阅 Binlog → 写归档库
  - 定时任务删除主库历史数据
  - 优点：不影响主库写入

方式 3：分区表
  - MySQL 分区表（PARTITION BY RANGE）
  - 老分区直接 DROP
  - 优点：原生支持、无锁
  - 限制：分区键必须是主键一部分
```

### 12.4 数据软删除 vs 硬删除

```
软删除（推荐）：
  - 加 deleted 字段
  - 查询加 WHERE deleted = 0
  - 定时任务定期物理删除
  - 优点：可恢复、审计友好

硬删除：
  - DELETE 物理删除
  - 优点：表小、性能好
  - 缺点：不可恢复

实践：先软删，定期（如 90 天后）硬删
```

### 12.5 数据膨胀控制

```
1. 日志表
   - 定期清理（如保留 30 天）
   - 大日志转 ES/HBase

2. 临时表
   - 用完即删
   - 命名带 expire

3. 大字段
   - 大文本/二进制单独表存储
   - 或转对象存储（OSS/S3）

4. 冗余字段
   - 定期审查冗余
   - 不必要的冗余清理
```

### 12.6 数据生命周期 Checklist

- [ ] 明确各类数据的保留周期
- [ ] 冷热分离方案落实
- [ ] 归档定时任务上线
- [ ] 归档数据可查询（历史查询接口）
- [ ] 软删除策略统一
- [ ] 大表有分区或归档
- [ ] 备份策略覆盖冷数据
- [ ] 合规要求满足（如金融数据保留 7 年）

---

## 13. MySQL/InnoDB 底层原理八股

> 本节梳理 MySQL 面试高频的底层原理。理解原理才能做好索引优化、事务设计、故障排查。

### 13.1 InnoDB 架构

```
InnoDB 内存结构：
  Buffer Pool：缓存数据页和索引页（核心）
  Change Buffer：缓存二级索引的修改
  Adaptive Hash Index：自适应哈希索引
  Log Buffer：redo log 缓冲

InnoDB 线程：
  Master Thread：核心后台（刷脏页、合并插入缓冲）
  IO Thread：异步 IO
  Purge Thread：清理 undo log
  Page Cleaner Thread：刷脏页

InnoDB 磁盘结构：
  系统表空间：ibdata1
  独立表空间：.ibd（每表一个）
  undo 表空间：undo log
  redo log 文件：ib_logfile
  临时表空间
```

### 13.2 B+ 树索引原理

```
B+ 树特点：
- 非叶子节点只存索引（不存数据）→ 扇出大
- 叶子节点存数据 + 有序链表 → 范围查询高效
- 树高通常 3-4 层 → 千万级数据 3-4 次 IO

InnoDB 索引：
- 聚簇索引（主键）：叶子节点存完整行数据
- 二级索引：叶子节点存主键值（需回表）

为什么用 B+ 树不用 B 树/红黑树/Hash？
- vs B 树：B+ 树非叶子不存数据，扇出更大，树更矮
- vs 红黑树：红黑树二叉，千万级数据树太高
- vs Hash：Hash 不支持范围查询

页（Page）：
- InnoDB 最小存储单位 16KB
- 一个页存多行数据
- 页是读写 IO 的基本单位
```

### 13.3 MVCC 实现原理

```
MVCC（多版本并发控制）核心：

每行隐藏字段：
- DB_TRX_ID：最后修改的事务 ID
- DB_ROLL_PTR：指向 undo log 的指针
- DB_ROW_ID：行 ID（无主键时）

undo log 版本链：
  当前数据 → undo log(旧版本1) → undo log(旧版本2) → ...

ReadView（读视图）：
  - m_ids：生成时活跃事务 ID 列表
  - min_trx_id：m_ids 最小值
  - max_trx_id：下一个事务 ID
  - creator_trx_id：当前事务 ID

可见性判断：
  版本的 trx_id < min_trx_id → 可见（已提交）
  版本的 trx_id >= max_trx_id → 不可见（未来事务）
  版本的 trx_id 在 m_ids 中 → 不可见（活跃中）
  版本的 trx_id 不在 m_ids 中 → 可见（已提交）
  否则不可见 → 沿 undo log 找上一个版本

RC vs RR：
- RC：每条 SELECT 生成新 ReadView → 看到最新已提交
- RR：事务开始生成 ReadView，整个事务复用 → 可重复读
```

### 13.4 锁机制详解

```
InnoDB 锁类型：

行锁：
- 记录锁（Record Lock）：锁单条记录
- 间隙锁（Gap Lock）：锁记录间的间隙（RR 才有）
- 临键锁（Next-Key Lock）：记录锁 + 间隙锁（RR 默认）

表锁：
- 意向锁（IS/IX）：行锁前先加表级意向锁
- AUTO-INC 锁：自增列

锁的兼容性：
- 共享锁（S）vs 排他锁（X）：互斥
- 意向锁之间兼容
- 意向锁与行级 S/X 不冲突（不同层级）

RR 的间隙锁：
  SELECT * FROM t WHERE id BETWEEN 10 AND 20 FOR UPDATE
  → 锁 (10, 20] + 间隙 (negative, 10) 和 (20, positive)
  → 阻止 INSERT id=15

死锁：
  - 交叉加锁顺序不同
  - 间隙锁冲突
  - InnoDB 自动检测死锁 → 回滚代价小的事务
```

### 13.5 日志系统（redo / undo / binlog）

```
三种日志：

redo log（InnoDB 引擎层）：
- 记录数据页的物理修改
- 保证 crash 安全（持久性）
- WAL（Write Ahead Log）：先写 redo log 再写数据页
- 循环写，固定大小

undo log（InnoDB 引擎层）：
- 记录修改前的数据
- 保证事务回滚（原子性）
- 支持 MVCC（版本链）
- 存于 undo 表空间

binlog（Server 层）：
- 记录所有写操作（逻辑日志）
- 用于复制和恢复
- 追加写，文件切换

两阶段提交（redo log + binlog）：
  1. 写 redo log（prepare 状态）
  2. 写 binlog
  3. 写 redo log（commit 状态）
  
  保证 redo log 和 binlog 一致
  crash 恢复时：
  - redo log prepare + binlog 完整 → commit
  - redo log prepare + binlog 不完整 → rollback
```

### 13.6 WAL 与刷脏页

```
WAL（Write Ahead Log）：
- 先写日志（redo log）再改数据页
- 数据页异步刷盘
- crash 后用 redo log 恢复

刷脏页（Buffer Pool → 磁盘）触发：
- redo log 满 → 强制刷脏页（可能卡顿）
- Buffer Pool 空间不足
- MySQL 正常关闭
- 后台线程定期刷

刷脏页性能：
- innodb_io_capacity：每秒刷页数（SSD 调高）
- 脏页比例 > 70% → 激进刷
- 平滑刷页避免卡顿
```

### 13.7 事务实现原理

```
ACID 实现：
- 原子性（A）：undo log（回滚）
- 持久性（D）：redo log（crash 恢复）
- 隔离性（I）：锁 + MVCC
- 一致性（C）：以上三者共同保证

隔离级别实现：
- READ UNCOMMITTED：无锁无 MVCC
- READ COMMITTED：MVCC（每语句新 ReadView）
- REPEATABLE READ：MVCC（首次 ReadView）+ 间隙锁
- SERIALIZABLE：所有读加锁
```

### 13.8 主从复制原理

```
复制流程：
1. 主库写 binlog
2. 从库 IO Thread 拉取 binlog → 写 relay log
3. 从库 SQL Thread 执行 relay log

复制方式：
- 异步复制：主库不等从库（默认）
- 半同步复制：主库等至少一个从库收到 binlog
- 全同步：主库等所有从库（性能差）

并行复制（5.7+）：
- 基于 GROUP COMMIT（LOGICAL_CLOCK）
- 从库多线程回放
- 8.0 WRITESET 进一步优化

复制格式：
- STATEMENT：记 SQL（小，但有些函数不安全）
- ROW：记行变更（大，但安全）
- MIXED：混合
```

### 13.9 高频面试题速记

```
Q: 为什么 InnoDB 用 B+ 树？
A: 扇出大、树矮、范围查询高效

Q: 聚簇索引 vs 二级索引？
A: 聚簇存数据，二级存主键需回表

Q: MVCC 怎么实现？
A: 隐藏字段 + undo log 版本链 + ReadView

Q: RR 如何防幻读？
A: 间隙锁 + MVCC

Q: redo log 和 binlog 区别？
A: redo 物理日志(引擎层)，binlog 逻辑日志(Server层)

Q: 为什么两阶段提交？
A: 保证 redo 和 binlog 一致

Q: 主从延迟原因？
A: 单线程回放（5.7前）、大事务、网络

Q: 索引失效场景？
A: 函数操作、隐式转换、OR、!=、LIKE '%x'
```

---

## 关联阅读

- [01-Redis 业务场景实战](./00-Redis业务场景实战.md) - 缓存设计
- [03-数据一致性专题](../02-数据一致性/00-数据一致性专题.md) - 主从一致性、缓存一致性
- [04-分布式系统核心设计模式](../03-分布式架构与设计/00-分布式系统核心设计模式.md) - 分布式 ID、性能优化
- [05-中间件组合架构实战](./03-中间件组合架构实战.md) - 数据同步架构、数据迁移
- [07-分布式定时任务与调度](../05-稳定性与可观测/00-分布式定时任务与调度.md) - 归档任务调度

