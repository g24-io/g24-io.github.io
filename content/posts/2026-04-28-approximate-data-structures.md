+++
title = "用近似换 1000 倍内存：HyperLogLog、Bloom Filter、CountMinSketch 在 ADX 里的真实用法"
description = "三件 RoaringBitmap 的「近亲」——把「绝对精确」的成本和「足够好」的代价摆到桌面上。"
date = 2026-04-28
slug = "approximate-data-structures"

[taxonomies]
tags = ["hyperloglog", "bloom-filter", "countminsketch", "rust", "adx", "data-structure", "production"]
series = ["adx-design"]

[extra]
author = "g24-io engineering"
tldr = [
  "**HyperLogLog**：12 KB 表示任意基数的独立用户数，用于频次帽——内存 1300× 压缩。",
  "**Bloom Filter**：85 MB 装 5000 万 IVT 黑名单——99.9% 走快路径、0.1% 兜底 Redis。",
  "**CountMinSketch**：4 KB 估算热门 publisher Top-K——比 HashMap 省 4000× 内存。",
  "三件 + RoaringBitmap 在 ADX 主链路上**协作而非替代**——精确路径与近似路径各司其职。",
  "**业务方对「误差」的接受度是谈出来的**——给他算「精确版每月多 ¥20 万」，多半就同意 1% 误差。",
]
+++

## RoaringBitmap 的近亲

[上一篇](/posts/roaringbitmap-in-production/)讲了 RoaringBitmap——
一个把<em>精确</em>集合代数做到极致的数据结构。

但很多业务问题<em>本身就不需要精确</em>。

- 频次帽 ±5% 误差对业务无感
- 黑名单查询允许 0.1% 假阳性（兜底 Redis 即可）
- Top-K 排名用估算就够，业务不在意"第 47 名比第 49 名"的精确差

强行精确 = 烧机器。
**承认"近似"，能省 100–1000 倍的内存**。这是这篇文章的全部主题。

下面三件：HyperLogLog、Bloom Filter、CountMinSketch——
都属于 _probabilistic data structure_（概率型数据结构）。
它们和 RoaringBitmap 是<em>同一类思想</em>的不同变体——

> 用<em>近似</em>换<em>体积</em>，用<em>体积</em>换<em>速度</em>。

广告系统的「高频写、海量数据、实时查」三角矛盾——
靠它们一并解决。

### 全文一张速查表

| 数据结构 | 回答的问题 | 误差类型 | 内存 |
|---|---|---|---:|
| **RoaringBitmap** | "X 是否在集合里？""集合大小？""集合交并？" | 0%（精确）| 1.2 KB / 万元素 |
| **HyperLogLog** | "去重计数大概是多少？" | ±0.5–2% | **12 KB / 任意基数** |
| **Bloom Filter** | "X<em>可能</em>在集合里吗？（不在则一定不在）" | 假阳性 0.1–1% | 1.2 KB / 万元素 |
| **CountMinSketch** | "X 的频度大概是多少？" | 高估（不会低估）| **几 KB / 流** |

下面三节分别讲它们在我们 ADX 里被用在哪。

## HyperLogLog · 频次帽的「近似但够用」

### 业务问题

「这个广告活动每天对每个用户最多展示 3 次。」——典型频次帽。
要在 BidRequest 进来时回答：<em>这个 plan 今天已经向当前用户展示过几次？</em>

精确做法：

```text
Redis: HINCRBY freq:plan{42}:date{20260428} user_hash{...} 1
```

每个 (plan, user, date) 一行。账：

- 100 万活动 × 1 亿日活 × 31 天保留 = ~3 万亿条记录
- 每行 hash entry ~30 byte → **理论占用 90 TB**
- 现实里只活跃 plan × 活跃 user 才存——但仍至少 **每天 16 GB 集群内存**

### 业务方真的需要"精确到 3"吗

「频次帽 3 次」的业务语义是：<em>不要让一个用户连续看到太多次同一活动</em>。
误差 ±5% 在这个语义下完全无感——
"用户被这个活动打了大约 3 次"和"打了正好 3 次"业务效果一致。

**这一步是与产品经理谈出来的**——
不是工程单方面"决定降级精度"。

