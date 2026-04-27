+++
title = "过滤上下文：fail-open 与合规 fail-close 的边界"
description = "规则匹配器、订阅型黑名单、IVT 模型、第三方品牌安全的接入——以及为什么大多数过滤要 fail-open，少数必须 fail-close。"
date = 2026-04-27
slug = "filtering-context"

[taxonomies]
tags = ["filtering", "ddd", "rust", "ivt", "compliance", "fail-safe"]
series = ["adx-design"]

[extra]
author = "g24-io engineering"
tldr = [
  "**3 个聚合根**：FilterRule（自家规则）/ Blocklist（订阅型黑名单）/ IvtModelDeployment（异常流量模型）。",
  "**统一接口 RuleMatcher**，7 类实现：InSet / IpRange / Regex / IabContains / MlScoreAbove / Composite / External。",
  "**默认 fail-open**——过滤器宕机宁可放过，不让正常流量被误杀。",
  "**合规 fail-close 例外**——少数与法律责任相关的规则（地域合规、未成年人保护）必须在故障时阻断。",
  "**双 ACL**：一条防腐层订阅外部黑名单（NCC / IAB），另一条 pre-fetch DV / IAS 品牌安全结果。",
]
+++

## 过滤的两难

把一个有问题的请求<em>错过滤掉</em>，会丢钱——这条流量本来能成交。
把一个有问题的请求<em>放过去</em>，会出事——可能是反作弊数据被污染，
也可能是品牌广告被投到不该出现的地方，最坏的是<strong>触发监管罚款</strong>。

这是过滤上下文每一天都在权衡的两件事。
工程上的做法不是"哪个更重要"——是<strong>分类</strong>：

- **大部分过滤** 出错时<em>放过</em>——也就是 _fail-open_。
  代价是偶尔放掉一两个 IVT 流量，对营收和品牌损失可控。
- **少数过滤**（合规、未成年人保护、地域限制）出错时<em>阻断</em>——也就是 _fail-close_。
  代价是这条流量丢钱，但与潜在监管成本相比可忽略。

> 默认 fail-open，例外 fail-close——而且<em>例外必须在聚合层面被显式标注</em>。
> 没有这条纪律，整个过滤链一旦下游慢下来，引擎就不知道该让流量过还是不过。

## 3 个聚合根，分别管 3 类来源

过滤要处理的"<em>是否合规</em>"信号来自三处：我们自己写的规则、订阅别人维护的黑名单、机器学习模型。
强行混在一起会让<em>更新节奏</em>互相牵制——所以拆成 3 个聚合根。

### 1. FilterRule · 自家维护的过滤规则

```rust
pub struct FilterRule {
    pub id: FilterRuleId,
    pub name: String,
    pub matcher: RuleMatcher,
    pub action: FilterAction,         // Block / Flag / RoutToReview
    pub failure_mode: FailureMode,    // FailOpen / FailClose
    pub compliance_tag: Option<ComplianceTag>, // 合规规则要打 tag
    pub priority: u16,
    pub effective_at: DateTime<Utc>,
    pub expires_at: Option<DateTime<Utc>>,
}
```

不变式：

- `failure_mode = FailClose` 时 `compliance_tag` 必填——逼运营在<em>编辑器</em>就回答清楚理由。
- 一个 `FilterRule` 仅持有一个 `RuleMatcher`——多个规则就拆多条；不允许嵌套到一条规则里"内部 OR"。
- `action = Block` 的规则到期后保留审计 90 天再 hard delete。

### 2. Blocklist · 订阅外部维护的黑名单

行业内的恶意域名、欺诈应用 bundle、知名 IVT IP 段——这些不是我们自己维护的，
而是从 NCC（中国互联网协会）、IAB、Pixalate 等订阅而来。

```rust
pub struct Blocklist {
    pub id: BlocklistId,
    pub source: BlocklistSource,      // NCC / IAB / Pixalate / Custom
    pub kind: BlocklistKind,          // Domain / Bundle / IpRange / DeviceId
    pub revision: BlocklistRevision,  // 上游版本
    pub fetched_at: DateTime<Utc>,
    pub entries: Vec<BlocklistEntry>,
}
```

