+++
title = "为什么我们用 DDD 重新设计 ADX"
description = "从一团乱麻的实时竞价系统出发，把 4 个核心域切清楚——bidding / pricing / targeting / filtering——为什么这是一笔划算的前期投入。"
date = 2026-04-26
updated = 2026-04-27
slug = "why-ddd-for-adx"

[taxonomies]
tags = ["ddd", "rust", "adx", "openrtb", "architecture"]
series = ["adx-design"]

[extra]
author = "g24-io engineering"
tldr = [
  "ADX 是 SSP 流量的拍卖撮合中心：上接 SSP / 下扇 N 个 DSP，端到端 SLA ≤ 100ms。",
  "我们用 **DDD** 把系统切成 10 个限界上下文（4 核心 / 3 支撑 / 3 通用），核心域是差异化护城河。",
  "**Rust + 六边形 + CQRS**：domain crate 禁依赖 tokio/sqlx，让边界不会随时间烂掉。",
  "运营后台只写 MySQL，引擎只读快照（ArcSwap），共享的不是库而是 **Protobuf Schema**。",
  "落地路线：6 个里程碑，先做 Bidding / Filtering / Targeting 三个核心域。",
]
+++

## 起点：一团乱麻的代码

这个项目的第一版能跑——单 SSP、单 DSP、几千 QPS 的内测流量、p99 60ms 上下，
看起来一切都好。但当我们准备接入第二个 SSP、第三个 DSP，
并且业务方要求「下周加一种新的频次策略」时，麻烦就来了。

> 加一个频次策略，需要改竞价主链路里的 7 个文件。

不是因为代码差——是因为「<em>这件事到底属于谁</em>」从来没被回答过。
频次策略是属于"定向"的吗？还是"过滤"的？还是"拍卖"的？
在第一版里，它三处都改了，因为没人确定边界在哪里。

于是我们停下来，决定花两周时间把它<em>切干净</em>。
切的工具是 **DDD**——具体地说是<strong>限界上下文 + 聚合根 + 上下文映射</strong>。
工具不是新的，但用在 ADX 这个领域，确实有它独有的味道。

> 这篇文章是<strong>为什么</strong>。后续 5 篇会一个一个讲<strong>怎么</strong>。

## 业务定位：ADX 是什么

ADX（Ad Exchange）在程序化广告生态里的位置，可以一句话说清——

<blockquote>
ADX 是<em>SSP 流量的拍卖撮合中心</em>：
上接 SSP / 媒体，下接 N 个 DSP，全部用 OpenRTB 协议沟通。
</blockquote>

它的价值取决于两件事：

1. **撮合效率**：SSP 的一个 imp 能不能在毫秒级让最合适的 DSP 出价并赢标
2. **生态信任**：DSP 信任拍卖透明、SSP 信任结算准确、合规方信任过滤到位

端到端 SLA：**≤ 100ms**。其中 DSP 侧分到的预算大概是 60–80ms，
ADX 自己的内部预算只有 20–30ms（含路由扇出、聚合、撮合、计价、回执构造）。

### 一次 BidRequest 的主链路

```
SSP → [接入&解析] → [流量过滤 / IVT / 品牌安全] → [定向 / 底价 / 频次]
    → [DSP 路由扇出] → [应答聚合] → [拍卖 / 计价] → [回执 SSP]
    → [展示曝光 / 点击 / 转化回调] → [计费 & 对账] → [日志 & 大数据]
```

每个箭头都不能慢，每个方括号都不能错。

### 四条平行的旁路

主链路是同步的、毫秒级的；但系统的另一半是异步的、秒到分钟级的——

- **配置下发**：运营后台 → 引擎热加载
- **实时日志**：引擎 → Kafka
- **计费扣款**：实时（win notice）+ T+1（对账）
- **风控反作弊**：在线特征 + 离线训练，模型双轨

主链路的设计目标是<em>快</em>，旁路的设计目标是<em>不丢</em>。
两套目标几乎对立——这正是为什么需要把它们切到不同的上下文里。

