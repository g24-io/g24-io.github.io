+++
title = "4 个核心域如何在一个进程里协同：集成总图"
description = "9 步同步主流程 + 6 条异步旁路——以及为什么我们坚持同进程而不是微服务化。"
date = 2026-04-27
slug = "integration-overview"

[taxonomies]
tags = ["integration", "ddd", "rust", "architecture", "performance"]
series = ["adx-design"]

[extra]
author = "g24-io engineering"
tldr = [
  "**4 个核心域跑在同一进程里**——不是因为不会拆，是因为 5ms 的延迟预算容不下 RPC。",
  "**主流程 9 步同步**：parse → filter → targeting → pricebook → fanout → collect → seal → decide → respond。",
  "**6 条异步旁路**：win notice / loss notice / impression / click / conversion / 配置下发。",
  "**SLO**：可用性 99.95%，端到端 p99 < 100ms，引擎内部 p99 < 30ms。",
  "**域间通信靠值对象传递**——没有 trait object dyn dispatch、没有 channel 排队、没有 mutex 等待。",
]
+++

## 集成的核心命题：在 5ms 内完成 4 个核心域的协作

前 4 篇分别讲了 Bidding / Pricing / Targeting / Filtering——
每个都拥有自己的聚合根、自己的不变式、自己的演进节奏。
这一篇要回答的是<strong>它们如何在一次实时竞价里彼此协作</strong>。

我们当时面临的设计抉择有两类：

- **微服务方案**：每个核心域自成进程，跨进程 gRPC——边界最干净，单核心可独立扩缩
- **同进程方案**：4 个核心域作为同一 binary 内的 crate，调用走函数——边界靠 crate 拓扑约束

工程上我们选了后者。理由很硬性：

> 内部预算 30ms。一次 gRPC（含序列化 + 网络往返 + 反序列化）现实开销 1.5–4ms，
> 4 个核心域之间最少 3 次跨进程调用——光通信开销就吃掉<em>三分之一</em>。
> 这笔账无论怎么算都不划算。

DDD 的边界<em>不要求</em>物理拆分。<strong>同进程也能 DDD</strong>——
靠的是 crate workspace、严格的依赖方向、值对象传递、不变 IO trait。
拆物理只在两种条件下做：(a) 单核心域真的做不下来需独立扩缩，
(b) 团队组织规模大到跨域协作成本超过 RPC 成本。我们暂不满足这两条。

## 主流程：9 步同步走完

一次 BidRequest 进来到回执出去，引擎内部走 9 步——每一步在哪个核心域、产生哪些副作用、
哪条异步事件被发出，都有明确归属。

```text
   ┌─ interface-rtb ─┐
   │   1. parse       │  ← OpenRTB 解码 + schema 校验（外层防腐）
   └────────┬─────────┘
            │ ParsedBidRequest
            ▼
   ┌─ filtering ─────┐
   │   2. filter      │  ← FilterChain 跑 7 类匹配 + DV/IAS 预查询
   └────────┬─────────┘
            │ FilteredRequest（含 ComplianceVerdict）
            ▼
   ┌─ targeting ─────┐
   │   3. match       │  ← TargetingIndex 倒排 + RoaringBitmap 求交
   └────────┬─────────┘
            │ CandidatePlanSet（位图 → Vec<PlanId>）
            ▼
   ┌─ pricing ───────┐
   │   4. pricebook   │  ← compute_reserve_price + 4 纯函数预备
   └────────┬─────────┘
            │ AuctionInputs（含 PriceBook 引用）
            ▼
   ┌─ bidding ───────┐
   │   5. start       │  ← Auction::start()：Initialized → Open
   └────────┬─────────┘
            │
            ▼
   ┌─ infra-dsp ─────┐
   │   6. fanout      │  ← FuturesUnordered 扇出 N 个 DSP，含 timeout
   └────────┬─────────┘
            │ DspBidStream（先到先得 + 超时剪枝）
            ▼
   ┌─ bidding ───────┐
   │   7. collect     │  ← Auction::collect_bid() 边收边校验 22 不变式
   └────────┬─────────┘
            │
            ▼
   ┌─ bidding ───────┐
   │   8. seal+decide │  ← Sealed → Decided/NoBid，winning_bid 落定
   └────────┬─────────┘
            │ AuctionOutcome
            ▼
   ┌─ interface-rtb ─┐
   │   9. respond     │  ← 构造 BidResponse + 出向 SSP
   └──────────────────┘
```

