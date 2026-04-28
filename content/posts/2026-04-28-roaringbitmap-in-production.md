+++
title = "把 RoaringBitmap 用对：一个 ADX 引擎里的全部细节"
description = "从基础选型到容量规划、跨语言互操作、一致性防御、多版本共存——一份生产级落地清单。"
date = 2026-04-28
slug = "roaringbitmap-in-production"

[taxonomies]
tags = ["roaringbitmap", "rust", "performance", "data-structure", "adx", "production"]
series = ["adx-design"]

[extra]
author = "g24-io engineering"
tldr = [
  "**RoaringBitmap 不是魔法**——它是把「高频读 + 低频写 + 集合运算」做到极致的数据结构。",
  "**单 ArcSwap 起步够**，扩到千万 plan 要建 IndexSlots、一致性 4 层防御、容量规划。",
  "**暂态开销 ≠ 稳态开销**：构建 / 反序列化 / drop 各自有 2–3× 的瞬时峰值。",
  "**跨语言走 portable**——u32 域 RoaringBitmap 三家二进制兼容，u64 RoaringTreemap 不要跨语言。",
  "**90% 现代项目终点是 Roaring**——不是因为最快，是因为生态最厚。",
]
+++

## 系列里那段被一笔带过的 RoaringBitmap

ADX 设计系列第 4 篇 [《定向上下文：用 RoaringBitmap 把 8 维筛选打成毫秒级》](/posts/targeting-context/)
里我们用一段话讲了为什么用 bitmap 倒排——那段大概是这样：

> 倒排索引 + 位图运算 + ArcSwap 快照——8 类 Criterion 在线匹配的工程做法。

干净、利落，但<em>不够</em>。这套在生产里跑了一年多，期间踩过的坑、做过的容量规划、
建过的一致性防御、和其它技术栈对比的判据——值得单独写一篇。
这篇就是把那一笔带过的东西<strong>全部摊开</strong>。

读者预期：

- 已经知道 RoaringBitmap 是什么、跑过 hello world
- 想搞清楚<em>它在生产里到底要做哪些工程功夫</em>
- 关心 5 ms RTB 预算下的工程纪律，而不仅是 API 调用

> 这篇不是教程——是<strong>事故日志 + 决策日志</strong>的结晶。
> 任何一段都对应某次真实在线工况下的取舍。

## RoaringBitmap 是怎么压紧的

把 32-bit 整数空间按高 16 位切成 2^16 个桶，每个桶按稀疏度<em>自适应</em>选 3 种存储：

- **Array container**（< 4096 个数）—— sorted u16 数组
- **Bitmap container**（≥ 4096 个数）—— 8 KB 固定位图
- **Run container**（长连续段）—— RLE 压缩

带来的<em>规模化</em>效应：

- 100 万 PlanId 集合内存 ~100 KB（vs `HashSet<u32>` ~16 MB）
- 集合 AND / OR / MINUS 跨容器走专用 SIMD 路径，百万级在 50–200 µs
- 序列化是 portable binary，可直接存对象存储 / 通过 Kafka 投递

API 速览：

```rust
use roaring::RoaringBitmap;

let mut bm = RoaringBitmap::new();
bm.insert(42);
bm.contains(42);                     // true
bm.len();                            // 1, O(1)，container 内置 cardinality 缓存

let a = RoaringBitmap::from_iter([1, 2, 3, 4]);
let b = RoaringBitmap::from_iter([3, 4, 5, 6]);

let i = &a & &b;                     // {3, 4}（owned 新 bitmap）
let mut x = a.clone();
x &= &b;                             // in-place，避免 clone

// portable 序列化，跨语言兼容
let mut buf = Vec::new();
bm.serialize_into(&mut buf)?;
let restored = RoaringBitmap::deserialize_from(&buf[..])?;
```

## 为什么不是 EWAH / BitMagic / Concise

Roaring 不是技术上最聪明的——是<em>每一项都不弱</em>，且生态最厚。
我们做过 head-to-head benchmark（1000 万 plan_id、8 维倒排）：