## 切成 10 个限界上下文

把上面主链路 + 旁路里所有职责盘点完，按<em>变化频率</em>和<em>业务依赖</em>聚合，
最终落到 **10 个限界上下文**——分三档：核心 / 支撑 / 通用。

> 这三档是 DDD 的<strong>战略分类</strong>，目的是回答「投多少工程力气」：
> 核心域投精兵、支撑域写工整、通用域能买就买。
> 它和限界上下文的<strong>边界划分</strong>是<em>正交的两件事</em>——
> 不要把 "核心域" 和 "上下文 A" 混为一谈。

### 核心域（4）：差异化护城河

竞争对手能不能做出同等水平的<em>这部分</em>——决定我们活不活下去。

| 上下文 | 聚合根 | 关键职责 |
|---|---|---|
| **Bidding** 竞价 | `Auction` | 拍卖编排、一价/二价、底价处理、winning bid 决策 |
| **Pricing** 定价 | `PriceBook` | 底价策略、币种、Bid Shading、margin、保留价 |
| **Targeting** 定向 | `TargetingPlan` | 地域/设备/受众/上下文/频次/时段定向匹配 |
| **Filtering** 流量过滤 | `FilterChain` | IVT、DV、品牌安全、黑白名单、域名/Bundle 校验 |

### 支撑域（3）：业务相关但非差异化

每家 ADX 都得做、做得好不会赢，做得不好会输。

| 上下文 | 聚合根 | 关键职责 |
|---|---|---|
| **Inventory** 流量供给 | `Publisher` / `AdSlot` | SSP / 媒体 / 广告位元数据 |
| **DSP Routing** 需求路由 | `DspEndpoint` | DSP 端点、扇出、超时熔断、QPS 配额、协议适配 |
| **Creative** 创意 | `Creative` | 创意素材、审核状态、尺寸/格式/物料校验 |

### 通用域（3）：可外采或平台化

业内通用，不用自己重新发明，能用 SaaS 或开源就用。

| 上下文 | 聚合根 | 关键职责 |
|---|---|---|
| **Identity** 身份 | `IdGraph` | IDFA / GAID / OAID / CAID、UID 解析、ID 映射 |
| **Billing** 计费 | `Account` / `LedgerEntry` | 账户余额、对账、win notice 入账 |
| **Tracking** 监测 | `TrackingEvent` | 曝光 / 点击 / 转化回调、宏替换、第三方监测 |

### Context Map：上下文之间的角色

切完 10 块还不够——必须画清楚<em>谁向谁要东西、谁给谁喂数据</em>。
我们用了三种 DDD 标准关系：

- **Customer / Supplier**：Bidding 是 Customer，向 Targeting / Pricing / Filtering / Inventory
  这几个 Supplier 拿能力。Supplier 改接口要打招呼。
- **Open Host Service（OHS）**：Creative / Inventory / DSP Routing 对引擎暴露<em>只读快照</em>——
  运行时引擎不再回查它们的数据库，全靠预热好的内存视图（`ArcSwap<Snapshot>`）。
- **Published Language**：跨上下文事件用 Protobuf schema 定义，所有上下文按这份契约工作。

> 上下文之间必要时通过 **ACL（Anti-Corruption Layer）防腐层** 隔离——
> 比如 Filtering 接 DV / IAS 这类第三方品牌安全 SDK，外部模型不会污染域内语言。

最关键的一对关系——**运营后台是所有上下文的写入端，引擎是所有上下文的读取端**。
这天然就是 CQRS 的形状，下文展开。

## 工程拓扑：六边形 + Crate Workspace

DDD 的边界是<em>逻辑</em>的。要让它在 Rust 代码里也站得住，还得落到<em>物理</em>边界——
也就是 crate 之间。我们用六边形（Ports & Adapters）+ Cargo workspace 做这件事。

### 四层架构

