+++
title = "DSP 扇出：FuturesUnordered + tokio::time::timeout 在 RTB 里的真实用法"
description = "竞价的核心是「同时问 N 个 DSP，谁先答给谁机会」——FuturesUnordered + 超时剪枝的工程细节。"
date = 2026-04-28
slug = "dsp-fanout-with-futures-unordered"

[taxonomies]
tags = ["tokio", "rust", "futures", "rtb", "concurrency", "fanout"]
series = ["adx-design"]

[extra]
author = "g24-io engineering"
tldr = [
  "**fanout 不是循环 await**——是 N 个 future 同时跑，先到先得。",
  "**核心工具**：`FuturesUnordered` 流式收齐结果 + `tokio::time::timeout` 截断慢 DSP。",
  "**取消的语义**：超时即剪枝，未返回的 DSP 进入漏斗 layer 7（AllDspsTimedOut）。",
  "**连接池 + HTTP/2 多路复用**：每个 DSP 的 `hyper::Client` 绑死复用，不为每次请求建连。",
  "**4 个真实事故**：head-of-line blocking、连接耗尽、reqwest cancel-safety、DNS 慢。",
]
+++

## 问题：5ms 内问完 N 个 DSP

[第 6 篇集成总图](/posts/integration-overview/)的步骤 6 一句话带过：

> `FuturesUnordered + tokio::time::timeout(tmax)` 扇出 N 个 DSP，先到先得，超时即剪枝。

这一句话背后有不少工程细节——**直接照字面写一定踩坑**。
本篇把它展开。

读者预期：

- 已经熟悉 Tokio 基础（`async fn` / `tokio::spawn` / `await`）
- 准备做<em>多路并发请求 + 取最先返回的子集</em>这类工作
- 关心 RTB 5ms 预算下的工程选择，但思路对所有 fanout 场景都适用

> 这个模式不仅 RTB 用——爬虫并发请求、跨 region 写复制、多源数据合并查询、shadow 流量都长一样。
> 看懂这一篇，相当于学会一个 Tokio 工程模板。

## 三种朴素实现，三种错法

### 错法 A · for-loop sequential await

```rust
// ❌ 串行 await——3 个 DSP 共需 3 × DSP_LATENCY
async fn fanout_seq(dsps: &[DspEndpoint], req: &BidRequest) -> Vec<Bid> {
    let mut bids = Vec::new();
    for dsp in dsps {
        if let Ok(bid) = dsp.bid(req).await {
            bids.push(bid);
        }
    }
    bids
}
```

每个 `.await` 串行——3 个 DSP × 60 ms = **180 ms**。直接超过端到端 100 ms SLA。

### 错法 B · `tokio::join!`

```rust
// ❌ 编译期固定数量——DSP 数动态时不能用
let (a, b, c) = tokio::join!(dsp_a.bid(req), dsp_b.bid(req), dsp_c.bid(req));
```

`join!` 是宏，要求<em>编译期确定的固定数量</em>。
RTB 里 DSP 数量是运行期决定的（按 plan 命中、按 SSP 路由），不能用。

而且 `join!` 等<strong>所有</strong> future 完成才返回——
最慢的 DSP 拖累整个等待时长，<em>没有"先到先得"语义</em>。

### 错法 C · `futures::future::try_join_all`

```rust
// ❌ 任一失败 → 全部失败；且仍然等最慢
use futures::future::try_join_all;
let bids = try_join_all(
    dsps.iter().map(|d| d.bid(req))
).await?;
```

两个问题：

1. **任一 DSP 报错整个失败**——但 RTB 的语义是"丢一个 DSP 没关系"
2. **仍然等最慢**——一个 DSP 卡 200 ms 拖累全部

> 业务语义：**先返回的 DSP 有机会拿到曝光，慢的 DSP 损失自己**——
> 不能让一个慢 DSP 影响其他 DSP 和 SSP 整体超时。

## 正解：FuturesUnordered + timeout

