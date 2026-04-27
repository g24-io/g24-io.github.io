+++
title = "竞价上下文：22 条不变式如何长出来"
description = "Auction、ImpressionBidding、Bid 三层聚合不是一开始就长这样的——它们是被一条又一条「不能让它发生」逐步压出来的。"
date = 2026-04-26
updated = 2026-04-27
slug = "bidding-context"

[taxonomies]
tags = ["bidding", "ddd", "rust", "openrtb", "state-machine"]
series = ["adx-design"]

[extra]
author = "g24-io engineering"
tldr = [
  "**Auction** 是聚合根，一次 SSP 请求映射一个 Auction；多 imp 用 ImpressionBidding 子实体承担。",
  "**不变式分两层**：跨 Imp 的（请求级时间窗、币种统一、tmax 上限）与 Imp 内的（出价 ≥ 底价、币种一致、bid_id 唯一）。",
  "**状态机驱动每一条不变式**：Initialized→Filtering→Open→Collecting→Sealed→Decided/NoBid/TimedOut/Failed，状态转换是不变式的「执法时机」。",
  "**NoBidReason 的两层枚举**：聚合内 7 类 ⊂ 应用层 18+ 类，前者保护建模简洁，后者撑漏斗报表的 n→m→m1。",
]
+++

## 从一段实时竞价的代码说起

最初的版本里，"一次 SSP 请求"和"一次出价"是混在一起的——
一个 `BidContext` 结构体扛起了上下文、出价、计价、回执所有职责。
在 imp 数量还小、SSP 还少的时候，这跑得很快。

但当我们开始接入第二个 SSP，并且需要支持_一个请求里多个 imp 各自独立竞价_时，
所有不变式都开始打架——
哪些状态属于"整个请求"、哪些属于"这个 imp"、底价是请求级还是 imp 级、
timeout 计时从什么时候开始算……每一个看似简单的问题，都在原结构里没有清晰答案。

于是我们停下来，在白板上把**聚合根 / 实体 / 值对象**重新切了一次。
切完之后，22 条原本散落在评论里的"不能让它发生"被收进一个清单——
本文展开讲它们如何彼此咬合、如何被状态机驱动、以及为什么不能再少一条。

> 在我们看来，**不变式不是约束，是设计的副产品**。
> 一个聚合的边界画对了，不变式自然就只能落在这条边界上；
> 反过来，如果你发现一条不变式跨越了两个聚合，那一定是边界画错了。

## 三层结构：Auction · ImpressionBidding · Bid

重切之后，整个上下文只有**一个聚合根**—— `Auction`。
它对应"一次 SSP 的 Bid Request"，承担跨 imp 的不变式（币种统一、tmax 一致、整体超时）。
每一个 imp 用 `ImpressionBidding` 子实体表示，
子实体之间不互相引用，所有跨 imp 的协调都从根上发出。

```rust
pub struct Auction {
    pub id: AuctionId,
    pub ssp_id: SspId,
    pub bid_request_id: BidRequestId,
    pub received_at: DateTime<Utc>,
    pub tmax_ms: u32,
    pub currency: Currency,
    pub state: AuctionState,
    /// SmallVec：单 imp 命中 inline，多 imp 退化为 heap
    pub impressions: SmallVec<[ImpressionBidding; 4]>,
}
```

`SmallVec<[T; 4]>` 是个工程取舍：实测中 _≥99% 的请求只有 1 个 imp_，
但协议允许多 imp，我们不能因此去堆分配——4 槽 inline 既能命中绝大多数路径，
又给极端情况（视频前贴片串播多 slot）留了退路。

### 子实体 ImpressionBidding 与值对象 Bid

`ImpressionBidding` 持有该 imp 的_底价_、_本 imp 状态_和收到的所有 `Bid`。
`Bid` 是值对象——它一旦诞生就不变，
所有"修订"其实都是产生一个新的 `Bid` 替换旧的，旧的进入审计流。