```text
┌────────────────────────────────────────────────────────────┐
│  Interface Layer  axum / hyper, gRPC, OpenRTB adapter     │
├────────────────────────────────────────────────────────────┤
│  Application Layer  Use Case 编排、事务边界                │
├────────────────────────────────────────────────────────────┤
│  Domain Layer       聚合 / 实体 / 值对象 / 领域服务         │  ← 纯 Rust，零 IO
├────────────────────────────────────────────────────────────┤
│  Infrastructure     Redis / MySQL / Kafka / HTTP client    │
└────────────────────────────────────────────────────────────┘
```

### Crate workspace 物理结构

```text
adx/
├── crates/
│   ├── domain-bidding/        # 纯领域，无 tokio 依赖
│   ├── domain-targeting/
│   ├── domain-pricing/
│   ├── domain-filtering/
│   ├── domain-creative/
│   ├── app-bidding/           # 用例编排
│   ├── infra-cache/           # ArcSwap + DashMap 配置缓存
│   ├── infra-kafka/           # rdkafka 事件发布
│   ├── infra-dsp-client/      # hyper 连接池
│   ├── infra-storage/         # sqlx / redis
│   ├── interface-rtb/         # OpenRTB HTTP 入口
│   ├── interface-tracking/    # 曝光点击回调
│   └── shared-kernel/         # 公共值对象（Money / AuctionId / …）
└── apps/
    ├── adx-engine/            # 竞价引擎 binary
    ├── adx-tracker/           # 监测回调 binary
    └── adx-billing/           # 计费 worker binary
```

### 一条死规矩

<blockquote>
<code>domain-*</code> crate 禁止依赖 <code>tokio</code> / <code>sqlx</code> / <code>reqwest</code>。
</blockquote>

这条死守。一旦某个 domain crate 因为「就调一次 HTTP」而引入了 reqwest，
两周之内整个 domain 层就开始<em>异步化</em>，
六个月之内你会重新看到「业务逻辑里夹着 await」的样子——也就是当初要逃离的状态。

把 IO 关在 infrastructure 层、把编排关在 application 层、让 domain 保持纯函数式——
这条边界是 DDD 在 Rust 里的<strong>物理体现</strong>。

## 高性能：按 5 层来设计

「Rust 快」是开始，不是结尾。50K QPS / 节点 + p99 内部 30ms 这种目标，
单凭语言层是凑不出来的。我们按<strong>硬件 / 进程 / I/O / 数据 / 算法</strong>五层逐层挤。

### 1. 硬件 / 运行时

- Tokio multi-threaded runtime，按 CPU 核心绑核
- `mimalloc` 或 `jemalloc` 替换默认分配器
- Linux 上启用 `io_uring`（`tokio-uring` 或 `monoio`）
- CPU 亲和性 + NUMA-aware 部署

### 2. 网络层

- `axum + hyper`（成熟）或 `monoio + glommio`（极致单核）
- HTTP keepalive + 连接池
- DSP 出向请求走 HTTP/2 多路复用

### 3. 数据访问

- 配置数据<strong>全量本地缓存</strong>：`ArcSwap<Snapshot>`，
  运营后台变更通过 Kafka 推送，引擎收到后<em>整体替换 Snapshot</em>——无锁读、原子切换
- 热数据（频次、预算）用 Redis Cluster + pipeline + Lua，本地 LRU 二级缓存
- JSON 解析：`simd-json`，或者直接走 Cap'n Proto / Protobuf

### 4. 算法 / 数据结构

- 定向匹配用 RoaringBitmap 或倒排索引
- 频次 / 预算计数用<em>分片计数器</em>（每核一个 slot，定时合并）
- DSP 扇出用 `FuturesUnordered + tokio::time::timeout`，先到先得，超时即剪枝

### 5. 可观测性

- Prometheus + OpenTelemetry，分位数延时（p50 / p95 / p99 / p999）
- 每条 BidRequest 埋 `trace_id`，串联 SSP → ADX → DSP 全链路

### 性能目标

| 指标 | 目标 |
|---|---|
| 单实例 QPS | 50K |
| 内部 p99 | < 30ms（不含 DSP 等待） |
| 端到端 p99（SSP 视角）| < 100ms |
| 内存常驻配置 | < 4GB / 实例 |

