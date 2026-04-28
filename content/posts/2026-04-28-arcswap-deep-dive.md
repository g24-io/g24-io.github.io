+++
title = "ArcSwap 深度：让百万 QPS 读路径无锁的 Rust 工具"
description = "从 Atomic Pointer 到 RCU——ArcSwap 在 ADX 引擎里到底做了什么、不能做什么、以及为什么它和 Mutex/RwLock 是不同物种。"
date = 2026-04-28
slug = "arcswap-deep-dive"

[taxonomies]
tags = ["arcswap", "rust", "concurrency", "performance", "rcu", "lock-free"]
series = ["adx-design"]

[extra]
author = "g24-io engineering"
tldr = [
  "**ArcSwap = Atomic Pointer + Arc**——不是锁，是「无锁原子替换整体快照」的工具。",
  "**读路径 5–15 ns**——一次原子指针读 + 一次 Arc 引用计数 inc，远低于 Mutex / RwLock。",
  "**写路径不修改原对象**——构建一个新 Arc，整体 swap；旧版引用计数到 0 自然 drop。",
  "**适合「读多写少 + 整体替换」**：配置快照、路由表、索引、模型权重——不适合细粒度状态机。",
  "**最容易踩的坑**：忘了大对象 drop 是 stop-the-world——drop 要交给后台 worker。",
]
+++

## 引擎里反复出现的那个名字

[第 4 篇 targeting-context](/posts/targeting-context/) 用 ArcSwap 装百万 plan 的索引。
[第 6 篇 integration-overview](/posts/integration-overview/) 提到运营后台变更通过 ArcSwap 切快照。
[实战篇第 1 篇](/posts/roaringbitmap-in-production/) 整篇都在围绕 ArcSwap 设计 IndexSlots。

每次都是<em>一笔带过</em>——读多写少、无锁、原子替换。

但 ArcSwap 这个工具到底<strong>怎么做到无锁</strong>？
什么时候用它<em>不对</em>？为什么它和 `Mutex` / `RwLock` 是<strong>完全不同的物种</strong>？
这一篇专门把它讲清楚。

读者预期：

- 已经在 Rust 项目里写过 `Mutex` / `RwLock`，但没用过 ArcSwap
- 想搞懂"读多写少"场景应该选什么，依据是什么
- 关心 RTB / 高频读场景的工程取舍

> 一句话定位：**ArcSwap 是 Rust 实现「RCU（Read-Copy-Update）模式」的标准工具**——
> 它不是锁，更接近"原子整体替换"。读这篇前先把"它是锁吗"这个误解扔掉。

## ArcSwap 内部到底是什么

简化模型：`ArcSwap<T>` 包装的是 **一个 `AtomicPtr<T>` 和一套引用计数协议**。
完整实现远比这复杂（涉及 hazard pointer / generation tracker / GC 节奏），
但概念可以用 30 行伪代码说清楚：

```rust
pub struct ArcSwap<T> {
    ptr: AtomicPtr<T>,         // 当前的 Arc 内部裸指针
    // ... 还有一些用于解决 ABA 问题的 generation 计数
}

impl<T> ArcSwap<T> {
    pub fn load(&self) -> Arc<T> {
        // 1. 原子读当前指针
        let raw = self.ptr.load(Ordering::Acquire);
        // 2. 安全地 inc 引用计数（细节：要解决 reader 拿到 ptr 但
        //    writer 已经 dec 了 count 的竞争条件 —— ArcSwap 用 hazard
        //    pointer 或者 epoch GC 来保证）
        unsafe { Arc::increment_strong_count(raw) }
        // 3. 重建 Arc——不再是裸指针
        unsafe { Arc::from_raw(raw) }
    }

    pub fn store(&self, new: Arc<T>) {
        let new_raw = Arc::into_raw(new);
        // 4. 原子 swap，拿到旧的裸指针
        let old_raw = self.ptr.swap(new_raw as *mut _, Ordering::AcqRel);
        // 5. 旧 Arc 不再被 ptr 引用——把这个引用计数减掉
        unsafe { Arc::decrement_strong_count(old_raw) }
        // 注意：旧 Arc 不一定立即 drop——
        // 如果还有其它 reader 持有，等他们都释放完才 drop。
    }
}
```