| 实现 | 索引体积 | match p99 | 内存峰值 | 跨语言 |
|---|---:|---:|---:|---|
| **Roaring** (`roaring` 纯 Rust) | 1.2 GB | 280 µs | 2.8 GB | ✓ portable |
| **Roaring** (`croaring` C 绑定) | 1.2 GB | **180 µs** | 2.6 GB | ✓ portable |
| EWAH | 0.95 GB | 720 µs | 2.0 GB | △（rare bindings）|
| BitMagic | 1.05 GB | 220 µs | **1.8 GB** | ✗（C++ only）|
| `HashSet<u32>` baseline | 18 GB | 950 µs | 19 GB | ✗ |

我们项目的实际选择：**`roaring` 纯 Rust 起步**。
理由：

1. **跨语言协议**——builder 在 Spark/Scala 上跑，引擎在 Rust 读，
   `roaring`(JVM) ↔ `roaring`(Rust) 二进制 portable 兼容
2. **生态厚**——Lucene/ES/Druid 内部都用，bug 早被踩过
3. **够快**——hot path 没卡住时不必上 `croaring` C 绑定

> 一条经验：**90% 现代项目的终点都是 Roaring**——
> 不是因为它每一项最强，是因为它<em>每一项都不弱</em>，且选它<em>解释成本</em>最低。

### 一条容易踩的兼容性陷阱

`RoaringBitmap`（u32 域）跨语言**完全 portable**。
但 64-bit 版本——
Java `Roaring64NavigableMap` / Rust `RoaringTreemap` / C `roaring64_bitmap_t`——
**三家的二进制布局不一样**。

我们的做法：**永远不跨语言传 64-bit bitmap**。
要表 u64 集合就在 builder 端先 split 高低 32 位，传两个 u32 bitmap 自己组装。

## 索引结构：8 维倒排 + 内部 idx 映射

bitmap 只能存 u32——但业务 PlanId 可能是 UUID 或 i64。
中间插一张「外部 id ↔ 内部 idx」映射表：

```rust
pub struct TargetingIndex {
    plan_idx:  HashMap<PlanId, u32>,             // 外部 id → 内部 u32 idx
    plan_list: Vec<Arc<TargetingPlan>>,           // idx → plan

    // 8 维 include 倒排：token → bitmap-of-internal-idx
    geo:        HashMap<GeoToken,        RoaringBitmap>,
    device:     HashMap<DeviceToken,     RoaringBitmap>,
    daypart:    HashMap<DaypartBucket,   RoaringBitmap>,
    contextual: HashMap<IabCategoryId,   RoaringBitmap>,
    audience:   HashMap<AudienceTagId,   RoaringBitmap>,
    lookalike:  HashMap<LookalikeModel,  RoaringBitmap>,
    ssp:        HashMap<SspId,           RoaringBitmap>,
    publisher:  HashMap<PublisherId,     RoaringBitmap>,

    // 8 维 exclude 倒排（同样形态，查询时做 MINUS）
    geo_excl:   HashMap<GeoToken, RoaringBitmap>,
    // … 其余 7 维 _excl

    // 关键：「未声明该维」的 plan 必须通过该维——而不是被误判为不命中
    geo_unconstrained:        RoaringBitmap,
    // … 其余 7 维 _unconstrained
}
```

这里 `_unconstrained` 是个容易漏的设计点——
**`Criterion::Geo` 在某 plan 里完全不存在 ≠ 该维度不命中**——
它的语义是「对该维度无要求」。查询时把这部分 plan 一并放过。

## 一次匹配：8 维 AND + MINUS