### 几个关键约束

**步骤 4 的 PriceBook 是引用而非拷贝**——`AuctionInputs` 持有 `Arc<PriceBook>`，
在 ArcSwap load 时就已经拿到。<em>整个 Auction 生命周期内 PriceBook 不变</em>，
保证步骤 7 / 8 用同一份规则做合规判断与最终撮合。

**步骤 6 的 fanout 是唯一异步点**——所有 DSP 调用并行发出，靠 `FuturesUnordered`
+ `tokio::time::timeout(tmax_ms)` 控制总时长。<em>超时即剪枝</em>，已收 Bid 进入步骤 7，
未收到的 DSP 进入 NoBidCause::AllDspsTimedOut 漏斗层。

**步骤 7 与 8 的不变式执法在聚合内**——状态机驱动，参考[竞价上下文](/posts/bidding-context/)。

**步骤 9 不再修改聚合**——它只读 `AuctionOutcome` 构造响应。
任何 SSP 协议特殊化（OpenRTB 2.5 vs 2.6、字段命名差异）都在 `interface-rtb` 这层处理，
不污染核心域。

### 域间通信只走值对象

每一步之间传递的<em>不是 trait object，不是 channel message，是值对象</em>——
`ParsedBidRequest` / `FilteredRequest` / `CandidatePlanSet` / `AuctionInputs` / `AuctionOutcome`。
它们都是 `Send + Sync` 的纯数据，函数调用时按引用或所有权传递。

```rust
pub async fn handle_bid_request(
    raw: RawBidRequest,
) -> Result<BidResponse, AdxError> {
    let parsed   = interface_rtb::parse(raw)?;                    // 1
    let filtered = filtering::run(&parsed, &filter_idx).await?;   // 2
    let plans    = targeting::match_plans(&filtered, &tgt_idx);   // 3
    let inputs   = pricing::build_inputs(&filtered, &plans, &book); // 4
    let auction  = bidding::start(inputs)?;                       // 5
    let bids     = dsp_routing::fanout(&auction).await;           // 6
    let auction  = bidding::collect(auction, bids)?;              // 7
    let outcome  = bidding::seal_and_decide(auction)?;            // 8
    interface_rtb::build_response(outcome)                        // 9
}
```

> 这段函数签名是<strong>整个引擎的灵魂</strong>。
> 每一步纯函数式，错误向上传播，无 mutex 无 channel——
> 这正是同进程能在 30ms 内跑完的根本原因。

## 6 条异步旁路：主流程之外的事

主流程必须在 30ms 内跑完——这意味着任何<strong>不必等结果的事</strong>都得撤出主路径。
我们识别出 6 条异步旁路：

| # | 旁路 | 触发点 | 节奏 | 终点 |
|---|---|---|---|---|
| 1 | **win notice 接收** | DSP 推送 `wnur` | 实时秒级 | Billing 入账 + Kafka |
| 2 | **loss notice 派发** | 步骤 8 后 | 实时秒级 | DSP 端点（best-effort） |
| 3 | **impression 回调** | 浏览器/SDK 上报 | 准实时 | Tracking 入 Kafka |
| 4 | **click 回调** | 浏览器/SDK 上报 | 准实时 | Tracking 入 Kafka |
| 5 | **conversion 归因** | 第三方监测推回 | 分钟级 | 大数据归因服务 |
| 6 | **配置下发** | 运营后台 → 引擎 | 1–60 秒 | 引擎热加载 ArcSwap |

### 旁路的 3 条共性原则

每条旁路都在<em>不阻塞主流程</em>的前提下满足业务——靠的是三条工程纪律：

1. **写入 Kafka 是<strong>异步非阻塞</strong>的**——主流程发出领域事件后立即返回，
   `rdkafka` Producer 在后台 batch + flush。
2. **失败靠<strong>重试 + DLQ</strong>而非<strong>报错</strong>**——
   一条 impression 推送失败不能影响下一次竞价，所以入 dead-letter topic 等离线处理。
3. **配置下发用<strong>整体替换</strong>而非<strong>字段更新</strong>**——
   ArcSwap 切快照那一刻，引擎读到的是<em>一致的全量</em>，不会出现"半新半旧"中间态。

> 旁路里发生什么，主流程都不该感知到。
> 这条原则一旦破了——比如「计费写失败要回滚 Auction」——整个延迟预算就失控了。