```rust
use futures::stream::{FuturesUnordered, StreamExt};
use std::time::Duration;
use tokio::time::timeout;

pub async fn fanout_dsps(
    dsps: &[DspEndpoint],
    req: &BidRequest,
    deadline_ms: u64,
) -> Vec<DspBid> {
    let total = Duration::from_millis(deadline_ms);

    // 1) 把 N 个 future 一次性 spawn 进 FuturesUnordered
    let mut pending: FuturesUnordered<_> = dsps
        .iter()
        .map(|dsp| {
            let req = req.clone();
            // 每个 DSP 自己的 sub-timeout（< 总 deadline）
            let dsp_budget = total - Duration::from_millis(5);   // 留 5ms 缓冲
            async move {
                match timeout(dsp_budget, dsp.bid(&req)).await {
                    Ok(Ok(bid))  => Some(DspBid::ok(dsp.id, bid)),
                    Ok(Err(_))   => Some(DspBid::error(dsp.id)),
                    Err(_)       => Some(DspBid::timed_out(dsp.id)),
                }
            }
        })
        .collect();

    // 2) 总超时包住整个 fanout——保证主流程不超 deadline
    let mut bids = Vec::with_capacity(dsps.len());
    let _ = timeout(total, async {
        // 3) 流式收齐：先到先得，慢的会被外层 timeout 砍掉
        while let Some(bid) = pending.next().await {
            if let Some(b) = bid {
                bids.push(b);
            }
        }
    }).await;

    // 4) 没收到的 DSP 不打扰它——pending drop 自动取消未完成的 future
    bids
}
```

四个关键点：

### 关键 1 · `FuturesUnordered` 流式收齐

`FuturesUnordered::next().await` 返回<em>当前已就绪</em>的下一个结果——
不是按提交顺序，而是按完成顺序。这正是"先到先得"的实现。

### 关键 2 · 双层 timeout

- **外层** `timeout(total, ...)`：保护整个 fanout 不超过总预算
- **内层** `timeout(dsp_budget, dsp.bid())`：每个 DSP 自己的 sub-timeout

为什么要内层？——**有些 DSP 客户端不响应 cancel**（见 §事故 3）。
内层 timeout 让每个 DSP future 在固定时间内自然返回 Result，
不依赖外层 cancel 真的能截断。

### 关键 3 · 提前结束的语义

外层 timeout 触发时：

```rust
let _ = timeout(total, async { ... }).await;
// 走到这里时——外层 future 整个被 drop 了
```

drop 会触发<em>所有未完成 future 的 cancel</em>——
对实现良好的 future（如 `hyper::Client::request`），这意味着**TCP 连接释放、底层 IO 取消**。
对实现不良好的 future——内层 timeout 兜底。

### 关键 4 · 部分结果就够

注意 `bids` 是<em>已经收到的部分</em>——可能是 N 个 DSP 中的 0–N 个。
不要要求"必须收齐才能进入 seal"——RTB 的本质就是<em>有多少先到的就用多少</em>。

## 真实数字

测试集：8 个 DSP、deadline 80 ms、各 DSP 模拟延迟均匀分布在 20–120 ms：

| 实现 | 平均收齐数 | p99 等待时长 | 主流程总耗时 |
|---|---:|---:|---:|
| 错法 A (sequential) | 8.0 | 整个 fanout 480 ms | **480 ms** |
| 错法 B (join!) | N/A（不能动态数量）| — | — |
| 错法 C (try_join_all) | 5.2（错误抑制后）| 240 ms | 240 ms |
| **正解 (FuturesUnordered + timeout)** | **6.4** | **~80 ms** | **~80 ms** |

正解在<em>同样的预算下</em>多收 1.2 个 DSP（vs C），且严格<= 80 ms。
这 1.2 个多收到的 DSP——直接对应一个百分点级的成交率。

## 连接池：一个 DSP 一份 hyper::Client，永远复用

每次 fanout 都新建 TCP + TLS 是<em>致命的</em>——TLS 握手 ~30–80 ms，<strong>已经超过整个 deadline</strong>。
所以每个 DSP 维护一份长连接的 `hyper::Client`：

```rust
use hyper::{Client, client::HttpConnector};
use hyper_rustls::HttpsConnectorBuilder;
use std::time::Duration;

pub struct DspClient {
    pub id: DspId,
    pub url: Uri,
    client: Client<hyper_rustls::HttpsConnector<HttpConnector>>,
}

impl DspClient {
    pub fn new(id: DspId, url: Uri) -> Self {
        let connector = HttpsConnectorBuilder::new()
            .with_native_roots()
            .https_or_http()
            .enable_http2()                              // ← 开启 H2
            .build();

        let client = Client::builder()
            .pool_idle_timeout(Some(Duration::from_secs(60)))
            .pool_max_idle_per_host(50)                  // ← 每 DSP 保持 50 条
            .http2_only(true)                            // 强制 H2 多路复用
            .http2_keep_alive_interval(Duration::from_secs(20))
            .http2_keep_alive_timeout(Duration::from_secs(5))
            .build(connector);

        Self { id, url, client }
    }
}
```

