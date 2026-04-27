+++
title = "定向上下文：用 RoaringBitmap 把 8 维筛选打成毫秒级"
description = "倒排索引 + 位图运算 + ArcSwap 快照——8 类 Criterion 在线匹配的工程做法，以及 DMP ACL 的多源接入策略。"
date = 2026-04-27
slug = "targeting-context"

[taxonomies]
tags = ["targeting", "ddd", "rust", "roaringbitmap", "arcswap", "performance"]
series = ["adx-design"]

[extra]
author = "g24-io engineering"
tldr = [
  "**TargetingPlan** 是聚合根；8 类 Criterion（Geo / Device / Daypart / Contextual / Audience / Lookalike / Ssp / Publisher）。",
  "**TargetingIndex 是读视图**：每维度一组倒排，召回用 RoaringBitmap 做位图交并差。",
  "**ArcSwap<TargetingIndex>**：运营变更触发整体替换，无锁读、原子切换；每次替换不影响在飞 Auction。",
  "**3 类 Policy（频次 / Pacing / 预算）单独存放**：状态写多读多，不放 ArcSwap，走 Redis Cluster + 分片计数器。",
  "**DMP 通过 ACL 多源接入**：极光 / TalkingData / LiveRamp 三家口径不同，用防腐层把它们映射到统一 AudienceTag。",
]
+++

## 召回三毫秒，定向二毫秒

竞价的内部预算大概是 20–30ms——其中**召回 + 定向匹配**通常分到 5ms 左右。
也就是说在<em>这 5 毫秒内</em>，引擎要回答以下八个问题：

1. 这个 imp 来自的城市，命中了哪些活动？
2. 这台设备的 OS / 制造商 / 屏幕尺寸，命中了哪些活动？
3. 当前时间属于哪些活动的投放时段？
4. 这条媒体内容的 IAB 分类与关键词，命中了哪些活动？
5. 当前用户的受众标签（DMP），命中了哪些活动？
6. Lookalike 模型对这个用户打分，与哪些活动的阈值匹配？
7. 该 SSP 在哪些活动的 SSP 白名单里？
8. 该 Publisher / AdSlot 在哪些活动的发布方白名单里？

每个问题独立都不难——难的是<strong>在 5ms 内并行答完</strong>，且活动量到达<em>百万级</em>。

第一版我们用了 hash 表 + 嵌套 for——单库 10K 活动时跑得动，
到 50K 活动 + 海外多区域全打开时，p99 直接爆到 200ms。

> 解法不是更聪明的 hash，是<em>换数据结构</em>——这就是定向上下文的核心工程命题。

## TargetingPlan 与 8 类 Criterion

聚合根是 `TargetingPlan`——一个广告活动一个 plan。
plan 包含<em>正向</em>和<em>反向</em>两组 Criterion——前者要命中，后者必须不命中。

```rust
pub struct TargetingPlan {
    pub id: TargetingPlanId,
    pub campaign_id: CampaignId,
    pub include: Vec<Criterion>,   // 任一不命中即整 plan 不命中
    pub exclude: Vec<Criterion>,   // 任一命中即整 plan 不命中
    pub state: PlanState,
    pub effective_at: DateTime<Utc>,
    pub expires_at: Option<DateTime<Utc>>,
}

pub enum Criterion {
    Geo(GeoSpec),                  // 国家 / 大区 / 城市 / 圆形围栏
    Device(DeviceSpec),            // OS / 制造商 / 屏幕 / 网络
    Daypart(DaypartSpec),          // 周几 / 时段 / 时区
    Contextual(ContextualSpec),    // IAB 分类 / 关键词 / 内容评级
    Audience(AudienceSpec),        // DMP 标签 ID 集
    Lookalike(LookalikeSpec),      // 模型 ID + 阈值
    Ssp(SspSpec),                  // SSP 白/黑名单
    Publisher(PublisherSpec),      // Publisher / AdSlot 白/黑名单
}
```