```rust
pub struct ImpressionBidding {
    pub imp_id: ImpId,
    pub floor: Money,
    pub state: ImpState,
    pub bids: Vec<Bid>,
    pub winning_bid: Option<BidId>,
}

#[derive(Debug, Clone)]
pub struct Bid {
    pub id: BidId,
    pub dsp_id: DspId,
    pub price: Money,
    pub creative_id: CreativeId,
    pub received_at: DateTime<Utc>,
}
```

> 一个判断聚合切对了的小测试：**能否只读根，就回答关于这次请求的所有合规问题？**
> 币种统一吗？tmax 多少？哪些 imp 已经定标？——全部从 `Auction` 根直接读出。
> 不需要 join、不需要旁路缓存。

## 状态机驱动不变式

把状态机放在聚合根上之后，**每一条不变式都有了一个明确的执法时机**——
要么在进入某状态时检查（前置条件），要么在状态内每次写操作时检查（持久条件），
要么在状态转出时检查（出口条件）。这条规矩让代码里再也不会出现
`if state == X && cond_y` 这种散落在各处的"巧合校验"。

```text
  Initialized
       │  start()
       ▼
  Filtering  ──reject──►  Failed
       │  open()
       ▼
  Open
       │  collect_bid()
       ▼
  Collecting  ──tick > tmax──►  TimedOut
       │  seal()
       ▼
  Sealed
       │  decide()
       ▼
  Decided     (or NoBid if no bid clears floor)
```

这 9 个状态可以分成三组：

- **过渡态**（`Initialized · Filtering · Open · Collecting · Sealed`）：在写路径上向前推进，每一步都有 invariant。
- **终态成功线**（`Decided · NoBid`）：竞价完成且对外可观测；不变式从"写"转向"只读 + 审计"。
- **终态异常线**（`TimedOut · Failed`）：fail-safe，对应漏斗里的两类损失，统计上必须可分。

### 哪些不变式在哪个状态执法

以下是几条代表性不变式与其执法点的对照——完整 22 条见下一节。

| #   | 不变式（节选） | 执法状态 | 执法时机 |
|-----|---|---|---|
| I3  | 请求级 `currency` 一旦确定不可变更 | `Initialized → Filtering` | 进入时 |
| I7  | 每个 `Bid.price` 必须 ≥ 该 imp 的 `floor` | `Collecting` | 每次 `collect_bid()` |
| I12 | elapsed 超 `tmax_ms` 时聚合必须可被 `tick` 进入 `TimedOut` | 除终态外的任意状态 | 每次 tick |
| I18 | 同一 `BidId` 在一次 Auction 内全局唯一 | `Open · Collecting` | 每次 `collect_bid()` |
| I22 | `tmax_ms` 不允许超过协议上限（当前 1000ms）| `Initialized` | 构造时 |

注意 **I12 是横切的**——它不绑定单一状态，而是"除终态外都成立"。
在代码里我们用一个统一的 `maybe_time_out(now)` 方法承载它，
每个状态内的写操作前都先调用一次。这样的好处：超时不会因为某个状态忘了写检查而漏掉。

## 22 条不变式清单

把上面散落的规则收拢，按_「这条规则保护谁」_分成 5 组。
完整 22 条排在下面——长，但每一条都有意义。

### A · 请求级 / 跨 imp（5）

- **I01** 一次 SSP Bid Request 映射唯一一个 `Auction`，不复用、不合并。
- **I02** Auction 受理后 `ssp_id` / `bid_request_id` 不可变更。
- **I03** Auction 的 `currency` 一旦确定不可变更，整次竞价币种统一（业界一般 USD）。
- **I04** `impressions` 至少 1 个 ImpressionBidding，`imp_id` 不允许重复。
- **I05** `received_at` 单调，所有内部时间戳基线由它派生（用于漏斗时间轴）。

### B · Imp 级（5）

