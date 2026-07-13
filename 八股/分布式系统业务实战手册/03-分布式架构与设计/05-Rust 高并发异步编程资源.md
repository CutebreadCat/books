# Rust 高并发异步编程资源与学习路径

> 针对 Go/Rust 后端开发者，整理 Rust 高并发异步编程的核心模式、学习资源与实战场景。覆盖 Channel/Actor、异步锁、无锁并发、分片策略、Copy-on-Write 等 6 种并发模型，附生产级库推荐与选型决策树。

---

## 目录

- [1. 学习路径总览](#1-学习路径总览)
- [2. 模式一：Channel / Actor 模型（最推荐）](#2-模式一channel--actor-模型最推荐)
- [3. 模式二：tokio::sync::Mutex（异步锁）](#3-模式二tokiosyncmutex异步锁)
- [4. 模式三：tokio::sync::RwLock（读写锁）](#4-模式三tokiosyncrwlock读写锁)
- [5. 模式四：分片（Sharding）](#5-模式四分片sharding)
- [6. 模式五：无锁并发数据结构](#6-模式五无锁并发数据结构)
- [7. 模式六：快照 / Copy-on-Write](#7-模式六快照--copy-on-write)
- [8. 选型决策树](#8-选型决策树)
- [9. 核心学习资源](#9-核心学习资源)
- [10. 生产踩坑与面试要点](#10-生产踩坑与面试要点)

---

## 1. 学习路径总览

```
Rust 异步并发学习路径（建议顺序）：

  阶段 1：基础（1-2 周）
    - 理解 Rust 所有权 + Send/Sync trait
    - 理解 async/await 语法糖与 Future 状态机
    - Tokio 运行时：多线程 vs 当前线程模式
    - 掌握 tokio::sync 全家桶：mpsc, oneshot, broadcast, watch

  阶段 2：共享状态（2-3 周）
    - 模式 A：Channel/Actor（消息传递，无共享状态）
    - 模式 B：Arc<Mutex<T>>（共享状态，需同步）
    - 区分 std::sync::Mutex vs tokio::sync::Mutex
    - 理解锁的粒度、死锁、饥饿

  阶段 3：高级并发（3-4 周）
    - 无锁数据结构：DashMap, papaya, flurry
    - 分片策略：手动 Sharding vs 库实现
    - Copy-on-Write：ArcSwap, im-rc
    - 结构化并发：JoinSet, CancellationToken, select!

  阶段 4：生产实战（持续）
    - 构建一个高并发服务（如 Redis/MQ 简化版）
    - 性能压测：criterion, tokio-console, perf
    - 故障注入：模拟死锁、内存泄漏、任务取消
```

---

## 2. 模式一：Channel / Actor 模型（最推荐）

### 2.1 核心思想

```
"Do not communicate by sharing memory; instead, share memory by communicating."
  — Go 语言设计哲学，Rust 同样适用

Actor 模型：
  每个 Actor 是独立的异步任务，持有私有状态
  Actor 之间通过消息（Channel）通信，不直接共享状态
  所有消息串行处理，天然避免数据竞争（Data Race）

为什么最推荐：
  1. 无锁：没有 Mutex/RwLock，没有死锁风险
  2. 清晰：状态隔离在 Actor 内部，逻辑边界明确
  3. 可测试：Actor 可独立测试，消息输入/输出明确
  4. 可扩展：可启动多个 Actor 实例，配合路由分发

适用场景：
  状态集中管理（如连接池、配置中心、会话管理）
  需要保证操作顺序（如库存扣减、订单状态机）
  中等并发量（Actor 是单线程串行处理，超高并发需多 Actor）
```

### 2.2 基础实现：tokio::sync::mpsc + oneshot

```rust
use tokio::sync::{mpsc, oneshot};

// 定义 Actor 能接收的命令
pub enum StorageCommand {
    Get { key: String, respond_to: oneshot::Sender<Option<String>> },
    Set { key: String, value: String },
    Delete { key: String },
}

// 启动 Actor，返回发送端句柄
pub async fn start_storage_actor() -> mpsc::Sender<StorageCommand> {
    let (tx, mut rx) = mpsc::channel::<StorageCommand>(64); // 有界通道，背压
    
    tokio::spawn(async move {
        let mut storage = std::collections::HashMap::new();
        
        while let Some(cmd) = rx.recv().await {
            match cmd {
                StorageCommand::Get { key, respond_to } => {
                    let _ = respond_to.send(storage.get(&key).cloned());
                }
                StorageCommand::Set { key, value } => {
                    storage.insert(key, value);
                }
                StorageCommand::Delete { key } => {
                    storage.remove(&key);
                }
            }
        }
    });
    
    tx
}

// 使用 Actor
pub async fn get_value(
    actor: &mpsc::Sender<StorageCommand>,
    key: &str
) -> Option<String> {
    let (tx, rx) = oneshot::channel();
    let cmd = StorageCommand::Get {
        key: key.to_string(),
        respond_to: tx,
    };
    
    // send().await 会阻塞直到通道有空间（背压）
    if actor.send(cmd).await.is_err() {
        return None; // Actor 已关闭
    }
    
    rx.await.ok().flatten()
}
```

### 2.3 关键设计点

```
有界通道（Bounded Channel）：
  mpsc::channel(64) 而非 unbounded_channel()
  原因：无界通道在消费者慢时会导致内存无限增长，最终 OOM
  有界通道提供背压（backpressure）：生产者 send().await 会等待

Oneshot 用于请求-响应：
  每个 Get 请求附带一个 oneshot::Sender
  Actor 处理完后通过 oneshot 返回结果
  oneshot 只能使用一次，天然适合单次请求响应

Actor 的生命周期：
  所有 Sender 被 drop 后，recv() 返回 None，Actor 退出循环
  优雅关闭：先 drop 所有 Sender，等待 Actor 自然结束
  强制关闭：用 CancellationToken 或 abort handle
```

### 2.4 多 Actor 扩展

```rust
// 多个 Actor 实例，按 key 路由
pub struct ActorPool {
    actors: Vec<mpsc::Sender<StorageCommand>>,
}

impl ActorPool {
    pub async fn new(count: usize) -> Self {
        let mut actors = Vec::with_capacity(count);
        for _ in 0..count {
            actors.push(start_storage_actor().await);
        }
        Self { actors }
    }
    
    fn route(&self, key: &str) -> &mpsc::Sender<StorageCommand> {
        use std::collections::hash_map::DefaultHasher;
        use std::hash::{Hash, Hasher};
        
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        let idx = (hasher.finish() as usize) % self.actors.len();
        &self.actors[idx]
    }
    
    pub async fn get(&self, key: &str) -> Option<String> {
        get_value(self.route(key), key).await
    }
}
```

### 2.5 Actor 框架（生产级）

```
tokio-actors（推荐）：
  GitHub: https://github.com/uwejan/tokio-actors
  特点：
    - 零仪式，Tokio-native（基于 tokio::task，无自定义调度器）
    - 强类型：Message 和 Response 编译时检查
    - 有界邮箱（默认 64），自然背压
    - OTP 风格监督（Supervisor）：OneForOne, OneForAll, RestForOne
    - 定时器漂移处理（MissPolicy: Skip/CatchUp/Delay）
  适用：需要监督树、定时器、生命周期管理的生产系统

actix（老牌，但较重）：
  GitHub: https://github.com/actix/actix
  特点：功能全面，但学习曲线陡，生态在萎缩
  适用：需要完整 Actor 框架的复杂系统

自定义 Actor（推荐从简单开始）：
  如上面的 mpsc + oneshot 实现，理解原理后再上框架
  大多数场景不需要完整框架，几十行代码即可
```

---

## 3. 模式二：tokio::sync::Mutex（异步锁）

### 3.1 核心问题：为什么不用 std::sync::Mutex？

```
std::sync::Mutex 在 async 中的问题：
  .lock() 阻塞当前 OS 线程
  在 Tokio 的多线程运行时中，一个线程被阻塞 = 该线程上的所有任务都卡住
  如果所有工作线程都被阻塞，整个运行时瘫痪

tokio::sync::Mutex 的解决：
  .lock().await 是异步的，不会阻塞线程
  等待锁时，当前任务被挂起（yield），线程去执行其他任务
  锁可用时，通过 Waker 唤醒任务

黄金法则：
  锁必须跨越 .await → 用 tokio::sync::Mutex
  锁不跨越 .await，且临界区极短 → 用 std::sync::Mutex（性能更好）
```

### 3.2 使用示例

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[tokio::main]
async fn main() {
    let data = Arc::new(Mutex::new(0u64));
    let mut handles = vec![];
    
    for _ in 0..10 {
        let d = data.clone();
        handles.push(tokio::spawn(async move {
            let mut guard = d.lock().await; // 异步等待，不阻塞线程
            *guard += 1;
            // guard 在这里 drop，自动释放锁
        }));
    }
    
    for h in handles { h.await.unwrap(); }
    println!("Result: {}", *data.lock().await);
}
```

### 3.3 关键陷阱

```
陷阱 1：在锁内执行 .await
  let mut guard = mutex.lock().await;
  some_async_op().await; // 危险！锁持有期间其他任务无法获取
  *guard += 1;
  
  解决：先完成异步操作，再获取锁
  let result = some_async_op().await;
  let mut guard = mutex.lock().await;
  *guard += result;

陷阱 2：锁跨越 .await 但用 std::sync::Mutex
  let guard = std_mutex.lock().unwrap();
  some_async_op().await; // 阻塞整个线程！
  
  解决：要么用 tokio::sync::Mutex，要么在 .await 前 drop guard

陷阱 3：死锁
  任务 A 持有锁 1，尝试获取锁 2
  任务 B 持有锁 2，尝试获取锁 1
  
  解决：统一锁获取顺序；或只用一个锁；或用 try_lock 失败则释放重试

陷阱 4：任务取消导致状态不一致
  select! {
      guard = mutex.lock().await => { /* 修改状态 */ }
      _ = cancellation.cancelled() => { /* 任务被取消，但状态可能已改一半 */ }
  }
  
  解决：使用 CancellationToken，在关键位置检查取消；或保证操作原子性
```

---

## 4. 模式三：tokio::sync::RwLock（读写锁）

### 4.1 适用场景

```
读多写少：
  80%+ 操作是读，写操作稀少
  例：配置缓存、路由表、特征开关

与 Mutex 的对比：
  Mutex：任何操作都独占，简单但读无法并行
  RwLock：读可并行，写独占，适合读密集场景

注意：
  写操作会阻塞所有读操作（写者优先或读者优先取决于实现）
  tokio::sync::RwLock 是写者优先（防止写饥饿）
  频繁写操作下，RwLock 性能可能不如 Mutex（调度开销）
```

### 4.2 使用示例

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

#[tokio::main]
async fn main() {
    let data = Arc::new(RwLock::new(vec![1, 2, 3]));
    
    // 多个读任务可并行
    let mut readers = vec![];
    for i in 0..5 {
        let d = data.clone();
        readers.push(tokio::spawn(async move {
            let read_guard = d.read().await;
            println!("Reader {} sees {:?}", i, *read_guard);
            // read_guard 自动 drop
        }));
    }
    
    // 写任务独占
    let writer = tokio::spawn(async move {
        let mut write_guard = data.write().await;
        write_guard.push(4);
        println!("Writer updated");
    });
    
    for r in readers { r.await.unwrap(); }
    writer.await.unwrap();
}
```

### 4.3 写饥饿问题

```
问题：如果读操作持续不断，写操作可能长期等待

tokio::sync::RwLock 的策略：
  写者优先：一旦有写者等待，新读者会被阻塞，直到写者完成
  这避免了写饥饿，但可能降低读吞吐量

如果读吞吐量更重要：
  考虑使用 Mutex + 克隆数据（读时克隆出副本，释放锁后读副本）
  或使用 Copy-on-Write 模式（见模式六）
```

---

## 5. 模式四：分片（Sharding）

### 5.1 核心思想

```
把数据分成多份，每份独立加锁，不同 key 的操作可并行

分片数量：
  经验：2 的幂（如 16, 32, 64, 128），方便位运算取模
  通常：CPU 核心数 × 2~4
  太多：内存开销大，cache locality 差
  太少：锁竞争仍严重
```

### 5.2 手动实现

```rust
use std::collections::HashMap;
use std::hash::{Hash, Hasher};
use std::sync::Arc;
use tokio::sync::Mutex;

const SHARD_COUNT: usize = 64;

pub struct ShardedMap<K, V> {
    shards: Vec<Mutex<HashMap<K, V>>>,
}

impl<K: Hash + Eq, V> ShardedMap<K, V> {
    pub fn new() -> Self {
        let mut shards = Vec::with_capacity(SHARD_COUNT);
        for _ in 0..SHARD_COUNT {
            shards.push(Mutex::new(HashMap::new()));
        }
        Self { shards }
    }
    
    fn shard_index(&self, key: &K) -> usize {
        let mut hasher = std::collections::hash_map::DefaultHasher::new();
        key.hash(&mut hasher);
        (hasher.finish() as usize) & (SHARD_COUNT - 1) // 位运算取模（要求 SHARD_COUNT 是 2 的幂）
    }
    
    pub async fn insert(&self, key: K, value: V) {
        let idx = self.shard_index(&key);
        let mut shard = self.shards[idx].lock().await;
        shard.insert(key, value);
    }
    
    pub async fn get(&self, key: &K) -> Option<V> where V: Clone {
        let idx = self.shard_index(key);
        let shard = self.shards[idx].lock().await;
        shard.get(key).cloned()
    }
}
```

### 5.3 与 Actor 模型结合

```
每个分片一个 Actor：
  比 Mutex 更优，因为 Actor 是无锁串行处理
  但 Actor 是单任务处理，分片内无法并行
  
  选择：
    分片 + Mutex：实现简单，分片内操作串行
    分片 + Actor：无锁，但 Actor 消息队列可能成为瓶颈
    分片 + RwLock：读多写少时读可并行

跨分片操作：
  KEYS * / 全局遍历：需要逐个分片收集，成本高
  事务：跨分片事务需要 2PC，极其复杂，尽量避免
```

---

## 6. 模式五：无锁并发数据结构

### 6.1 DashMap（最常用）

```
特点：
  内部基于分片，每个分片一个 RwLock
  API 接近 HashMap，易用性好
  读多数情况下无全局锁（分片锁）

使用：
  cargo add dashmap
  
  let map = DashMap::new();
  map.insert("key", "value");
  if let Some(entry) = map.get("key") {
      println!("{}", entry.value());
  }

注意：
  不要在持有 DashMap entry 时调用另一个 DashMap 操作（可能死锁）
  不要在 .await 点持有 entry guard（会阻塞其他操作）
  快速 clone 或 copy 值，然后释放 guard

适用：高并发读写，键空间大，不需要跨 key 事务
```

### 6.2 papaya（真正无锁）

```
特点：
  基于 epoch-based memory reclamation，真正无锁
  支持 async：pin_owned() guard 可 Send，可跨越 .await
  compute() API：基于 CAS 的原子操作

使用：
  cargo add papaya
  
  let map = papaya::HashMap::new();
  let guard = map.pin();
  guard.insert("key", "value");

优势：
  无死锁（没有锁）
  可跨越 .await（pin_owned guard）
  高并发读性能极佳

劣势：
  写操作开销高于 DashMap（CAS 循环）
  内存占用稍高（epoch reclamation）
  较新，生态不如 DashMap 成熟

适用：读极多、写少、需要跨 await 访问的场景
```

### 6.3 flurry（Java ConcurrentHashMap 风格）

```
特点：
  epoch-based GC，lock-free reads
  API 类似 Java ConcurrentHashMap
  适合读多写少

与 DashMap 对比：
  DashMap：有锁（分片 RwLock），但 API 更简单
  flurry：无锁读，但需管理 guard（epoch）

选型：
  简单场景：DashMap
  极致读性能：papaya 或 flurry
  需要跨 await：papaya（pin_owned）
```

---

## 7. 模式六：快照 / Copy-on-Write

### 7.1 核心思想

```
数据不可变，每次修改产生新副本，读操作读旧快照

实现方式：
  Arc<Storage> 全局指针
  读：直接克隆 Arc，读快照（无锁）
  写：克隆 Arc → 修改 → 原子替换全局指针（Arc::make_mut 或 Arc::clone + 修改）

关键：Arc::swap 或 Arc::store 是原子操作，读和写不会互相阻塞
```

### 7.2 使用示例

```rust
use std::sync::Arc;
use std::collections::HashMap;
use arc_swap::ArcSwap;

pub struct SnapshotStorage {
    inner: ArcSwap<HashMap<String, String>>,
}

impl SnapshotStorage {
    pub fn new() -> Self {
        Self {
            inner: ArcSwap::new(Arc::new(HashMap::new())),
        }
    }
    
    // 读：完全无锁，克隆 Arc 即可
    pub fn get(&self, key: &str) -> Option<String> {
        self.inner.load().get(key).cloned()
    }
    
    // 写：克隆 + 修改 + 原子替换
    pub fn insert(&self, key: String, value: String) {
        let mut new_map = (*self.inner.load()).clone();
        new_map.insert(key, value);
        self.inner.store(Arc::new(new_map));
    }
}
```

### 7.3 适用与局限

```
适用：
  读极多、写极少（如配置中心、白名单、路由表）
  数据量不大（克隆成本低）
  可容忍最终一致（写操作后，正在进行的读可能读到旧数据）

局限：
  写频繁：每次克隆整个数据结构，CPU 和内存开销大
  大数据量：克隆 GB 级数据不可接受
  需要强一致：写后读可能读到旧数据

优化：
  使用持久化数据结构（如 im-rc, rpds），结构共享，克隆只需 O(log n) 或 O(1)
  批量写：积累多个修改后一次性替换，减少克隆次数
```

### 7.4 arc-swap 库

```
cargo add arc-swap

特点：
  专为 Copy-on-Write 设计，比 AtomicPtr<Arc<T>> 更安全
  支持多读多写，读无锁，写原子替换
  有 Guard 机制，保证读期间数据不被释放

对比 AtomicPtr：
  arc-swap 处理内存安全（旧 Arc 的释放时机）
  AtomicPtr 需要手动管理内存，容易 UAF
```

---

## 8. 选型决策树

```
是否需要共享可变状态？
  ├─ 否 → 用 Channel/Actor（消息传递，无共享）
  │        适用：状态集中管理，操作需串行化
  │        推荐：tokio::sync::mpsc + oneshot，或 tokio-actors
  │
  └─ 是 → 数据量多大？
           ├─ 小（< 1000 条目）→ 用 Copy-on-Write
           │        适用：配置、白名单、路由表
           │        推荐：arc-swap + 持久化数据结构
           │
           └─ 大 → 读写比例？
                    ├─ 读 >> 写（> 10:1）→ 用无锁并发 Map
                    │        适用：缓存、会话存储、元数据索引
                    │        推荐：DashMap（简单）/ papaya（极致读性能）
                    │
                    ├─ 读 > 写（> 3:1）→ 用 RwLock 或分片
                    │        适用：一般业务数据
                    │        推荐：tokio::sync::RwLock 或手动 Sharding
                    │
                    └─ 读写均衡 或 写为主 → 用 Mutex 或分片
                             适用：计数器、队列、状态机
                             推荐：tokio::sync::Mutex + 缩小临界区
                             或：分片 + Mutex（提升并发度）

特殊场景：
  需要事务/原子操作多个 key → Actor 模型（单线程串行保证原子性）
  需要跨 await 访问共享状态 → tokio::sync::Mutex 或 papaya（pin_owned）
  极致性能，无锁必需 → papaya / flurry（但写性能可能下降）
```

---

## 9. 核心学习资源

### 9.1 官方文档与教程

| 资源 | 类型 | 说明 |
|------|------|------|
| [Tokio Tutorial](https://tokio.rs/tokio/tutorial) | 官方教程 | 从基础到 Channel、I/O、Framing，必读 |
| [tokio::sync docs](https://docs.rs/tokio/latest/tokio/sync/) | API 文档 | mpsc, oneshot, broadcast, watch, Mutex, RwLock, Semaphore |
| [Async Rust Book](https://rust-lang.github.io/async-book/) | 官方书籍 | async/await 原理、Pin、Future、Stream |
| [Rust Atomics and Locks](https://marabos.nl/atomics/) | 书籍 | 并发原语底层原理，Mutex、RwLock、Atomics、Memory Ordering |

### 9.2 推荐书籍

| 书名 | 作者 | 重点 |
|------|------|------|
| **Async Rust** | Maxwell Flitton, Caroline Morton | 2024 新书，Tokio 运行时、Actor 模型、异步 TCP 服务器从零构建 |
| **Rust Atomics and Locks** | Mara Bos | 并发原语圣经：Mutex、Condvar、RwLock、Atomics、Memory Ordering 深入讲解 |
| **Programming Rust** | Jim Blandy, Jason Orendorff | 系统级 Rust，并发章节讲 Send/Sync、Channel、Mutex 非常清楚 |
| **Zero To Production In Rust** | Luca Palmieri | 生产级 Rust 后端，含并发、测试、部署 |

### 9.3 优质 GitHub 项目

| 项目 | 说明 |
|------|------|
| [tokio-actors](https://github.com/uwejan/tokio-actors) | 生产级 Actor 框架，OTP 监督、有界邮箱、定时器 |
| [mini-redis](https://github.com/tokio-rs/mini-redis) | Tokio 官方示例项目，Channel、并发、协议解析的最佳实践 |
| [dashmap](https://github.com/xacrimon/dashmap) | 并发 HashMap，学习分片锁设计 |
| [papaya](https://github.com/ibraheemdev/papaya) | 无锁 HashMap，学习 epoch-based reclamation |
| [arc-swap](https://github.com/vorner/arc-swap) | Copy-on-Write 原子指针，学习无锁读设计 |
| [flurry](https://github.com/jonhoo/flurry) | Java ConcurrentHashMap 的 Rust 移植，epoch-based GC |
| [tokio-console](https://github.com/tokio-rs/console) | 异步运行时调试工具，可视化任务、锁、Channel 状态 |

### 9.4 博客与文章

| 文章 | 重点 |
|------|------|
| [Async Rust Without the Footguns](https://muhammadamal.my.id/blog/async-rust-tokio-patterns-2024/) | 生产级 Tokio 模式：有界通道、结构化并发、Cancel Safety |
| [Designing A Fast Concurrent Hash Table](https://ibraheem.ca/posts/designing-papaya/) | papaya 设计原理，无锁 HashMap 的深入讲解 |
| [Rust Async Concurrency Patterns](https://lobehub.com/bg/skills/gar-ai-mallorn-rust-async-concurrency) | Channel 选择、Semaphore、Stream 处理 |
| [DashMap vs HashMap](https://users.rust-lang.org/t/dashmap-vs-hashmap/122953) | 社区讨论，死锁警告、替代方案 |

### 9.5 视频与课程

| 资源 | 平台 | 说明 |
|------|------|------|
| [RustConf 2025 Async Fundamentals](https://github.com/thebracket/rustconf2025_async_fundamentals) | GitHub + 视频 | Tokio 宏、Actor Model、Axum 集成 |
| [Jon Gjengset 的 Rust 视频](https://www.youtube.com/c/JonGjengset) | YouTube | 深入讲解 Rust 并发、Tokio、Pin、Future 原理 |
| [Tokio 官方 YouTube](https://www.youtube.com/@tokio-rs) | YouTube | 运行时设计、最佳实践 |

### 9.6 调试与性能工具

| 工具 | 用途 |
|------|------|
| [tokio-console](https://github.com/tokio-rs/console) | 可视化异步任务、锁竞争、Channel 积压 |
| [cargo flamegraph](https://github.com/flamegraph-rs/flamegraph) | 生成火焰图，定位 CPU 热点 |
| [criterion](https://github.com/bheisler/criterion.rs) | 基准测试，对比不同并发方案性能 |
| [loom](https://github.com/tokio-rs/loom) | 并发代码测试，模拟各种线程交错，发现数据竞争 |
| [miri](https://github.com/rust-lang/miri) | Rust 未定义行为检测，检测 unsafe 代码问题 |

---

## 10. 生产踩坑与面试要点

### 10.1 常见陷阱清单

```
1. 在 async 中用 std::sync::Mutex 跨越 .await
   → 阻塞线程，运行时瘫痪
   → 解决：用 tokio::sync::Mutex，或 .await 前 drop guard

2. 无界通道导致 OOM
   → unbounded_channel() 在消费者慢时内存无限增长
   → 解决：用 mpsc::channel(N)，利用背压

3. DashMap 死锁
   → 持有 entry guard 时调用另一个 DashMap 操作
   → 解决：快速 clone/copy 值，释放 guard 后再操作

4. 任务取消导致状态不一致
   → select! 中一个分支完成，另一个分支的 Future 被 drop
   → 解决：用 CancellationToken，关键操作检查取消状态

5. 忘记 Send/Sync 约束
   → 把 !Send 类型（如 Rc）跨任务传递
   → 编译错误，但新手可能困惑
   → 解决：用 Arc 替代 Rc，用 Mutex 包装非 Sync 类型

6. 锁粒度太粗
   → 一个 Mutex 保护整个数据结构，所有操作串行
   → 解决：分片、细粒度锁、或无锁数据结构

7. 在锁内做 I/O
   → 网络请求、文件读写放在锁内
   → 解决：I/O 在锁外做，锁只保护数据修改
```

### 10.2 面试高频问题

**Q1：tokio::sync::Mutex 和 std::sync::Mutex 的区别？**
> std::sync::Mutex 的 .lock() 阻塞 OS 线程，在 async 中跨越 .await 会卡住整个线程上的所有任务。tokio::sync::Mutex 的 .lock().await 是异步的，等待时当前任务挂起（yield），线程去执行其他任务，锁可用时通过 Waker 唤醒。但 tokio::sync::Mutex 也有开销，如果临界区极短且不跨越 .await，std::sync::Mutex 性能更好。

**Q2：Actor 模型和共享内存模型怎么选？**
> Actor 模型适合状态集中管理、需要保证操作顺序的场景（如库存扣减、状态机），无锁、无死锁、逻辑清晰，但单 Actor 是串行处理，超高并发需多 Actor 实例。共享内存模型（Mutex/RwLock）适合数据分散、需要随机访问的场景，但需处理锁竞争、死锁、粒度问题。Rust 中推荐优先尝试 Actor，复杂时再考虑共享状态。

**Q3：DashMap 和 Mutex<HashMap> 的区别？**
> DashMap 内部分片（Sharding），每个分片一个 RwLock，不同 key 的操作可并行，并发度高。Mutex<HashMap> 是整个 HashMap 一把锁，所有操作串行。DashMap API 接近 HashMap，但需注意死锁（持有 entry 时调用另一个 DashMap 操作）。高并发读写场景 DashMap 性能显著优于 Mutex<HashMap>。

**Q4：什么是 Copy-on-Write，适用场景是什么？**
> Copy-on-Write 是数据不可变，读操作直接读当前快照（无锁），写操作克隆一份修改后原子替换指针。适用读极多、写极少、数据量不大的场景（如配置中心、路由表）。不适合写频繁或数据量大的场景，因为每次写都要克隆整个数据结构。可用 arc-swap 库实现，或用持久化数据结构（im-rc）优化克隆成本。

**Q5：Rust 中如何避免死锁？**
> 1. 统一锁获取顺序（所有任务按相同顺序获取多把锁）；2. 减少锁的数量（用一个锁替代多个锁）；3. 使用 try_lock，失败时释放已持有的锁并重试；4. 优先使用无锁数据结构（DashMap、papaya）；5. 使用 Actor 模型，无共享状态自然无死锁；6. 用 loom 测试并发代码，模拟各种线程交错。

**Q6：有界通道和无界通道的区别？为什么生产环境要用有界？**
> 有界通道 mpsc::channel(N) 在满时 send().await 会等待，提供背压（backpressure），防止内存无限增长。无界通道 unbounded_channel() 在消费者慢时队列无限增长，最终 OOM。生产环境必须用有界通道，背压是流控的重要机制，让慢的消费者自然阻塞快的生产者。

**Q7：什么是 Cancel Safety？**
> 在 select! 或超时中，一个 Future 被选中执行，其他 Future 被 drop。如果被 drop 的 Future 已经产生了副作用（如修改了状态、发送了消息），但 select 选择了另一个分支，会导致状态不一致。Cancel-safe 的 Future 在被 drop 时不会产生副作用，或副作用可安全忽略。设计异步操作时要考虑 Cancel Safety，关键操作用 CancellationToken 控制。

---

## 一句话总结

> Rust 高并发异步 = Channel/Actor（无锁首选）→ tokio::sync::Mutex（跨 await 共享）→ RwLock（读多写少）→ 分片（提升并发度）→ 无锁 Map（DashMap/papaya，极致性能）→ Copy-on-Write（读极多写极少）。核心原则：能消息传递就不共享状态，能无锁就不加锁，锁粒度越细越好，有界通道防 OOM。
