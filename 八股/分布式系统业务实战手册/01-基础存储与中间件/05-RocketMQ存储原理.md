# RocketMQ 存储原理深度解析

> RocketMQ 是国内电商场景最广泛使用的消息队列，其存储设计（CommitLog + ConsumeQueue + IndexFile）比 Kafka 更偏向业务可靠性。面试中从"为什么用 RocketMQ"追问到"CommitLog 怎么设计"，本篇把底层讲透。

---

## 目录

- [1. RocketMQ 与 Kafka 存储的核心差异](#1-rocketmq-与-kafka-存储的核心差异)
- [2. CommitLog：消息主体存储](#2-commitlog消息主体存储)
- [3. ConsumeQueue：消费队列索引](#3-consumequeue消费队列索引)
- [4. IndexFile：消息索引](#4-indexfile消息索引)
- [5. 刷盘策略与可靠性](#5-刷盘策略与可靠性)
- [6. 主从同步机制](#6-主从同步机制)
- [7. 事务消息原理](#7-事务消息原理)
- [8. 踩坑与反模式](#8-踩坑与反模式)
- [9. 面试追问应对](#9-面试追问应对)

---

## 1. RocketMQ 与 Kafka 存储的核心差异

```
Kafka 存储模型：
  - 每个 Partition 是一个独立的日志文件
  - Partition 数量多时，文件数量多，随机 I/O 增加
  - 消费者拉取时，直接从 Partition 的 .log 文件读取

RocketMQ 存储模型：
  - 所有 Topic 的所有消息写入同一个 CommitLog（顺序写）
  - 每个 Topic 的每个 Queue 有独立的 ConsumeQueue（索引文件）
  - 消费者通过 ConsumeQueue 找到消息在 CommitLog 中的位置，再去 CommitLog 读取

核心差异对比：

| 维度 | Kafka | RocketMQ |
|------|-------|----------|
| 存储结构 | 每个 Partition 独立文件 | 统一 CommitLog + 索引分离 |
| 写入方式 | 多文件顺序写 | 单文件顺序写（极致顺序 I/O） |
| 读取方式 | 直接从 Partition 读取 | 先读索引再读 CommitLog（两次 I/O） |
| 消息查找 | 按 Offset 二分查找 | 按索引直接定位 |
| 可靠性 | 依赖 ACK 配置 | 原生同步刷盘 + 事务消息 |
| 适用场景 | 日志/大数据 | 业务消息（订单、支付） |
```

---

## 2. CommitLog：消息主体存储

```
CommitLog：所有消息的唯一存储文件，所有 Topic 共用

文件结构：
  /store/commitlog/
    ├── 00000000000000000000  ← 第一个文件，起始物理偏移 0
    ├── 00000000000001000000  ← 第二个文件，起始物理偏移 1048576 (1MB)
    ├── 00000000000002000000  ← 第三个文件，起始物理偏移 2097152 (2MB)
    └── ...

  文件命名：20 位数字，表示该文件起始的物理偏移量（绝对偏移）
  文件大小：默认 1GB（可配置 mappedFileSizeCommitLog）
  写满后：创建新文件，继续追加

消息格式：
  ┌────────────────────────────────────────────────────┐
  │ Msg Total Size (4 bytes)                            │
  │ Magic Code (4 bytes) = 0xAABBCCDD                   │
  │ Body CRC (4 bytes)                                   │
  │ Queue ID (4 bytes)                                   │
  │ Flag (4 bytes)                                       │
  │ Queue Offset (8 bytes)                               │
  │ Physical Offset (8 bytes)                            │
  │ Sys Flag (4 bytes)                                   │
  │ Born Timestamp (8 bytes)                             │
  │ Born Host (8 bytes)                                  │
  │ Store Timestamp (8 bytes)                            │
  │ Store Host (8 bytes)                                 │
  │ Reconsume Times (8 bytes)                            │
  │ Prepared Transaction Offset (8 bytes)                  │
  │ Body Length (4 bytes) + Body                         │
  │ Topic Length (1 byte) + Topic                         │
  │ Properties Length (2 bytes) + Properties               │
  └────────────────────────────────────────────────────┘

关键字段：
  Physical Offset：消息在 CommitLog 中的物理偏移量（全局唯一）
  Queue Offset：消息在逻辑队列中的偏移量（按 Topic+Queue 独立）
  Store Timestamp：消息存储时间（用于按时间查询）

写入方式：
  1. 获取当前 CommitLog 的 MappedFile（内存映射文件）
  2. 在 MappedFile 末尾追加消息（顺序写）
  3. 更新 WrotePosition（写入位置）
  4. 提交刷盘请求（同步或异步）
```

---

## 3. ConsumeQueue：消费队列索引

```
ConsumeQueue：每个 Topic 的每个 Queue 的独立索引文件

目录结构：
  /store/consumequeue/
    ├── topicA/
    │   ├── 0/                    ← Queue 0 的索引
    │   │   ├── 00000000000000000000
    │   │   └── ...
    │   ├── 1/                    ← Queue 1 的索引
    │   └── ...
    └── topicB/
        └── ...

文件格式：
  每个条目固定 20 字节：
  ┌────────────────────┬────────────────────┬────────────────────┐
  │ CommitLog Offset (8) │ Message Size (4)   │ Message Tag Hash (8)│
  └────────────────────┴────────────────────┴────────────────────┘

  CommitLog Offset：消息在 CommitLog 中的物理偏移量
  Message Size：消息总大小
  Tag Hash：消息 Tag 的哈希值（用于过滤）

ConsumeQueue 文件大小：
  默认 5.72MB（30 万条 × 20 字节）
  写满后创建新文件

消费者消费流程：
  1. 消费者知道 Topic + Queue + QueueOffset（如 topicA/Queue0/Offset=100）
  2. 读取 ConsumeQueue 的第 100 个条目（固定 20 字节，随机读）
  3. 得到 CommitLog Offset = 12345678，Message Size = 500
  4. 去 CommitLog 的 Offset 12345678 处读取 500 字节的消息内容
  5. 返回给消费者

关键：
  ConsumeQueue 是索引，只存位置信息，不存消息内容
  消息内容只在 CommitLog 中存一份，节省空间
  ConsumeQueue 文件小，可以常驻内存，读取快
```

---

## 4. IndexFile：消息索引

```
IndexFile：按 Key 或时间戳查找消息的索引文件

目录结构：
  /store/index/
    └── 20240707120000000  ← 文件名 = 创建时间戳

用途：
  1. 按 Message Key 查找消息（如根据订单号查找消息）
  2. 按时间范围查找消息（如查找某段时间内的消息）

文件结构：
  Header (40 bytes) + Slot Table (500 万 × 4 bytes) + Index Linked List

  Slot Table：
    - 500 万个 Slot，每个 Slot 4 字节
    - 根据 Key 的 Hash 值定位到 Slot
    - Slot 值指向该 Hash 对应的第一个 Index 条目位置

  Index 条目：
    ┌────────────────────┬────────────────────┬────────────────────┬────────────────────┐
    │ Key Hash (4)       │ CommitLog Offset (8)│ Time Diff (4)       │ Next Index Offset (4)│
    └────────────────────┴────────────────────┴────────────────────┴────────────────────┘

    Time Diff：消息存储时间与 IndexFile 创建时间的差值（毫秒）
    Next Index Offset：同一 Hash 的下一个 Index 条目位置（链表）

查找流程（按 Key）：
  1. 计算 Key 的 Hash 值
  2. 在 Slot Table 中找到对应 Slot
  3. 从 Slot 指向的 Index 条目开始遍历链表
  4. 匹配 Key Hash 和 Time Diff，找到对应的 CommitLog Offset
  5. 去 CommitLog 读取消息

限制：
  - 一个 IndexFile 只保存创建时间前后一定范围内的消息（默认 24 小时）
  - 超过范围后，查询需要遍历多个 IndexFile
```

---

## 5. 刷盘策略与可靠性

```
RocketMQ 提供两种刷盘方式：

方式1：同步刷盘（SYNC_FLUSH）
  flushDiskType = SYNC_FLUSH

  流程：
    1. 消息写入 MappedFile（Page Cache）
    2. 立即调用 force() 刷盘
    3. 刷盘成功后返回成功给生产者

  优点：消息不丢（只要返回成功，消息一定在磁盘上）
  缺点：性能低（每次写都要等磁盘 I/O）
  适用：订单、支付等关键业务

方式2：异步刷盘（ASYNC_FLUSH）
  flushDiskType = ASYNC_FLUSH（默认）

  流程：
    1. 消息写入 MappedFile（Page Cache）
    2. 立即返回成功给生产者（不等待刷盘）
    3. 后台线程定期刷盘（每 500ms 或积累一定量）

  优点：性能高（内存操作速度）
  缺点：可能丢消息（刷盘前 Broker 宕机，Page Cache 丢失）
  适用：日志、埋点等非关键业务

刷盘线程：
  GroupCommitService（同步刷盘）：
    - 收集刷盘请求，批量 force()
    - 减少单次 force() 开销

  FlushRealTimeService（异步刷盘）：
    - 每 500ms 检查一次，如果有新数据就刷盘
    - 积累一定量（如 4 页）也刷盘

性能对比：
  同步刷盘：约 5k ~ 10k TPS
  异步刷盘：约 50k ~ 100k TPS（取决于内存和磁盘）
```

---

## 6. 主从同步机制

```
Broker 角色：
  ASYNC_MASTER：主从异步复制（默认）
  SYNC_MASTER：主从同步复制
  SLAVE：从节点

同步复制（SYNC_MASTER）：
  流程：
    1. 生产者发送消息到 Master
    2. Master 写入 CommitLog
    3. Master 同步消息到 Slave（通过网络发送）
    4. Slave 写入本地 CommitLog 后返回确认
    5. Master 收到 Slave 确认后，返回成功给生产者

  优点：Master 宕机后，Slave 数据完整，不丢消息
  缺点：性能低（等 Slave 确认）

异步复制（ASYNC_MASTER）：
  流程：
    1. 生产者发送消息到 Master
    2. Master 写入 CommitLog 后立即返回成功
    3. Master 后台异步同步到 Slave

  优点：性能高
  缺点：Master 宕机且未同步到 Slave 时，可能丢消息

复制方式：
  CommitLog 实时同步：Master 写入后，通过 HAConnection 推送给 Slave
  Slave 收到后：写入本地 CommitLog，更新 ConsumeQueue

消费者读取：
  默认：Master 提供读服务
  配置：readFromSlave = true，Slave 也可提供读服务（分担 Master 压力）
  注意：如果 Slave 同步延迟，可能读到旧数据

故障切换：
  Master 宕机后：
    1. NameServer 检测到 Master 心跳超时
    2. Slave 自动提升为 Master（如果配置了自动切换）
    3. 或人工将 Slave 提升为 Master
  注意：
    异步复制下，Master 宕机可能有数据未同步，切换后可能有消息丢失或重复
    同步复制下，数据一致，切换安全
```

---

## 7. 事务消息原理

```
RocketMQ 事务消息：解决"业务操作和消息发送"的原子性问题

问题：
  业务操作（如写订单表）和消息发送（通知库存服务）是两个动作
  → 如果订单写成功但消息发送失败，库存不扣，数据不一致
  → 如果消息发送成功但订单回滚，库存扣了但订单不存在，超卖

RocketMQ 方案：两阶段提交 + 回查机制

流程：
  阶段一：半消息（Half Message）
    1. 生产者发送"半消息"到 Broker
    2. Broker 存储半消息，但标记为"不可消费"（对消费者不可见）
    3. Broker 返回半消息发送成功

  阶段二：本地事务
    1. 生产者执行本地事务（如写订单表）
    2. 本地事务成功 → 发送 Commit 给 Broker
    3. 本地事务失败 → 发送 Rollback 给 Broker

  阶段三：消息确认/回滚
    1. Broker 收到 Commit → 半消息变为"可消费"（对消费者可见）
    2. Broker 收到 Rollback → 删除半消息

  阶段四：回查机制（兜底）
    1. 如果 Broker 长时间没收到 Commit 或 Rollback（如生产者挂了）
    2. Broker 主动回查生产者："你的本地事务执行成功了吗？"
    3. 生产者查询本地事务状态（如查订单表是否存在）
    4. 返回状态：COMMIT / ROLLBACK / UNKNOWN（稍后重试）

关键设计：
  1. 半消息对消费者不可见，即使本地事务回滚，消费者也不会消费到
  2. 回查机制保证最终一致性，即使生产者中途崩溃
  3. 回查频率：默认 1 分钟后第一次回查，之后间隔递增（1m, 2m, 4m...）
  4. 最多回查 15 次，如果一直是 UNKNOWN，最终删除半消息

与本地消息表的对比：
  ┌────────────┬────────────────────┬────────────────────┐
  │ 维度       │ 本地消息表         │ RocketMQ 事务消息  │
  ├────────────┼────────────────────┼────────────────────┤
  │ 侵入性     │ 高（需改业务代码） │ 低（框架封装）     │
  │ 依赖       │ 业务数据库 + MQ      │ 仅依赖 RocketMQ   │
  │ 回查       │ 需自己实现定时任务   │ Broker 自动回查  │
  │ 性能       │ 中（需查消息表）     │ 高（半消息直接）  │
  │ 适用       │ 通用 MQ 均可        │ 仅 RocketMQ      │
  └────────────┴────────────────────┴────────────────────┘
```

---

## 8. 踩坑与反模式

### 坑 1：异步刷盘 + 异步复制双重风险
```
问题：
  flushDiskType = ASYNC_FLUSH
  brokerRole = ASYNC_MASTER
  → 消息在 Page Cache 中，Master 还没同步到 Slave 就挂了
  → 消息丢失，且无法恢复
解决：关键业务至少用一个同步（SYNC_FLUSH 或 SYNC_MASTER）
```

### 坑 2：ConsumeQueue 和 CommitLog 不一致
```
问题：
  CommitLog 写入成功，但 ConsumeQueue 写入失败（如磁盘满）
  → 消息存在，但消费者消费不到
解决：
  RocketMQ 有 ReputService 线程，定时扫描 CommitLog，补写 ConsumeQueue
  但修复需要时间，期间消息不可消费
```

### 坑 3：IndexFile 满了后查询慢
```
问题：
  按 Key 查询消息时，如果消息时间跨度大，需要遍历多个 IndexFile
  → 查询性能差
解决：
  IndexFile 只适合近实时查询（如 24 小时内）
  历史查询建议走业务数据库或日志系统，不要用 IndexFile
```

### 坑 4：事务消息回查逻辑不完善
```
问题：
  生产者实现了回查接口，但查询逻辑有误（如查询条件不对）
  → 回查返回 UNKNOWN，Broker 反复回查，消息长期不可消费
解决：
  回查逻辑必须幂等、准确，查询条件必须能覆盖所有事务状态
  如果回查失败，要有告警和人工介入机制
```

### 坑 5：Queue 数量过多导致内存溢出
```
问题：
  一个 Topic 创建几百个 Queue，每个 Queue 都有独立的 ConsumeQueue 文件
  → 内存中映射的文件过多，OOM
解决：
  Queue 数量合理设计：根据消费者并发数，一般 8~16 个 Queue 足够
  不要为每个用户/每个订单创建一个 Queue
```

### 坑 6：主从切换后消息重复
```
问题：
  Master 宕机，Slave 提升为 Master
  异步复制下，Slave 可能缺少部分消息
  或消费者已经消费了 Master 上的消息，新 Master 又推送了一次
解决：
  消费者必须幂等（业务唯一键/幂等表）
  关键业务用 SYNC_MASTER
```

---

## 9. 面试追问应对

**Q1：RocketMQ 和 Kafka 的存储有什么区别？**
> RocketMQ 把所有消息写入统一的 CommitLog，所有 Topic 共用，实现极致的顺序写 I/O。每个 Topic 的每个 Queue 有独立的 ConsumeQueue 索引文件，消费者通过索引定位到 CommitLog 中的消息位置。Kafka 是每个 Partition 一个独立的日志文件，Partition 多时文件数量多。RocketMQ 更适合业务消息（支持同步刷盘、事务消息），Kafka 更适合大数据/日志场景。

**Q2：CommitLog 和 ConsumeQueue 怎么配合工作？**
> 生产者发送消息时，Broker 把消息追加写入 CommitLog（全局顺序写）。同时，后台线程 ReputService 从 CommitLog 中读取新消息，为每个 Topic 的每个 Queue 生成索引条目，写入 ConsumeQueue。索引条目只有 20 字节：CommitLog 偏移量 + 消息大小 + Tag 哈希。消费者消费时，先读 ConsumeQueue 得到消息位置，再去 CommitLog 读取完整消息内容。这样消息主体只存一份，索引文件小且可常驻内存。

**Q3：RocketMQ 的刷盘策略怎么选？**
> 关键业务（订单、支付）用 SYNC_FLUSH + SYNC_MASTER，保证消息不丢。非关键业务（日志、埋点）用 ASYNC_FLUSH + ASYNC_MASTER，追求性能。如果只能选一个同步，优先 SYNC_FLUSH（刷盘到本地磁盘），因为网络同步到 Slave 的延迟比磁盘刷盘更大。但最安全的是双同步，性能最低。

**Q4：RocketMQ 事务消息怎么保证最终一致性？**
> 两阶段提交 + 回查。阶段一：生产者发送半消息到 Broker，半消息对消费者不可见。阶段二：生产者执行本地事务，成功发送 Commit，失败发送 Rollback。Broker 收到 Commit 后消息变为可见，收到 Rollback 后删除。如果 Broker 长时间没收到确认，会主动回查生产者，生产者查询本地事务状态后返回。最多回查 15 次，确保最终要么提交要么回滚。

**Q5：RocketMQ 主从切换后会不会丢消息？**
> 看配置。如果是 SYNC_MASTER，Master 写入后必须同步到 Slave，Slave 数据完整，切换不丢消息。如果是 ASYNC_MASTER，Master 可能有没有同步到 Slave 的消息，切换后这些消息丢失。另外，异步刷盘下，如果 Master 宕机时 Page Cache 未刷盘，也会丢消息。所以关键业务必须双同步（SYNC_FLUSH + SYNC_MASTER）。

---

## 关联阅读

- [03-中间件组合架构实战](./03-中间件组合架构实战.md) — RocketMQ 选型与组合使用
- [02-数据一致性/04-分布式事务与Seata](../02-数据一致性/04-分布式事务与Seata.md) — 事务消息与分布式事务对比
- [02-数据一致性/14-消息可靠性保障](../02-数据一致性/14-消息可靠性保障.md) — 消息不丢、不重、不积压

---

> **一句话总结**：RocketMQ 的存储 = 统一 CommitLog 顺序写 + ConsumeQueue 索引分离 + IndexFile 辅助查询 + 同步刷盘/同步复制保可靠。面试深挖会从"CommitLog 设计"问到"ConsumeQueue 索引"问到"事务消息两阶段提交"问到"主从同步"。
