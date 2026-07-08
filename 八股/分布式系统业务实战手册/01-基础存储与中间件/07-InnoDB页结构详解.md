# MySQL InnoDB 页结构详解

> InnoDB 的可靠性不是凭空而来的，而是基于页结构、Change Buffer、Double Write Buffer、Redo Log 等一系列精密机制。面试中从"InnoDB 为什么可靠"追问到"Redo Log 的 LSN 怎么工作"，本篇把底层讲透。

---

## 目录

- [1. InnoDB 逻辑存储结构](#1-innodb-逻辑存储结构)
- [2. 页（Page）结构](#2-页page结构)
- [3. B+ 树在页中的组织](#3-b-树在页中的组织)
- [4. Change Buffer](#4-change-buffer)
- [5. Double Write Buffer](#5-double-write-buffer)
- [6. Redo Log 与 LSN](#6-redo-log-与-lsn)
- [7. Undo Log 与回滚段](#7-undo-log-与回滚段)
- [8. 踩坑与反模式](#8-踩坑与反模式)
- [9. 面试追问应对](#9-面试追问应对)

---

## 1. InnoDB 逻辑存储结构

```
InnoDB 的存储层次：

  表空间（Tablespace）
    ├── 段（Segment）
    │     ├── 数据段（叶子节点段）
    │     ├── 索引段（非叶子节点段）
    │     └── 回滚段（Undo 段）
    │           └── 区（Extent）
    │                 ├── 页（Page）1
    │                 ├── 页（Page）2
    │                 └── ...（64 个页 = 1MB）

关键概念：
  表空间：
    - 系统表空间（ibdata1）：共享表空间，存数据字典、Undo、Change Buffer 等
    - 独立表空间（*.ibd）：每张表一个文件（innodb_file_per_table = ON）
    - 通用表空间：多张表共享一个表空间

  区（Extent）：
    - 1 个区 = 64 个页 = 1MB（默认页大小 16KB）
    - 申请空间时按区分配，保证 B+ 树叶子节点的物理连续性

  页（Page）：
    - InnoDB 的最小 I/O 单位，默认 16KB
    - 所有数据（索引、行数据、Undo）都以页为单位存储
```

---

## 2. 页（Page）结构

```
InnoDB 页大小：默认 16KB（可配置为 4KB/8KB/32KB，但 16KB 最常用）

页结构：
  ┌────────────────────────────┐
  │ File Header (38 bytes)     │  ← 文件头
  ├────────────────────────────┤
  │ Page Header (56 bytes)     │  ← 页头
  ├────────────────────────────┤
  │ Infimum + Supremum Records │  ← 虚拟记录（最小/最大边界）
  ├────────────────────────────┤
  │ User Records               │  ← 用户数据（行记录）
  │  ...                       │
  ├────────────────────────────┤
  │ Free Space                 │  ← 空闲空间
  ├────────────────────────────┤
  │ Page Directory             │  ← 页目录（槽数组）
  ├────────────────────────────┤
  │ File Trailer (8 bytes)     │  ← 文件尾（校验和）
  └────────────────────────────┘

File Header（文件头）：
  ┌────────────────────┬────────────┐
  │ FIL_PAGE_SPACE     │ 4 bytes    │  ← 表空间 ID
  │ FIL_PAGE_OFFSET    │ 4 bytes    │  ← 页偏移量（页号）
  │ FIL_PAGE_PREV      │ 4 bytes    │  ← 上一页指针（双向链表）
  │ FIL_PAGE_NEXT      │ 4 bytes    │  ← 下一页指针（双向链表）
  │ FIL_PAGE_LSN       │ 8 bytes    │  ← 最后修改该页的 LSN
  │ FIL_PAGE_TYPE      │ 2 bytes    │  ← 页类型（数据页/索引页/Undo 页等）
  │ FIL_PAGE_ARCH_LOG_NO│ 4 bytes   │  ← 归档日志号
  │ FIL_PAGE_FILE_FLUSH_LSN│ 8 bytes│  ← 表空间刷盘 LSN
  │ FIL_PAGE_SPACE_ID  │ 4 bytes    │  ← 表空间 ID（重复）
  └────────────────────┴────────────┘

Page Header（页头）：
  ┌────────────────────┬────────────┐
  │ PAGE_N_DIR_SLOTS   │ 2 bytes    │  ← 页目录槽数
  │ PAGE_HEAP_TOP      │ 2 bytes    │  ← 堆顶位置（Free Space 起始）
  │ PAGE_N_HEAP        │ 2 bytes    │  ← 记录数（含 Infimum/Supremum）
  │ PAGE_FREE          │ 2 bytes    │  ← 删除记录链表头
  │ PAGE_GARBAGE       │ 2 bytes    │  ← 已删除记录占用的字节数
  │ PAGE_LAST_INSERT   │ 2 bytes    │  ← 最后插入位置
  │ PAGE_DIRECTION     │ 2 bytes    │  ← 插入方向（LEFT/RIGHT/NO_DIR）
  │ PAGE_N_DIRECTION   │ 2 bytes    │  ← 同一方向插入次数
  │ PAGE_N_RECS        │ 2 bytes    │  ← 用户记录数
  │ PAGE_MAX_TRX_ID    │ 8 bytes    │  ← 修改该页的最大事务 ID
  │ PAGE_LEVEL         │ 2 bytes    │  ← 页在 B+ 树中的层级（0 = 叶子）
  │ PAGE_INDEX_ID      │ 8 bytes    │  ← 索引 ID
  │ PAGE_BTR_SEG_LEAF  │ 10 bytes   │  ← 叶子节点段信息
  │ PAGE_BTR_SEG_TOP   │ 10 bytes   │  ← 非叶子节点段信息
  └────────────────────┴────────────┘

User Records（用户记录）：
  - 行记录按主键顺序存储
  - 每条记录有记录头（5 bytes），包含：
    - 删除标记（1 bit）
    - 最小记录标记（1 bit）
    - 记录拥有的列数（n_owned，4 bits）
    - 堆序号（13 bits）
    - 记录类型（3 bits：0 普通 / 1 B+ 树节点指针 / 2 Infimum / 3 Supremum）
    - 下一记录偏移量（16 bits）
  - 记录之间用链表连接（按主键排序）

Page Directory（页目录）：
  - 解决在页内查找记录的问题（页内记录是链表，O(N) 查找慢）
  - 将页内记录分组，每组最后一个记录的偏移量放入槽（Slot）
  - 槽是有序数组，支持二分查找
  - 每组记录数：4 ~ 8 条（InnoDB 规定）
  - 查找过程：
    1. 二分查找 Page Directory，定位到包含目标记录的槽
    2. 在槽对应的记录组中顺序遍历（最多 8 条）
    3. 时间复杂度：O(log N) + O(8)，近似 O(log N)

File Trailer（文件尾）：
  - 前 4 字节：校验和（Checksum）
  - 后 4 字节：LSN（与 File Header 中的 FIL_PAGE_LSN 最后 4 字节一致）
  - 作用：校验页是否完整（磁盘写入时可能只写了一半，校验和不匹配则页损坏）
```

---

## 3. B+ 树在页中的组织

```
B+ 树节点 = 页

非叶子节点页：
  - 存储索引键值 + 子页指针
  - PAGE_LEVEL > 0
  - 键值按顺序排列，指向下一层的页

叶子节点页（PAGE_LEVEL = 0）：
  - 存储实际行数据
  - 页之间用双向链表连接（FIL_PAGE_PREV / FIL_PAGE_NEXT）
  - 支持范围查询（顺序遍历页链表）

插入操作：
  1. 找到目标叶子节点页
  2. 如果页有空闲空间（Free Space）：
     - 在合适位置插入记录，更新链表指针
     - 更新 Page Directory（必要时增加槽）
  3. 如果页已满（无 Free Space）：
     - 页分裂：申请新页，将原页一半记录移到新页
     - 更新父节点的指针（如果父节点也满了，继续分裂）
     - 最坏情况：B+ 树层级增加

删除操作：
  1. 找到记录，标记删除位（记录头删除标记 = 1）
  2. 记录移入 Free 链表（PAGE_FREE），不立即物理删除
  3. 更新 PAGE_GARBAGE（记录已删除空间）
  4. 不立即压缩页（惰性删除，后续插入可重用空间）
  5. 如果页利用率低于阈值（如 50%），可能触发页合并

页合并：
  - 删除后页利用率太低，与相邻页合并
  - 释放一个页，减少空间碎片
  - 如果合并导致父节点记录太少，可能继续合并或调整
```

---

## 4. Change Buffer

```
问题：
  插入/更新/删除操作通常需要更新二级索引（非主键索引）
  如果二级索引页不在 Buffer Pool 中，需要从磁盘读取，I/O 开销大

Change Buffer（原名 Insert Buffer）的解法：
  - 如果二级索引页不在 Buffer Pool：
    1. 不立即读取磁盘页
    2. 将变更操作缓存在 Change Buffer（存在于共享表空间 ibdata1）
    3. 等该页被读取到 Buffer Pool 时，再合并（Merge）Change Buffer 中的变更
    4. 或者后台线程定期合并

Change Buffer 结构：
  - 也是 B+ 树结构，存储在共享表空间
  - 键：Space ID + 页号（标识是哪个索引的哪个页）
  - 值：该页上待应用的变更记录列表

适用条件：
  - 只对二级索引生效（主键索引是顺序写，不需要 Change Buffer）
  - 只适用于非唯一索引（唯一索引需要查磁盘确认唯一性，无法缓冲）

优点：
  - 减少随机 I/O（二级索引更新是随机写）
  - 批量合并时效率更高

缺点：
  - 占用共享表空间（ibdata1）
  - 如果 Change Buffer 过大，合并时可能阻塞
  - 宕机恢复时，需要应用 Change Buffer 中的记录（增加恢复时间）

配置：
  innodb_change_buffering = all  // 缓冲所有操作（INSERT/UPDATE/DELETE）
  innodb_change_buffer_max_size = 25  // 占 Buffer Pool 的 25%
```

---

## 5. Double Write Buffer

```
问题：
  InnoDB 页大小 16KB，操作系统页大小 4KB
  如果写入 16KB 的页时，操作系统只写了 4KB 就断电了
  → 页只写了一半（Partial Write），数据损坏
  → 即使 Redo Log 存在，也无法恢复（因为原始页已经坏了）

Double Write Buffer 的解法：
  1. 脏页刷盘时，先把页写入 Double Write Buffer（共享表空间中的 2MB 连续区域）
  2. 等 Double Write Buffer 写满或需要刷盘时，一次性写入数据文件
  3. 如果写入数据文件时发生 Partial Write：
     - 从 Double Write Buffer 中读取完整页，覆盖损坏的页
     - 然后应用 Redo Log 恢复

Double Write Buffer 结构：
  - 位于共享表空间（ibdata1）或独立文件（ibdblwr）
  - 大小：2MB（128 个 16KB 页）
  - 分为两部分：
    - 内存中的 Double Write Buffer（2MB）
    - 磁盘上的 Double Write 区域（2MB）

写入流程：
  1. 脏页写入内存 Double Write Buffer（顺序写，快）
  2. 内存 Double Write Buffer 写满后，刷盘到磁盘 Double Write 区域（顺序写，快）
  3. 然后再把脏页写入数据文件的对应位置（随机写，慢）
  4. 如果步骤 3 失败（Partial Write），用步骤 2 的副本修复

为什么叫"Double Write"？
  - 页被写了两次：一次到 Double Write Buffer，一次到数据文件
  - 虽然多了写入，但 Double Write Buffer 是顺序写，开销不大

适用场景：
  - 有独立表空间（file_per_table）时启用
  - 如果用了原子写（如 Fusion-io 硬件），可以关闭 Double Write
  - 默认开启，保障数据安全

注意：
  - Double Write 不是备份，而是防止 Partial Write 的修复机制
  - 如果数据文件写入成功，Double Write 中的副本就不再需要了
  - 下次写入时会被覆盖
```

---

## 6. Redo Log 与 LSN

```
Redo Log（重做日志）：
  - 物理日志：记录的是"对哪个页做了什么修改"
  - 作用：崩溃恢复时，重放已提交事务的修改，保证数据不丢
  - 位置：ib_logfile0, ib_logfile1（循环写入）

Redo Log 结构：
  - 每个日志记录：
    - 表空间 ID
    - 页号
    - 页内偏移量
    - 修改长度
    - 修改前的数据（用于校验）
    - 修改后的数据

LSN（Log Sequence Number）：
  - 单调递增的 64 位整数，标识日志位置
  - 每个 Redo Log 记录都有一个 LSN
  - 每写一条日志，LSN 增加

关键 LSN 值：
  - checkpoint_lsn：已经刷盘的页对应的最新 LSN
  - last_checkpoint_lsn：上次 checkpoint 时的 LSN
  - current_lsn：当前日志写入位置
  - flushed_to_disk_lsn：已刷盘到磁盘文件的 LSN

Checkpoint 机制：
  - 目的：标记哪些日志已经不需要（对应页已刷盘）
  - 过程：
    1. 找到 Buffer Pool 中已修改页的最小 LSN（最早修改的页）
    2. 将该 LSN 之前的所有日志标记为已过期（可以覆盖）
    3. 写入 Checkpoint 记录
  - 触发条件：
    - 日志文件快满时（循环写入需要回收空间）
    - 定时触发（每几秒）
    - 脏页比例过高时

崩溃恢复：
  1. 读取 Checkpoint 记录，找到 last_checkpoint_lsn
  2. 从该 LSN 开始扫描 Redo Log
  3. 对每个 Redo Log 记录：
     - 读取对应页，检查页的 FIL_PAGE_LSN
     - 如果页 LSN >= Redo LSN，说明该修改已写入页，跳过
     - 如果页 LSN < Redo LSN，说明修改未写入页，应用 Redo
  4. 所有 Redo 应用完成后，数据库恢复一致

Redo Log 与 Binlog 的区别：
  ┌────────────┬────────────────────┬────────────────────┐
  │ 维度       │ Redo Log           │ Binlog             │
  ├────────────┼────────────────────┼────────────────────┤
  │ 层级       │ 存储引擎层（InnoDB）│ 服务器层（MySQL）  │
  │ 内容       │ 物理修改（页级别）  │ 逻辑 SQL（语句/行）│
  │ 用途       │ 崩溃恢复            │ 主从复制、数据恢复 │
  │ 写入时机   │ 事务执行时持续写入  │ 事务提交时一次性写入│
  │ 文件形式   │ 循环文件（固定大小）│ 追加文件（可增长） │
  │ 格式       │ 二进制物理记录      │ 文本/二进制逻辑记录│
  └────────────┴────────────────────┴────────────────────┘

两阶段提交（保证 Redo 和 Binlog 一致）：
  1. 事务执行：Redo Log 写入（Prepare 状态）
  2. 事务提交前：Binlog 写入
  3. Binlog 写入成功：Redo Log 更新为 Commit 状态
  4. 如果中途崩溃：
     - Redo 是 Prepare，Binlog 无记录 → 回滚
     - Redo 是 Prepare，Binlog 有记录 → 提交（Redo 补充提交标记）
```

---

## 7. Undo Log 与回滚段

```
Undo Log（回滚日志）：
  - 逻辑日志：记录的是"如何撤销这个修改"
  - 作用：
    1. 事务回滚：撤销未提交事务的修改
    2. MVCC：为读事务提供历史版本数据

Undo Log 类型：
  - Insert Undo：插入操作的回滚记录（事务提交后即可删除）
  - Update Undo：更新/删除操作的回滚记录（需要保留到没有读事务需要它时）

Undo Log 结构：
  - 存储在回滚段（Rollback Segment）中
  - 回滚段在 Undo 表空间（undo_001, undo_002）或系统表空间中
  - 每个事务有一个 Undo 段（Undo Segment），包含多个 Undo 页

MVCC 与 Undo Log：
  - 每行记录有隐藏字段：
    - DB_TRX_ID（6 bytes）：最后修改该记录的事务 ID
    - DB_ROLL_PTR（7 bytes）：回滚指针，指向 Undo Log 记录
    - DB_ROW_ID（6 bytes）：行 ID（无主键时生成）
  - 读事务创建 Read View（一致性视图），包含：
    - 活跃事务 ID 列表
    - 最小活跃事务 ID
    - 最大已分配事务 ID
  - 读取记录时：
    - 如果 DB_TRX_ID 在 Read View 中（事务未提交），沿 DB_ROLL_PTR 找历史版本
    - 直到找到已提交版本

Purge 线程：
  - 后台线程，清理不再需要的 Undo Log
  - 当没有读事务需要某个历史版本时，删除对应的 Undo Log
  - 防止 Undo 表空间无限增长

注意：
  - 长事务（不提交的读事务）会阻止 Purge，导致 Undo 表空间膨胀
  - 这是大事务/长事务导致 MySQL 性能下降的主要原因之一
```

---

## 8. 踩坑与反模式

### 坑 1：忽略 Change Buffer 对性能的影响
```
问题：大量二级索引更新，Change Buffer 合并时阻塞
  → 写入性能突然下降
解决：监控 innodb_change_buffer_max_size，如果频繁合并，适当调大或关闭
```

### 坑 2：长事务导致 Undo 表空间膨胀
```
问题：
  读事务开始后不提交，持续数小时
  → Purge 无法清理该事务开始后的 Undo Log
  → Undo 表空间从 1GB 涨到 100GB
  → MySQL 性能急剧下降
解决：
  1. 避免长事务（读事务也要及时提交）
  2. 监控 trx_history_list_length（Undo 未清理长度）
  3. 如果已膨胀，触发 Purge 或重建表空间
```

### 坑 3：Redo Log 太小导致频繁 Checkpoint
```
问题：
  innodb_log_file_size = 48MB（默认）
  写入量大时，Redo Log 很快写满
  → 频繁触发 Checkpoint，刷脏页
  → 写入性能抖动
解决：
  innodb_log_file_size = 2GB（根据写入量调整）
  innodb_log_files_in_group = 2  // 或 3
  Redo Log 总大小 = 4GB ~ 6GB，减少 Checkpoint 频率
```

### 坑 4：关闭 Double Write 以为提升性能
```
问题：
  innodb_doublewrite = 0  // 关闭 Double Write
  → 如果发生 Partial Write，数据页损坏，无法恢复
  → 数据库无法启动，只能恢复备份
解决：
  除非有原子写硬件（如 Fusion-io），否则不要关闭 Double Write
  性能损失很小（顺序写），但安全性极高
```

### 坑 5：大表删除数据后空间不释放
```
问题：
  DELETE 删除 1000 万行，但 ibd 文件大小没变
  → 因为 InnoDB 是页组织，删除只是标记记录头删除位
  → 页内产生碎片，但页本身不释放
解决：
  1. OPTIMIZE TABLE 重建表（但会锁表）
  2. pt-online-schema-change 在线重建
  3. 或者使用独立表空间，定期归档+重建
```

### 坑 6：页分裂导致索引性能退化
```
问题：
  主键不是自增，随机插入（如 UUID）
  → 频繁页分裂和页合并
  → 页内碎片多，查询性能下降
解决：
  主键尽量用自增整数（AUTO_INCREMENT），保证顺序插入
  如果必须用 UUID，用 UUID_TO_BIN() 或调整为顺序 UUID
```

---

## 9. 面试追问应对

**Q1：InnoDB 的页结构是怎样的？**
> InnoDB 页默认 16KB，结构包括：File Header（38 字节，含页类型、前后指针、LSN）、Page Header（56 字节，含记录数、空闲空间、目录槽数）、Infimum/Supremum 虚拟记录（页边界）、User Records（按主键排序的行记录链表）、Free Space（空闲空间）、Page Directory（页目录槽数组，支持二分查找）、File Trailer（8 字节校验和）。查找页内记录时，先在 Page Directory 中二分定位到槽，再在槽对应的记录组中顺序遍历，近似 O(log N)。

**Q2：Double Write Buffer 是干什么的？**
> 防止 Partial Write。InnoDB 页是 16KB，但操作系统页是 4KB，如果写入 16KB 时只写了 4KB 就断电，页会损坏。Double Write Buffer 在脏页刷盘时，先把页顺序写入共享表空间的 Double Write 区域（2MB），然后再写入数据文件。如果数据文件写入时发生 Partial Write，可以从 Double Write 区域读取完整副本修复。虽然多写一次，但 Double Write 是顺序写，性能开销很小。

**Q3：Redo Log 和 Binlog 有什么区别？**
> Redo Log 是 InnoDB 存储引擎层的物理日志，记录"对哪个页做了什么修改"，用于崩溃恢复。Binlog 是 MySQL 服务器层的逻辑日志，记录"执行了什么 SQL"，用于主从复制和数据恢复。Redo Log 是循环写入，Binlog 是追加写入。两者通过两阶段提交保证一致性：事务先写 Redo Log（Prepare），再写 Binlog，最后 Redo Log 更新为 Commit。

**Q4：Change Buffer 是什么？什么时候生效？**
> Change Buffer 用于优化二级索引的更新。如果二级索引页不在 Buffer Pool 中，InnoDB 不立即读磁盘，而是把变更缓存在 Change Buffer（共享表空间的 B+ 树）中。等该页被加载到 Buffer Pool 时，再合并变更。只对非唯一二级索引生效，因为唯一索引需要查磁盘确认唯一性。优点是减少随机 I/O，缺点是可能增加合并时的阻塞和恢复时间。

**Q5：MVCC 怎么实现的？**
> MVCC 通过 Undo Log 和每行的隐藏字段实现。每行记录有 DB_TRX_ID（最后修改事务 ID）和 DB_ROLL_PTR（回滚指针）。读事务创建 Read View（包含活跃事务列表），读取时如果记录的事务 ID 在活跃列表中（未提交），就沿 DB_ROLL_PTR 找到 Undo Log 中的历史版本，直到找到已提交的版本。Purge 线程后台清理不再需要的 Undo Log。长事务会阻止 Purge，导致 Undo 表空间膨胀。

---

## 关联阅读

- [01-MySQL 业务场景实战](./01-MySQL业务场景实战.md) — SQL 层优化、索引设计
- [02-存储引擎深度对比](./02-存储引擎深度对比.md) — B+ Tree vs LSM Tree
- [02-数据一致性/04-分布式事务与Seata](../02-数据一致性/04-分布式事务与Seata.md) — 事务一致性

---

> **一句话总结**：InnoDB 的可靠性 = 16KB 页结构（有序存储 + 校验和） + Change Buffer（减少随机 I/O） + Double Write Buffer（防 Partial Write） + Redo Log（崩溃恢复） + Undo Log（回滚 + MVCC）。面试深挖会从"页结构"问到"Double Write"问到"Redo Log LSN"问到"MVCC 实现"。