关键不变式（节选）：

- 同一 plan 同一 Criterion 类型最多一条——比如不能同时存在两条 `Geo`，
  否则语义模糊（"两条都要命中"还是"任一命中"？编辑器拒绝保存）。
- `Lookalike` 引用的模型 ID 必须存在于 PriceBook 同期 ShadingModel 列表中——
  防止模型下线但 plan 还在引用。
- `effective_at < expires_at`，且 plan 状态为 `Active` 时这两个时间戳已锁定不可改。

> 这 8 类不是一次划全的——是被业务一次次"加一个新的定向维度"逼出来的。
> 但每加一类前必须答清：<em>这条维度能不能用现有 Criterion 表达？</em>——能，就不加新类。

## TargetingIndex · 把 8 类压成一组倒排

匹配的瓶颈不是「一条 Criterion 是否命中」——是「<em>百万条 plan 里哪些命中</em>」。
直接遍历 plan 是 O(n)；倒排索引可以做到 O(候选集大小)，差距能到三个数量级。

```rust
/// 8 维倒排，每维一张 token → bitmap 表。
pub struct TargetingIndex {
    pub geo:        InvertedIndex<GeoToken,        PlanIdSet>,
    pub device:     InvertedIndex<DeviceToken,     PlanIdSet>,
    pub daypart:    InvertedIndex<DaypartBucket,   PlanIdSet>,
    pub contextual: InvertedIndex<IabCategoryId,   PlanIdSet>,
    pub audience:   InvertedIndex<AudienceTagId,   PlanIdSet>,
    pub lookalike:  InvertedIndex<LookalikeModel,  PlanIdSet>,
    pub ssp:        InvertedIndex<SspId,           PlanIdSet>,
    pub publisher:  InvertedIndex<PublisherId,     PlanIdSet>,

    /// PlanId → 完整 TargetingPlan 引用（命中后的取数）。
    pub plans: HashMap<PlanId, Arc<TargetingPlan>>,

    /// 这份 Index 对应哪一批运营变更——审计锚点。
    pub revision: IndexRevision,
}

pub type PlanIdSet = RoaringBitmap;
```

### 一次匹配走 3 步

```rust
pub fn match_plans(
    idx: &TargetingIndex,
    ctx: &ImpContext,
) -> RoaringBitmap {
    // 1. 每维度按 token 取出该维命中的 plan 位图集合
    let geo_hit       = idx.geo.lookup(&ctx.geo_tokens);
    let device_hit    = idx.device.lookup(&ctx.device_tokens);
    let daypart_hit   = idx.daypart.lookup(&[ctx.daypart_bucket()]);
    // … 其余 5 维同理

    // 2. 对 8 张位图求交（include 维度做 AND）
    let mut hit = geo_hit;
    hit &= device_hit;
    hit &= daypart_hit;
    // … 其余维度继续 AND-into

    // 3. 减去 exclude 维度（位图差集）
    hit -= idx.geo_excl.lookup(&ctx.geo_tokens);
    hit -= idx.device_excl.lookup(&ctx.device_tokens);
    // …

    hit
}
```

`RoaringBitmap` 上的 `&=` 与 `-=` 是<em>常数时间级</em>的操作（实测百万 PlanId
求交在 100µs 内）。倒排表里每个 token 的 bitmap 是预压缩的，
求交不会展开成 Vec——一次匹配真正的开销在<em>取倒排</em>，而非<em>做集合运算</em>。

> 这一步是定向上下文的<strong>性能护城河</strong>：
> 数据结构选对了，再多活动也是<em>线性</em>而非<em>组合</em>地变慢。

## ArcSwap 快照：写少读多的最优解

`TargetingIndex` 是<em>读多写少</em>的——每秒被读几十万次，
每分钟被运营改 1–2 次。这种场景的标准答案不是锁，是**快照替换**。