```rust
pub fn match_plans(idx: &TargetingIndex, ctx: &ImpContext) -> RoaringBitmap {
    // 1) 每维 dim_match = lookup(token) | unconstrained
    fn dim<K: Eq + Hash>(
        table: &HashMap<K, RoaringBitmap>,
        unconstrained: &RoaringBitmap,
        tokens: &[K],
    ) -> RoaringBitmap {
        let mut acc = RoaringBitmap::new();
        for t in tokens {
            if let Some(bm) = table.get(t) { acc |= bm; }
        }
        acc |= unconstrained;
        acc
    }

    let dims = [
        dim(&idx.geo,        &idx.geo_unconstrained,        &ctx.geo),
        dim(&idx.device,     &idx.device_unconstrained,     &ctx.device),
        dim(&idx.daypart,    &idx.daypart_unconstrained,    &[ctx.daypart]),
        dim(&idx.contextual, &idx.contextual_unconstrained, &ctx.contextual),
        dim(&idx.audience,   &idx.audience_unconstrained,   &ctx.audience),
        dim(&idx.lookalike,  &idx.lookalike_unconstrained,  &ctx.lookalike),
        dim(&idx.ssp,        &idx.ssp_unconstrained,        &[ctx.ssp]),
        dim(&idx.publisher,  &idx.publisher_unconstrained,  &[ctx.publisher]),
    ];

    // 2) 跨维 AND——按基数升序排，让 AND 在最小的集合上推进
    let mut dims: Vec<_> = dims.into_iter().collect();
    dims.sort_by_key(|bm| bm.len());
    let mut hit = dims.into_iter().next().unwrap();
    for bm in &dims[1..] {
        hit &= bm;
        if hit.is_empty() { return hit; }      // 提前剪枝
    }

    // 3) MINUS exclude
    hit -= &lookup_dim(&idx.geo_excl,    &ctx.geo);
    hit -= &lookup_dim(&idx.device_excl, &ctx.device);
    // … 其余 6 维 _excl
    hit
}
```

3 个性能护栏：

- **按基数升序 AND**——先用最小集合参与运算，让结果集只可能更小，配合 `is_empty()` 短路
- **`&=` / `-=` in-place**——bitmap 一旦定型不再分配
- **lookup miss 返回空 bitmap**——别 panic，让该 token 没倒排自然降级为 0 命中

## 两段式过滤：把"高频写状态"剥出 ArcSwap

bitmap 倒排是「构一次、读一万次」的结构。但<em>频次帽 / Pacing / 预算</em>完全相反：
每次成交都要 +1，秒级几十万次写。这些必须从倒排索引里抽出来，单独走 Redis Cluster。

```rust
pub async fn final_match(
    &self,
    ctx: &ImpContext,
    user: &UserKey,
) -> Vec<Arc<TargetingPlan>> {
    // ─── 段 1：纯 bitmap，0 IO，~50–280 µs ────────────
    let snap = self.index.load();
    let bm   = match_plans(&snap, ctx);
    if bm.is_empty() { return vec![]; }

    let plans: Vec<_> = bm.iter()
        .map(|i| snap.plan_list[i as usize].clone())
        .collect();

    // ─── 段 2：批量查 Redis（pipeline + Lua），~1–2 ms ──
    let plan_ids: Vec<PlanId> = plans.iter().map(|p| p.id).collect();
    let states = self.runtime_state
        .batch_check(&plan_ids, user)
        .await?;

    plans.into_iter()
         .zip(states)
         .filter(|(_, st)| st.frequency_ok && st.pacing_ok && st.budget_ok)
         .map(|(p, _)| p)
         .collect()
}
```

段 1 的输出是<em>理论上能投</em>的 plan 集合；段 2 的输出是<em>此刻真能投</em>。
两段拆开的好处：

- **bitmap 快照永远静态**，运营改频次帽不会引爆 ArcSwap 重建
- **Redis 调用只发生在有候选 plan 的请求**——段 1 空集直接短路
- **Redis 调用是 batch 的**——一次往返查 N 个 plan_id，不是逐个 RTT

实测端到端 targeting：

| 阶段 | p50 | p99 |
|---|---:|---:|
| 段 1 (bitmap match) | 95 µs | 280 µs |
| 段 1 (bitmap → Vec\<PlanId\>) | 15 µs | 65 µs |
| 段 2 (Redis pipeline) | 700 µs | 1.4 ms |
| **合计** | **0.85 ms** | **1.85 ms** |

5 ms RTB 内部预算下，bitmap 路径只吃 **40%**——还有 3 ms 留给其它环节。

