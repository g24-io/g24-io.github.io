+++
title = "定价上下文：底价、Bid Shading 与利润策略的解耦"
description = "把价格从竞价里彻底拎出来——四个独立写聚合 + 一份 PriceBook 读视图，让定价规则的演进不被竞价上下文牵着走。"
date = 2026-04-27
slug = "pricing-context"

[taxonomies]
tags = ["pricing", "ddd", "rust", "bid-shading", "cqrs"]
series = ["adx-design"]

[extra]
author = "g24-io engineering"
tldr = [
  "**4 个独立写聚合**：FloorRule（底价）/ ShadingModel（出价削减）/ MarginPolicy（利润）/ RevenueShareContract（分账）。",
  "**一份 PriceBook 读视图**：把 4 个聚合在 Auction 时刻的快照冻结，用于幂等计价。",
  "**4 个纯函数**：compute_reserve_price / apply_shading / apply_margin / compute_payout——可单独单元测试，无 IO。",
  "**这套切分让 Bidding 上下文不必关心价格怎么算**——它只问 PriceBook 要"这个 imp 此刻的最终底价是多少"。",
]
+++

## 价格不是一件事

第一版里，"价格"在代码里只有一个名字——`final_price`。
它由 `BidContext::settle()` 里的一段几十行算出，
里面塞着底价取整、Bid Shading 削减、margin 加价、分账拆分四件事。

业务上，这四件事的演进节奏完全不同：

- **底价**——业务/媒体侧每周都在调，要支持"按时段"和"按设备"的差异
- **Bid Shading**——算法侧迭代，需要 A/B 实验
- **Margin**——财务侧定，季度调一次
- **分账**——合同级，按 SSP / Publisher 协议固定

让这四件事共用一段代码，意味着<em>任何一件事的修改都要考虑另外三件</em>。
两个月内，这段代码就成了团队最不敢动的地方。

> 我们的判断：**这不是一个上下文里的"几个步骤"——是 4 个独立上下文，被错误地塞在了一起。**

于是有了这次拆分。

## 4 个独立写聚合

每件事变成自己的聚合，自己的演进节奏，自己的不变式集合。
它们彼此**互不引用**——共享的只有<em>读视图</em>（下一节讲）。

### 1. FloorRule · 底价规则

底价不是一个数字，是一组规则——按 SSP / Publisher / AdSlot / 时段 / 设备类型分层。

```rust
pub struct FloorRule {
    pub id: FloorRuleId,
    pub scope: FloorScope,           // ssp / publisher / slot / global
    pub conditions: Vec<Condition>,  // daypart / device / geo …
    pub floor: Money,
    pub priority: u16,               // 多规则命中时按优先级取最高底价
    pub effective_at: DateTime<Utc>,
    pub expires_at: Option<DateTime<Utc>>,
}
```

规则演进的不变式（节选）：

- 同一 scope 下命中多条规则时，<em>取最高底价</em>而不是最先匹配——避免"故意造一条小规则绕过主底价"。
- `effective_at` / `expires_at` 必须单调，过期规则只能新增不能改写——保证审计可追溯。
- 优先级冲突时编辑器拒绝保存，必须先调整。

### 2. ShadingModel · Bid Shading 模型

Bid Shading 把 DSP 的出价"削"到接近 SSP 真实底价以上一点点——
对 DSP 来说赢率提高，对 SSP 来说成交价更接近真实意愿。

```rust
pub struct ShadingModel {
    pub id: ShadingModelId,
    pub dsp_id: DspId,
    pub strategy: ShadingStrategy,   // linear / quantile / ml-blob
    pub params: ShadingParams,       // 系数 / 分位数表 / 模型 ref
    pub trained_at: DateTime<Utc>,
    pub experiment_tag: Option<ExperimentTag>,
}
```

不变式：

- `experiment_tag` 不空时，模型仅作用于该 tag 命中的流量——A/B 实验隔离写在聚合里。
- ML blob 引用走对象存储，**模型文件指纹必须存在 PriceBook 里**——上线后才能溯源到具体权重。
- 同一 DSP 同一时刻只能有一个生产 ShadingModel + N 个实验 ShadingModel。

