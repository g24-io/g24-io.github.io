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

<!-- 后续：22 条清单 / NoBidReason / 收尾 见下一轮 -->