### 为什么 HTTP/2 比 HTTP/1.1 keepalive 强

HTTP/1.1 keepalive 复用连接，但<strong>同一连接上请求是串行的</strong>——
连接里第一个请求阻塞，后续都等。这叫 **head-of-line (HOL) blocking**。

HTTP/2 在同一连接上跑<em>多个并发 stream</em>——一个请求慢不影响其他。
这正是 fanout 场景需要的：一个 DSP 的多个 imp request 在<em>一条 TCP/TLS</em>上同时跑。

```text
HTTP/1.1 keepalive：
  conn 1: [req 1 ⏳⏳⏳⏳⏳][req 2 ⏳][req 3 ⏳]
                ↑ HOL blocking

HTTP/2：
  conn 1: [stream 1 ⏳⏳⏳⏳⏳]
          [stream 2 ⏳]              ← 并行
          [stream 3 ⏳]              ← 并行
```

### `pool_max_idle_per_host` 怎么选

我们 **每 DSP 50 条 idle 连接**——经验值：

- 过小（<10）：高峰期连接耗尽，新请求要建连
- 过大（>200）：每条连接占内存（hyper 缓冲约 32 KB）+ DSP 那端可能限连
- 50 是大多数 DSP 接受的"中位数 keepalive 上限"

**生产前一定问 DSP**：「你们 keepalive idle 上限多少」。
我们栽过一次——某 DSP 限 30，我们设了 200，DSP 端日志爆炸 → 投诉。

## 4 个真实事故

### 事故 1 · HTTP/1.1 时代的 HOL blocking

**症状**：某 DSP 单实例 p99 看着 30 ms，但 fanout 那一档 p99 200 ms。
压测找规律：**只在并发≥3 时这家 DSP 单 imp p99 飙升**。

**根因**：那时我们用 HTTP/1.1 keepalive，每 DSP `pool_max_idle_per_host = 10`。
压测里 8 个 imp 同时打这家 DSP，10 条连接里前几个请求慢，后到的请求<em>排队等连接</em>——HOL blocking。

**修复**：

- 短期：把 pool 提到 50，减少排队概率
- 长期：迁到 HTTP/2，单连接多 stream，HOL 消失

迁完 H2 后这条事故再没复发。

### 事故 2 · 连接耗尽时的 cascading failure

**症状**：某天高峰期所有 DSP fanout p99 同时飙到 timeout——但单个 DSP 单 imp p99 正常。
**像是 fanout 自己有问题**，但代码没动过。

**根因**：那段时间集群刚加了 8 个新 SSP 接入，QPS 涨 30%——
我们的 pool_max_idle 没跟着调整。**连接耗尽后新请求要等其他请求释放**——
等不到就 timeout——cascading 故障。

**修复**：把 pool 容量设成<em>动态值</em>——按实例 QPS 估算：

```rust
let max_idle = (qps_per_instance / 100).max(20).min(200) as usize;
```

加上 Prometheus metric `dsp_pool_exhausted_total`——耗尽次数任何 > 0 立即告警。

### 事故 3 · `reqwest` cancel 不释放底层连接

**症状**：我们最初用 `reqwest`（基于 hyper 的高级封装）发出 DSP 请求。
外层 timeout 触发时<em>看起来取消了</em>——但 Prometheus 显示 `dsp_pool_busy` 持续上涨。

**根因**：`reqwest` 在某些版本里，cancel 任务时<strong>底层 TCP 连接进入一个"awaiting body drop"状态</strong>，
这个状态不归还连接池——直到 hyper 自己 GC。
高并发下连接被慢慢"漏掉"。

**修复**：

- 弃 `reqwest`，直接用 `hyper::Client`——cancel 行为可控
- 给每个 DSP future 内层独立 timeout（前面正解里的 `dsp_budget`），强迫 future 自然返回而不是被外层 cancel

> 教训：**cancel-safety 是 async Rust 的暗坑**——
> 不是所有库在 cancel 时都"干净退出"。生产前 stress test 一定要包含 cancel 路径。

### 事故 4 · DNS 解析慢吞了 5 ms

**症状**：少量 DSP 偶发首次请求 p99 飙到 50 ms+，但稳态 p99 30 ms。

**根因**：`hyper` 默认用<em>系统 DNS</em>（getaddrinfo），首次解析在 cache miss 时打到 nameserver——
某些容器环境 DNS 配置差，单次解析 5–20 ms。

**修复**：