### 3. MarginPolicy · 利润策略

ADX 自身的利润率——不是按 SSP 也不是按 DSP，而是按**业务线**分。

```rust
pub struct MarginPolicy {
    pub id: MarginPolicyId,
    pub line_of_business: BizLine,   // brand / performance / dr / video
    pub margin_pct: Decimal,         // 0.10 = 10%
    pub min_margin: Money,           // 兜底绝对值
    pub effective_at: DateTime<Utc>,
}
```

`margin_pct` + `min_margin` 双轨——百分比和绝对值取较高的。
财务季度调整一次，几乎不参与算法实验。所以它单独成为聚合，不被 Shading 的 A/B 拖累。

### 4. RevenueShareContract · 分账合同

跟 SSP 或 Publisher 签的真实合同——成交后怎么把钱分回去。

```rust
pub struct RevenueShareContract {
    pub id: ContractId,
    pub counterparty: Counterparty,  // SSP::X / Publisher::Y
    pub formula: ShareFormula,       // 固定比例 / 阶梯 / 保底+分成
    pub currency: Currency,
    pub valid_from: DateTime<Utc>,
    pub valid_to: Option<DateTime<Utc>>,
    pub signed_doc_ref: DocRef,      // 法务存档链接
}
```

合同级数据最少改但最严肃——**所有改动都触发法务签字 + 财务复核**，
聚合层面用一条不变式守住：`signed_doc_ref` 不为空才允许 `effective_at` 落地。

### 这 4 个聚合互不引用

注意——**没有任何一个聚合的字段是另一个聚合的引用**。
要把它们关联起来，靠的是<em>读视图</em>，下一节讲。

## PriceBook · 把 4 个写聚合冻成一份读视图

竞价上下文要的不是"最新规则"——是<em>这一次 Auction 受理那一刻所适用的规则快照</em>。
两次相邻的 Auction，如果中间运营改了底价，
两份计价结果<strong>必须可以独立复算</strong>——这就是审计的基本要求。

我们用 `PriceBook` 承担"快照 + 索引"。

```rust
pub struct PriceBook {
    /// 这份 PriceBook 对应的 Auction 受理时刻；快照基准。
    pub as_of: DateTime<Utc>,

    /// 已展开的底价：按 (ssp, publisher, slot, daypart, device) 索引。
    pub floor_index: FloorIndex,

    /// DSP → 当前生效的 Shading 配置（含模型文件指纹）。
    pub shading_index: HashMap<DspId, ShadingSnapshot>,

    /// 业务线 → MarginPolicy 快照。
    pub margin_index: HashMap<BizLine, MarginSnapshot>,

    /// 对手方 → 分账合同快照。
    pub revshare_index: HashMap<Counterparty, RevShareSnapshot>,

    /// 引用追踪——这份 PriceBook 用了哪些聚合的哪个版本。
    pub source_versions: SourceVersionSet,
}
```

`source_versions` 是审计的关键——它记录"这次定价是基于
FloorRule#42 v7 + ShadingModel#mlA v2026-04-26-1500 + …"。
半年后查"这次成交为什么这个价"，靠它一查就能复现整个上下文。

### PriceBook 怎么生成

不是每次 Auction 都重新算——成本太高。我们走<strong>双轨</strong>：

- **基础 PriceBook**：每分钟由 `pricebook-builder` worker 预生成 + 推送给引擎，作为热基线
- **Auction 时增量**：进 Bidding 时，引擎把 `as_of` 与基础 PriceBook 比对，
  只对<em>过期那部分索引</em>做增量补算

> 这是 CQRS 的味道——4 个写聚合管演进，PriceBook 管查询。
> 写读分开后，写聚合不必关心查询性能，读视图也不必参与编辑校验。

## 4 个纯函数：让计算可单测、可重放

定价的<strong>逻辑</strong>抽到 4 个纯函数里——它们不接 IO，
只吃 PriceBook + 当前请求的事实，吐结果。

