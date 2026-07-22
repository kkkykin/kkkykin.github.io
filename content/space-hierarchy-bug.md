+++
title = "Space 子 Room 隐身之谜：一个 Matrix Hierarchy Bug 的跨客户端追踪"
date = 2026-07-22
description = "追踪一个 Continuwuity hierarchy API 的已知 bug——部分子 room 在 rooms[] 中消失，而 children_state 完好。为什么 Cinny 没事，ement.el 和 FluffyChat 却中招？"

[taxonomies]
tags = ["matrix", "continuwuity", "bug", "spaces", "hierarchy"]
categories = ["技术"]
+++

## 现象

某个 Matrix Space 里有两个子 room，但其中一個在某些客户端里就是不显示。

- **Cinny**：正常，两个 room 都看得见
- **ement.el**：少一个
- **FluffyChat**：也少一个

抓包确认 hierarchy API 的响应里确实没有缺失的那个 room——不是客户端渲染问题，是服务端就没返回。

<!-- more -->

## 分析

### 三个客户端处理方式的差异

把三个客户端的源码拉下来对比，发现了一个关键的模式差异：

| 客户端 | Space room 列表构建方式 |
|---|---|
| **Cinny** | 先从本地 `m.space.child` state 事件构建 child 列表，再用 hierarchy API 补充房间摘要 |
| **ement.el** | space 专页只读 hierarchy 响应中的 `rooms[]` |
| **FluffyChat** | space 页面也只读 hierarchy 响应 |

Cinny 的代码路径是：
1. 遍历本地 `m.space.child` state 得到所有 child room ID
2. 对每个 child，尝试从已有 room 数据中获取信息
3. 用 hierarchy API 补充缺失的摘要（头像、成员数等）

所以即使 hierarchy API 漏掉了某个 room，只要本地 state 里有它的 `m.space.child` 事件，Cinny 仍然能展示它。

而 ement.el 和 FluffyChat 直接信任 hierarchy 响应——`rooms[]` 里有什么就显示什么，少了就真的不显示。

### 根因：Continuwuity hierarchy bug

查看 Continuwuity（conduwuit 的社区续作，一个 Rust Matrix homeserver）的 hierarchy 实现源码，发现一个已知 issue：

> **[Issue #1789 — hierarchy 只返回部分 rooms](https://forgejo.ellis.link/continuwuation/continuwuity/issues/1789)**
>
> 现象完全吻合：`children_state` 包含所有 child，但顶层的 `rooms[]` 只有一部分。

截止分析时该 issue 仍为 open 状态。

### Continuwuity hierarchy 处理流程

源码显示 hierarchy 的 child 处理分四个阶段：

1. **Phase 1** — 读取 space 的 `m.space.child` state，过滤 `via` 为空或无效的 child
2. **Phase 2** — 对每个 child 尝试解析房间信息（优先查本地 DB，不在本地则走 federation 查询 `via`）
3. **Phase 3** — 权限/可见性检查，决定哪些 room 进入 `rooms[]`
4. **Phase 4** — 递归处理 subspace（当 `max_depth > 1` 时）

bug 出现在 Phase 2→Phase 3 的过渡阶段：声明通过了（所以 `children_state` 里有），但摘要解析或可见性判断出了问题（所以 `rooms[]` 里没有）。

## 总结

这是一个 Continuwuity 服务端的 hierarchy bug（Issue #1789），不是客户端的问题。三个客户端的行为差异完全可以用它们对 hierarchy API 的信任程度来解释——Cinny 用本地 state 兜底，所以不受影响；ement.el 和 FluffyChat 直接渲染 `rooms[]`，bug 一览无余。