**关键点**：

- 读路径只有<em>一次原子指针读</em>+<em>一次原子计数 inc</em>——**没有锁、没有等待**
- 写路径是<em>整体替换</em>，不修改原对象——这是 RCU（Read-Copy-Update）的精神
- 旧版本由 **reader 引用计数** 决定何时回收——writer 不知道 reader 何时完成

> ArcSwap 不是抢占式（preemptive）——它是 cooperative：
> reader 拿到 Arc 之后想用多久用多久，writer 不会打断。
> 这是它和"锁"的根本区别。

## 真正的实测延迟

`AtomicPtr::load` 加 `Arc::clone`（即 `increment_strong_count`）实测：

| 操作 | 典型延迟 | 是否 wait-free |
|---|---:|---|
| `ArcSwap::load` | **5–15 ns** | ✓ |
| `Arc::clone`（已持有 Arc）| 2–5 ns | ✓ |
| `Mutex::lock`（无竞争）| 20–40 ns | ✗ |
| `Mutex::lock`（有竞争）| 100ns – 数 ms | ✗ |
| `RwLock::read`（无竞争）| 30–60 ns | ✗ |
| `RwLock::read`（有竞争写）| 100ns – 数 ms | ✗ |

差距在<em>有竞争</em>时显著放大——这是 ArcSwap 在百万 QPS 读路径上不可替代的根本原因。

## 与 Mutex / RwLock 的对比

它们看似都解决"多线程访问共享数据"的问题，但<strong>语义完全不同</strong>：

| 维度 | `Mutex<T>` | `RwLock<T>` | `ArcSwap<T>` |
|---|---|---|---|
| 读阻塞写？| 是 | 否（读读并行）| **否** |
| 写阻塞读？| 是 | 是 | **否** |
| 写写阻塞？| 是 | 是 | 否（但有"丢失更新"风险）|
| 数据可变？| 直接修改 | 直接修改 | **不可变**（每次 store 整个新对象）|
| 读延迟（无竞争）| 20–40 ns | 30–60 ns | **5–15 ns** |
| 适合粒度 | 细粒度 | 中粒度 | 粗粒度 |
| 适合什么 | 写多 / 状态机 | 读多写少（数据可变）| 读极多 / 全量替换 |

### 一个直观的反例：什么时候 ArcSwap 是错的

假设你要做一个 `HashMap<UserId, FrequencyCounter>`——每次成交 +1，每次查询 read。
**这是 ArcSwap 的反例**：

```rust
// ❌ 错误用法
struct UserCounters {
    map: ArcSwap<HashMap<UserId, u64>>,
}

impl UserCounters {
    fn increment(&self, uid: UserId) {
        let cur = self.map.load_full();
        let mut new = (*cur).clone();      // ← 整个 HashMap clone！
        *new.entry(uid).or_insert(0) += 1;
        self.map.store(Arc::new(new));     // 写入新版本
    }
}
```

每次 `increment` clone 整张 HashMap——百万 entry 的 HashMap 一次 clone 几百毫秒。
**ArcSwap 不适合"细粒度可变状态"**——它的写代价是<em>整体重建</em>。

正确做法：用 `DashMap` 或 `Mutex<HashMap>` 的分片版（`sharded_hashmap`）。

### ArcSwap 适合什么

- **配置 / 路由表 / 索引快照**——读极多、写每分钟一次
- **模型权重**——一个 ML 模型 800 MB，每天替换一次
- **大型只读字典**——比如 GeoIP 数据库
- **DSP 端点列表**——50–500 个 endpoint，每分钟刷新

> 黄金法则：**写代价 = 重建整个对象的代价**。
> 这个代价能接受时用 ArcSwap，否则别用。

