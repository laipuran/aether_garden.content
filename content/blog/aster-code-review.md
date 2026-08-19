---
slug: aster-code-review
title: tatetian 关于 aster-code-review 的讲座
date: 2026-08-18
excerpt: tatetian 根据 Asterinas 的实践，让 AI 从维护者的角度来审阅代码，包括与 Linux 类似模块的对比。
tags:
  - open-source
  - os
  - skills
status: published
updatedAt: 2026-08-19
---

参加了 CCF 开源贡献赛并进入了决赛，因而得到了去蹭 CCF 论坛的机会，15号下午去听了几场，一场是本文，另一场比较有意思的是 AI 背景下代码的复用，和如何使用 LLM 编写软件测试，在此作一个备忘。

### 宗旨

目的是从维护者的角度，尽可能多的提出问题，他们团队为 skill 设定了不同的 persona，用于不同维度的评审，同时设定了大量的，针对不同 scope 的 code guidelines，来确保 AI 对特定代码使用正确的评审规则。另一个则是将规则制定为行为，这样可以固化 AI 行为，减少幻觉。

### A New Mindset

"think like a scientist"：observer → theory → experiment → science，像科学家一样思考。三条落地实践：

- **expose the full text of guidelines on demand** — 按需暴露指南全文：先给 gist，疑似违规才展开完整规则，而不是一次性全塞给 LLM。
- **make review guidance actionable** — 指南要写得可执行、能直接照做，而不是泛泛的建议。
- **keep a log of review items** — 保存评审记录，包括被驳回的结论。

### 如何进一步改进

- **Making the Git history an asset** — benchmark 的每个评审问题都携带一个从真实 PR 评审历史中收集的已知缺陷，让仓库自身的 Git 历史成为度量素材。
- **Measured, not asserted** — 质量靠 benchmark 度量，不靠断言；指南与 harness 的改进以证据为准。
- **Recall is the headline metric** — 召回率是首要指标，宁可激进；误报率留作下一步再量化。
- **Every miss is an actionable signal** — 每条漏报都指向缺失、含糊或划界错误的指南，或 harness 的弱点；修好它，就是改进的具体路径。

### 和 Linux 的对比

Linux 对应的是 Sashiko（刺し子）：一个由 Linux Foundation 托管的 agentic 内核评审系统，监听 linux-kernel 邮件列表上的每个新补丁，跑一个模拟评审团队的多阶段流水线，算力资源由平台方提供。它只保留能通过具体代码级冲突检查和验证的结论，评审比较保守，保证意见站得住。

而 Asterinas 选择自研这个 skill，可以在本地跑节省团队开销，也可以在 PR 中线上调用；并且主动发出基于指南的主观意见，采用激进策略报告问题——因为结果给发起 PR 的人看，所以对 PR owner 提出了明辨真伪的要求。