数字不是终点，是<em>每次回归测试要守住的护栏</em>。

## 运营后台 × 引擎：天然 CQRS

ADX 引擎（在线）和运营后台（管理）的职责完全不同——
一个追求毫秒延时，一个追求审核可控。强行让它们共享一份代码或数据库，
两边都会拧巴。我们走的是<strong>命令查询职责分离（CQRS）</strong>。

```text
┌──────────────┐      ┌──────────────┐
│  运营后台    │──写─→│  MySQL       │  ← 写库，强一致
│  (Rust)      │      │  (主)        │
└──────────────┘      └───┬──────────┘
                          │ binlog / CDC
                          ▼
                      ┌──────────────┐
                      │  Kafka       │  ← Domain Event Bus
                      │  config-topic│
                      └───┬──────────┘
                          │ 订阅
                          ▼
                  ┌────────────────────┐
                  │  ADX Engine (Rust) │
                  │  - 启动全量拉取    │
                  │  - 运行期增量      │
                  │  - ArcSwap<Cfg>    │
                  └────────────────────┘
```

### 三条核心约定

1. **运营后台不调引擎接口**——它只写 MySQL，不知道引擎在哪、有几个实例、健康不健康。
2. **引擎只读**——启动时全量拉取，运行期订阅 Kafka 增量；
   每天兜底再做一次全量 reconcile，治可能的事件丢失。
3. **共享的不是数据库，是 Schema**——这是 DDD 里的 *Published Language*。
   用 Protobuf 定义事件契约，运营后台和引擎各自生成代码，编译期就能发现不兼容修改。

> 这条比看上去重要。共享数据库相当于让所有上下文都能伸手到别家厨房；
> 共享 Schema 相当于在边界处放一份合同，谁改谁负责发新版。

### 运营后台都管些什么

- 广告主 / DSP 资质审核
- 创意审核（人工 + AI 双轨）
- 底价 / 流量包 / 黑白名单管理
- 报表查询——<em>走 OLAP，不走引擎</em>
- 财务对账

「报表不走引擎」这条容易被忽略。引擎是为了出价的，不是为了 group by 的；
所有跨时段的聚合查询都应该打到大数据侧的 ClickHouse / StarRocks，
让引擎专心做<em>毫秒级</em>的事。

## 大数据：事件驱动，不反查

```text
ADX Engine (Rust) ──→ Kafka (高吞吐, 7d 保留)
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
  ClickHouse /    Spark / Flink   Druid /
  StarRocks       (T+H 离线)      StarRocks
  (实时多维)                       (实时大盘)
```

### 几条原则

- **引擎只发事件，不查询大数据**——保持引擎轻量。
  哪怕需要"昨天的频次"这种数据，也是<em>大数据预聚合</em>后通过配置下发回来。
- **事件 schema 用 Protobuf + Schema Registry**，向后兼容。
  字段加可选、不删字段、改义就改名——这套规矩不灵活，但 7 天保留期内任何下游都能消费。
- **反作弊 / 出价模型在大数据侧训练**，模型文件通过对象存储下发到引擎，
  引擎本地用 `candle` / `ort` / `tract` 加载 ONNX 推理——保证<em>线下实验</em>和<em>线上推理</em>用的是同一份权重。

### 标准领域事件

```rust
pub enum AdxEvent {
    BidRequestReceived  { req_id, ssp_id, received_at, .. },
    AuctionCompleted    { auction_id, winning_bid, nbr, .. },
    ImpressionDelivered { imp_id, ts, .. },
    ClickTracked        { imp_id, ts, .. },
    ConversionAttributed{ imp_id, attribution_window, .. },
}
```

每个变体都有 `trace_id` 串起 SSP→ADX→DSP→曝光→点击→转化的全链路。
报表的"漏斗"和审计的"追踪"，本质都是对这个事件流的 group by。

## 几个我们清楚知道的风险