## 典型用法：从基础到 IndexSlots

### 最小可用模式

```rust
use arc_swap::ArcSwap;
use std::sync::Arc;

struct Service {
    config: ArcSwap<Config>,
}

impl Service {
    fn new(initial: Config) -> Self {
        Self { config: ArcSwap::from(Arc::new(initial)) }
    }

    fn current_threshold(&self) -> u32 {
        self.config.load().threshold       // ← 注意 load() 返回的不是 Arc<T>
    }

    fn reload(&self, new: Config) {
        self.config.store(Arc::new(new));   // 整体替换
    }
}
```

### `load()` 与 `load_full()` 的差异——这是高频踩坑点

```rust
// load() 返回 Guard<Arc<T>>——它是个临时引用，drop 后释放
let guard = self.config.load();
let t = guard.threshold;                   // OK——Guard 还活着
println!("{}", guard.endpoint);            // 仍然 OK

// load_full() 返回 owned Arc<T>——可以传出函数
fn current_config(&self) -> Arc<Config> {
    self.config.load_full()                // ← 而不是 self.config.load().clone()
}
```

**两者性能差异**：

- `load()` 返回 Guard——**几纳秒**，最快路径
- `load_full()` 返回 Arc clone——多一次 inc，**~10 ns**
- `load().clone()`（Guard.clone()）——和 `load_full()` 等价

**什么时候用哪个**：

- 读路径单次使用 → `load()` Guard 即可
- 要把数据传给其它函数或 spawn task → `load_full()`，避免 Guard 借用问题
- 在长时间 hot-loop 里 → 用 `Cache` 包装（见进阶段）

### 可空版本 `ArcSwapOption`

某些场景"可能没有当前值"——如 IndexSlots 的 canary 槽：

```rust
use arc_swap::ArcSwapOption;

struct IndexSlots {
    primary:  ArcSwap<Index>,
    canary:   ArcSwapOption<Index>,        // ← 可能没有 canary
}

impl IndexSlots {
    fn maybe_use_canary(&self) -> Option<Arc<Index>> {
        self.canary.load_full()             // 返回 Option<Arc<Index>>
    }

    fn deploy_canary(&self, new: Arc<Index>) {
        self.canary.store(Some(new));
    }

    fn abort_canary(&self) {
        self.canary.store(None);
    }
}
```

`ArcSwap<Option<Arc<T>>>` 是错的（多一层 Arc）。
**用 `ArcSwapOption<T>`**——它内部直接管 `Option<Arc<T>>`。

## 三个真实事故

### 事故 1 · drop 大对象时整线程被卡

[实战篇第 1 篇](/posts/roaringbitmap-in-production/) 已经讲过这条事故，这里换一个角度复盘：
**ArcSwap 切换瞬间的 stop-the-world 不在 ArcSwap 自己，而在被替换对象的 drop**。

时序：

```text
   t0  builder 算出新 index
   t1  arc_swap.store(new_arc)
       ├── 原子指针 swap：~5 ns
       └── 旧 Arc 引用计数 dec：~3 ns
   t2  最后一个持有旧 Arc 的 reader 释放
       └── 触发 drop(old_arc) ← jemalloc 释放数千个 chunk，卡 30–80 ms
```

**ArcSwap 自己零开销**——但<em>触发的 drop 不是它能控制的</em>。
解法（前面已经讲过）：

```rust
struct Service {
    config: ArcSwap<Config>,
    drop_tx: mpsc::UnboundedSender<Arc<Config>>,
}

impl Service {
    fn replace(&self, new: Arc<Config>) {
        let old = self.config.swap(new);
        let _ = self.drop_tx.send(old);     // 异步丢给后台 worker
    }
}
```

> 重点不在 ArcSwap，在<em>大对象的释放成本</em>。
> 任何被 ArcSwap 装的对象，都该问一句：**它 drop 要多久？**

### 事故 2 · Guard 借用导致 holding-while-await