```rust
use arc_swap::ArcSwap;

pub struct TargetingService {
    /// 整张 index 用 ArcSwap 装着——读路径不需要任何锁。
    index: ArcSwap<TargetingIndex>,
}

impl TargetingService {
    /// 读路径：load() 是无锁的，拿到 Arc 之后随便用。
    pub fn match_plans(&self, ctx: &ImpContext) -> RoaringBitmap {
        let idx = self.index.load();
        match_plans(&idx, ctx)
    }

    /// 写路径：把新 index 整个 swap 进来，旧的让 Arc 引用计数自己回收。
    pub fn replace_index(&self, new_idx: Arc<TargetingIndex>) {
        self.index.store(new_idx);
    }
}
```

读路径的开销只有一次原子指针读 + 一次 `Arc::clone`（增计数）。
**没有锁、没有等待、没有重试**——这是 ArcSwap 适合做"配置型读"的根本原因。

写路径不是<em>原地改</em>，而是<em>构建一个完整的新 index 然后整体替换</em>。
听起来很贵，但 1 分钟构一次 vs 每秒读几十万次——这账怎么算都是赚的。

### 增量与全量

`replace_index` 看起来"全量"，但实际上 builder 内部走<em>增量</em>：

- 启动时：从 MySQL 拉全量构出 V1
- 运行期：订阅 Kafka 的 plan 变更事件，<em>差量更新一份内部 mutable 结构</em>，
  到节奏点（默认 60s）打成新 V2 freeze 出 `Arc<TargetingIndex>` 然后 store
- 每天凌晨：兜底再做一次完整 reconcile，对账后再 store

> 别在读路径上做"更新"——任何时候发现想用 `RwLock<TargetingIndex>`，
> 都该停下来问：是不是把数据结构选错了？

## 3 类 Policy：写多读多，单独处理

`TargetingPlan` 之外，还有三类<strong>状态型数据</strong>：频次帽、Pacing 节奏、预算。
它们和 plan 不同——既要被读（每次 Auction 检查），也要被写（每次成交累加）。
这些<strong>不能</strong>放进 `TargetingIndex`——快照模型 + 高频写是矛盾的。

```rust
pub struct FrequencyPolicy {
    pub plan_id: PlanId,
    pub cap_per_user_per_day: u32,
    pub cap_per_user_per_session: u32,
    pub window_strategy: WindowStrategy, // sliding / tumbling
}

pub struct PacingPolicy {
    pub plan_id: PlanId,
    pub mode: PacingMode,                // ASAP / Even / Front-Loaded
    pub target_curve: TargetCurve,       // 24 个 1 小时桶的目标占比
}

pub struct BudgetPolicy {
    pub plan_id: PlanId,
    pub daily_budget: Money,
    pub lifetime_budget: Option<Money>,
    pub overspend_grace_pct: Decimal,    // 允许超出 N% 后再硬阻断
}
```

它们的<em>状态</em>（已花多少 / 已展示多少 / 当前节奏分布）在 Redis Cluster 里——
用<strong>分片计数器</strong>（每核一个 slot，每秒合并一次到 Redis）做高并发累加，
读用 pipeline + Lua 一次拉 N 项。

### 为什么三个 Policy 不合并

第一版我们把这三类塞进同一张 `RuntimeState` 表里——结果运营调"<em>只改频次帽不动预算</em>"
也要锁整张表。拆开后，三类各走自己的 key 空间，编辑互不影响。

> 同一个东西被多人按不同节奏改，就是切边界的信号。

## DMP 接入：用 ACL 隔离 3 家不同口径

定向上下文里最不"我们自己"的部分，是 `Audience` 与 `Lookalike` ——
它们的数据来自外部 DMP。我们当前接了三家：极光、TalkingData、LiveRamp。
三家的<em>语义</em>都是"用户身上的标签"，但<strong>口径完全不同</strong>：