- **I06** `floor` 必须为正（货币最小单位 ≥ 1）。
- **I07** 收入 `bids` 的每个 `Bid.price` 必须 ≥ 该 imp 的 `floor`。
- **I08** `winning_bid` 仅在 Decided 状态可设，且必须指向 `bids` 内已存在的 `Bid.id`。
- **I09** ImpressionBidding 状态必须与 Auction 状态对齐——不存在"Auction Sealed 但 imp 仍 Open"。
- **I10** 同一 imp 的所有 `Bid` 价格币种与 `Auction.currency` 一致（不允许混币）。

### C · Bid 级（5）

- **I11** `Bid` 是值对象，进入 `bids` 列表后不允许就地修改（替换型才允许）。
- **I12** `Bid.id` 在该次 Auction 内全局唯一（跨 imp 也唯一）。
- **I13** `Bid.dsp_id` 必须存在于 Auction 创建时通告的候选 DSP 集合。
- **I14** `Bid.creative_id` 必须存在且在该 `dsp_id` 名下可用、未被风控冻结。
- **I15** `Bid.received_at` 必须晚于该 Auction 的 `filter_done_at`。

### D · 时间窗与 tmax（4）

- **I16** `tmax_ms` ∈ [50, 1000]——下限保护 DSP 思考时间，上限挡 SSP 越界请求。
- **I17** elapsed > `tmax_ms` 时聚合必须能 tick 到 `TimedOut`，无论当前非终态是哪个。
- **I18** 进入 `Sealed` 之后任何 `collect_bid` 都必须失败，不写入 `bids`。
- **I19** 进入终态后所有写操作幂等失败（返回 NoOp，不修改聚合，不发新事件）。

### E · 终态与对外（3）

- **I20** `NoBid` 与 `Decided` 在同一 imp 上互斥；Auction 整体可以"部分 Decided + 部分 NoBid"。
- **I21** 终态确定后 `nbr_code` 必须落在已声明的两层枚举内（聚合内 7 类 ⊂ 应用层 18+ 类）。
- **I22** 任意终态发出的事件必须携带完整 funnel 标签（Layer 0–12 + DSP×Imp 子漏斗 B1–B8）。

### 三个判断「这条该不该是不变式」的快测

1. **违反它会让对外语义出错吗？**（不只是数据不一致——是真的会让 SSP 收到错的回执、漏斗算错、分账不对）
2. **它能在聚合根内被本地检查吗？**（如果需要看其他聚合或外部服务，那它不属于这里——属于一致性流程而非不变式）
3. **它能在某个具体的状态/操作被列出执法时机吗？**（如果列不出来，说明它太抽象，需要先拆）

22 条全过这三关。

## NoBidReason 的两层枚举

我们走过的弯路：第一版 `NoBidReason` 只有 4 个枚举值，
看起来很干净——直到运营同事来要漏斗：「_这个 SSP 这小时丢了 30% 流量，到底丢在哪一步？_」。
4 个值答不出。把它扩到 18 个又破坏了聚合的简洁——
DDD 一直强调聚合内的语言要稳定，不能为了报表把根撑爆。

最后落到**两层枚举**：聚合内只看 7 类粗粒度原因（语义稳定），
应用层在序列化为事件时_注入_更细的 18+ 类细分原因——
通过一个 `impl From<NoBidCause> for NoBidReason` 的多对一映射保证一致。

### 聚合内：NoBidReason

```rust
/// 聚合内可见的「为什么没出价」——粗粒度，语义稳定。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum NoBidReason {
    /// 没有任何 DSP 给出 Bid（含未召回、超时、显式 NoBid）。
    NoBids,
    /// 收到 Bid 但全部低于该 imp 的 floor。
    AllBelowFloor,
    /// 整个 Auction 超时，已收 Bid 不再开标。
    AuctionTimedOut,
    /// 收到的 Bid 全部因校验失败被剔除（创意/风控/币种…）。
    AllInvalid,
    /// 进入 Filtering 后整请求被拒（合规、黑名单、ICP 等）。
    FilteredOut,
    /// 协议解析或 schema 不通过。
    ProtocolError,
    /// 内部异常（不应对外细化原因）。
    InternalError,
}
```

