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

### 其他已知相关 bug

- **[Issue #1522](https://forgejo.ellis.link/continuwuation/continuwuity/issues/1522)** — `roomid_spacehierarchy_cache` 未随 room event 失效，旧版 `v0.5.x` 可能持续返回过时的 children/member count
- **`v0.5.10` federation 限制** — 只检查 `FuturesUnordered::next()` 的第一个完成结果。最快完成的 `via` 如果失败，其他仍可能成功的请求也会被放弃。新版 `v26.6.2` 已修复为逐个尝试全部 `via`
- **`max_depth=1` off-by-one** — 曾导致只返回 space 本身不返回 child（已在 2025 年修复）

## 判断方法

一次 hierarchy 请求的两个位置提供了诊断线索：

1. **child 不在 `children_state`** → 检查 `via` 是否缺失/为空/格式非法、suggested_only 过滤
2. **child 在 `children_state`，但不在 `rooms[]`** → 声明已通过，问题在摘要解析、federation 或可见性判断

同时在服务端日志中可以观察到：
- `Preparing local summary for room`：走本地路径，`via` 不应影响结果
- `Asking for room summary over federation ... via=[...]`：服务端未识别为本地 room，所有列出的 `via` 都失败后会静默省略（新版已修复单个 `via` 失败就放弃的问题）

## 总结

这是一个 Continuwuity 服务端的 hierarchy bug（Issue #1789），不是客户端的问题。三个客户端的行为差异完全可以用它们对 hierarchy API 的信任程度来解释——Cinny 用本地 state 兜底，所以不受影响；ement.el 和 FluffyChat 直接渲染 `rooms[]`，bug 一览无余。

对日常使用影响不大：Cinny 正常，Element 等其他客户端的行为取决于它们是否也做了本地 state 回退。Bug 修复等待上游发布即可。

如果你想亲自验证自己的 space 是否受影响，对比 hierarchy 响应中的 `children_state` 和 `rooms[]` 是最快的判断方式。
