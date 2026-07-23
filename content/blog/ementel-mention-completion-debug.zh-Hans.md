+++
title = "调试 Ement.el：为什么 @mention 不渲染、TAB 补全不带 @ 就失效"
date = 2026-07-23
description = "一个 Emacs Matrix 客户端的小 Bug 排查，附一则补全行为的说明。"

[taxonomies]
tags = ["emacs", "matrix", "ement", "debugging"]
categories = ["tech"]
+++

## 起因

最近在折腾 [ement.el](https://github.com/alphapapa/ement.el)（Emacs 上的 Matrix 客户端），发现两个诡异行为：

1. 输入 `@用户名` 发出去——收到的只是纯文本，不能点击。但回复功能正常，回复头里的 `@` 链接完美可点。
2. TAB 补全用户名——不加 `@` 时好好的，加了 `@` 要么 "No match"，要么补出裸 MXID 而不是显示名。

追了一遍代码后，来龙去脉都清楚了。

<!-- more -->

## @mention 发出去不渲染

### Matrix 协议背景

在 Matrix 里，"可点击的提及"不是魔法——它只是消息 `formatted_body`（HTML）里的一个 `<a>` 标签：

```html
<a href="https://matrix.to/#/@user:server">显示名</a>
```

纯文本 `body` 里写 `@user:server` 写再多也没用，它就是文本。只有 `formatted_body` 里有这个链接，客户端才会渲染成可点击的药丸（pill）。

### 代码路径

Ement 的交互式发送命令 `ement-room-send-message` 调用 `ement-send-message` 时**只传了 `:body`，没有 `:formatted-body`**：

```elisp
;; ement-room.el:2141
(list ement-room ement-session :body body)
```

进到 `ement-send-message`（`ement-lib.el:1133`）后，提及链接化的步骤被**门控在 `formatted-body` 非空**这个条件上：

```elisp
(formatted-body (when formatted-body              ; ← nil → 整段跳过
                  (ement--format-body-mentions formatted-body room)))
```

因为它是 nil，`ement--format-body-mentions`——那个真正把 `@名字` 转成 `<a href="...">` 链接的函数——**根本就没被调用**。消息只有 `body` 没有 `formatted_body` 就发出去了，接收端只能看到纯文本。

而且 `ement-room-send-message-filter` 默认也是 `nil`，所以那个*会*走 HTML 导出 + 提及链接化的 Org 过滤器路径也一样不可用。

### 为什么回复正常

回复路径调用 `ement--add-reply`（`ement-lib.el:1144-1146`），它**无条件**拼出 `<mx-reply>` HTML 块，里面硬编码了 `matrix.to` 链接：

```elisp
;; ement-lib.el:1190-1199
"<mx-reply><blockquote>
   <a href=\"https://matrix.to/#/%s/%s\">In reply to</a>
   <a href=\"https://matrix.to/#/%s\">%s</a> ..."
```

不论 `formatted_body` 是不是 nil，这段 HTML 都会追加进去。所以回复始终带有一个指向被回复者的可点击链接——无需任何配置。

### 应急方案

设置 `ement-room-send-message-filter` 为 `ement-room-send-org-filter` 即可：Org 过滤器在发送前对 body 做 HTML 导出并调用 `ement--format-body-mentions`。

---

## 关于 TAB 补全的行为说明

### 不带 @ 时为什么好使

补全函数 `ement-room--complete-members-at-point` 计算补全区域起点的方式是：向后搜索最近的词边界，再向前跳过空白。对于输入 ` alice`，起点落在 `a` 上，补全串是裸词 `alice`。

候选表返回两类值：

```elisp
collect id     ; "@alice:matrix.org"    — 带 @
collect name   ; "Alice Cooper"         — 不带 @
```

默认的前缀补全要求候选以输入串开头。`alice` → 前缀匹配 `Alice Cooper` ✓，所以工作正常。

### 加了 @ 后发生了什么

输入 ` @alice` 时，起点的计算把 `@` 也包括了进去——补全串是 `@alice`。此时候选匹配情况：

- `Alice Cooper` ❌ — 开头没 `@`
- `@alice:matrix.org` ⚠️ — 只有 localpart 正好匹配才命中

当显示名 ≠ localpart（比如显示名 `Alice Cooper`、localpart `acooper`），输入 `@Alice` 两头不靠，结果就是 "No match"。

### 对比：房间补全是好的

房间补全 `ement-room--complete-rooms-at-point` 的正则明确消费 `#`/`!` sigil，候选表里的 alias/ID **也以 `#`/`!` 开头**。两侧一致，所以没问题。

### 简单说明

如果要让 `@名字` 也能补全，在候选表里加一项 `(concat "@" name)` 即可——这样 `@Alice` 能命中 `@Alice Cooper`，同时 `@alice`（localpart 搜索）仍能命中 `@alice:matrix.org`，两边都保住。

---

## 小结

- **够不到的门控功能不是死代码，是缺失的默认值。** `ement--format-body-mentions` 存在、工作正常，但被一个普通发送路径永远满足不了的 nil 守卫挡住了。
- **最小的 diff 修最困惑的 UX。** 有时修复只是一行代码的事。

---

*本文来自对自建 Matrix 服务器上 ement.el 的调试。所有代码引用对应仓库 commit `790aee7`。*
