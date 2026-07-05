# 中文业务场景与技术选型资源推荐

> 本清单聚焦**业务场景、技术选型、面试八股文**，总结"什么时候用什么、为什么用、怎么用"，而非底层源码或操作系统原理。资源以中文为主，持续更新，适合面试突击与日常技术选型参考。

---

## 一、综合八股文 / 面试知识库（网站类）

| 资源 | 地址 | 特点 |
|------|------|------|
| **JavaGuide** | [javaguide.cn](https://javaguide.cn) | 国内最权威的 Java 后端面试指南，覆盖 Java 基础、Spring、MySQL、Redis、分布式、系统设计等全链路八股文，有开源免费版和知识星球付费版（含场景题专项） |
| **小林coding** | [xiaolincoding.com](https://xiaolincoding.com) | 图解计算机基础（网络、OS、数据库、Redis），八股文讲得非常细且生动，有专门的系统设计面试题板块（秒杀、点赞等） |
| **CS-Notes** | [cyc2018.xyz](http://www.cyc2018.xyz) / [GitHub](https://github.com/CyC2018/CS-Notes) | 143k+ Star 的面试笔记仓库，技术面试必备基础知识，涵盖算法、OS、网络、系统设计、数据库等，结构清晰，适合快速查阅 |
| **advanced-java** | [doocs.github.io/advanced-java](https://doocs.github.io/advanced-java) | 互联网 Java 工程师进阶知识完全扫盲：高并发、分布式、高可用、微服务、海量数据处理，有大量场景对比（如 Kafka vs RabbitMQ、Redis 分布式锁 vs Zookeeper） |
| **pdai.tech** | [pdai.tech](https://pdai.tech) | 全栈技术知识库，非常深入，包含技术选型文章（如向量数据库对比、知识图谱可视化技术选型），适合想"吊打面试官"的深度复习 |
| **Java八股文** | [github.com/CoderLeixiaoshuai/java-eight-part](https://github.com/CoderLeixiaoshuai/java-eight-part) | 专注 Java 面试套路，包含大量场景题和进阶学习路线，有并发编程、Java 8、设计模式等专题 |
| **tobebetterjavaer.com** | [tobebetterjavaer.com](https://tobebetterjavaer.com) | 另一个质量不错的 Java 八股文站点，有面试场景题整理 |

---

## 二、业务场景题 / 系统设计专项

### 1. 小林coding 系统设计面试题

**地址**：[xiaolincoding.com/interview/systemdesign.html](https://www.xiaolincoding.com/interview/systemdesign.html)

包含大量真实业务场景：
- **秒杀系统**：Redis 原子扣减 + Lua 脚本 + MQ 异步落库 + 乐观锁兜底
- **点赞系统**：MySQL 持久化 + Redis 缓存 + 逐步优化抗高并发
- **10w 人抢 10 个库存**：分三层（入口限流 → Redis 原子扣减 → MQ 异步落库）

### 2. JavaGuide 场景题小册

**《后端面试高频系统设计&场景题》**：30+ 道高频系统设计和场景面试题，涵盖秒杀、IM、短链、订单系统等当下中大厂面试趋势。

### 3. 大厂面试实录文章（CSDN / 掘金系列）

搜索"互联网大厂Java面试实录"能找到大量场景对话，例如：
- **内容社区 UGC**：Redis List/Hash 缓存评论、Spring Security 鉴权
- **电商支付**：Kafka 订单异步、Redis 购物车、分布式事务
- **智慧物流**：Elasticsearch + Flink 实时检索、Spring Cloud 微服务

---

## 三、具体技术场景总结

### Redis 业务场景总结

| 场景 | 方案 |
|------|------|
| **布隆过滤器** | 缓存穿透防护：预加载有效 Key，查询先过布隆，不存在直接返回，存在再查 Redis/MySQL；也可用于推荐去重、URL 去重、邮件黑名单 |
| **分布式锁** | Redisson 实现、Lua 脚本保证原子性；对比 Zookeeper 分布式锁的适用场景 |
| **缓存策略** | 缓存预热、缓存雪崩/穿透/击穿的区分与应对、多级缓存（本地 Caffeine + Redis） |
| **计数/限流** | Redis + Lua 原子操作实现秒杀库存扣减、滑动窗口限流 |

### MySQL 业务场景总结

| 场景 | 方案 |
|------|------|
| **索引优化** | B+树索引、覆盖索引、最左前缀、索引下推；何时用 B+树 vs Hash 索引 |
| **分库分表** | 按用户 ID 哈希分片、二级索引优化、避免热点问题；配合 Redis 缓存热门元数据 |
| **读写分离** | 主从复制、延迟问题处理、强制走主库场景 |
| **事务与锁** | 乐观锁（版本号/CAS）vs 悲观锁（SELECT FOR UPDATE）、MVCC 实现、死锁判定 |
| **与缓存一致性** | Cache Aside、延迟双删、分布式锁保证一致性 |

### 消息队列选型总结

| 对比维度 | Kafka | RabbitMQ | 适用场景 |
|----------|-------|----------|----------|
| **吞吐量** | 极高（百万级） | 中等 | Kafka 适合日志/大数据流；RabbitMQ 适合复杂路由/任务调度 |
| **延迟** | 毫秒级 | 微秒级 | RabbitMQ 适合实时性要求高的金融交易 |
| **可靠性** | 高（多副本） | 高（ACK机制） | 电商订单用 Kafka 削峰；库存扣减用 RabbitMQ 确保可靠 |
| **复杂度** | 高（需调优） | 低（开箱即用） | 中小规模用 RabbitMQ；超大规模用 Kafka |

---

## 四、技术选型与对比类文章

| 资源 | 内容 |
|------|------|
| **小林coding 向量数据库选型** | Chroma vs Qdrant vs Milvus vs pgvector，从数据规模、部署方式、混合检索三个维度分析 |
| **advanced-java 分布式锁对比** | Redis 分布式锁 vs Zookeeper 分布式锁：实现复杂度、性能、可靠性、适用场景对比表 |
| **advanced-java 分布式 ID 生成** | Leaf（美团）vs UidGenerator（百度）：吞吐量、易用性、依赖组件、适用场景对比 |
| **advanced-java 分布式事务** | XA（2PC）vs TCC vs Saga vs 本地消息表：一致性级别、性能、复杂度、适用场景 |

---

## 五、推荐组合（按需求）

### 快速过一遍八股文 + 场景题
1. **JavaGuide**（javaguide.cn）打基础
2. **小林coding**（xiaolincoding.com）补图解和场景题
3. **advanced-java**（doocs.github.io）看分布式选型对比

### 专门准备业务场景面试
1. **小林coding 系统设计专题**
2. **JavaGuide《后端面试高频系统设计&场景题》**
3. 牛客网搜"场景题"看最新面经

### 技术选型总结（如 Redis 布隆过滤器、MySQL 分库分表）
1. **advanced-java** 的分布式系列
2. **pdai.tech** 的深度技术选型文章
3. CSDN 搜具体场景（如"Redis 布隆过滤器 业务场景"）

---

## 说明

- 所有资源均聚焦在**业务场景、技术选型、面试八股**层面
- 不涉及操作系统线程调度、内存页管理、系统调用等底层机制
- 大部分资源免费且持续更新，非常适合面试突击和日常技术选型参考
- 建议结合多个站点交叉验证，获取最新面经与场景题

---

*Generated on 2026-07-05*