`Blocklist` 自己<em>不写</em>——它只被订阅源更新。运营后台能看，但只能切换"是否启用"。
每次启用变更，引擎重新构建 `BlocklistIndex`（同 `TargetingIndex` 那套 ArcSwap 玩法）。

### 3. IvtModelDeployment · 反作弊模型上线

机器学习模型的<strong>部署</strong>是一个 DDD 概念上独立的事——
模型本身的训练在大数据侧，<em>这次上线哪个版本、给哪些 SSP 用、阈值多少</em>是定向上下文之外的另一份数据。

```rust
pub struct IvtModelDeployment {
    pub id: DeploymentId,
    pub model_ref: ObjectStorageRef,  // 模型文件指纹（与 PriceBook 共享同一套 ref）
    pub model_kind: IvtModelKind,     // Bot / Spoofing / FarmDevice / Anomaly
    pub threshold: f32,
    pub scope: DeploymentScope,       // 全量 / 按 SSP / 按 SspGroup
    pub experiment_tag: Option<ExperimentTag>,
    pub failure_mode: FailureMode,    // 默认 FailOpen
    pub deployed_at: DateTime<Utc>,
}
```

模型推理在引擎本地（`candle` / `ort` / `tract` 加载 ONNX），不打跨进程 RPC——
否则 5ms 内吃不消。

> 三个聚合根的演进节奏完全不同：
> FilterRule 每天改、Blocklist 每周拉一次、IvtModelDeployment 每两周上一版。
> 把它们切开，意味着<em>每一种节奏的变化都不会震动其他两类</em>。

## RuleMatcher · 7 类实现

`FilterRule` 持有的 `RuleMatcher` 是个 trait——7 类常用实现覆盖 95%+ 的过滤场景：

```rust
pub trait RuleMatcher {
    fn matches(&self, ctx: &FilterContext) -> MatchOutcome;
}

pub enum MatchOutcome {
    Match,
    NoMatch,
    Inconclusive,    // 用于 fail-open / fail-close 的兜底分支
}

pub enum RuleMatcherKind {
    /// 字段值在常量集合内（domain ∈ {…}）。
    InSet           { field: ContextField, set: HashSet<String> },
    /// IP 在 CIDR 段内（含 RoaringTreemap 加速大量段查询）。
    IpRange         { ranges: IpRangeIndex },
    /// 字段匹配正则（compile 一次缓存）。
    Regex           { field: ContextField, regex: Regex },
    /// IAB 分类树包含某节点（含子树）。
    IabContains     { category: IabCategoryId },
    /// ML 模型推理分高于阈值。
    MlScoreAbove    { deployment: DeploymentId, threshold: f32 },
    /// And/Or/Not 组合（嵌套深度 ≤ 3）。
    Composite       { op: CompositeOp, children: Vec<RuleMatcherKind> },
    /// 调用外部服务（DV / IAS）—— 需要 pre-fetch + 短超时。
    External        { provider: ExternalProvider, timeout: Duration },
}
```

### 关键约束：External 永远 fail-open

`External` 类匹配器调用 DV / IAS 等第三方品牌安全 API。
即便上游 timeout 或返回错误，<strong>结果都 map 到 `MatchOutcome::Inconclusive`</strong>，
然后由 `FilterRule.failure_mode` 决定放/挡。

但我们对 `External` 强制规定 `failure_mode = FailOpen`——
原因：<em>第三方服务挂掉的概率远高于我们漏过一两条不安全流量的损失</em>。

> 别让外部依赖能把整个引擎拖下水。这是 5ms 预算线下的生存原则。

## 双 ACL：两条防腐线各管一头

过滤上下文有<strong>两条</strong>外部依赖路径——它们方向相反、节奏不同，所以走两条独立 ACL。

### ACL-A · Blocklist 订阅

外部 → 我们：从 NCC、IAB、Pixalate 等订阅黑名单。

- **节奏**：天到周级，批量下载
- **失败容忍度**：高——上次成功的 `Blocklist` 仍然有效，新一批失败仅记录告警
- **形态**：拉取 + 解析 + 转成内部 `BlocklistEntry`，<em>不让外部数据格式渗进核心域</em>