## 跨语言：Spark/Scala 算 + Rust 读

builder 我们跑在 Spark 上（Scala 写 UDAF），引擎跑 Rust 反序列化读。
两端依赖同一个 portable 协议——用法一致：

| 语言 | 库 | portable 序列化 |
|---|---|---|
| Java / Scala | [RoaringBitmap](https://github.com/RoaringBitmap/RoaringBitmap) | `serialize(DataOutput)` |
| Rust | `roaring` | `serialize_into` / `deserialize_from` |
| Go | `github.com/RoaringBitmap/roaring` | `WriteTo` / `ReadFrom` |
| C / C++ | CRoaring | `roaring_bitmap_portable_serialize` |

### Scala/Spark 端：UDAF 写入

```scala
import org.apache.spark.sql.expressions.Aggregator
import org.apache.spark.sql.{Encoder, Encoders}
import org.roaringbitmap.RoaringBitmap

object FastRoaringAggregator extends Aggregator[Int, RoaringBitmap, Array[Byte]] {
  override def zero: RoaringBitmap = new RoaringBitmap()
  override def reduce(buf: RoaringBitmap, idx: Int): RoaringBitmap = { buf.add(idx); buf }
  override def merge(b1: RoaringBitmap, b2: RoaringBitmap): RoaringBitmap = { b1.or(b2); b1 }
  override def finish(buf: RoaringBitmap): Array[Byte] = {
    buf.runOptimize()                          // ← 务必先压缩
    val baos = new java.io.ByteArrayOutputStream(buf.serializedSizeInBytes)
    buf.serialize(new java.io.DataOutputStream(baos))
    baos.toByteArray
  }
  override def bufferEncoder: Encoder[RoaringBitmap] = Encoders.kryo[RoaringBitmap]
  override def outputEncoder: Encoder[Array[Byte]]  = Encoders.BINARY
}
```

> 关键：BUF 用 Kryo `Encoders.kryo[RoaringBitmap]`，不要每次 reduce 都
> deserialize/serialize——naive 实现 11 分钟，Kryo 优化 35 秒。

### Rust 端：流式反序列化

```rust
use roaring::RoaringBitmap;

let bin: &[u8] = row.get_bytes("bin")?;        // parquet 字段
let mut bm = RoaringBitmap::deserialize_from(bin)?;
bm.run_optimize();                              // ← 反序列化后再压一次
```

两端的 bytes 二进制兼容——**不用任何 schema 协商**。

### 三条死规矩

1. **永远 `runOptimize` 一次再序列化**——否则读端解析路径分支可能踩稀疏度边界 bug
2. **C 实现额外有 frozen / native 格式**——跨语言<em>绝对要走 portable</em>，不要用 frozen
3. **每次升级 jar / crate 跑一个 round-trip 测试**——Rust 写 → Scala 反序列化 → 再写 → Rust 反序列化，bitmap 必须 bit-for-bit 等同

## 暂态开销：bitmap 在<em>构建 / 反序列化 / drop</em>时的瞬时峰值

bitmap 稳态紧凑——但<em>暂态</em>开销和稳态完全不在一个数量级。
我们栽进去过两次，都是<em>暂态</em>问题，不是<em>稳态</em>问题。

### 事故 A · ArcSwap 切换瞬间的"看不见的 GC"

**症状**：每分钟 builder `store()` 之后约 50–120 ms 内，
0.3% 请求 p99 从 25 ms 飙到 50–80 ms。Prometheus 上每分钟一个尖刺，
时间点和 builder 节奏完全对齐。

**误判路径**：第一反应"ArcSwap stop-the-world"——查源码根本没有，
`store` 是一次原子指针 swap + 旧 Arc 计数 dec。

**根因**：旧 `Arc<TargetingIndex>` 在最后一个 reader 释放时计数到 0 → **触发 drop**。
TargetingIndex 内部是一大堆 HashMap + RoaringBitmap，bitmap 内部又是几万个小 chunk。
**释放路径走了几万次 `free()`**——jemalloc 拿内部锁，碰巧打到的请求线程被卡 30–80 ms。

**修复**：把 drop 让到专门的后台 worker：

```rust
pub struct TargetingService {
    index: ArcSwap<TargetingIndex>,
    drop_tx: mpsc::UnboundedSender<Arc<TargetingIndex>>,
}

impl TargetingService {
    pub fn replace(&self, new_idx: Arc<TargetingIndex>) {
        let old = self.index.swap(new_idx);
        let _ = self.drop_tx.send(old);          // 扔给后台 worker
    }
}

async fn drop_worker(mut rx: mpsc::UnboundedReceiver<Arc<TargetingIndex>>) {
    while let Some(arc) = rx.recv().await {
        tokio::time::sleep(Duration::from_millis(100)).await; // 让 reader 自然释放
        drop(arc);                                // 大对象释放在专用线程
    }
}
```

修复后每分钟尖刺消失，p99 整体回到 22 ms 平稳。

> 教训：**大对象的 `drop()` 永远是隐形的 stop-the-world**——
> 任何 ArcSwap 模式都该把这件事关进后台。

### 事故 B · bitmap 反序列化峰值 OOM

**症状**：引擎冷启动从对象存储拉昨天的 index 时容器 OOMKilled，
但 final index 只有 1.2 GB（容器 limit 4 GB）。

**根因**：

- 反序列化期间，每个 RoaringBitmap 的临时 buffer + 最终结构同时存活——**峰值 2.3× final**
- 8 维 + 8 _excl + 8 unconstrained = **24 张大表**同时反序列化
- jemalloc 不会立即 unmap 临时 buffer

**修复**：串行反序列化 + 立即 `run_optimize` + 立即释放原始字节

```rust
for dim in DIMENSIONS {
    let raw = object_store.get(&format!("idx/{dim}.parquet")).await?;
    let mut bm_map = HashMap::new();
    let reader = ParquetRecordReader::try_new(Cursor::new(&raw))?;
    for row in reader {
        let token = row.get_typed("token")?;
        let bin: &[u8] = row.get_bytes("bin")?;
        let mut bm = RoaringBitmap::deserialize_from(bin)?;
        bm.run_optimize();                       // ← 立即压缩
        bm_map.insert(token, bm);
    }
    drop(raw);                                    // ← 释放原始字节
    builder.set_dim(dim, bm_map);
}
```

容器 memory limit 提到 6 GB 留 1.5× headroom。Cold start 峰值降到 ~3.8 GB，正常运行 1.2 GB。

> 教训：**反序列化的瞬时内存峰值要单独 budget**——
> 不能拿稳态体积当容器 limit 的依据。

### 这两个事故的共同点

它们看起来一个是 drop、一个是反序列化，本质<strong>同一件事</strong>——
RoaringBitmap 这种"超紧凑数据结构"的<em>暂态</em>开销，
和它的<em>稳态</em>开销不在一个数量级。

工程上只看 benchmark 平均数会骗自己。**真正会出事的是 p99 和 startup**——
后台 drop worker、流式反序列化、容器 limit 留 1.5–2× headroom，三件都得做。

## 容量规划：1× → 10× 业务的真实账

工程上最容易低估的是「同一套架构能撑多少业务」。
RoaringBitmap 给了我们一个**线性而非组合爆炸**的扩展曲线——但<em>线性也有斜率</em>。

下面是基于 100 万 plan 真实数字外推到 1000 万 plan 的估算：

### 内存

| 项 | 100 万 plan | 1000 万 plan | 倍数 |
|---|---:|---:|---:|
| 索引常驻 | 1.2 GB | ~10.6 GB | ~9× |
| 冷启动反序列化峰值（×2.3）| 2.8 GB | ~24 GB | — |

bitmap 倒排不严格 10×——RoaringBitmap 在密度变高时<em>压缩率反而提升</em>，10× plan 数下倒排只膨胀 ~9×。
但 cold start 反序列化峰值是<em>真问题</em>——容器 limit 必须从 4 GB 提到 32 GB。

### 查询路径（每次 BidRequest）

```text
                       100 万 plan       1000 万 plan       倍数
段 1 (bitmap match) p99   280 µs              ~600 µs        2.1×
段 2 (Redis pipeline) p99 1.4 ms              ~3 ms          2.1×
合计 p99                  1.85 ms             ~3.8 ms        ~2×
```

**10× plan 数只带来 ~2× 查询延迟**——bitmap 集合代数的复利效应。

### 月度账单

```
                100 万 plan       1000 万 plan       倍数
引擎单实例规格    c6i.2xlarge       r6i.4xlarge       ~3× 单价
引擎实例数        16                16-24             1-1.5×
Spark builder     32 cores          64 cores          2×
S3 跨 region 出   $0                $6,600/天         ∞
月度总成本        $X                ~$3.5X            **3.5×**
```

**10× 业务规模 = ~3.5× 成本**——这是 RoaringBitmap 给我们的<em>规模经济</em>。
hash + nested loop 的实现 10× 业务等于 ~10× CPU + ~10× 内存。

### 容量预警的三个早期信号

把容量规划从"季度规划会"改成"日常监控告警"：

1. **单维度 token 数突破 50 万**——HashMap lookup 进入 hot path，要考虑分片
2. **cold start 反序列化峰值占容器 limit > 80%**——下次副本扩容就会崩
3. **builder 单次构建 > 60 s**——跨过分钟节奏，规则更新滞后

## 一致性 4 层防御：让漏数据被引擎主动察觉

builder 直接挂掉好处理（`index_age_seconds` 涨上去秒级告警）。
真正难处理的是 builder<em>看起来在跑、但产出的 index 缺了一些 plan</em>——
症状是<em>某些 plan 长时间不出价</em>，但运营找过来时已过去几小时。

我们栽进去过两次：

- **case A**：Spark `flatMap` 阶段某节点 OOM 但没失败重试，1.2% plan 缺席
- **case B**：Kafka schema-registry 升级后降级到旧 schema，新加字段静默丢失

后来我们建了 4 层防御：

### 第 1 层 · manifest 签名校验

builder 写 manifest 时把<em>每维统计</em>都签上：

```json
{
  "version": "2026-04-28T14:32:00Z",
  "plan_count": 1042573,
  "dim_stats": {
    "geo":      { "tokens": 18247,  "total_postings": 8240122 },
    "audience": { "tokens": 187302, "total_postings": 22300018 }
  },
  "checksum_sha256": "..."
}
```

引擎拿到 index 后<strong>把数字算一遍</strong>对比 manifest——不一致拒绝 swap。
抓什么：builder 自报和实际产出不一致。

### 第 2 层 · 与上一版 diff（趋势）

```rust
// 单次更新里 plan 数变化 > 5% 几乎一定异常
let ratio = (added + removed) as f64 / prev.plan_count() as f64;
ensure!(ratio <= 0.05, "suspicious churn: {:.2}%", ratio * 100.0);

// 单维 token 变化 > 20% 也警告
for dim in DIMENSIONS {
    let p = prev.dim_token_count(dim);
    let n = next.dim_token_count(dim);
    ensure!((p as f64 - n as f64).abs() / p as f64 <= 0.20,
            "{dim}: {} -> {}", p, n);
}
```

抓什么：builder 自报数字也错的<em>系统性偏差</em>。case A 在这层抓住。

### 第 3 层 · canary 流量回放

50 条人工挑选的代表性 fixture，每次 swap 前都跑：

```rust
const CANARY: &[CanaryCase] = &[
    CanaryCase { name: "cn_ios_news",   ctx: ..., min: 80, max: 200 },
    CanaryCase { name: "us_android_dr", ctx: ..., min: 30, max: 90  },
    // ... 48 more
];

for case in CANARY {
    let n = match_plans(idx, &case.ctx).len();
    ensure!(n >= case.min && n <= case.max,
            "canary {} hit count {} out of [{}, {}]",
            case.name, n, case.min, case.max);
}
```

> **canary case 必须独立写**——不能从 builder 输出反推。它们是<em>外部不可变的合同</em>。

抓什么：单一维度数据漏（其他维正常）。case B 在这层抓住——
"美国 Android" 的 imp hit 数从 65 跌到 12（Geo 维度数据丢了）。

### 第 4 层 · 生产 1% shadow 双计算

最严密但最贵——**生产流量在新旧 index 上各跑一遍**，diff worker 异步对比：

```rust
let hit_main = match_plans(&svc.index.load(), &ctx);

if should_sample() {                         // 1% 流量打开 shadow
    if let Some(prev) = svc.previous_index.load_full() {
        let hit_shadow = match_plans(&prev, &ctx);
        diff_tx.try_send(DiffEvent {
            ctx_hash: ctx.hash(),
            main_hit: hit_main.iter().collect(),
            shadow_hit: hit_shadow.iter().collect(),
        }).ok();
    }
}
```

抓什么：极少数 plan 漏（< 0.1%），上面 3 层都抓不住的小尺度问题。

### 4 层防御的累积效果

| 漏数据形态 | L1 签名 | L2 diff | L3 canary | L4 shadow |
|---|:---:|:---:|:---:|:---:|
| builder 输出明显错 | ✓ | ✓ | ✓ | ✓ |
| builder 自报数字也错 | ✗ | ✓ | ✓ | ✓ |
| 单维度数据漏 | ✗ | △ | ✓ | ✓ |
| 极少数 plan 漏（< 0.1%）| ✗ | ✗ | △ | ✓ |

**单层都有盲区，4 层叠加几乎不可能全漏**。
按业务规模递进上：< 100 万 plan 用 L1 已足够，> 500 万再加 L4。

> 工程哲学：**引擎不该被动等运营反馈，而是主动验证自己拿到的 index**。
> 任何一层不通过，引擎<strong>拒绝</strong> swap，旧 index 继续服务，告警让人介入。
> 这种设计叫 _fail-stop with degraded continuity_——出错时停止前进而不是盲目接受。

## 多版本共存：IndexSlots 把切换/回滚/canary 收成一套机制

单 ArcSwap 装索引能跑——但工程上需要<strong>多个版本同时活着</strong>，理由有四：

1. **drop worker 延迟**——旧版要再多活一会儿等 reader 自然释放（前面事故 A 的修复）
2. **shadow 双计算**——跑 1% 流量在<em>上一版</em>上对比 hit
3. **canary release**——新版先承接 5% 流量，30 分钟无异常再 100%
4. **回滚**——新版上线发现告警，立即切回上一版（不重新拉旧 index）

如果每个用途各自维护一份"上一版引用"，代码会乱成一锅粥。
我们做了一个统一的<em>四槽 index 管理器</em>：

```rust
pub struct IndexSlots {
    primary:   ArcSwap<TargetingIndex>,                       // 99% 流量
    previous:  ArcSwap<Option<Arc<TargetingIndex>>>,           // shadow + 回滚
    canary:    ArcSwap<Option<Arc<TargetingIndex>>>,           // 灰度
    drop_tx:   mpsc::UnboundedSender<Arc<TargetingIndex>>,    // 后台 worker
}
```

每个槽<strong>唯一</strong>用途，不复用。

### promote 流程：从 canary 到 primary

builder 推一份新 index 时，**不是直接覆盖 primary**，先进 canary：

```rust
impl IndexSlots {
    pub fn deploy_canary(&self, new: Arc<TargetingIndex>) {
        self.canary.store(Some(new));
    }

    pub fn promote(&self) -> Result<()> {
        let canary = self.canary.load_full()
            .ok_or(anyhow!("no canary to promote"))?;
        let old_primary = self.primary.swap(canary);
        let old_previous = self.previous.swap(Some(old_primary));
        if let Some(too_old) = old_previous {
            let _ = self.drop_tx.send(too_old);
        }
        self.canary.store(None);
        Ok(())
    }

    pub fn abort_canary(&self) {
        if let Some(c) = self.canary.swap(None) {
            let _ = self.drop_tx.send(c);
        }
    }
}
```

整个生命周期：

```text
   builder 推送
        ▼
    canary slot ─────────► (5min → 1% → 5% → 25% → 30min 观测)
                                │
                  ┌─────────────┴────────────┐
                  ▼                          ▼
            promote()                  abort_canary()
                  │                          │
                  ▼                          ▼
            primary ← canary             canary 进 drop queue
            previous ← old primary       （primary 不动）
```

### 流量分配：按 ctx hash 稳定打到 canary

```rust
pub fn match_plans(&self, ctx: &ImpContext) -> RoaringBitmap {
    let canary_pct = self.canary_pct.load(Ordering::Relaxed);
    let bucket = ctx.stable_hash() % 100;          // ctx hash 稳定，重试落同槽
    let use_canary = bucket < canary_pct
        && self.slots.canary.load().is_some();

    let idx = if use_canary {
        self.slots.canary.load_full().unwrap()
    } else {
        self.slots.primary.load_full()
    };

    match_plans(&idx, ctx)
}
```

### rollback：< 50 ms 一次原子 swap

```rust
pub fn rollback(&self) -> Result<()> {
    let prev = self.previous.load_full()
        .ok_or(anyhow!("no previous version"))?;
    let bad_primary = self.primary.swap(prev);
    let _ = self.drop_tx.send(bad_primary);
    self.previous.store(None);
    Ok(())
}
```

回滚是 **一次原子 swap**——没有重新构索引、没有 S3 拉数据、没有反序列化。
事故应对从「几分钟重建」压到「几十毫秒切换」——这是多版本共存最值钱的一处。

### 内存账：极端情况 ~5 GB

每个 `Arc<TargetingIndex>` 是 1.2 GB。三槽全满：

| 状态 | 内存占用 |
|---|---|
| 稳态（仅 primary）| 1.2 GB |
| 部署中（primary + canary）| 2.4 GB |
| promote 后短暂（primary + previous）| 2.4 GB |
| 极端（三槽满 + drop queue 排队）| **~5 GB** |

容器 limit 必须按<em>极端情况</em>设——4 GB 实例容不下，r6i.4xlarge 32 GB 才稳。

> 不是所有项目都需要四槽。**适合**：业务<em>不能停服</em>、index 出错代价高于多花点内存。
> **不适合**：内部工具、流量小、偶尔停服可接受——单 ArcSwap 已够用。

## 一段话收尾：三条工程哲学

整个文章想传递的不是 bitmap 知识，是<em>用什么样的工程姿态对待数据结构</em>。三条提炼：

### 哲学 1 · 暂态开销和稳态开销不在一个数量级

bitmap 稳态紧凑 + 快——但<em>构建、反序列化、drop</em>的瞬时成本可能是稳态的 2-3 倍。
**永远要为暂态留预算**——后台 drop worker、流式反序列化、容器 limit 留 1.5–2× headroom。

### 哲学 2 · "能跑" 比 "完美" 重要

builder 漏数据时引擎不该崩，应该<em>用旧版继续跑</em>+告警。
fail-stop with degraded continuity——保住可用性比保住时效性优先。

### 哲学 3 · 数据结构选对了，工程问题大半就解决了

把"百万 plan 实时匹配"拆成"集合代数"问题——
就有 Roaring 这个现成的数据结构能用。
**很多看似工程难题的，本质是<em>数据结构选错了</em>导致的复杂度**——
退一步重新审视数据形态，往往能省下半年的工程力气。

---

RoaringBitmap 不是「魔法」——它是把"高频读 + 低频写 + 集合运算"做到极致的数据结构。
当业务恰好长这个形状时，它能让你在<em>不加机器</em>的前提下把规则量再涨 10 倍；
当业务不长这个形状（写读比 1:1、或者只有几百条规则），它就是个炫技的麻烦。

工程不是用最酷的技术——是用<em>最适合当下问题的最熟悉的技术</em>。

---

<small>
本篇是 ADX 设计系列的<strong>实战篇第 1 篇</strong>，承接[第 4 篇定向上下文](/posts/targeting-context/)的实现细节。
更多实战记录后续会更新到
<a href="/series/adx-design/">/series/adx-design/</a>。
</small>