### HyperLogLog 怎么工作

直觉：哈希一个元素，看哈希值<em>开头有几个 0</em>。
开头有 5 个 0 的元素——表示哈希空间里大约 1/32 的概率事件——
意味着集合里大约有 32 个不同元素。

实际算法用 **m = 2^k 个独立的哈希桶**，
每个桶记录"看到的最大 leading-zeros 数"，
最后把 m 个桶的值用调和平均合一——误差 ≈ 1.04 / √m。

| 桶数 m | 内存 | 误差 |
|---:|---:|---:|
| 1024 | 1 KB | 3.25% |
| 4096 | 4 KB | 1.6% |
| **16384** | **12 KB** | **0.81%** |
| 65536 | 64 KB | 0.40% |

我们项目用 m = 16384——12 KB 表示<em>任意基数</em>的独立用户数。
百万级 × 亿级集合也是 12 KB。

### Rust API

```rust
use hyperloglog::HyperLogLog;

let mut hll = HyperLogLog::new(0.01);    // 1% 误差 → m ≈ 16384

for user_hash in users {
    hll.insert(&user_hash);
}

let unique_users = hll.count();           // 估算独立用户数
```

合并两个 HLL：

```rust
let mut a = HyperLogLog::new(0.01);
let mut b = HyperLogLog::new(0.01);
a.merge(&b);                              // 取每个桶的 max
// 这是 HLL 的杀手锏——分布式场景下各自独立计数，最后合并
```

> **HLL 合并是 idempotent 的**——`a.merge(b).merge(b) == a.merge(b)`，
> 重复合并不影响结果。这让分布式去重变得简单。

### Redis 内置就有 HLL

Redis 自带 `PFADD` / `PFCOUNT` / `PFMERGE`：

```text
PFADD   freq:plan{42}:date{20260428} user_hash_001 user_hash_002 ...
PFCOUNT freq:plan{42}:date{20260428}    # 估算独立用户数
PFMERGE merged freq:plan{42}:date{20260428} freq:plan{42}:date{20260427}
```

Redis 的 HLL 实现稍优于学术原版（叫 HLL++），固定每个 key 12 KB。
**100 万活动 × 31 天 = 3100 万 key × 12 KB ≈ 370 GB**——
比精确版（90 TB）省约 **240×**。

### 我们的双轨方案

```text
              ┌── 严格频次帽（用户级精确度）
              │   单 plan + 单天，HLL 12 KB / 精度 ±1%
频次维度 ────┤
              └── 粗频次桶（业务大盘）
                  plan_group × hour-bucket，HLL 4 KB / 精度 ±5%
```

严格的留给"3 次帽"硬阈值，粗的给报表大盘。
两轨各自占资源，互不影响。

### 一个跨语言的坑

HLL 的二进制格式 **没有跨语言标准**——
Redis 的 HLL 格式、Java `stream-lib`、Rust `hyperloglog`、Python `pyhll`——
四家**两两不兼容**。

我们的做法：

- **跨语言传 HLL** → 不传二进制，传<em>每桶值的 JSON 数组</em>，两端各自 fold
- **同语言内** → 用 portable bytes 直接传

这套约定 hard-code 在 shared kernel crate。代价是 JSON 解析慢一点，
收益是<em>跨语言不再有版本兼容陷阱</em>。

> 教训：probabilistic 数据结构的"标准化"远不如 RoaringBitmap 成熟——
> 跨语言要么自定义 wire format，要么在同语言内闭环。

## Bloom Filter · IVT 黑名单前置过滤

### 业务问题

第三方 IVT 反作弊厂商每天给我们一份恶意 device_id 清单——**5000 万条左右**。
每个 BidRequest 进来要查：「该 device 是否在黑名单」。

精确做法：`HashSet<DeviceId>`

- 5000 万 × 32 byte（DeviceId UUID + hash overhead）= **1.6 GB**
- 16 个引擎实例 × 1.6 GB = **每集群 26 GB**——只为了查一下黑名单

### Bloom Filter 怎么工作

一个 m-bit 的位数组 + k 个独立哈希函数。