```rust
pub trait BlocklistFeed {
    fn fetch_latest(&self) -> Result<RawBlocklist, FeedError>;

    /// 把外部格式翻译成我们内部的 BlocklistEntry。
    fn map_entries(&self, raw: &RawBlocklist) -> Vec<BlocklistEntry>;
}

pub struct NccBlocklistFeed     { /* … */ }
pub struct IabBlocklistFeed     { /* … */ }
pub struct PixalateFeed         { /* … */ }
```

### ACL-B · 第三方品安预查询

我们 → 外部：对每条流量打 DV / IAS 预查询拿到「是否品牌安全」「是否 IVT」结果。

- **节奏**：实时——每个 BidRequest 都要查
- **失败容忍度**：低延迟约束（`timeout` ≤ 8ms），结果存入本次 `FilterContext` 内
- **形态**：把外部 SDK 的多变接口包成统一 trait

```rust
pub trait BrandSafetyProvider {
    /// 异步预查询。timeout 由调用方传入。
    async fn evaluate(
        &self,
        ctx: &PreFilterContext,
        timeout: Duration,
    ) -> ProviderResult;
}

pub struct DvProvider  { /* DoubleVerify SDK 封装 */ }
pub struct IasProvider { /* IAS SDK 封装 */ }
```

> 两条 ACL 的<em>共同点</em>：核心域看到的都不是外部数据格式，
> 只是<strong>翻译过的内部类型</strong>。FilterRule 永远不直接 import DV 或 IAS 的 SDK 类型。

## 合规：极少数 fail-close 的归属

整个过滤上下文里，<strong>fail-close 的规则不会超过 10 条</strong>。它们极少，但极重要——

| 合规来源 | 失败模式 | 例子 |
|---|---|---|
| 地域强制（中国境内服务）| fail-close | `compliance_tag = ICP_REQUIRED` 的 publisher 没有有效 ICP 时阻断 |
| 未成年人保护 | fail-close | content rating ≥ Mature 的 imp 在儿童活动上必须挡 |
| 禁投行业 | fail-close | 烟酒/博彩等敏感品类对应国家不允许时挡 |
| 数据隐私（GDPR / CCPA）| fail-close | 用户已撤回同意时不允许下发 audience 标签 |

每一条 fail-close 规则都<strong>必须填 `compliance_tag`</strong>。
这个 tag 不是装饰——它是让法务可以事后审计「<em>这个时段哪些请求被以哪条合规规则挡掉了</em>」的查询锚点。

> Fail-close 是一种<em>主动放弃部分营收以避免更大风险</em>的设计——
> 越少越好，但每一条都<strong>必须</strong>能解释清楚理由。

## 小结：分类思考，例外标注

过滤上下文的工程命题不是"过滤多少种"——是<em>每种过滤<strong>怎么失败</strong>都讲清楚</em>。

1. **三类来源各自成聚合**：自家规则 / 订阅黑名单 / IVT 模型——演进节奏不同。
2. **7 类 RuleMatcher 通过 trait 统一**：足够灵活，又不会无序膨胀。
3. **默认 fail-open**：放过偶尔的坏请求，远比阻断正常流量便宜。
4. **fail-close 是稀有例外**：必须在聚合层显式标注，必须挂 `compliance_tag` 让法务可审计。
5. **双 ACL**：Blocklist 走拉取式 ACL，第三方品安走 pre-fetch ACL，外部数据格式不污染核心域。

下一篇是这个系列的最后一站——
**集成总图**：4 个核心域如何在一个进程里同步协作，
9 步同步主流程怎么走，6 条异步旁路如何各司其职，以及为什么我们坚持<em>同进程</em>而非<em>微服务化</em>。

---

<small>
本篇是 ADX 设计系列第 5 篇，系列归档于
<a href="/series/adx-design/">/series/adx-design/</a>。
原始内部讨论文档保留在
<a href="https://github.com/g24-io/conversations-archive">g24-io/conversations-archive</a>。
</small>
