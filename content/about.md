+++
title = "about"
description = "Who we are, what we publish, and how to talk to us."
template = "page.html"
date = 2026-04-27

[extra]
author = "g24-io engineering"
+++

## g24-io-tech 是什么

g24-io 工程团队的公开笔记本——
把构建一个高 QPS、低延迟、可演进的 ADX 过程中的**设计取舍、不变式、以及踩过的坑**，
整理成可以反复读的文档。

我们不写「最佳实践」、不卖课、不带货。每一篇都是先在内部需要解释清楚，
然后才公开出来——所以即使你不在广告行业，也应该能看出我们当时面对的真问题是什么。

## 我们都写什么

- **ADX 的领域设计**：DDD bounded contexts、聚合切分、不变式收敛、状态机驱动
- **Rust 在线侧的工程实现**：`ArcSwap` 快照、`RoaringBitmap` 倒排、`SmallVec` 零堆分配、零拷贝路径
- **大数据后端的契约**：Spark/Scala 解析海量竞价日志，StarRocks/ClickHouse 撑漏斗报表
- **小经验**：不那么"AI"，但都是真实踩坑后得到的

## 怎么联系我们

- GitHub: [github.com/g24-io](https://github.com/g24-io)
- 文章评论 / 提问：在对应文章页底点 _discuss on github_，会跳到该文 markdown 源文件，开 Issue 即可
- 写错了？欢迎 PR

## 内容许可

- **代码片段** · MIT
- **文章正文** · CC BY 4.0
- 转载时请保留作者署名与原文链接，欢迎翻译

## 关于这个站点

- 静态生成器：[Zola](https://www.getzola.org/)（Rust 写的）
- 字体：[Inter](https://rsms.me/inter/) · [JetBrains Mono](https://www.jetbrains.com/lp/mono/)
- 部署：GitHub Pages，源码就是这个仓
- 主题：自己写的，仓内 `templates/` + `sass/`，欢迎 fork