- **写入** `add(x)`：算 k 个哈希，把对应的 k 个 bit 置 1
- **查询** `contains(x)`：算 k 个哈希，<em>所有</em>对应 bit 都是 1 → "可能在"；任一为 0 → "一定不在"

关键性质：

- **假阴性 0**——它说"不在"就一定不在
- **假阳性可控**——n 个元素插入 m bits 用 k 个哈希时：

```text
P(false-positive) ≈ (1 - e^(-kn/m))^k
```

最优 k = (m/n) · ln 2 时假阳性率最低。
工程上不算公式，用反推：**n 个元素 + 期望误差 ε → m ≈ -n · ln(ε) / (ln 2)²**。

| n | ε | m (bits) | 内存 | k (最优) |
|---:|---:|---:|---:|---:|
| 5,000 万 | 1% | 480 Mbit | 60 MB | 7 |
| 5,000 万 | **0.1%** | **720 Mbit** | **86 MB** | 10 |
| 5,000 万 | 0.01% | 960 Mbit | 115 MB | 13 |

5000 万 IVT 黑名单 + 0.1% 假阳性 ≈ **每实例只需 ~86 MB**——比 HashSet 省 19×。

### Rust API

```rust
use bloomfilter::Bloom;

// 容量 5000 万、目标假阳性 0.1%
let mut bloom = Bloom::new_for_fp_rate(50_000_000, 0.001);

for bad_device in feed.iter() {
    bloom.set(bad_device);
}

// 查询：99.9% 一次内存读就答
if !bloom.check(&req.device_id) {
    // 一定不在黑名单 → fast path
} else {
    // 可能在 → 兜底查 Redis 精确确认
}
```

### 双层防御：bloom 快路径 + Redis 慢路径

```rust
async fn is_blacklisted(
    bloom: &Bloom<DeviceId>,
    redis: &RedisClient,
    device: &DeviceId,
) -> bool {
    if !bloom.check(device) {
        return false;                    // 99.9% 走这里，纳秒级
    }
    // 0.1% 假阳性兜底——Redis SISMEMBER 精确查
    redis.sismember("ivt:blacklist", device).await.unwrap_or(false)
}
```

平均延迟 = 99.9% × 50ns + 0.1% × 1ms ≈ **51ns 平均、1ms p999**。
完全在 RTB 5ms 预算内吃得起。

> 假阳性<em>不是 bug</em>——它是<strong>设计选择</strong>。
> 假阳性的代价是"多一次 Redis 校验"，不是"误判"。
> 业务安全性靠 Redis 兜底保证，bloom 只负责<em>放过 99.9%</em>的快路径。

### 哪些坑要躲

#### 坑 1 · Bloom 不支持删除

Bloom 写过的 bit 一直是 1。如果 IVT 厂商更新了清单（移除了一些 device），
**没有任何方法把这些 device 从 bloom 里去掉**——
某 bit 可能被 N 个其他 device 共享。

**解法**：Bloom 不变更——每天直接 rebuild 一份新 bloom 整体替换。
ArcSwap 装着，写路径完全是 builder 的事，引擎只读。

#### 坑 2 · Counting Bloom 能删但贵

`Counting Bloom Filter` 把每个 bit 改成 4-bit counter——支持 `+1` / `-1`。
代价：**内存 4×**。除非真的需要在线删，否则不值得。

#### 坑 3 · Cuckoo Filter 是更好的现代替代品

Cuckoo Filter 与 Bloom 接近的内存占用 + 真正的删除支持 + 假阳性率随负载上升更平稳：

- **支持 delete**——每个槽存元素的指纹（fingerprint）而非 bit
- **lookup p99 略快**——k 次哈希降到 2 次
- **库的成熟度比 Bloom 弱**——选 Bloom 仍是最稳妥

我们目前**用 Bloom**——主要因为生态：Rust `bloomfilter` / Redis `BF.*`（RedisBloom 模块）/ Java `Guava BloomFilter` 都很稳定。
将来如果黑名单需要在线删（业务方需求），切到 Cuckoo Filter。

#### 坑 4 · 容量上限要监控