DDD 不是免费的。这次重构我们做了五项<em>明知有代价但仍然选了</em>的决定，列在这里——
不是为了说服自己，是为了在三个月后回头能识别"哪个代价是真的"。

1. **DDD 学习成本**——团队里不是每个人都熟。第一版我们只把 **Bidding / Filtering / Targeting**
   三个核心域做透；其余先用贫血模型，业务清楚了再二次重构。
2. **Rust 生态在广告领域偏薄**——OpenRTB SDK、DV / IAS SDK 多数是 Java/Go/C++。
   部分要自研、部分要 FFI 包一层、部分接受用 sidecar 进程隔离。
3. **配置全量加载会爆内存**——SKU 量到一定规模时，单实例 4GB 守不住。
   预案：分片（按媒体或区域）、冷热分层、把超大维度（创意、域名黑名单）走外部存储 + 本地 LRU。
4. **二价 / GSP 实现细节多**——一价（First-Price）先做，验证撮合主链路稳定；
   二价 / GSP 在第二个迭代叠加，附带 bid shading。
5. **DSP 接入永远是体力活**——DSP Routing 上下文做好<em>适配器模式</em>，
   每接一个新 DSP 就只动这一个 crate。核心域绝不感知具体 DSP 谁是谁。

> "我们知道自己不知道"和"我们不知道自己不知道"是两件不同的事。
> 上面 5 条属于第一种，是<em>可控风险</em>。第二种我们留了 5% 的迭代预算应付意外。

## 落地路线：6 个里程碑

| 阶段 | 范围 | 周期 |
|---|---|---|
| **M1** | 单 SSP × 单 DSP，OpenRTB 直通，无定向 | 4 周 |
| **M2** | Targeting + Pricing + Filtering 三大核心域上线 | 6 周 |
| **M3** | 运营后台 + 配置推送（CQRS 闭环） | 4 周 |
| **M4** | 多 DSP 扇出 + 拍卖（一价 → 二价）+ 计费 | 6 周 |
| **M5** | 监测回调 + Kafka 事件流 + 实时大盘 | 4 周 |
| **M6** | 反作弊模型 + 智能底价 + 性能调优 | 持续 |

每个里程碑结束都做一次<strong>架构回顾</strong>——
检查 domain crate 有没有偷渡 IO 依赖、上下文之间有没有意外耦合、
性能指标是否还守在<em>预定的护栏之内</em>。

<blockquote>
回顾的目的不是写漂亮文档，是<em>让边界继续是边界</em>。
</blockquote>

## 接下来的 5 篇文章

这篇是 ADX 设计系列的 overview。后续 5 篇会一篇一个核心域 + 一篇集成总图：

1. **[Bidding] 竞价上下文：22 条不变式如何长出来** —— `Auction` 聚合根、状态机、漏斗
2. **[Pricing] 定价上下文：底价、Bid Shading 与利润策略的解耦** —— 4 个独立写聚合 + PriceBook 读视图
3. **[Targeting] 定向上下文：用 RoaringBitmap 把 8 维筛选打成毫秒级** —— 倒排索引 + DMP ACL
4. **[Filtering] 过滤上下文：fail-open 与合规 fail-close 的边界** —— 规则匹配器、IVT、第三方品安
5. **[Integration] 4 个核心域如何在一个进程里协同：集成总图** —— 9 步同步主流程 + 6 条异步旁路

每一篇都从<em>这个上下文当时让我们最难受的那个问题</em>讲起——
而不是"这个上下文是什么"。这是这套博客唯一的写作纪律。

读到这里，你应该已经感觉到——
**DDD 在 ADX 里不是教条，是为了让一个高 QPS 系统在多人协作下<em>不烂</em>的工程手段**。
它是工具，不是宗教。

---

<small>
本系列基于 g24-io 工程团队 2026 年 4 月对内部 ADX 重构的归档。
原始讨论文档保留在
<a href="https://github.com/g24-io/conversations-archive">g24-io/conversations-archive</a>
仓内，时间戳与本博客一致。
</small>