```rust
// ❌ 这段代码会编不过——但即使你强行用 unsafe 绕过，行为也是错的
async fn handle_request(svc: &Service, ctx: &Ctx) -> Result<()> {
    let cfg = svc.config.load();           // Guard 借用了 svc
    let res = downstream.call(&cfg.url).await?;  // ← Guard 跨 await
    Ok(res)
}
```

Guard 跨 await 持有意味着——**这个请求每一刻都在让 svc 不能被 drop**。
对 RTB 的 hot path 影响：写路径切换后旧 Arc 一直被 await 中的 Guard 持有，
延迟整个 drop 链路。

**正确做法**：在 await 前用 `load_full()` 拿 owned Arc：

```rust
async fn handle_request(svc: &Service, ctx: &Ctx) -> Result<()> {
    let cfg = svc.config.load_full();      // owned Arc，不借用 svc
    drop(cfg.as_ref());                     // 取完字段就丢——这步可选
    let res = downstream.call(&cfg.url).await?;
    Ok(res)
}
```

> 实战经验：**Tokio 任务里几乎永远用 `load_full()` 而不是 `load()`**——
> Guard 的"零拷贝优势"在异步场景里被借用问题抵消。

### 事故 3 · 多次 store 之间的"ABA 错觉"

```rust
// 场景：reader 想知道"配置在我处理期间变过吗"
let snap1 = svc.config.load_full();
do_work().await;
let snap2 = svc.config.load_full();
if Arc::ptr_eq(&snap1, &snap2) {
    // 这两次拿到的是同一个 Arc——配置没变？
}
```

**陷阱**：如果在 `do_work` 期间发生过 A → B → A 的 store 序列，
ArcSwap 第二次 load 出来的 Arc 是<strong>新构造的 A</strong>——和第一次不是同一个 Arc。
`Arc::ptr_eq` 返回 `false`，但<em>语义上配置没变</em>。

工程上：

- 不要用 `Arc::ptr_eq` 做"配置变更检测"——那是 implementation detail
- 真要做变更检测，<em>显式版本号</em>：`AtomicU64` 单独维护一个 generation counter

```rust
struct Service {
    config: ArcSwap<Config>,
    gen: AtomicU64,
}

impl Service {
    fn replace(&self, new: Arc<Config>) {
        self.config.store(new);
        self.gen.fetch_add(1, Ordering::AcqRel);
    }

    fn current_gen(&self) -> u64 {
        self.gen.load(Ordering::Acquire)
    }
}
```

reader 拿 generation 比较——这个语义就稳了。

## 进阶：3 个让 ArcSwap 更顺手的模式

### 模式 1 · `arc_swap::Cache`：在 hot loop 里去掉每次 load 的 atomic op

`load()` 的 5–15 ns 单次看不算什么——但如果你在 hot loop 里<em>每次迭代都 load</em>，
就值得用 `Cache`：

```rust
use arc_swap::{ArcSwap, Cache};

fn process_batch(svc: &Service, batch: &[Item]) {
    let mut cache = Cache::new(&svc.config);

    for item in batch {
        let cfg: &Config = cache.load();    // ← 复用已 cache 的 Arc，零原子操作
        process_one(item, cfg);
    }
}
```

`Cache` 内部记着上次 load 时的 generation，
只有当 ArcSwap 真的 `store` 过新版本时才重新拿 Arc——
**循环里 load 的 amortized 成本压到 ~1 ns**。

适合：批处理 worker、定时聚合 task。
不适合：单次请求路径——单次成本本来就低。

### 模式 2 · `ArcSwapAny` + 自定义 Storage

默认 `ArcSwap<T>` 等价于 `ArcSwapAny<Arc<T>>`。
`ArcSwapAny` 是泛型版本——可以装 `Arc<T>` 也可以装其它 RC 类型：

```rust
use arc_swap::{ArcSwapAny, RefCnt};

// 用 triomphe::Arc（移除 weak count，速度略快）
type FastArcSwap<T> = ArcSwapAny<triomphe::Arc<T>>;
```