n 超过设计容量，假阳性率会**飙升**。
工程上：
- 设计容量按"最坏情况 1.2×"预留
- Prometheus 监控当前已插入元素数 `bloom_set_count_total / bloom_capacity`
- 超过 80% 触发告警，让 builder 重建一份更大的

### 在 ADX 主链路上的位置

```text
BidRequest 进来
     │
     ▼
┌─ Bloom Filter ─┐
│  device 黑名单 │  ← 99.9% 走快路径
│  bundle 黑名单 │
│  IP 黑名单     │
└────────┬───────┘
         │ 0.1% 假阳性
         ▼
   Redis SISMEMBER 兜底
         │ 真在黑名单
         ▼
     拒绝出价（NoBidCause::ContentBlocked）
```

我们部署了 3 个独立 Bloom：`device_blacklist` / `bundle_blacklist` / `ip_blacklist`，
合计内存 **~250 MB / 实例**——比 HashSet 实现省 ~5 GB。

> **Bloom 的工程价值：把 99.9% 的查询消化在内存读 + 几次哈希里**，
> 不打 Redis、不查 SQL、不跨进程——RTB 路径上每一纳秒都值得这种节省。

## CountMinSketch · Top-K 频度估算

### 业务问题

实时大盘要展示「过去 5 分钟出价 / 中标最频繁的 publisher Top-100」。
精确做法：`HashMap<PublisherId, u64>` 每出价 +1。

- publisher 数 ~200 万 → 16 MB hash 看起来还行
- 但要维护**滚动窗口**——每分钟一份 hash，6 份窗口共 96 MB
- 而且单 hash 的 contention 在百万级 QPS 下堪忧

### CountMinSketch 怎么工作

数据结构是 **d × w 二维数组**——d 个独立哈希函数 + 每个 hash 一行 w 列计数器。

写入 `add(x, 1)`：

```text
对每个 hash 函数 h_i：
    counters[i][h_i(x) mod w] += 1
```

查询 `estimate(x)`：

```text
return min(counters[i][h_i(x) mod w] for i in 0..d)
```

取所有行的<em>最小值</em>。这保证了估算<strong>永不低估</strong>——
某行可能因为哈希碰撞被同名计数污染从而高估，
但 d 行同时被同一组其他元素污染的概率极低 → 取 min 排掉污染。

### 误差公式

```text
误差 ε ≤ e/w   概率 ≥ 1 - 1/2^d
```

工程上选参数：

| w | d | 内存 (32-bit) | 误差上界 | 概率 |
|---:|---:|---:|---:|---:|
| 1024 | 4 | **4 KB** | ε ≈ 0.3% | 93.75% |
| 2048 | 5 | 10 KB | 0.13% | 96.875% |
| 4096 | 6 | 24 KB | 0.07% | 98.4% |

我们用 w=1024, d=4——**4 KB 描述任意 cardinality 的频度分布**。
单分钟一个 CMS、保留 6 份、每分钟 drop 最旧的——总内存 24 KB
（vs HashMap 96 MB）→ **省 4000×**。

### Rust API

```rust
use count_min_sketch::CountMinSketch32;

// width=1024, depth=4
let mut cms = CountMinSketch32::new(1024, 4);

for ev in stream {
    cms.increment(&ev.publisher_id);
}

// 查询
let estimated = cms.estimate(&publisher_id);  // 高估，永不低估
```

### Top-K 配套：CMS + Heap

CMS 给单个元素的<em>频度</em>，但不直接给 Top-K——
需要外配一个 **min-heap (size=K)** 维护当前最高的 K 个：

```rust
use std::collections::BinaryHeap;

struct TopK {
    cms: CountMinSketch32<PublisherId>,
    heap: BinaryHeap<(Reverse<u32>, PublisherId)>,  // 最小堆
    seen: HashSet<PublisherId>,
    k: usize,
}

impl TopK {
    fn add(&mut self, item: PublisherId) {
        self.cms.increment(&item);
        let est = self.cms.estimate(&item);

        if !self.seen.contains(&item) {
            if self.heap.len() < self.k {
                self.heap.push((Reverse(est), item.clone()));
                self.seen.insert(item);
            } else if est > self.heap.peek().unwrap().0.0 {
                let (_, old) = self.heap.pop().unwrap();
                self.seen.remove(&old);
                self.heap.push((Reverse(est), item.clone()));
                self.seen.insert(item);
            }
        }
    }
}
```