```rust
/// 根据 (ssp, publisher, slot, daypart, device) 算出该 imp 的最终底价。
pub fn compute_reserve_price(
    book: &PriceBook,
    ctx: &ImpContext,
) -> ReservePrice;

/// 把 DSP 的原始出价削成"接近 floor 但不破"的影子价格。
pub fn apply_shading(
    book: &PriceBook,
    bid: &RawBid,
    floor: &ReservePrice,
) -> ShadedBid;

/// 在 ADX 自身侧加 margin。
pub fn apply_margin(
    book: &PriceBook,
    line: BizLine,
    raw: Money,
) -> MarginedAmount;

/// 计算最终分账：DSP 实付 / SSP 应得 / 平台留存。
pub fn compute_payout(
    book: &PriceBook,
    contract_key: Counterparty,
    settled_price: Money,
) -> PayoutBreakdown;
```

这 4 个函数的特点都一样：

- **没有 `&mut`**——不修改 PriceBook，不修改入参
- **没有 `async fn`**——不打 IO
- **可重放**——同样的 PriceBook + 同样的输入 = 同样的输出，<em>跨进程跨机器都成立</em>
- **可单测**——构造一份手写 PriceBook 当 fixture，case 写多少都不卡 CI

> 这 4 个函数是定价上下文的<strong>价值密度</strong>所在。
> 一切 IO（读 MySQL、订阅 Kafka、对账落库）都在它们外面，让它们保持纯。

## 30 条不变式：分布在 5 处

定价上下文在切干净后，<em>每条不变式都能找到唯一的归属</em>——
4 个写聚合各自负责一组，PriceBook 视图负责一组跨聚合的<em>组合规则</em>。
完整 30 条不展开，按归属列出每组的<strong>守护对象</strong>：

| 归属 | 数量 | 守护对象 |
|---|---:|---|
| **FloorRule** | 7 | 底价单调性、scope 优先级、生效时间区间、currency 与 scope 一致、规则不可改写已过期项 |
| **ShadingModel** | 6 | 实验 tag 隔离、模型指纹必登记、生产模型唯一、参数边界、训练时间不可超前、A/B 流量割裂 |
| **MarginPolicy** | 4 | margin_pct ∈ [0, 1)、min_margin 与 currency 一致、按业务线唯一生效、effective_at 单调 |
| **RevenueShareContract** | 5 | 法务签章必填、起止时间区间合法、formula 自洽（保底+分成总和合理）、对手方在白名单内、合同到期前 30 天预警 |
| **PriceBook**（组合） | 8 | as_of 单调、source_versions 完备、floor_index 命中率不低于阈值、shading 与 margin 顺序固定（先 shade 后 margin）、payout 总额守恒等 |

> 30 条听起来很多——但当每条都<em>只能在某一个聚合里被违反</em>时，
> 维护成本是线性的。如果它们都散落在原来那段 `BidContext::settle()` 里，
> 维护成本是<strong>组合爆炸</strong>。

## 小结：把 4 件事拆开，让 4 个聚合各自演进

回头看这次重构：

1. **业务节奏不同的逻辑必须切到不同的聚合**——底价、Shading、margin、分账各自独立。
2. **写聚合彼此不引用**——它们只共享一份<em>读视图</em>，写读分离让两边都不必妥协。
3. **PriceBook 是审计的锚点**——`source_versions` 让任何成交都能复现当时的规则全集。
4. **4 个纯函数是价值密度所在**——可单测、可重放、跨机器结果一致；IO 关在外面。
5. **不变式按归属分布**——每条只能在一个聚合内被违反，维护成本线性而非组合爆炸。

下一篇我们走到**定向上下文**——8 维 Criterion 在线匹配怎么打成毫秒级，
为什么 RoaringBitmap + 倒排索引 + ArcSwap 是这件事的标准答案，
以及 DMP 那一堆历史包袱怎么用 ACL 隔离。

---

<small>
本篇是 ADX 设计系列第 3 篇，系列归档于
<a href="/series/adx-design/">/series/adx-design/</a>。
原始内部讨论文档保留在
<a href="https://github.com/g24-io/conversations-archive">g24-io/conversations-archive</a>。
</small>
