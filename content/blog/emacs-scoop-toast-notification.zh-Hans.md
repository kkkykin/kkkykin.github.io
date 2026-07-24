+++
title = "Scoop Emacs 的 w32-notification-notify 不弹通知——快捷方式问题"
date = 2026-07-24
description = "排查 Emacs 的 w32-notification-notify 返回正常但通知死活不弹的问题，最后发现是 scoop 安装缺少开始菜单快捷方式。Windows Toast 通知的一些机制值得记一笔。"

[taxonomies]
tags = ["emacs", "scoop", "windows"]
categories = ["tech"]
+++

## 现象

在 Emacs 里调用 `w32-notification-notify`（Emacs C 层封装的 Windows Toast 通知 API）时，返回了一个看起来正常的数值：

```elisp
(w32-notification-notify :title "Emacs Alert" :body "Task completed!")
;; => 43
```

但通知就是不弹——屏幕右下角什么都没有，`Win + N` 通知中心也空空如也。

以前是好的。经历了什么？

<!-- more -->

## 排查过程

### 排除 Windows 设置

- 系统通知总开关 ✅
- 专注助手/请勿打扰 ✅
- 通知历史 ✅
- 逐个排查下来，Windows 自身的通知设置没有异常

### 关键线索

1. `w32-notification-notify` 确认是 Emacs **内置 C 函数**（`symbol-file` 显示 `C source code`）
2. `Get-StartApps | findstr emacs` —— 看不到 Emacs 条目
3. `reg query HKCU\Software\Classes\AppUserModelId` —— 没有 Emacs 的相关项
4. 用的是 Scoop 安装的 **emacs-k**（来自 `kiennq/scoop-misc` bucket）

不看不知道——真正的症结在 Scoop manifest 里。

## 根因：manifest 没写快捷方式

看一下 `emacs-k.json`：

```json
{
  "bin": [
    "bin\\runemacs.exe",
    "bin\\emacs.exe",
    "bin\\emacsclient.exe",
    "bin\\emacsclientw.exe",
    ["bin\\emacsclientw.exe", "emw", "-c -n -a \"\""]
  ]
  // 没有 shortcuts 这个字段
}
```

有 `bin`，有启动器入口，**唯独没有 `shortcuts`**。

对比正常的 Scoop 应用，形如：

```json
{
  "shortcuts": [
    ["bin/emacs.exe", "Emacs"]
  ]
}
```

少了这个，Scoop 就不会在开始菜单里创建 `.lnk` 快捷方式。

### 为什么快捷方式影响通知？

Windows Toast 通知的链路是这样的：

```
应用调用 Toast API
       ↓
系统查询进程的 AppUserModelID (AUMID)
       ↓
AUMID 对应到一个开始菜单快捷方式 (.lnk)
       ↓
快捷方式指向的 exe 被允许显示通知
```

这是一个 **身份证明** 机制：Windows 需要知道"谁在发通知"。对 Win32 桌面程序来说，最稳定（在大部分 Windows 版本上也是唯一有效）的身份注册方式就是开始菜单快捷方式。

Emacs 的 `w32-notification-notify` 内部没有硬编码一个固定的 AUMID（比如 `org.gnu.Emacs`），而是走的默认路径——根据当前 exe 路径推导 AUMID。如果这个 AUMID 找不到对应的 `.lnk`，Windows 会**默默地丢弃**通知。API 调用本身是成功的（所以返回了 `43`），但通知不会出现在屏幕上。

## 解决办法

在 `%APPDATA%\Microsoft\Windows\Start Menu\Programs\` 下创建一个指向 `emacs.exe` 的快捷方式即可。

PowerShell 一行搞定：

```powershell
$shell = New-Object -ComObject WScript.Shell
$link = $shell.CreateShortcut("$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Emacs.lnk")
$link.TargetPath = "$env:USERPROFILE\scoop\apps\emacs-k\current\bin\emacs.exe"
$link.Save()
```

或者右键 `shell:programs` → 新建快捷方式 → 指向 `emacs.exe`。

这个快捷方式 **只是给 Windows 注册身份用的**，不需要用户手动去点它启动 Emacs。事后甚至可以删掉——只要 Windows 缓存里的 AUMID 关联还在，通知仍然能弹。

## 附录：排查中值得记录的 Windows 通知机制

- `Get-StartApps` 输出的第二列不是 exe 路径，而是 AUMID（只是大部分 Win32 应用的 AUMID 长得像路径）
- `HKCU\Software\Classes\AppUserModelId` 不是 AUMID 的唯一存储位置
- `.lnk` 文件本身包含 AUMID 属性，这是 Win32 桌面程序注册身份的规范方式
- 只往注册表写值、不创建 `.lnk`，通知通常还是不工作

---

*排查源于自用的 Scoop Emacs-K。所有 `emacs-k.json` 代码引用对应 [`kiennq/scoop-misc`](https://github.com/kiennq/scoop-misc/blob/master/bucket/emacs-k.json) 仓库。*