CMS 占 4 KB，heap 占 K × 16 byte ≈ 1.6 KB（K=100）——
**整套 Top-100 < 6 KB**。

### CMS 的真正强项：相对排名稳定

CMS 估算<em>偏高</em>，但**所有元素一起被偏高**——相对顺序基本不变。
对 Top-K 这种排名查询无关紧要。

> 业务方关心"哪个 publisher 第一"远多于"它的精确次数是 482731 还是 482795"。
> CMS 让<em>排名稳</em>而<em>绝对值随意</em>——这正是 Top-K 的需求。

### 一个特别注意的坑

**低频元素估算**会被显著高估——因为污染来自所有元素，对小数字影响更大。
例：

- 真实出现 1 次的元素，CMS 可能估算成 100 次
- 真实出现 1000 次的元素，CMS 估算成 1050 次（基本准确）

工程上：CMS **只用于 Top-K**——大数字稳，小数字不准但不影响 Top-K。
**不要用 CMS 直接当"是否出现过"判断**——那是 Bloom 的活。

## 四件套的组合拳：ADX 主链路上的实际位置

```text
┌──────────────────────────────────────────────────────────────┐
│  BidRequest 进入                                              │
│       │                                                      │
│       ├─ Bloom Filter: 黑名单查询    → 99.9% 直接放行         │
│       │                                                      │
│       ▼                                                      │
│  RoaringBitmap: 8 维倒排匹配         → plan_id 集合           │
│       │                                                      │
│       ▼                                                      │
│  Redis pipeline + HyperLogLog: 频次帽 → 状态校验             │
│       │                                                      │
│       ▼                                                      │
│  最终 plan 进入拍卖 → DSP fanout → 出价                      │
│       │                                                      │
│       └─ CountMinSketch 异步: 实时 publisher Top-K 大盘       │
└──────────────────────────────────────────────────────────────┘
```

四件各司其职：

| 数据结构 | 路径位置 | 职责 | 精度要求 |
|---|---|---|---|
| **Bloom Filter** | 入口 | 前置过滤"绝大概率不在" | 0% 假阴性，可容假阳性 |
| **RoaringBitmap** | 主路径 | 核心规则匹配 | **精确** |
| **HyperLogLog** | Redis 状态 | 频次帽近似 | ±1-2% 业务可接受 |
| **CountMinSketch** | 旁路 | 实时排名统计 | 高估容许 |

> 这是这套架构的<em>关键洞察</em>——**精确路径与近似路径并存**，<strong>互补而非替代</strong>。
> 把"哪些查询需要精确、哪些可以近似"想清楚——
> 整个系统能省掉 80-90% 的内存和延迟。

## 一段话收尾：选对工具的两个判断

如果你的业务也面临"高频写、海量数据、实时查"的三角矛盾，
看到这里应该已经能本能地判断每个查询该用哪个工具：

### 判断 1 · 精确还是近似？

- 法律 / 合规 / 财务对账 → **精确**（RoaringBitmap / HashMap）
- 频次帽 / Top-K / 黑名单查询 → **近似**（HLL / CMS / Bloom）

### 判断 2 · 假阴性可容忍吗？

- 黑名单查询：**绝不容忍假阴性**（漏过坏请求）→ Bloom（保证假阴性 0）
- 频次帽：**可容忍小幅高估**（多挡几次）→ HLL / CMS

把这两条想清楚，就能在合适的位置嵌入合适的工具。

> 工程不是用最酷的技术——是<em>把每个数据点的"精度需求"和"成本"摆到桌面上谈</em>。
> 业务方对误差的接受度，往往比工程师以为的高。
> 你的工作是<strong>给他算账</strong>，让他<em>知情地</em>选近似。

---

<small>
本篇是 ADX 设计系列的<strong>实战篇第 2 篇</strong>，与
<a href="/posts/roaringbitmap-in-production/">实战篇第 1 篇 (RoaringBitmap)</a>
互补阅读。
</small>