绝大多数项目<em>不需要</em>这个——`ArcSwap<T>` 已经够。
仅在 hot loop 里 `Arc::clone` 占了 profile 显著比例时考虑。

### 模式 3 · 与 generation counter 显式协作

如果你的业务需要做<em>变更检测</em>（cache invalidation、watermark 推进、对账），
ArcSwap 单独不够——配套 `AtomicU64` 维护 generation：

```rust
pub struct VersionedSwap<T> {
    inner: ArcSwap<T>,
    gen:   AtomicU64,
}

impl<T> VersionedSwap<T> {
    pub fn store(&self, new: Arc<T>) -> u64 {
        self.inner.store(new);
        self.gen.fetch_add(1, Ordering::AcqRel) + 1
    }

    pub fn load(&self) -> (u64, arc_swap::Guard<Arc<T>>) {
        // ⚠️ 注意：先读 gen 还是先读 inner？两者都不能完美原子——
        // 工程上接受 gen 比 inner 略落后或略超前一帧
        let g = self.gen.load(Ordering::Acquire);
        let v = self.inner.load();
        (g, v)
    }
}
```

> Generation 与 ArcSwap 之间的"原子性"——严格说不可达。
> 但<strong>因果一致</strong>足够：旧 gen 配旧 inner、新 gen 配新 inner，
> 中间态对业务无感。

### 模式 4（隐含）· Tokio task 与 ArcSwap 的协作

把 `Arc<ArcSwap<T>>` clone 到 spawn 的 task 里——
**用 `Arc<ArcSwap<T>>` 而不是 `ArcSwap<Arc<T>>`**：

```rust
use std::sync::Arc;

let svc = Arc::new(MyService::new());                    // Arc<MyService>
                                                          // svc.config: ArcSwap<Config>

let svc_clone = svc.clone();
tokio::spawn(async move {
    loop {
        let cfg = svc_clone.config.load_full();
        do_work(&cfg).await;
    }
});
```

每个 task 拿一份 `Arc<MyService>`——所有 task 共享同一个 ArcSwap，
write path 一次 store 所有 task 立刻看到新版本。

## 一段话收尾：3 条心法

### 心法 1 · ArcSwap 不是锁，是 RCU

把 `ArcSwap<T>` 当 `Mutex<T>` 的"快版"用——会在第二天发现细粒度可变状态写不出来。
**它的写代价是<em>整体重建</em>**——这是它的特征，不是缺陷。
适合粗粒度、不适合细粒度。

### 心法 2 · 关心被装的东西的 drop 成本

ArcSwap 自己<em>零开销</em>——但被装的对象 drop 不是。
任何 `ArcSwap<BigThing>` 都该想清楚：BigThing 释放时会触发多少 free？
答案大于"几百微秒"，就把 drop 关进后台 worker。

### 心法 3 · 异步路径用 `load_full()`，避免 Guard 跨 await

`load()` 的 Guard 在借用规则下会让函数签名复杂、跨 await 时会让旧 Arc 滞留。
**Tokio 任务里默认 `load_full()`**——多 5 ns 换一切都能 owned 流转，值得。

---

ArcSwap 是 Rust 生态里"被低估的小工具"——
大多数 Rust 教程都会教 Mutex / RwLock，
**但教 ArcSwap 的教程屈指可数**。
原因是它的适用场景<em>窄</em>——读极多 + 整体替换 + 大对象。
RTB 引擎正是<em>每一项都满足</em>的场景，所以它在这里几乎无可替代。

如果你的项目正好有这个形状的需求——这篇文章帮你绕过我们当年的所有坑。

---

<small>
本篇是 ADX 设计系列的<strong>实战篇第 3 篇</strong>，承接
<a href="/posts/roaringbitmap-in-production/">实战篇第 1 篇</a>
中反复出现的 ArcSwap 工具，单独深入。
</small>