## 部署拓扑：4 个核心域 + 1 个 binary

```text
┌─────────────────────────────────────────────────┐
│                 adx-engine binary                │
│  ┌──────────────────────────────────────────┐   │
│  │  interface-rtb       ← OpenRTB 入口      │   │
│  │  ─────────────────────────────────────   │   │
│  │  app-bidding         ← Use Case 编排     │   │
│  │  ─────────────────────────────────────   │   │
│  │  domain-bidding      ┐                   │   │
│  │  domain-pricing      │  4 个核心域       │   │
│  │  domain-targeting    │  纯 Rust，零 IO   │   │
│  │  domain-filtering    ┘                   │   │
│  │  ─────────────────────────────────────   │   │
│  │  infra-cache (ArcSwap snapshots)         │   │
│  │  infra-dsp-client (hyper pool)           │   │
│  │  infra-kafka (rdkafka)                   │   │
│  │  infra-storage (sqlx / redis)            │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
                  │           │
       ┌──────────┘           └────────────┐
       ▼                                   ▼
   ┌─────────┐                       ┌─────────┐
   │  Kafka  │ ← 事件总线             │  Redis  │ ← 频次/Pacing/预算
   └─────────┘                       └─────────┘
       │                                   │
   ┌───┴────────┐                          │
   ▼            ▼                          │
adx-tracker   adx-billing  ← 独立 binary，对外抗压隔离
```

`adx-engine` 是主竞价 binary，<em>4 个核心域都在它的进程内</em>。
`adx-tracker`（处理曝光/点击）和 `adx-billing`（处理 win notice / 对账）是<strong>独立 binary</strong>——
它们不参与主竞价、不进 30ms 预算，崩了也不影响实时出价。

部署上：

- `adx-engine`：8–32 实例，各自独立 50K QPS / 节点，集群总容量 1M+ QPS
- `adx-tracker`：4–8 实例，按曝光峰值横向伸缩
- `adx-billing`：2 实例热备，状态全在 Redis Cluster + 异步落 MySQL

## SLO：守住护栏

| 指标 | 目标 | 当前 |
|---|---|---|
| 端到端可用性 | 99.95%（每月最多 22 分钟不可用）| 99.97% |
| 端到端 p99（SSP 视角）| < 100ms | 84ms |
| 引擎内部 p99（不含 DSP 等待）| < 30ms | 22ms |
| 单实例 QPS | 50K | 测试 58K，生产稳定 42K |
| 内存常驻配置 | < 4GB / 实例 | 2.8GB |
| 错误预算消耗（月）| ≤ 100% | 38% |

每个数字都是<strong>每次回归测试要守住的护栏</strong>——
任何一项跌出阈值都会触发 release rollback 而非"先上再优化"。

## 小结：让边界继续是边界

回到最开始那个问题：4 个核心域怎么协作。
答案是<strong>同进程 + 值对象传递 + 严格的 crate 拓扑</strong>。

- **同进程**：不是因为不会拆，是因为 5ms 容不下 RPC。
- **值对象传递**：每一步之间是纯数据流，不是 channel 排队，不是 mutex 等待。
- **crate 拓扑**：`domain-*` 禁止依赖 tokio/sqlx，让 IO 永远关在 infra 层。
- **异步旁路**：6 条次要事都撤出主路径，写 Kafka 失败靠重试 + DLQ 而不是阻塞。
- **SLO 是合同**：每个数字是回归护栏，跌出就 rollback。

> 整个系列读下来，你会发现没有任何一处用了<em>「最佳实践」</em>这个词——
> 我们只讲<strong>"我们当时面对什么、为什么这样选、代价是什么"</strong>。
> 这就是 g24-io-tech 的写作纪律。

写到这里 ADX 设计系列首批 6 篇全部交付完成。
后续我们会基于这套架构继续写实战记录——
比如<em>第 100 万 QPS 那天我们调了什么</em>、<em>第一次模型上线踩了哪些坑</em>。
欢迎在 GitHub 上和我们聊。

---

<small>
本篇是 ADX 设计系列第 6 篇 / 收官篇，系列归档于
<a href="/series/adx-design/">/series/adx-design/</a>。
原始内部讨论文档保留在
<a href="https://github.com/g24-io/conversations-archive">g24-io/conversations-archive</a>。
</small>