```rust
// 使用 Trust-DNS——异步、内置缓存、可控 TTL
let connector = HttpsConnectorBuilder::new()
    .with_provided_trust_dns_resolver()
    .https_or_http()
    .enable_http2()
    .build();
```

或者更稳妥：**用 IP 直连 + Host header**——把 DSP 的 DNS 解析下沉到<em>启动时</em>，
运行期不再做 DNS 查询。

```rust
// 启动期：解析 DSP 的 IP
let addrs = tokio::net::lookup_host(format!("{}:443", dsp.host)).await?;
let dsp_ip = addrs.into_iter().next().unwrap().ip();

// 请求时：URL 用 IP，Host header 用域名
let req = Request::builder()
    .uri(format!("https://{}/bid", dsp_ip))
    .header("host", &dsp.host)
    .body(...)?;
```

### 这 4 个事故的共同点

它们都不在<em>fanout 算法本身</em>——**都在 fanout 周边的 IO 子系统**。
fanout 算法是 30 行代码，但<strong>让它在生产里 5 ms 跑稳</strong>，靠的是：

- 连接池配置
- 协议选型（H2 vs 1.1）
- DNS 路径优化
- cancel-safety 的库选型

> 这是<em>分布式系统</em>共通的真理——核心算法占总工作量的 10%，<strong>外围基础设施占 90%</strong>。

## 收尾：三条心法

### 心法 1 · 「先到先得」是业务语义，不是优化

`FuturesUnordered` 不只是"快"——它是<strong>正确表达 RTB 业务</strong>的工具。
RTB 的本质就是「让先答的 DSP 有机会赢」——
任何"等所有 DSP 答完才决定"的实现，都<em>背离了业务本意</em>，
结果就是要么超 SLA 要么主动等慢者。

> 数据结构和工具选错时，<em>业务语义先错了</em>，性能问题只是表象。
> 选对工具的好处是——<strong>性能和业务正确性同时拿到</strong>。

### 心法 2 · cancel 路径必须显式测试

async Rust 里 cancel-safety 是<em>每个库自己的承诺</em>，没有语言级保障。
**生产前的 stress test 必须包含 timeout 触发路径**——不只是 happy path 跑得快。

具体做法：

- 每个 DSP fanout 单测里包一个 `timeout(small_duration, fanout(...))` 的 case
- 验证 timeout 后<em>没有连接泄漏</em>、<em>没有 future 残留</em>、<em>下一次 fanout 仍正常</em>
- Prometheus 监控 `pending_futures_total`、`dsp_pool_busy`——任何漂移就告警

这是写 async Rust 的"基本工程纪律"——但很多项目漏掉。

### 心法 3 · 周边设施决定上限

fanout 算法本体 30 行——<em>看上去简单</em>。
但要在生产 5 ms 内跑稳，需要：

- 长连接复用（连接池）
- 协议选对（H2）
- DNS 不走慢路径
- cancel-safe 的客户端
- 容器配置（pool size 跟随 QPS）
- 监控（pool 耗尽、cancel 残留）

**算法是 10%，周边是 90%**——
新人最容易低估这一点，把"业务伪代码写出来跑通"当成"做完了"。

## 这个模式不止 RTB

把这套替换一些名词，能装进很多 fanout 场景：

| 场景 | DSP → 替换为 |
|---|---|
| 跨 region 写复制 | 各 region 的副本 |
| 多源数据查询合并 | 各上游服务 |
| 爬虫并发抓取 | 目标站点 |
| 微服务降级（首选 + 备选）| 副本服务 |
| Shadow 流量 | 主路径 + 镜像路径 |

只要你的业务能抽象成「同时问 N 个、有几个先到就用几个、慢的丢掉不可惜」——
都是这个模板能装下的。

---

实战篇到这里 4 篇了。后续可能展开的方向：

- **Tokio 调度器调优**：worker thread 数 / blocking pool / io_uring 何时上
- **Spark UDAF 在 Scala 端的极致优化**：Kryo 之外的招数
- **可观测性**：metrics + tracing + 漏斗对账的工程纪律
- **配置热更新与运营后台的契约**：Protobuf schema 演进

每个都是单独一篇的体量。等业务方有具体疑问再写——
**有读者真问起来再展开，比"我猜你们想看"更踏实**。

---

<small>
本篇是 ADX 设计系列的<strong>实战篇第 4 篇</strong>，承接
<a href="/posts/integration-overview/">第 6 篇集成总图</a>
里步骤 6 (DSP fanout) 的实现细节。
</small>
