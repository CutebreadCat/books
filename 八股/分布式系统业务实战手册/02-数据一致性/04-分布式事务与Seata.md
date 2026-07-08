# 分布式事务与 Seata

> 微服务跨服务一致性是工程中最复杂的分布式事务问题。本篇从理论方案到 Seata 框架实战完整梳理，含 Saga 高级隔离模式。

---

## 目录

- [1. 问题背景](#1-问题背景)
- [2. 方案对比](#2-方案对比)
- [3. 2PC](#3-2pc)
- [4. TCC](#4-tcc)
- [5. Saga](#5-saga)
- [6. 本地消息表](#6-本地消息表)
- [7. MQ 事务消息](#7-mq-事务消息)
- [8. Seata 框架实战](#8-seata-框架实战)
- [9. Saga 高级隔离模式](#9-saga-高级隔离模式)

---

## 1. 问题背景

微服务架构下，一个业务操作跨多个服务，每个服务有自己的数据库，如何保证数据一致性？

```
订单服务 → 创建订单
库存服务 → 扣减库存
支付服务 → 处理支付
积分服务 → 增加积分

如果其中一步失败，前面的操作怎么办？
```

---

## 2. 方案对比

| 方案 | 一致性 | 性能 | 实现复杂度 | 适用场景 |
|------|--------|------|------------|----------|
| 2PC | 强一致 | 低 | 中 | 数据库层面 XA 事务 |
| 3PC | 强一致 | 低 | 高 | 2PC 改进，实际少用 |
| TCC | 最终一致 | 中 | 高 | 资金类高要求 |
| Saga | 最终一致 | 高 | 中 | 长流程业务 |
| 本地消息表 | 最终一致 | 中 | 低 | 大多数业务 |
| MQ 事务消息 | 最终一致 | 高 | 中 | 异步场景 |

---

## 3. 2PC

```
阶段一：准备阶段（Prepare）
  协调者 → 所有参与者：准备提交
  参与者 → 执行事务，但不提交 → 锁定资源
  参与者 → 协调者：准备就绪/失败

阶段二：提交阶段（Commit）
  - 所有参与者就绪 → 协调者发送 Commit → 参与者提交
  - 任一参与者失败 → 协调者发送 Rollback → 参与者回滚

优点：强一致性
缺点：同步阻塞、协调者单点、参与者超时导致不一致
适用：同库跨表事务，少用
```

---

## 4. TCC

```
Try：预留资源
  - 冻结余额（不真正扣减）
  - 预扣库存（不真正扣减）
  - 检查资源是否充足

Confirm：确认操作
  - 真正扣减冻结余额
  - 真正扣减预扣库存

Cancel：取消预留
  - 解冻余额
  - 释放预扣库存
```

### 4.1 TCC 三个关键问题

1. **幂等性**：Try/Confirm/Cancel 都必须幂等
2. **空回滚**：Try 未执行但收到 Cancel → 要能处理
3. **悬挂控制**：Cancel 先于 Try 到达 → Try 不能再执行

详见 [08-异步长链路一致性](./08-异步长链路一致性.md) 中的进阶讨论。

---

## 5. Saga

```
长事务拆分为多个本地事务：
  T1 → T2 → T3 → T4

每个本地事务有对应的补偿操作：
  C1 ← C2 ← C3 ← C4

正向执行失败时，反向补偿：
  T1(成功) → T2(成功) → T3(失败)
  → 执行 C2 → C1

两种实现：
  1. 事件驱动（Choreography）
  2. 命令驱动（Orchestration）
```

---

## 6. 本地消息表（最实用方案）

```
服务 A（订单服务）：
  1. 业务操作 + 写消息表（同一本地事务）
     BEGIN;
     INSERT orders ... ;
     INSERT message_table (msg_id, content, status='NEW') ... ;
     COMMIT;

  2. 后台线程定时扫描消息表
     → status='NEW' 的消息 → 发送到 MQ
     → 发送成功 → 更新 status='SENT'

服务 B（库存服务）：
  1. 消费 MQ 消息 → 本地事务处理
  2. 处理成功 → ACK
  3. 处理失败 → MQ 重投（需幂等）

优点：不依赖 MQ 事务消息、实现简单、可靠
缺点：消息表需定期清理、有延迟、需消费者幂等
```

---

## 7. MQ 事务消息（RocketMQ）

```
1. 发送方发送"半消息"到 MQ（消费者不可见）
2. MQ 返回发送成功
3. 发送方执行本地事务
4. 发送方根据结果：Commit（消费者可见）/ Rollback（删除）
5. 若发送方未返回 → MQ 回查事务状态

优点：保证本地事务与消息发送原子性
缺点：需实现事务回查接口、仅 RocketMQ 支持
```

---

## 8. Seata 框架实战

### 8.1 Seata 四种模式

| 模式 | 对应理论 | 一致性 | 侵入性 | 适用 |
|------|----------|--------|--------|------|
| AT | 改进 2PC | 最终一致 | 极低（注解） | 通用首选 |
| TCC | TCC | 最终一致 | 高 | 资金、强一致 |
| Saga | Saga | 最终一致 | 中 | 长流程 |
| XA | 2PC/XA | 强一致 | 低 | 多 DB 强一致 |

### 8.2 AT 模式（最常用，几乎零侵入）

```java
@GlobalTransactional  // 只需这一个注解
public void createOrder(OrderDTO dto) {
    orderMapper.insert(dto);
    storageFeignClient.deduct(dto.getProductId(), dto.getCount());
    accountFeignClient.debit(dto.getUserId(), dto.getMoney());
}
```

原理：拦截 SQL，生成 before/after image 存 undo_log，回滚时用 before image 还原。

### 8.3 TCC 模式（资金类首选）

```java
@LocalTCC
public interface AccountTccAction {
    @TwoPhaseBusinessAction(name="debit", commitMethod="confirm", rollbackMethod="cancel")
    boolean prepare(BusinessActionContext ctx, ...);
    boolean confirm(BusinessActionContext ctx);
    boolean cancel(BusinessActionContext ctx);
}
```

### 8.4 Seata 架构角色

```
TC（Transaction Coordinator）：事务协调者，独立部署
TM（Transaction Manager）：事务管理器，业务服务集成
RM（Resource Manager）：资源管理器，业务服务集成
```

### 8.5 何时不用 Seata

- 性能要求极高（有 undo_log/全局锁开销）
- 跨语言系统（Seata 主要支持 Java）
- 简单异步场景（MQ 事务消息更轻）
- 长流程且可容忍不一致（本地消息表 + 对账）

---

## 9. Saga 高级隔离模式

> Saga 无数据库隔离 → 中间状态可见。本节讲如何用**语义锁**补偿隔离性。

### 9.1 Saga 的隔离性问题

```
Saga T1 → T2 → T3（失败）→ C2 → C1

中间状态问题：
  T1 完成、T2 进行中时，其他事务能看到 T1 的结果
  但整个 Saga 可能最终回滚

  → 脏读：读到可能回滚的数据
  → 丢失更新：并发 Saga 互相覆盖
```

### 9.2 语义锁（Semantic Lock）

```
方案：处理中的数据加"处理中"标志

  T1（创建订单）：
    INSERT orders(..., status='PENDING')
    → status='PENDING' 即语义锁

  其他事务看到 PENDING：
    → 拒绝操作（如不能基于未确定订单发货）
    → 或等待（如查询等待完成）

  Confirm/Cancel 后清除：
    Confirm → status='CONFIRMED'
    Cancel → status='CANCELLED'
```

### 9.3 其他 Saga 隔离模式

| 模式 | 说明 |
|------|------|
| 语义锁 | 处理中加标志位 |
| 交换式更新 | 用新记录而非更新，避免覆盖 |
| 悲观视图 | 读时重读确认 |
| 重读 | 关键操作前重读数据 |
| 版本值 | 版本号控制 |

---

## 关联阅读

- [01-一致性理论基础](./01-一致性理论基础.md) - 一致性模型
- [06-对账补偿与最终一致](./06-对账补偿与最终一致.md) - 补偿机制
- [08-异步长链路一致性](./08-异步长链路一致性.md) - 长链路事务
- [10-场景实战与选型面试](./10-场景实战与选型面试.md) - 选型决策

