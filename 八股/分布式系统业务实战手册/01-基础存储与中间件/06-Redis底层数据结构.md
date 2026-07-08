# Redis 底层数据结构详解

> Redis 的速度不仅来自单线程，更来自底层数据结构的高度优化。面试中从"Redis 为什么快"追问到"SDS 和 C 字符串有什么区别""ZSet 底层是跳表还是压缩列表"，本篇把每个数据结构讲透。

---

## 目录

- [1. 为什么需要了解底层数据结构](#1-为什么需要了解底层数据结构)
- [2. SDS（简单动态字符串）](#2-sds简单动态字符串)
- [3. dict（字典 / Hash 表）](#3-dict字典--hash-表)
- [4. ziplist（压缩列表）](#4-ziplist压缩列表)
- [5. quicklist（快速列表）](#5-quicklist快速列表)
- [6. skiplist（跳表）](#6-skiplist跳表)
- [7. intset（整数集合）](#7-intset整数集合)
- [8. 对象编码系统](#8-对象编码系统)
- [9. 内存淘汰算法](#9-内存淘汰算法)
- [10. 面试追问应对](#10-面试追问应对)

---

## 1. 为什么需要了解底层数据结构

```
面试中的追问链：

面试官：Redis 为什么快？
候选人：单线程、内存操作、IO 多路复用。
面试官：单线程为什么快？
候选人：没有线程切换开销。
面试官：那如果操作 100 万个元素的 Hash 呢？
候选人：...可能慢？
面试官：为什么？底层是什么结构？
候选人：...

了解底层数据结构的意义：
  1. 预估性能：知道某个操作的时间复杂度（如 HGETALL 在大 Hash 时有多慢）
  2. 避坑：知道什么场景会触发编码转换（如 ziplist 转 hashtable）
  3. 调优：知道怎么配置编码阈值，减少内存占用
  4. 面试：这是从"会用"到"懂原理"的分水岭
```

---

## 2. SDS（简单动态字符串）

```
Redis 的字符串不是 C 语言的原生字符串（char*），而是自己实现的 SDS。

C 字符串的问题：
  1. 获取长度 O(N)：需要遍历到 '\0'
  2. 缓冲区溢出：strcat 不检查目标空间
  3. 二进制不安全：内容中不能有 '\0'，不能存图片/序列化数据
  4. 修改效率低：每次修改都要重新分配内存

SDS 结构：
  struct sdshdr {
      uint32_t len;      // 已使用长度（O(1) 获取）
      uint32_t free;     // 剩余空间（预分配，减少内存分配次数）
      char buf[];        // 实际数据（柔性数组）
  };

SDS 的优势：
  1. O(1) 获取长度：直接读 len 字段
  2. 预分配空间：修改时如果 free 足够，不需要重新分配
     策略：如果 len < 1MB，预分配同等空间；如果 len >= 1MB，预分配 1MB
  3. 惰性释放：缩短字符串时，不立即释放内存，只增加 free
  4. 二进制安全：数据中间可以有 '\0'，用 len 判断结束

面试要点：
  "SDS 和 C 字符串的区别" → 答四个优势
  "Redis 能存二进制数据吗" → 能，因为 SDS 是二进制安全的
```

---

## 3. dict（字典 / Hash 表）

```
dict 是 Redis 的 Hash 表实现，用于 Hash 对象和全局键空间。

结构：
  typedef struct dict {
      dictType *type;     // 类型特定函数（如哈希函数）
      void *privdata;     // 私有数据
      dictht ht[2];       // 两个哈希表（渐进式 rehash 用）
      long rehashidx;     // rehash 进度，-1 表示未在进行
      int iterators;      // 正在运行的迭代器数量
  } dict;

  typedef struct dictht {
      dictEntry **table;  // 哈希表数组
      unsigned long size; // 数组大小（2 的幂）
      unsigned long sizemask; // 掩码 = size - 1
      unsigned long used; // 已有节点数
  } dictht;

  typedef struct dictEntry {
      void *key;          // 键（SDS）
      union {
          void *val;
          uint64_t u64;
          int64_t s64;
          double d;
      } v;                // 值
      struct dictEntry *next; // 拉链法解决冲突
  } dictEntry;

渐进式 Rehash：
  问题：当负载因子 > 1 时，需要扩容（size * 2）
  如果一次性把所有数据迁移到新表，单线程会阻塞很长时间

  Redis 的解法：渐进式 Rehash
    1. 创建新表 ht[1]，大小为 ht[0].size * 2
    2. 保持 ht[0] 和 ht[1] 同时可用
    3. 每次增删查改时，从 ht[0] 迁移 N 个桶到 ht[1]
    4. 查询时，先查 ht[0]，再查 ht[1]
    5. 全部迁移完成后，ht[0] = ht[1]，释放旧表

  特点：
    - 没有一次性 rehash 的阻塞问题
    - 但 rehash 期间内存占用是平时的 2 倍
    - rehash 期间的查询需要查两个表

面试要点：
  "Redis 是单线程，Hash 扩容时为什么不阻塞？"
  → 渐进式 Rehash，分批迁移，每次只迁移少量桶
```

---

## 4. ziplist（压缩列表）

```
ziplist：一种紧凑的线性结构，用于存储少量且长度较小的元素。

用途：
  - List 元素少且短时：List 的编码为 ziplist
  - Hash 元素少且短时：Hash 的编码为 ziplist
  - ZSet 元素少且短时：ZSet 的编码为 ziplist

结构：
  <zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>

  zlbytes：4 字节，整个 ziplist 占用的字节数
  zltail：4 字节，尾节点距离起始地址的偏移量（O(1) 定位尾部）
  zllen：2 字节，节点数量（超过 65535 时需要遍历统计）
  entry：节点（变长）
  zlend：1 字节，结束标记 = 0xFF

entry 结构：
  <prevlen> <encoding> <content>

  prevlen：前驱节点的长度（用于从后往前遍历）
  encoding：编码方式，标识 content 的类型和长度
    |00xxxxxx|：6 位整数，1 字节
    |01xxxxxx|：14 位整数，2 字节
    |10000000|：32 位整数，5 字节
    |11000000|：int16_t，3 字节
    |11100000|：int24_t，4 字节
    |11110000|：int64_t，9 字节
    |0xxxxxxx|：长度 <= 63 的字符串，1 字节
    |10xxxxxx| xxxxxxxx：长度 <= 16383 的字符串，2 字节
    |11111110|：长度 > 16383 的字符串，5 字节

ziplist 的优点：
  - 紧凑：无额外指针，内存利用率极高
  - 缓存友好：数据连续存储，CPU Cache 命中率高

ziplist 的缺点：
  - 插入/删除时可能需要连锁更新（prevlen 变长，影响后续节点）
  - 查找是 O(N)，不适合大量元素
  - 每次修改都要重新分配内存（如果空间不够）

编码转换阈值（Redis 7.0 前）：
  list-max-ziplist-entries = 512     // 元素超过 512 个，转 linkedlist
  list-max-ziplist-value = 64       // 单个元素超过 64 字节，转 linkedlist
  hash-max-ziplist-entries = 512    // Hash 元素超过 512 个，转 hashtable
  hash-max-ziplist-value = 64       // Hash 值超过 64 字节，转 hashtable
  zset-max-ziplist-entries = 128    // ZSet 元素超过 128 个，转 skiplist
  zset-max-ziplist-value = 64       // ZSet 值超过 64 字节，转 skiplist

注意：Redis 7.0+ ziplist 被 listpack 取代，但面试中 ziplist 仍常被问
```

---

## 5. quicklist（快速列表）

```
quicklist：Redis 3.2+ 引入的 List 编码，是 ziplist + linkedlist 的混合体。

背景：
  - ziplist 内存紧凑但插入慢
  - linkedlist（双向链表）插入快但内存开销大（每个节点有前后指针）
  - quicklist 取两者之长

结构：
  typedef struct quicklist {
      quicklistNode *head;        // 头节点
      quicklistNode *tail;        // 尾节点
      unsigned long count;        // 元素总数
      unsigned long len;          // 节点总数
      int fill : QL_FILL_BITS;    // 每个节点的 ziplist 大小限制
      unsigned int compress : QL_COMP_BITS; // 压缩深度（两端不压缩，中间压缩）
  } quicklist;

  typedef struct quicklistNode {
      struct quicklistNode *prev;
      struct quicklistNode *next;
      unsigned char *zl;           // ziplist 或 listpack 数据
      unsigned int sz;             // ziplist 大小
      unsigned int count : 16;     // ziplist 中的元素数
      int encoding : 2;            // RAW 或 LZF 压缩
      int container : 2;           // PLAIN 或 PACKED
      int recompress : 1;          // 是否被解压过
      int attempted_compress : 1;  // 测试用
  } quicklistNode;

设计：
  - 宏观上是双向链表（quicklistNode 之间有 prev/next 指针）
  - 每个节点内部是一个 ziplist（或 listpack），存储多个元素
  - 插入时：如果目标节点的 ziplist 还有空间，直接插入 ziplist；否则新建节点
  - 查找时：先定位到节点（O(N) 节点数），再在 ziplist 内查找（O(M) 元素数）

优点：
  - 内存比纯 linkedlist 节省（每个节点存多个元素，减少指针开销）
  - 插入删除比纯 ziplist 快（只需修改局部）
  - 支持压缩（中间节点压缩，减少内存）

配置：
  list-max-listpack-size = -2  // 每个 listpack 节点大小：-2 表示 8KB
  list-compress-depth = 0     // 压缩深度：0 不压缩，1 两端各 1 个不压缩
```

---

## 6. skiplist（跳表）

```
skiplist：Redis 的 ZSet 底层实现之一，用于有序集合。

结构：
  typedef struct zskiplist {
      struct zskiplistNode *header, *tail;  // 头尾节点
      unsigned long length;                // 节点总数
      int level;                           // 最大层数
  } zskiplist;

  typedef struct zskiplistNode {
      sds ele;                             // 成员（元素）
      double score;                        // 分数（排序依据）
      struct zskiplistNode *backward;      // 后退指针（只有 1 层）
      struct zskiplistLevel {
          struct zskiplistNode *forward;   // 前进指针
          unsigned int span;               // 跨度（到下一个节点的距离）
      } level[];                            // 变长数组，层数随机
  } zskiplistNode;

跳表原理：
  - 多层链表，每层都是有序链表
  - 最底层（level 1）包含所有元素
  - 上层是下层的"快速通道"，元素更少
  - 查找时从顶层开始，快速跳过大量元素

层数随机生成：
  - 每个节点层数随机，概率为 1/2 的幂
  - level 1：概率 1
  - level 2：概率 1/2
  - level 3：概率 1/4
  - ...
  - 最大层数 32

时间复杂度：
  - 查找：O(log N)
  - 插入：O(log N)
  - 删除：O(log N)
  - 范围查询：O(log N + M)，M 是返回元素数

为什么用跳表而不是红黑树？
  1. 实现简单：跳表代码比红黑树简单很多，调试容易
  2. 范围查询快：ZREVRANGE 按排名取区间，跳表直接定位后顺序遍历
  3. 并发友好：跳表插入只需修改局部指针，红黑树需要旋转，影响更大
  4. 内存可控：层数随机，平均内存开销稳定

ZSet 的编码选择：
  - 元素少且短：ziplist（或 listpack）
  - 元素多或长：skiplist + dict（dict 存 ele->score 的映射，O(1) 查分数）
  - 注意：ZSet 实际上同时用 skiplist + dict，dict 用于 O(1) 查 score，skiplist 用于排序
```

---

## 7. intset（整数集合）

```
intset：Set 对象在元素全是整数且数量少时的编码。

结构：
  typedef struct intset {
      uint32_t encoding;   // 编码方式：INT16 / INT32 / INT64
      uint32_t length;     // 元素个数
      int8_t contents[];   // 实际数据（柔性数组，按 encoding 对齐）
  } intset;

设计：
  - 有序数组存储整数（二分查找，O(log N)）
  - 编码升级：如果插入 int32，当前是 int16，则全部升级为 int32
  - 没有降级（删除后不会降级）

编码转换：
  set-max-intset-entries = 512  // 超过 512 个元素，转 hashtable

优点：
  - 内存紧凑：比 hashtable 节省大量内存（无指针、无哈希桶）
  - 查找快：有序数组 + 二分查找

缺点：
  - 只支持整数
  - 插入可能触发编码升级（O(N) 全部拷贝）
```

---

## 8. 对象编码系统

```
Redis 对象系统：
  typedef struct redisObject {
      unsigned type : 4;      // 对象类型：STRING / LIST / HASH / SET / ZSET
      unsigned encoding : 4; // 编码方式：见下表
      unsigned lru : 24;      // 最近访问时间（LRU 或 LFU）
      int refcount;           // 引用计数
      void *ptr;              // 指向底层数据的指针
  } redisObject;

编码对照表：

  ┌────────────┬────────────────────┬────────────────────┐
  │ 对象类型    │ 编码方式            │ 底层数据结构        │
  ├────────────┼────────────────────┼────────────────────┤
  │ STRING     │ RAW                │ SDS                │
  │ STRING     │ INT                │ 整数（直接存在 ptr）│
  │ STRING     │ EMBSTR             │ 嵌入式 SDS（≤ 44B）│
  │ LIST       │ QUICKLIST          │ quicklist          │
  │ LIST       │ ZIPLIST（旧）       │ ziplist            │
  │ HASH       │ HASHTABLE          │ dict               │
  │ HASH       │ ZIPLIST（旧）       │ ziplist            │
  │ SET        │ HASHTABLE          │ dict（值都是 NULL）│
  │ SET        │ INTSET             │ intset             │
  │ ZSET       │ SKIPLIST           │ skiplist + dict    │
  │ ZSET       │ ZIPLIST（旧）       │ ziplist            │
  └────────────┴────────────────────┴────────────────────┘

编码转换：
  - 写入时根据数据特征选择最紧凑的编码
  - 数据增长后自动升级到更高效的编码（不可逆）
  - 例如：Hash 开始用 ziplist，元素超过 512 个后转为 dict

EMBSTR vs RAW：
  - EMBSTR：redisObject 和 SDS 连续分配在一块内存中（适合短字符串）
  - RAW：redisObject 和 SDS 分开分配（适合长字符串）
  - 阈值：44 字节（Redis 64 位系统）
```

---

## 9. 内存淘汰算法

```
Redis 内存淘汰策略：

  maxmemory-policy：
    noeviction：不淘汰，内存满时写操作报错（默认）
    allkeys-lru：所有 Key 按 LRU 淘汰
    allkeys-lfu：所有 Key 按 LFU 淘汰（Redis 4.0+）
    allkeys-random：随机淘汰
    volatile-lru：只淘汰设置了 TTL 的 Key，按 LRU
    volatile-lfu：只淘汰设置了 TTL 的 Key，按 LFU
    volatile-random：只淘汰设置了 TTL 的 Key，随机
    volatile-ttl：只淘汰设置了 TTL 的 Key，优先淘汰 TTL 短的

LRU 实现：
  - 不是精确 LRU（需要维护全局链表，O(N) 开销大）
  - 近似 LRU：
    1. 每个 redisObject 有 lru 字段（24 位，记录秒级时间戳）
    2. 随机采样 N 个 Key（默认 5 个）
    3. 从中淘汰 lru 最小的（最久未访问）
  - 优点：O(1) 开销，内存占用小
  - 缺点：不是全局 LRU，可能淘汰最近访问的 Key（但概率低）

LFU 实现（Redis 4.0+）：
  - lru 字段复用：高 16 位记录分钟级时间戳，低 8 位记录访问计数器
  - 计数器增长：每次访问 +1，但增长速度随值增大而减缓（非线性）
  - 计数器衰减：随时间推移，计数器值衰减（不常访问的 Key 计数下降）
  - 更精准地反映"访问频率"而非"最近访问"

面试要点：
  "LRU 和 LFU 的区别"
  → LRU：最近没访问的淘汰（适合热点数据）
  → LFU：访问次数少的淘汰（适合长期高频数据）
  "Redis 的 LRU 是精确的吗"
  → 不是，是近似 LRU，随机采样 5 个选最久未访问的
```

---

## 10. 面试追问应对

**Q1：Redis 的 String 底层是什么？**
> Redis 的 String 不是 C 字符串，而是自己实现的 SDS（简单动态字符串）。SDS 结构包含 len（已使用长度）、free（剩余空间）和 buf（数据缓冲区）。优势：O(1) 获取长度、预分配空间减少内存分配、惰性释放避免频繁回收、二进制安全（可以存储包含 \0 的数据）。如果字符串是整数，直接用 INT 编码，不存 SDS。

**Q2：Hash 的底层是什么？什么时候会转换编码？**
> Hash 的底层编码有两种：ziplist（元素少且短时）和 dict（元素多或长时）。默认当元素数超过 512 个或单个值超过 64 字节时，从 ziplist 转换为 dict。ziplist 是紧凑的线性结构，内存省但插入慢；dict 是哈希表，查找插入快但内存开销大。转换是单向的，一旦转为 dict，不会回退到 ziplist。

**Q3：ZSet 的底层是什么？为什么用跳表而不是红黑树？**
> ZSet 的底层是 skiplist + dict。skiplist 负责按 score 排序和范围查询，dict 负责按 member 查 score（O(1)）。用跳表而不是红黑树的原因：1. 实现简单，代码量少，调试容易；2. 范围查询快（ZREVRANGE 直接定位后顺序遍历）；3. 插入删除只需修改局部指针，不需要像红黑树那样旋转；4. 内存可控，层数随机生成，平均内存开销稳定。

**Q4：Redis 的 LRU 是怎么实现的？**
> Redis 用的是近似 LRU，不是精确 LRU。每个 redisObject 有一个 24 位的 lru 字段，记录秒级时间戳。淘汰时，随机采样 5 个 Key，从中淘汰 lru 值最小的（最久未访问的）。优点是 O(1) 开销，内存占用小；缺点是可能不是全局最久未访问的，但概率低。Redis 4.0+ 还引入了 LFU，按访问频率淘汰，更适合长期高频数据的场景。

**Q5：Ziplist 有什么缺点？**
> Ziplist 是紧凑的线性结构，内存利用率高，但有两个主要缺点：1. 插入和删除时可能触发连锁更新（prevlen 字段变长，导致后续节点都要更新），最坏 O(N^2)；2. 查找是 O(N)，不适合大量元素。所以 Redis 设置了编码转换阈值，当元素数量或长度超过阈值时，自动转为更高效的编码（如 dict、skiplist）。Redis 7.0+ 用 listpack 替代了 ziplist，解决了连锁更新问题。

---

## 关联阅读

- [00-Redis 业务场景实战](./00-Redis业务场景实战.md) — 命令层和应用场景
- [02-数据一致性/03-缓存一致性](../02-数据一致性/03-缓存一致性.md) — 缓存与数据库的一致性

---

> **一句话总结**：Redis 的速度来自底层数据结构的高度优化：SDS 替代 C 字符串、渐进式 Rehash 替代一次性扩容、ziplist 节省内存、跳表实现高效排序、近似 LRU 实现低成本淘汰。面试深挖一定会从命令层问到编码层。
