# Redis 业务场景实战

> 本篇聚焦 Redis 在真实业务场景中的设计与使用，从"为什么用"到"怎么用"到"怎么避坑"。

---

## 目录

- [1. 布隆过滤器 —— 防缓存穿透](#1-布隆过滤器--防缓存穿透)
- [2. 分布式锁 —— 防并发重复操作](#2-分布式锁--防并发重复操作)
- [3. 限流器 —— 接口保护](#3-限流器--接口保护)
- [4. 排行榜 —— ZSet 实战](#4-排行榜--zset-实战)
- [5. 消息队列 —— 异步解耦](#5-消息队列--异步解耦)
- [6. 延迟队列 —— 超时处理](#6-延迟队列--超时处理)
- [7. GEO 服务 —— 位置相关业务](#7-geo-服务--位置相关业务)
- [8. 会话管理 —— 登录态与Token](#8-会话管理--登录态与token)
- [9. 库存扣减 —— 高并发抢购](#9-库存扣减--高并发抢购)
- [10. HyperLogLog —— UV 统计](#10-hyperloglog--uv-统计)
- [11. 位图 Bitmap —— 用户行为标记](#11-位图bitmap--用户行为标记)
- [12. 发布订阅 Pub/Sub —— 实时通知](#12-发布订阅pubsub--实时通知)
- [13. 缓存三大问题 —— 穿透/雪崩/击穿](#13-缓存三大问题--穿透雪崩击穿)
- [14. Lua 脚本实战要点](#14-lua-脚本实战要点)
- [15. 集群模式与选型](#15-集群模式与选型)
- [16. 持久化策略](#16-持久化策略)

---

## 1. 布隆过滤器 —— 防缓存穿透

### 1.1 业务痛点

用户查询一个数据库中绝对不存在的数据，缓存层也没有，每次请求都打到数据库，形成"穿透"。典型场景：

- **恶意攻击**：黑客构造大量不存在的 ID 发起查询
- **爬虫遍历**：系统 ID 被枚举式遍历
- **业务误操作**：前端传错参数，ID 格式不合法
- **已删除数据查询**：商品下架后仍被访问

### 1.2 设计方案

```
请求 → 布隆过滤器(判断可能存在) → Redis缓存 → MySQL
                    ↓ 不存在
              直接返回空（不查DB）
```

**核心思路**：布隆过滤器说"不存在"则**一定不存在**，说"存在"则**可能存在**（有误判率）。

### 1.3 原理与参数

布隆过滤器由一个位数组（bit array）和 k 个哈希函数组成：

```
添加元素 x：
  对 x 用 k 个哈希函数得到 k 个位置
  把位数组这 k 个位置都置为 1

查询元素 y：
  对 y 用 k 个哈希函数得到 k 个位置
  如果所有位置都是 1 → "可能存在"（可能误判）
  如果有任一位置是 0 → "一定不存在"
```

**关键参数计算**（已知预期元素数 n 和误判率 p）：

```
位数组大小：m = -n * ln(p) / (ln2)^2
哈希函数数：k = (m/n) * ln2

示例：1 亿用户 ID，0.01% 误判率
  m ≈ 1.37 亿 bit ≈ 17 MB
  k ≈ 7
```

### 1.4 实现要点

**RedisBitmap 实现布隆过滤器**：

```python
# 关键参数预先计算好
BIT_SIZE = 134_000_000     # 1.37亿bit
HASH_FUNC_COUNT = 7

def bloom_add(key, value):
    for i in range(HASH_FUNC_COUNT):
        # 多次哈希（实际可用 murmur3 + 不同种子）
        pos = hash_func(value, i) % BIT_SIZE
        # 注意：SETBIT 是原子操作，可并发
        redis.execute('SETBIT', key, pos, 1)

def bloom_exists(key, value):
    for i in range(HASH_FUNC_COUNT):
        pos = hash_func(value, i) % BIT_SIZE
        if redis.execute('GETBIT', key, pos) == 0:
            return False  # 一定不存在
    return True  # 可能存在
```

**Lua脚本原子添加**（避免多次往返网络）：

```lua
-- bloom_add.lua
local key = KEYS[1]
local val = ARGV[1]
local k = tonumber(ARGV[2])
local m = tonumber(ARGV[3])

for i = 0, k-1 do
    -- 实际 hash 算法在脚本内实现，或用现成 murmur3
    local hash = crc32(val .. tostring(i)) % m
    redis.call('SETBIT', key, hash, 1)
end
return 1
```

### 1.5 业务决策要点

| 决策项 | 选项 | 推荐 |
|--------|------|------|
| 误判率设定 | 0.1% / 0.01% / 0.001% | 电商推荐 0.01%；金融类严格 0.001% |
| 数据更新 | 定期重建 / 实时添加 | 冷数据定期重建；热数据实时添加 |
| 溢出处理 | 自动扩容 / 固定容量 | 固定容量 + 定期重建更可控 |
| 空值缓存 | 同时缓存空结果 | 配合使用，布隆挡大部分，空值缓存兜底 |

### 1.6 常见踩坑

1. **布隆过滤器不能删除元素**
   - 问题：多个元素可能哈希到同一位，删除会影响其他元素
   - 解决：使用 Counting Bloom Filter（每个 bit 变成计数器），或定期全量重建

2. **误判率上升**
   - 问题：元素数超过预期后误判率急剧上升
   - 解决：监控实际元素量，超阈值时触发扩容或重建

3. **重建期间穿透**
   - 问题：重建过程中部分请求可能"漏网"
   - 解决：双布隆过滤器交替重建（A 在线服务，B 后台重建，完成后切换）

4. **Redis 内存占用评估**
   - 1 亿 ID 约 17 MB，可接受
   - 10 亿级需要评估，可能需要分片

### 1.7 扩展应用

- **URL 去重**：爬虫场景已爬取 URL 去重
- **推荐去重**：用户已推荐过的内容不再推荐
- **邮件/手机号黑名单**：亿级黑名单快速判断
- **缓存预热判断**：哪些 key 应该预加载

---

## 2. 分布式锁 —— 防并发重复操作

### 2.1 业务痛点

- 防止用户重复提交订单（前端防抖不够，后端必须兜底）
- 防止定时任务多实例重复执行
- 防止库存并发超卖
- 防止同一用户并发修改数据
- 防止配置并发修改冲突

### 2.2 方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| Redis SETNX | 性能高，实现简单 | 单点故障风险，无续约机制 | 低可靠性要求 |
| Redis + Redisson | 看门狗续约，可重入 | 仍为 AP 系统，极端情况可能丢锁 | 大多数业务场景 |
| Redis RedLock | 多节点容错 | 争议大，时钟漂移问题 | 高可靠性但非强一致 |
| ZooKeeper | 强一致，临时节点自动释放 | 性能低，运维复杂 | 强一致性要求 |
| etcd | 强一致，Lease 机制 | 运维复杂 | K8s 生态 |
| MySQL 行锁 | 天然事务保障 | 性能低，DB 压力大 | 已在事务中的场景 |

### 2.3 Redisson 分布式锁实战

**加锁流程**：
```
SET lock_key unique_value NX PX 30000
  → 成功：获得锁
  → 失败：自旋等待 / 快速失败
```

**看门狗续约机制**（Redisson 默认）：
```
锁默认过期时间 30 秒
看门狗每 10 秒（leaseTime/3）检查：
  → 锁仍被当前线程持有 → 续期到 30 秒
  → 锁已释放 → 停止续约
```

**释放锁（Lua 原子操作）**：
```lua
-- 释放锁必须校验 unique_value，防止误删他人锁
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
```

### 2.4 业务决策要点

| 场景 | 锁策略 | 说明 |
|------|--------|------|
| 用户防重复提交 | 超时自动释放（3-5s） | 用户体验优先，宁可短暂锁定 |
| 定时任务互斥 | 看门狗续约 | 任务可能跑很久，需要续约 |
| 库存扣减 | 乐观锁兜底 + 分布式锁 | 锁只是防线之一，最终靠 DB |
| 数据修改 | 可重入锁 | 同一用户多次进入需可重入 |
| 配置变更 | 公平锁 | 按请求顺序获取，避免饥饿 |

### 2.5 常见踩坑

1. **锁过期但业务未完成**
   - 问题：业务执行时间超过锁过期时间，锁被自动释放，其他线程获取到锁
   - 解决：看门狗续约机制（Redisson 默认开启）

2. **误删他人锁**
   - 问题：A 持有锁过期释放，B 获取到锁，A 业务完成后删除了 B 的锁
   - 解决：释放时必须校验 unique_value（Lua 原子操作）

3. **锁等待超时**
   - 问题：高并发下大量线程等待锁，可能引发雪崩
   - 解决：设置合理等待时间，业务端要有降级策略

4. **RedLock 的争议**
   - Martin Kleppmann 指出：时钟漂移可能导致锁失效
   - Antirez 反驳：实际场景时钟漂移可控
   - 实践建议：单节点 Redisson 锁已够用；强一致需求用 ZK

5. **可重入锁的实现陷阱**
   - 问题：客户端崩溃后重入计数无法释放
   - 解决：使用 Hash 结构存储（key=锁名，field=客户端ID，value=重入次数）

---

## 3. 限流器 —— 接口保护

### 3.1 业务痛点

- 秒杀/抢购场景瞬时流量暴增
- OpenAPI 调用次数限制（如每用户每天 100 次）
- 防爬虫、防刷接口
- 系统容量有限，需要平滑限流
- 防止下游服务被压垮

### 3.2 限流算法对比

| 算法 | 特点 | 临界问题 | 适用场景 |
|------|------|----------|----------|
| 固定窗口计数 | 简单 | 窗口边界突发 2 倍流量 | 低精度要求 |
| 滑动窗口计数 | 平滑 | 内存开销稍大 | 中精度要求 |
| 滑动窗口日志 | 精确 | 内存开销大 | 精确限流 |
| 令牌桶 | 允许突发 | 实现稍复杂 | API 限流（允许短突发） |
| 漏桶 | 匀速输出 | 不允许突发 | 流量整形（匀速消费） |

### 3.3 Redis 实现滑动窗口限流

**Lua 脚本原子操作**：

```lua
-- sliding_window_rate_limit.lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])    -- 窗口大小（秒）
local threshold = tonumber(ARGV[3]) -- 阈值

-- 1. 清除窗口外的记录
redis.call('ZREMRANGEBYSCORE', key, 0, now - window * 1000)

-- 2. 统计当前窗口内请求数
local count = redis.call('ZCARD', key)

-- 3. 判断是否超过阈值
if count >= threshold then
    return 0  -- 拒绝
end

-- 4. 记录本次请求
redis.call('ZADD', key, now, now .. '-' .. math.random())
redis.call('EXPIRE', key, window)

return 1  -- 允许
```

### 3.4 令牌桶限流（Redis + Lua）

```lua
-- token_bucket.lua
local key = KEYS[1]
local max_tokens = tonumber(ARGV[1])     -- 桶容量
local refill_rate = tonumber(ARGV[2])    -- 每秒补充速率
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])      -- 本次请求令牌数

local last_refill = tonumber(redis.call('HGET', key, 'last_refill')) or now
local current_tokens = tonumber(redis.call('HGET', key, 'tokens')) or max_tokens

-- 计算补充量
local elapsed = now - last_refill
local refill = elapsed * refill_rate / 1000  -- ms 转 秒
current_tokens = math.min(max_tokens, current_tokens + refill)

if current_tokens >= requested then
    current_tokens = current_tokens - requested
    redis.call('HMSET', key, 'tokens', current_tokens, 'last_refill', now)
    return 1  -- 允许
else
    redis.call('HMSET', key, 'tokens', current_tokens, 'last_refill', now)
    return 0  -- 拒绝
end
```

### 3.5 业务决策要点

| 场景 | 算法 | 粒度 | 说明 |
|------|------|------|------|
| 秒杀抢购 | 令牌桶 | 全局 + 用户双重 | 全局限总 QPS，用户限单人频率 |
| OpenAPI | 令牌桶 | 按用户/按 key | 允许短时突发，但总量受限 |
| 防爬虫 | 滑动窗口 | 按 IP | 精确统计，封禁阈值 |
| 消息消费 | 漏桶 | 消费者维度 | 匀速消费，保护下游 DB |
| 内部服务调用 | 固定窗口 | 按服务 | 简单够用 |

### 3.6 多级限流设计

```
第一级：网关层限流（Nginx/网关）
  - 全局 QPS 上限
  - IP 维度限流（防爬虫）

第二级：应用层限流（业务服务）
  - 用户维度限流（防刷）
  - 接口维度限流（保护慢接口）

第三级：资源层限流（DB/Redis）
  - 连接池限流
  - 慢查询限流
```

---

## 4. 排行榜 —— ZSet 实战

### 4.1 业务场景

- 游戏积分排行
- 电商销量排行
- 社交热度排行（热搜、热帖）
- 用户活跃度排行
- 投资收益排行
- 签到连续天数排行

### 4.2 核心操作

```redis
# 添加/更新分数
ZADD leaderboard <score> <member>

# 获取 Top N（从高到低）
ZREVRANGE leaderboard 0 N-1 WITHSCORES

# 获取用户排名
ZREVRANK leaderboard <member>

# 获取用户分数
ZSCORE leaderboard <member>

# 分数增量（如签到 +1 分）
ZINCRBY leaderboard <increment> <member>

# 获取分数区间用户数
ZCOUNT leaderboard 100 200

# 移除过期数据
ZREMRANGEBYSCORE leaderboard 0 <threshold>
```

### 4.3 进阶设计

**多维排行榜**：同一维度用单个 ZSet，多维度多个 ZSet

```redis
# 销量排行
ZADD sales_rank <sales_count> <product_id>

# 评分排行（浮点精度处理：乘以 1000 转整数）
ZADD rating_rank <avg_rating*1000> <product_id>

# 综合排行：加权计算后写入
# composite_score = sales * 0.6 + rating * 0.4
ZADD composite_rank <composite_score> <product_id>
```

**分页排行**：大排行榜的分页

```redis
# 方案一：ZREVRANGE + 偏移量（数据量大时偏移性能差）
ZREVRANGE leaderboard 100000 100020 WITHSCORES  -- 10 万偏移很慢！

# 方案二：记录上一页最后一个元素的 score，用 ZREVRANGEBYSCORE
ZREVRANGEBYSCORE leaderboard <last_score> -INF WITHSCORES LIMIT 0 20
```

**周期性排行榜**：每日/每周/每月排行

```redis
# 按日期建 key
leaderboard:2024-01-15
leaderboard:2024-01-16
...

# 每日零点定时任务：合并为周榜
ZUNIONSTORE leaderboard:weekly 7 daily:2024-01-15 daily:2024-01-16 ... WEIGHTS 1 1 1 1 1 1 1

# 月度同理
ZUNIONSTORE leaderboard:monthly 30 daily_keys WEIGHTS ...
```

### 4.4 常见踩坑

1. **分数精度**：ZSet score 是 double，整数安全但浮点有精度问题 → 乘以 1000 转整数
2. **大偏移分页**：ZREVRANGE 大偏移量 O(log(N)+M) 偏移 → 改用 score 范围查询
3. **成员重复**：同一 member 只能出现一次，分数会被覆盖 → 需要区分时用复合 key 如 `user_id:period`
4. **冷数据堆积**：无限增长的排行榜 → 定期 ZREMRANGEBYSCORE 清理低分数据
5. **排行榜数据迁移**：member 改名后需要重新建 ZSet → 设计好 member 命名规范

---

## 5. 消息队列 —— 异步解耦

### 5.1 Redis Stream vs List 对比

| 特性 | List | Stream |
|------|------|---------|
| 消息持久化 | AOF/RDB | 专门持久化 |
| 消费者组 | 不支持 | XGROUP 支持 |
| 消息 ID | 无自动 ID | 自动时间戳 ID |
| 消息确认 | 不支持 | XACK 支持 |
| 消息回溯 | 不支持 | XRANGE 可回溯 |
| 阻塞消费 | BLPOP | XREAD GROUP BLOCK |
| 消息堆积 | 只能 LRANGE | 自动 Trim |

### 5.2 Stream 消费者组模式

```redis
# 创建消费者组
XGROUP CREATE mystream mygroup $ MKSTREAM

# 生产消息
XADD mystream * field1 value1 field2 value2

# 消费消息（组内分配，自动负载均衡）
XREADGROUP GROUP mygroup consumer1 COUNT 1 BLOCK 2000 STREAMS mystream >

# 确认消息
XACK mystream mygroup <message_id>

# 查看待处理消息（未确认）
XPENDING mystream mygroup

# 死信处理：超过 N 次未确认的消息重新分配
XCLAIM mystream mygroup consumer2 <idle_time> <message_id>
```

### 5.3 业务适用场景

| 场景 | 选择 | 说明 |
|------|------|------|
| 简单异步任务 | List | 如短信发送、邮件通知 |
| 可靠消息消费 | Stream | 如订单状态变更通知 |
| 多消费者负载均衡 | Stream + Consumer Group | 如日志多端处理 |
| 延时任务 | Stream + 定时扫描 | 轻量延时队列 |

### 5.4 重要提示

Redis 消息队列**适合轻量级场景**。正式业务推荐 RocketMQ/Kafka：

- Redis 丢消息风险（内存有限，AOF 非强一致）
- Redis 无事务性消息（如订单创建 + 消息发送需一致性）
- Redis 消费者故障恢复不如专业 MQ
- Redis 无死信队列、重试机制、消息轨迹等高级特性

详见 [05-中间件组合架构](./03-中间件组合架构实战.md) 中的 MQ 选型。

---

## 6. 延迟队列 —— 超时处理

### 6.1 业务场景

- 订单超时未支付自动取消
- 活动到期自动关闭
- 预约提醒（提前 30 分钟通知）
- 数据延迟同步
- 退款延迟审核
- 定时任务调度

### 6.2 方案对比

| 方案 | 精度 | 可靠性 | 实现复杂度 | 适用场景 |
|------|------|--------|------------|----------|
| Redis ZSet 轮询 | 秒级 | 中 | 低 | 中小规模 |
| Redis Stream 延时 | 秒级 | 中 | 中 | 需要消费者组 |
| RocketMQ 延时消息 | 秒级 | 高 | 低（内置） | 生产级 |
| RabbitMQ TTL+DLX | 秒级 | 高 | 中 | 已用 RabbitMQ |
| 时间轮算法 | 毫秒级 | 依赖存储 | 高 | 单机高性能 |
| 数据库定时扫描 | 分钟级 | 高 | 低 | 兜底方案 |
| Quartz 调度 | 分钟级 | 中 | 中 | 复杂调度 |

### 6.3 Redis ZSet 延迟队列设计

```redis
# 添加延迟任务
ZADD delay_queue <execute_timestamp> <task_id>

# 消费者轮询（Lua 原子操作）
ZRANGEBYSCORE delay_queue 0 <now_timestamp> LIMIT 0 <batch_size>
ZREM delay_queue <task_id_1> <task_id_2> ...
-- Lua 保证原子性：取到 + 移除必须原子
```

**完整 Lua 脚本**：

```lua
-- delay_queue_consume.lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local batch_size = tonumber(ARGV[2])

-- 取出到期任务
local tasks = redis.call('ZRANGEBYSCORE', key, 0, now, 'LIMIT', 0, batch_size)
if #tasks == 0 then
    return {}
end

-- 原子移除
for i, task in ipairs(tasks) do
    redis.call('ZREM', key, task)
end

return tasks
```

### 6.4 订单超时取消的完整设计

```
创建订单时：
  1. 写入 MySQL（订单状态 = 待支付）
  2. 写入 Redis ZSet delay_queue（30 分钟后执行）
  3. 可选：发 RocketMQ 延时消息（双保险）

30 分钟后消费延迟消息：
  1. 查 MySQL 订单状态
  2. 若仍为"待支付" → 更新为"已取消" + 释放库存
  3. 若已支付 → 忽略

兜底机制：
  - MySQL 定时任务每 5 分钟扫描超时未支付订单
  - 防 Redis 不可用时的遗漏
```

### 6.5 常见踩坑

1. **延迟任务堆积**：ZSet 无限增长 → 设置 max 常量或定期清理已执行任务
2. **轮询空转**：无任务时频繁轮询浪费 CPU → 使用阻塞等待或降低轮询频率
3. **任务重复消费**：多消费者竞争，ZREM 原子性保障 → 但要注意网络抖动导致取到但 REM 失败
4. **时间精度**：Redis 时间戳依赖系统时钟 → 确保各节点时钟同步（NTP）
5. **任务丢失**：Redis 宕机未持久化 → 重要任务用 MQ 方案

---

## 7. GEO 服务 —— 位置相关业务

### 7.1 业务场景

- 附近的人/商家/车辆
- 范围搜索（3 公里内的门店）
- 距离计算（两点之间）
- 热力图数据
- 配送范围判定
- 用户轨迹

### 7.2 核心操作

```redis
# 添加位置
GEOADD locations <longitude> <latitude> <member>

# 获取坐标
GEOPOS locations <member>

# 两点距离
GEODIST locations <member1> <member2> [km|m|mi|ft]

# 范围搜索（圆形）
GEORADIUS locations <longitude> <latitude> <radius> km
  [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT <n>] [ASC|DESC]

# 成员周围搜索
GEORADIUSBYMEMBER locations <member> <radius> km

# GeoHash 值
GEOHASH locations <member>
```

### 7.3 设计要点

1. **GEO 底层是 ZSet**：member=对象 ID，score=GeoHash 编码值 → 可以用 ZSet 命令辅助操作
2. **GeoHash 精度**：范围越大精度越低 → 短距离（<1km）精确，长距离（>100km）有误差
3. **大数据量**：百万级位置数据 → 考虑分区域存储不同 key（如 `locations:beijing`）
4. **更新频率**：车辆实时位置 → 合理设置更新间隔（如每 10 秒），避免频繁写入

### 7.4 业务实战案例：外卖配送范围

```redis
# 门店入库
GEOADD stores:city 116.397 39.908 "store_001"

# 用户搜索附近门店（3km 内，按距离排序，最多 20 个）
GEORADIUS stores:city 116.405 39.915 3 km WITHDIST COUNT 20 ASC

# 判断门店是否在配送范围内
GEODIST stores:city "store_001" "user_location" km
→ 结果 < 3.0 → 在配送范围内
```

### 7.5 进阶：海量位置数据

- 百万级以下：单 Redis 实例 GEO 命令
- 千万级：按城市分片存储（如 `locations:beijing`, `locations:shanghai`）
- 亿级：考虑专业空间数据库（PostGIS）或专门的 LBS 服务

---

## 8. 会话管理 —— 登录态与Token

### 8.1 业务场景

- 用户登录 Token 存储
- 多端登录互踢
- 单点登录（SSO）
- 登录态过期管理
- 验证码存储（短信/邮件验证码）

### 8.2 设计方案

```redis
# Token 存储
SET user:token:<user_id> <token_value> EX 7200  -- 2 小时过期

# 多端登录记录
SET user:session:<user_id>:<device_type> <session_info> EX 7200

# 互踢设计：新登录时
1. GET user:token:<user_id> → 获取旧 token
2. DEL user:token:<user_id>  → 删除旧 token（旧端自动失效）
3. SET user:token:<user_id> <new_token> EX 7200  → 写入新 token

# 验证码
SET verify:code:<phone> <code> EX 300  -- 5 分钟过期
-- 限制发送频率：INCR verify:limit:<phone> EX 60 → 每分钟最多 1 次
```

### 8.3 SSO 设计（基于 Redis）

```
登录成功：
1. 生成全局 token: SET sso:global:<token> <user_json> EX 7200
2. 各子系统验证: GET sso:global:<token>
3. 子系统可建立本地 session 缓存

单点登出：
DEL sso:global:<token>  → 所有子系统检查时发现 token 失效
```

### 8.4 Token 续期策略

| 策略 | 说明 | 适用 |
|------|------|------|
| 固定过期 | 到期重新登录 | 安全性高的场景 |
| 滑动过期 | 每次访问续期 | 用户体验优先 |
| 双 Token | access_token 短期 + refresh_token 长期 | 移动端常用 |

---

## 9. 库存扣减 —— 高并发抢购

### 9.1 业务痛点

秒杀/抢购场景下，大量请求同时扣减同一商品库存，容易出现超卖。

### 9.2 设计方案（多层防线）

```
第一层：Redis 预扣减（原子操作）
  DECR stock:<product_id>
  → 结果 >= 0：预扣减成功，允许下单
  → 结果 < 0：库存不足，直接拒绝

第二层：异步下单
  预扣减成功 → 发消息到 MQ → 订单服务异步创建订单

第三层：MySQL 最终确认
  UPDATE product_stock SET stock = stock - 1 WHERE id = ? AND stock > 0
  → 乐观锁兜底，Redis 预扣减可能因回滚而多扣

第四层：库存回补
  订单取消/支付超时 → Redis INCR stock:<product_id> + MySQL 回补
```

### 9.3 关键细节

```redis
# Redis 库存初始化（活动开始前）
SET stock:<product_id> <actual_stock_count>
```

**预扣减 + 限购（Lua 原子操作）**：

```lua
local stock_key = KEYS[1]
local limit_key = KEYS[2]       -- 用户限购 key
local user_id = ARGV[1]
local quantity = tonumber(ARGV[2])
local limit_num = tonumber(ARGV[3])

-- 检查限购
local bought = tonumber(redis.call('GET', limit_key) or '0')
if bought + quantity > limit_num then
    return -2  -- 超过限购
end

-- 检查库存
local stock = tonumber(redis.call('GET', stock_key) or '0')
if stock < quantity then
    return -1  -- 库存不足
end

-- 扣减库存 + 记录限购
redis.call('DECRBY', stock_key, quantity)
redis.call('INCRBY', limit_key, quantity)
redis.call('EXPIRE', limit_key, 86400)  -- 活动期间限购
return 1  -- 成功
```

### 9.4 常见踩坑

1. **Redis 预扣减与 MySQL 不一致** → Redis 扣了但订单失败，需要回补机制
2. **库存预热时机** → 活动开始前从 MySQL 同步到 Redis，注意同步时停止下单
3. **大 key 问题** → 单商品库存是简单数字，但限购记录用 Hash 更高效
4. **Redis 宕机** → 降级到纯 MySQL 乐观锁方案（性能下降但不超卖）
5. **热点商品** → 单 key 成为热点，考虑分片（如 `stock:product_001:1`, `stock:product_001:2`）

---

## 10. HyperLogLog —— UV 统计

### 10.1 业务场景

- 页面 UV 统计（独立访客数）
- 活动参与人数统计
- 搜索关键词独立用户数
- 广告曝光独立用户数

### 10.2 为什么不用 SET？

```redis
# SET 方案：存储每个用户 ID
SADD page_uv:<page_id> <user_id>
SCARD page_uv:<page_id>  -- 统计 UV

问题：
  - 100 万 UV → SET 占约 40 MB 内存
  - 1 亿 UV → SET 占约 4 GB 内存

# HyperLogLog 方案
PFADD page_uv:<page_id> <user_id>
PFCOUNT page_uv:<page_id>  -- 统计 UV（有 0.81% 标准误差）

优势：
  - 固定 12 KB 内存，无论多少 UV
  - 标准误差 0.81%，百万级 UV 可接受
```

### 10.3 设计要点

1. **误差 0.81%** → 百万 UV 误差约 8000，适合趋势统计，不适合精确计数
2. **不可取原始数据** → 只能计数，不能知道具体是谁访问了
3. **合并统计** → PFMERGE 合并多个 key → 如合并每日 UV 得到周 UV
4. **精确方案**：小规模用 SET，大规模用 HLL + 抽样校验

### 10.4 业务实战

```redis
# 每日 UV
PFADD page_uv:2024-01-15:<page_id> <user_id>

# 周 UV（合并 7 天）
PFMERGE page_uv:weekly:2024-W02 page_uv:2024-01-15:page_1 page_uv:2024-01-16:page_1 ...

# 月 UV
PFMERGE page_uv:monthly:2024-01 page_uv:2024-01-*:page_1
```

---

## 11. 位图 Bitmap —— 用户行为标记

### 11.1 业务场景

- 用户签到统计（连续签到天数）
- 用户特征标签（是否新用户、是否 VIP 等）
- 活跃用户统计（某日是否登录）
- 权限标记（某功能是否开通）

### 11.2 签到设计

```redis
# 用户每日签到（offset = 日期在年内的天数）
SETBIT sign:<user_id>:<year> <day_of_year> 1

# 统计本月签到天数
BITCOUNT sign:<user_id>:2024 <start_offset> <end_offset>

# 连续签到天数（从今天往前数连续 1 的个数）
-- 需要 Lua 脚本逐位检查

# 整体活跃统计
BITCOUNT active:2024-01-15  -- 当日活跃用户数
BITOP OR weekly_active active:mon active:tue ...  -- 周活跃
BITCOUNT weekly_active
```

### 11.3 内存优势

```
# 1 亿用户的日活跃标记
# SET 方案: SADD → 约 40 MB/天
# Bitmap 方案: SETBIT → 约 12 MB/天 (1 亿 bit)

# 一年 365 天的日活跃数据
# Bitmap → 约 4.4 GB (365 * 12 MB)
# SET → 约 14.6 GB
```

### 11.4 进阶：用户群组分析

```redis
# 计算连续 7 天活跃用户数
BITOP AND active_7days active:day1 active:day2 ... active:day7
BITCOUNT active_7days

# 计算流失用户（30 天前活跃，最近 7 天不活跃）
BITOP AND lost_users active_30days_ago NOT_active_7days
```

---

## 12. 发布订阅 Pub/Sub —— 实时通知

### 12.1 业务场景

- 配置变更实时推送（多节点同步）
- 实时消息通知（WebSocket 后端推送）
- 状态变更广播（订单状态变化通知）
- 日志实时收集

### 12.2 注意事项

```redis
# Pub/Sub 的特点：
1. 消息不持久化 → 订阅者离线时消息丢失
2. 无消费者组 → 所有订阅者都收到相同消息
3. 无确认机制 → 不知道谁收到了
4. 生产可靠 → 发布者不关心是否有订阅者
```

**适用**：
- 实时通知（丢失可接受）
- 配置广播（可配合持久化存储做兜底）

**不适用**：
- 重要业务消息（订单、支付等）→ 用 MQ
- 需要确认消费的消息 → 用 Stream

---

## 13. 缓存三大问题 —— 穿透/雪崩/击穿

### 13.1 三大问题对比

| 问题 | 描述 | 触发条件 | 危害 |
|------|------|----------|------|
| 穿透 | 查询不存在的数据 | 数据库也没有 | 每次请求打 DB |
| 雪崩 | 大量缓存同时失效 | 过期时间相同 | DB 瞬间压力暴增 |
| 击穿 | 热点 key 过期 | 单个热 key 失效 | 大量请求打 DB |

### 13.2 缓存穿透解决方案

```
方案一：布隆过滤器（详见 §1）
  → 拦截大部分不存在的 key

方案二：缓存空值
  → 数据库查不到也缓存空值（如 NULL），TTL 短（如 60s）
  SET cache:key "" EX 60

方案三：接口层校验
  → ID 格式、参数合法性校验，过滤掉明显非法请求

方案四：限流
  → 同一 IP/用户的查询频率限制
```

### 13.3 缓存雪崩解决方案

```
方案一：过期时间随机
  → SET key value EX <base + random(0, 300)>
  → 避免大量 key 同时过期

方案二：多级缓存
  → 本地缓存（Caffeine）+ Redis
  → Redis 全部失效时本地缓存兜底

方案三：永不过期 + 主动更新
  → 缓存不设过期时间
  → 后台任务定期更新

方案四：Redis 高可用
  → 主从 + 哨兵 + 集群，避免 Redis 整体宕机
```

### 13.4 缓存击穿解决方案

```
方案一：互斥锁（推荐）
  → 缓存失效时只允许一个线程查 DB 重建缓存
  → 其他线程等待或返回旧值

  伪代码：
  value = redis.get(key)
  if value == null:
      if redis.setnx(lock_key, 1, 10s):  # 获取锁
          try:
              value = db.query(key)
              redis.set(key, value, 600s)
          finally:
              redis.del(lock_key)
      else:
          sleep(50ms)
          return get(key)  # 重试

方案二：永不过期
  → 热点 key 不设过期时间
  → 后台异步更新

方案三：逻辑过期
  → value 中包含过期时间字段
  → 读取时检查逻辑过期，过期则异步更新
```

---

## 14. Lua 脚本实战要点

### 14.1 为什么用 Lua

- **原子性**：脚本内多条命令作为一个整体执行，不会被其他命令打断
- **减少网络往返**：多条命令一次发送
- **复杂逻辑**：支持条件判断、循环

### 14.2 注意事项

1. **不要执行耗时操作**：Lua 脚本会阻塞整个 Redis 实例
2. **脚本要幂等**：Redis 主从同步时副本会执行脚本，必须幂等
3. **参数传递**：用 KEYS 和 ARGV 传递，避免硬编码
4. **脚本缓存**：用 EVALSHA 替代 EVAL，避免重复传输脚本

### 14.3 常用 Lua 脚本场景

- 分布式锁释放（校验 + 删除原子）
- 限流计数（统计 + 判断 + 记录原子）
- 库存扣减（检查 + 扣减原子）
- 延迟队列消费（取 + 删原子）

---

## 15. 集群模式与选型

### 15.1 Redis 部署模式对比

| 模式 | 特点 | 适用场景 |
|------|------|----------|
| 单机 | 简单，无高可用 | 开发测试 |
| 主从 | 读写分离，故障手动切换 | 读多写少 |
| 哨兵 Sentinel | 主从 + 自动故障转移 | 中小规模生产 |
| Cluster | 分片 + 高可用 | 大规模生产 |

### 15.2 Cluster 注意事项

1. **key 分布**：用 CRC16 算法分到 16384 个 slot
2. **跨 slot 操作**：mset、事务等跨 slot 操作受限
3. **hash tag**：用 `{tag}` 强制多个 key 在同一 slot
4. **扩缩容**：迁移 slot 时部分 key 不可用

### 15.3 选型建议

- 日数据量 < 10 GB：主从 + 哨兵
- 日数据量 10-100 GB：Cluster
- 日数据量 > 100 GB：Cluster + 多机房部署

---

## 16. 持久化策略

### 16.1 RDB vs AOF

| 特性 | RDB | AOF |
|------|-----|-----|
| 触发方式 | 定时快照 | 实时追加 |
| 文件大小 | 小（压缩） | 大（命令日志） |
| 恢复速度 | 快 | 慢 |
| 数据安全 | 可能丢失最后一次快照 | 最多丢 1 秒（everysec） |
| CPU 占用 | 高（fork 子进程） | 低 |

### 16.2 推荐配置

- **生产环境**：RDB + AOF 混用
- **缓存场景**：可以不持久化（数据可重建）
- **重要数据**：AOF everysec + 定期 RDB 备份

### 16.3 数据迁移

```
# 方式一：Redis 迁移工具（redis-shake）
# 方式二：主从同步
# 方式三：RDB 文件恢复
# 方式四：SCAN + DUMP + RESTORE（小规模）
```

---

## 17. Redis 7.0+ 新特性

Redis 7.0/8.0 引入多项重要特性，是面试加分点，也是生产优化方向。

### 17.1 Redis Functions（替代 Lua 脚本）

```
痛点：Lua 脚本分散、难管理、难复用

Redis 7.0 引入 Functions：
  - 用 Lua 写函数库，注册到 Redis 持久化
  - 函数有名字、可管理（LIST/DELETE）
  - 支持版本管理
  - 比 EVALSHA 更规范

示例：
  FUNCTION LOAD '#!lua name=mylib
    redis.register_function("myadd", function(keys, args)
      return redis.call("INCRBY", keys[1], args[1])
    end)'

  FCALL myadd 1 mykey 10

优势：
  - 函数集中管理
  - 重启不丢失（持久化）
  - 团队复用
```

### 17.2 多线程 IO（性能提升）

```
Redis 6.0 引入多线程 IO，7.0 进一步优化：

原理：
  - 命令执行仍单线程（保证原子性）
  - 网络读写（协议解析、响应发送）多线程
  - 适合网络 IO 密集场景

配置：
  io-threads 4
  io-threads-do-reads yes

效果：
  - 高并发短命令场景 QPS 提升 50-100%
  - 长命令无提升（瓶颈在单线程执行）

注意：
  - 不解决单线程命令执行瓶颈
  - 大 value / 慢命令仍需优化
```

### 17.3 ACL 权限控制

```
Redis 6.0+ ACL：精细化权限控制

场景：
  - 不同服务不同账号
  - 限制命令（如只读账号）
  - 限制 key 前缀

配置：
  ACL SETUSER readonly_user on >password ~order:* +get +hget -@write

  → 用户 readonly_user
  → 密码 password
  → 只能访问 order:* 开头的 key
  → 只允许 get/hget
  → 禁止所有写命令

优势：
  - 安全性提升（最小权限原则）
  - 多租户隔离
  - 审计友好
```

### 17.4 客户端缓存（Tracking）

```
Redis 6.0+ 客户端缓存：服务端主动通知缓存失效

痛点：传统客户端缓存无法感知服务端数据变更

方案：
  CLIENT TRACKING ON
  → 客户端 GET key 后，Redis 记录
  → key 被修改时，Redis 主动通知客户端失效

效果：
  - 减少网络往返
  - 缓存一致性更好（服务端推送失效）
  - 适合热点 key

模式：
  - 普通模式：客户端收到失效通知后主动失效
  - 广播模式：按 key 前缀订阅变更（适合多客户端）
```

### 17.5 Sharded Pub/Sub

```
Redis 7.0 Sharded Pub/Sub：

痛点：原 Pub/Sub 在 Cluster 下要广播到所有节点

Sharded Pub/Sub：
  - 消息只在 key 所属分片传播
  - 减少跨节点流量
  - 命令：SSUBSCRIBE / SPUBLISH

适用：Cluster 模式下的 Pub/Sub 场景
```

### 17.6 Multi-part AOF

```
Redis 7.0 AOF 改进：

原 AOF：
  - 单文件，重写时体积大、慢
  - 重写期间 fork 子进程开销大

Multi-part AOF：
  - AOF 拆分为 base + increment
  - 重写只需生成新 base，无需重写历史
  - 恢复更快、重写更高效
```

### 17.7 新特性选型建议

| 特性 | 推荐度 | 适用场景 |
|------|--------|----------|
| Functions | ⭐⭐⭐ | 复杂脚本、团队复用 |
| 多线程 IO | ⭐⭐⭐ | 高并发短命令 |
| ACL | ⭐⭐⭐ | 多服务、多租户 |
| 客户端缓存 | ⭐⭐ | 热点 key、一致性要求高 |
| Sharded Pub/Sub | ⭐⭐ | Cluster + Pub/Sub |
| Multi-part AOF | ⭐⭐ | 生产必备（自动受益） |

---

## 18. Redis 避坑与选型速查

> 本节把 Redis 相关的选型决策和踩坑总结在一起，方便快速查阅。选型不脱离场景：先看业务诉求，再定方案。

### 17.1 何时该用 Redis，何时不该用

```
该用 Redis：
  ✓ 缓存（读多写少、可容忍短暂不一致）
  ✓ 计数器/限流（高频读写）
  ✓ 排行榜（ZSet 天然适合）
  ✓ 分布式锁（AP 场景）
  ✓ 会话存储（TTL 自动过期）

慎用 / 不该用 Redis：
  ✗ 强一致数据存储（Redis 是内存，AP 优先）→ 用 MySQL
  ✗ 大 Value 存储（单 key > 10MB）→ 用 MongoDB/对象存储
  ✗ 复杂查询（多条件组合、JOIN）→ 用 MySQL/ES
  ✗ 资金账户主存储（不能丢）→ Redis 只做缓存，DB 才是主
  ✗ 消息队列（重要业务消息）→ 用 RocketMQ/Kafka
```

### 17.2 Redis 部署模式选型

| 数据量 | 推荐模式 | 说明 |
|--------|----------|------|
| < 10 GB | 主从 + 哨兵 | 简单够用 |
| 10-100 GB | Cluster | 分片 + 高可用 |
| > 100 GB | Cluster + 多机房 | 大规模 |

### 17.3 Redis 常见踩坑速查

| 坑 | 表现 | 解决 |
|----|------|------|
| 大 KEY | 单 value > 10KB，操作阻塞 | 拆分 / 压缩 / 迁移 |
| 热 KEY | 单 key QPS 过万打满单节点 | 本地缓存 / 分片副本 |
| 慢查询 | KEYS、FLUSHALL 阻塞 | 禁用 / 用 SCAN |
| 内存溢出 | OOM | 设 maxmemory + 淘汰策略 |
| 持久化丢数据 | 宕机丢数据 | AOF everysec + RDB |
| 缓存穿透 | 不存在 key 打 DB | 布隆过滤器 + 空值缓存 |
| 缓存雪崩 | 大量 key 同时失效 | TTL 随机 + 多级缓存 |
| 缓存击穿 | 热 key 失效打 DB | 互斥锁 + 永不过期 |
| 跨 slot 操作 | Cluster 下 mset 报错 | hash tag `{tag}` |
| 主从切换丢数据 | 异步复制延迟 | 半同步 / 业务容忍 |

### 17.4 数据结构选型速查

| 业务需求 | 数据结构 | 关键命令 |
|----------|----------|----------|
| 缓存对象 | String / Hash | SET / HSET |
| 计数器 | String | INCR / DECR |
| 去重 | Set | SADD / SISMEMBER |
| 排行榜 | ZSet | ZADD / ZREVRANGE |
| 消息队列 | List / Stream | LPUSH / XADD |
| 位置服务 | Geo | GEOADD / GEORADIUS |
| UV 统计 | HyperLogLog | PFADD / PFCOUNT |
| 签到/标记 | Bitmap | SETBIT / BITCOUNT |
| 限流 | ZSet / List | ZADD / Lua |

---

## 19. Redis 底层原理八股

> 本节梳理 Redis 面试高频的底层原理。理解原理才能在选型和排障时心里有底。

### 19.1 数据结构底层实现

| Redis 类型 | 底层结构 | 关键点 |
|-----------|----------|--------|
| String | SDS（简单动态字符串） | 预分配空间、O(1) 长度、二进制安全 |
| List | quicklist（3.2+） | 双向链表 + ziplist 节点，兼顾内存和性能 |
| Hash | ziplist / hashtable | 小用 ziplist 省内存，大转 hashtable |
| Set | intset / hashtable | 全整数用 intset，否则 hashtable |
| ZSet | ziplist / skiplist+dict | 小用 ziplist，大用跳表+字典双结构 |
| Stream | radix tree | 基数树，支持范围查询 |

**SDS（Simple Dynamic String）**：
```
struct sdshdr {
    int len;      // 已使用长度
    int free;     // 剩余空间
    char buf[];   // 字符数组
}

优势：
- O(1) 获取长度（strlen）
- 预分配：扩容时多分配，减少 realloc
- 惰性释放：缩短时不立即释放
- 二进制安全：'\0' 不作为结束符
```

**跳表（SkipList）**：
```
为什么 ZSet 用跳表不用红黑树？
- 范围查询友好（ZRANGE 高效）
- 实现简单
- 并发友好（局部锁）
- 内存换性能（多级索引）

跳表 + dict 双结构：
- dict：O(1) 查 member 的 score
- skiplist：O(logN) 范围查询和排名
```

### 19.2 持久化原理

**RDB（快照）**：
```
触发：BGSAVE / 定时 / SHUTDOWN
原理：fork 子进程 → 遍历数据写 RDB 文件
优点：体积小、恢复快
缺点：fork 开销、可能丢最后一次快照后的数据

fork 的 COW（Copy-On-Write）：
- fork 时父子共享内存
- 父写 → 复制对应页
- 大内存实例 fork 慢
```

**AOF（追加日志）**：
```
原理：记录每条写命令到 AOF 文件
刷盘策略：
- always：每条都刷（最安全，最慢）
- everysec：每秒刷（默认，平衡）
- no：由 OS 决定（最快，可能丢）

AOF 重写：
- 命令合并（如 INCR 100 次 → SET 100）
- fork 子进程重写
- 重写期间新命令同时写旧 AOF 和缓冲
- 重写完成替换
```

### 19.3 事件模型

```
Redis 单线程模型：
- 命令执行单线程（避免锁竞争）
- IO 多路复用（epoll）处理网络事件
- 6.0+ 多线程 IO（网络读写多线程，命令仍单线程）

为什么单线程还快？
- 内存操作（纳秒级）
- 无锁、无上下文切换
- epoll 高效 IO
- 数据结构高效

单线程的瓶颈：
- CPU 密集命令（如大 ZUNIONSTORE）
- 大 key 操作
- fork 阻塞
```

### 19.4 过期策略

```
过期 key 删除策略：
1. 惰性删除：访问时检查过期 → 删
   - 优点：CPU 友好
   - 缺点：冷数据永远不删，占内存

2. 定期删除：每 100ms 随机抽样检查
   - 抽样一部分过期 key 删除
   - 过期 key 比例高则继续删
   - 限制每次执行时长

Redis 组合使用：惰性 + 定期

内存淘汰策略（内存满时）：
- noeviction：不淘汰，写入报错
- allkeys-lru：所有 key LRU
- volatile-lru：设了过期的 LRU
- allkeys-lfu：所有 key LFU（4.0+）
- volatile-lfu：设了过期的 LFU
- allkeys-random：随机
- volatile-random：随机
- volatile-ttl：即将过期的优先

业务选型：
- 缓存：allkeys-lru / allkeys-lfu
- 混合存储：volatile-lru（只淘汰可过期的）
```

### 19.5 主从复制原理

```
全量同步：
1. 从库发送 PSYNC 命令
2. 主库 BGSAVE 生成 RDB
3. 主库发送 RDB 给从库
4. 从库加载 RDB
5. 期间主库新命令写复制缓冲区
6. RDB 加载完 → 发送缓冲区命令

增量同步（断线重连）：
1. 从库带 replid 和 offset
2. 主库判断 replid 匹配 + offset 在积压缓冲区
3. 发送 offset 之后的命令
4. 否则全量同步

复制积压缓冲区：
- 环形缓冲区（默认 1MB）
- 太小 → 断线重连容易全量
- 建议：估算断线时间 × 峰值写入量
```

### 19.6 哨兵原理

```
哨兵职责：
- 监控：检测主从存活
- 通知：异常通知运维
- 自动故障转移：主挂自动选新主
- 配置提供：客户端从哨兵获取主地址

故障转移流程：
1. 主观下线（SDOWN）：单哨兵认为主挂
2. 客观下线（ODOWN）：多数哨兵确认
3. 选举领头哨兵
4. 领头选新主（优先级 → offset → runid）
5. 通知从库跟随新主
6. 通知客户端
7. 旧主恢复后变从库
```

### 19.7 Cluster 原理

```
Cluster 分片：
- 16384 个 slot
- CRC16(key) % 16384 → slot
- 每节点负责一部分 slot

客户端路由：
- 计算 slot → 找对应节点
- 节点不在本节点 → MOVED 重定向
- 迁移中 → ASK 临时重定向

高可用：
- 每个分片主从
- 主挂 → 从自动升级（Gossip 协议检测）
- 集群半数以上节点通信正常才可用
```

### 19.8 内存管理

```
内存划分：
- 自身内存（Redis 进程）
- 数据内存（主要占用）
- 缓冲内存（客户端缓冲、复制缓冲）
- 内存碎片（频繁修改产生）

内存优化：
- 合理选择数据结构（小 Hash 用 ziplist）
- 控制 key 数量和大小
- 设置 maxmemory + 淘汰策略
- 定期内存碎片整理（activedefrag）

内存监控：
- used_memory：实际使用
- used_memory_rss：OS 分配
- mem_fragmentation_ratio：碎片率（>1.5 需整理）
```

### 19.9 高频面试题速记

```
Q: Redis 为什么快？
A: 内存 + 单线程无锁 + epoll + 高效数据结构

Q: RDB 和 AOF 怎么选？
A: 生产混合用，AOF(everysec) + 定期 RDB

Q: 缓存击穿/穿透/雪崩区别？
A: 击穿=热key失效；穿透=查不存在；雪崩=批量失效

Q: 跳表 vs 红黑树？
A: 跳表范围查询好、实现简单、ZSet 用跳表

Q: 主从延迟怎么解决？
A: 关键读主、半同步、业务容忍

Q: 大 key 怎么处理？
A: 拆分、压缩、监控、删除用 UNLINK
```

---

## 关联阅读

