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
  "**状态机驱动每一条不变式**：Initialized→Filtering→Open→Collecting→Sealed→Decided/NoBid/TimedOut/Failed，状态转换是不变式的"执法时机"。",
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

<!-- 后续：三层结构 / 状态机 / 22 条清单 / NoBidReason / 收尾 见下一轮 -->