### 应用层：NoBidCause

```rust
/// 应用层可见的细分原因——给漏斗报表用，可演进、可加项。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum NoBidCause {
    // L0–L4 入口段
    SspNotIntegrated, ProtocolError, SchemaInvalid,
    ContentBlocked, PolicyRejected,
    // L5–L7 召回段
    NoMatchingDsp, AllDspsTimedOut, AllDspsNoBid,
    // L8–L10 撮合段
    AllBidsBelowFloor, AuctionTimedOut, ShadingFailed,
    // L11–L12 出口段
    PostFilterRemoved, ResponseEmitFailed,
    // 维护类
    InternalError, DownstreamUnavailable,
    RateLimited, CircuitBroken, Unknown,
}

impl From<NoBidCause> for NoBidReason {
    fn from(c: NoBidCause) -> Self {
        use NoBidCause::*;
        match c {
            SspNotIntegrated | ProtocolError | SchemaInvalid
                => NoBidReason::ProtocolError,
            ContentBlocked | PolicyRejected | PostFilterRemoved
                => NoBidReason::FilteredOut,
            NoMatchingDsp | AllDspsTimedOut | AllDspsNoBid
                => NoBidReason::NoBids,
            AllBidsBelowFloor | ShadingFailed
                => NoBidReason::AllBelowFloor,
            AuctionTimedOut
                => NoBidReason::AuctionTimedOut,
            _ => NoBidReason::InternalError,
        }
    }
}
```

### 这两层在漏斗里的作用

漏斗的精髓在于_每一层都能算出转化率_，且每一层失败的请求都能归因到某个 `NoBidCause`。
于是当运营问"这小时这个 SSP 丢了多少"，我们能精确指着_第 7 层 AllDspsTimedOut_说：
"DSP-X 整体抖了 4 倍 latency"。

```text
  Layer 0  receive request          ██████████████████████████████  100%   N=812k
  Layer 2  schema valid             █████████████████████████████    98%
  Layer 4  policy passed            ████████████████████████████     95%
  Layer 5  matched dsp candidates   ██████████████████████████       88%   ← n
  Layer 7  dsp returned bid         ██████████████████████           75%   ← m
  Layer 8  bid above floor          █████████████████████            71%
  Layer 10 won & shaded             ██████████████████               62%
  Layer 12 response emitted ok      ████████████████                 54%   ← m1
                                    ▲
                                    └── 8 percentage points lost here
```

n→m→m1 三个箭头是产品最常问的三个数：

- **n** 是"我们准备分发给多少个 DSP"
- **m** 是"最终有多少 DSP 真给了 Bid"
- **m1** 是"最终回执给 SSP 时还剩多少"

两层枚举的存在让 m1/n 的_每一个百分点_都能溯源到某条 NoBidCause。

## 小结：边界画对了，规则就长出来

回头看这次重构留下的几个判断标准：

1. **聚合切对了的标志**——只读根能回答关于该业务事实的所有合规问题。
2. **不变式不是约束，是设计的副产品**——它们必须能在某个状态/操作上找到执法时机，否则就是错误地分类。
3. **状态机是执法时机的载体**——把规则绑在状态转换上，而不是散落在条件判断里。
4. **对外暴露的细节单独建模**——NoBidReason 与 NoBidCause 的两层枚举，是"聚合稳定"与"报表演进"之间最干净的解。

下一篇我们走到**定价上下文**——把"价格"从竞价里彻底拎出来，
让底价、Bid Shading、利润策略各自独立演进，
再用一份 `PriceBook` 把它们的快照按 Auction 时间冻结下来。
竞价上下文之所以能这么干净，部分原因正是它_不再操心_价格怎么算。

---

<small>
本篇是 ADX 设计系列第 2 篇，系列归档于
<a href="/series/adx-design/">/series/adx-design/</a>。
原始内部讨论文档保留在
<a href="https://github.com/g24-io/conversations-archive">g24-io/conversations-archive</a>。
</small>