| 维度 | 极光 | TalkingData | LiveRamp |
|---|---|---|---|
| 标签 ID 格式 | 数字 | UUID | 字符串 hash |
| 标签层级 | 三级树 | 平铺 | 二级树 |
| 用户主键 | OAID / 手机号 hash | TDID（自有 ID）| RampID（identifier graph）|
| 更新频率 | T+1 批量 | 准实时 | T+1 批量 |
| 失效语义 | 标签消失即用户不再属于 | 显式过期时间戳 | 永久（直到再签收） |

如果我们让<em>核心域</em>直接看到这种参差，定向上下文的代码会被三家供应商的细节<strong>污染</strong>——
每加一家就要在 plan 编辑器、引擎匹配、报表三处同步改。

DDD 给这种情况一个标准答案：**ACL（Anti-Corruption Layer）防腐层**。

### 统一抽象：AudienceTag

```rust
/// 引擎核心域只认 AudienceTag——所有外部 DMP 都先映射到这个统一形态。
pub struct AudienceTag {
    pub id: AudienceTagId,         // 我们自己分配的稳定 ID
    pub source: DmpSource,         // 哪家 DMP 来的（仅供溯源）
    pub display_name: String,      // 用于运营后台 UI 展示
    pub user_key_kind: UserKeyKind,// OAID / TDID / RampID / …
    pub effective_at: DateTime<Utc>,
    pub expires_at: Option<DateTime<Utc>>,
}

pub trait DmpAdapter {
    /// 拉取该 DMP 的新增/变更标签，转成 AudienceTag 列表。
    fn fetch_tag_updates(&self, since: Cursor) -> Vec<AudienceTagDelta>;

    /// 给定一个用户主键，查询命中了哪些该 DMP 的标签 ID。
    fn lookup_user_tags(&self, key: &UserKey) -> Vec<DmpRawTagId>;

    /// 把外部原始标签 ID 翻译成我们内部的 AudienceTagId。
    fn map_tag_id(&self, raw: DmpRawTagId) -> Option<AudienceTagId>;
}

pub struct JiguangAdapter   { /* 极光特定逻辑 */ }
pub struct TalkingDataAdapter { /* TD 特定逻辑 */ }
pub struct LiveRampAdapter  { /* LiveRamp 特定逻辑 */ }
```

### 关键约束：核心域不感知 DmpSource

`Criterion::Audience` 里只引用 `AudienceTagId`。**没有任何 plan 直接写"极光的 12345 号标签"**——
它写的是<em>我们内部 ID</em>。这条边界一旦守住，再加第四家、第五家 DMP 都只是新写一个 Adapter，
不动定向上下文的核心代码。

> ACL 不是装饰——它是<em>防止外部供应商的数据模型反向定义你的核心域</em>的物理屏障。
> 看不到 `JiguangTagId` 出现在 plan 里，就是这道屏障在工作的证据。

## 小结：三件事的合力

定向上下文的工程价值，来自三件事的合力——

1. **数据结构**：8 维倒排 + RoaringBitmap，让百万 plan 匹配从线性扫到位图运算。
2. **并发模型**：ArcSwap 把"读多写少"做到物理上无锁；三类高频写 Policy 单独走 Redis。
3. **边界纪律**：DMP 用 ACL 隔离，核心域只认统一抽象 `AudienceTag`，不被供应商口径污染。

任何一件事缺位都会塌——
没有倒排，性能塌；没有 ArcSwap，并发塌；没有 ACL，演进塌。

下一篇我们走到**过滤上下文**——
为什么大部分过滤要 fail-open（识别错宁可放过），少数（合规相关）必须 fail-close，
以及如何把内部规则匹配器、订阅型黑名单、IVT 模型、第三方品牌安全（DV / IAS）四种来源
统一成一条链。

---

<small>
本篇是 ADX 设计系列第 4 篇，系列归档于
<a href="/series/adx-design/">/series/adx-design/</a>。
原始内部讨论文档保留在
<a href="https://github.com/g24-io/conversations-archive">g24-io/conversations-archive</a>。
</small>
