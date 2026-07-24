+++
title = "Emacs w32-notification-notify Silent Failure on Scoop — Missing Start Menu Shortcut"
date = 2026-07-24
description = "Debugging why w32-notification-notify returned success but no notification appeared. The root cause: the scoop manifest lacked the `shortcuts` field, so Windows had no AUMID registration for Emacs."

[taxonomies]
tags = ["emacs", "scoop", "windows"]
categories = ["tech"]
+++

## The Problem

Calling `w32-notification-notify` — Emacs's C-level wrapper for the Windows Toast notification API — returned a seemingly valid value:

```elisp
(w32-notification-notify :title "Emacs Alert" :body "Task completed!")
;; => 43
```

But no toast appeared. The notification center (`Win + N`) was empty too. It used to work — what changed?

<!-- more -->

## Investigation

### Windows settings — ruled out

- System notification toggle: ✅ on
- Focus Assist / Do Not Disturb: ✅ off
- Notification history: ✅ empty
- Nothing unusual in Windows notification settings.

### Key clues

1. `w32-notification-notify` is confirmed to be Emacs's **built-in C function** (`symbol-file` returns `C source code`)
2. `Get-StartApps | findstr emacs` — no Emacs entry
3. `reg query HKCU\Software\Classes\AppUserModelId` — no Emacs entry
4. Emacs installed via Scoop (**emacs-k** from the `kiennq/scoop-misc` bucket)

The real culprit turned out to be the Scoop manifest itself.

## Root Cause: Missing `shortcuts` in the Manifest

Here's the `emacs-k.json`:

```json
{
  "bin": [
    "bin\\runemacs.exe",
    "bin\\emacs.exe",
    "bin\\emacsclient.exe",
    "bin\\emacsclientw.exe",
    ["bin\\emacsclientw.exe", "emw", "-c -n -a \"\""]
  ]
  // no shortcuts field
}
```

It has `bin` entries, launchable from the command line — but **no `shortcuts`**. A typical Scoop GUI app would have:

```json
{
  "shortcuts": [
    ["bin/emacs.exe", "Emacs"]
  ]
}
```

Without `shortcuts`, Scoop never creates a `.lnk` in the Start Menu.

### Why does a shortcut matter for notifications?

Windows Toast notifications work like this:

```
App calls Toast API
       ↓
System queries the process's AppUserModelID (AUMID)
       ↓
AUMID resolves to a Start Menu shortcut (.lnk)
       ↓
The exe the shortcut points to is allowed to show toasts
```

This is an **identity** mechanism: Windows needs to know "who is sending this notification." For Win32 desktop applications, the most reliable way (and on many Windows versions, the *only* way) to register that identity is through a Start Menu shortcut.

Emacs's `w32-notification-notify` doesn't hardcode a fixed AUMID (like `org.gnu.Emacs`). Instead, it follows the default path — the AUMID is derived from the executable's path. When no `.lnk` corresponds to that AUMID, Windows **silently drops** the notification. The API call itself succeeds (hence the `43` return value), but the toast never appears on screen.

## The Fix

Create a shortcut in `%APPDATA%\Microsoft\Windows\Start Menu\Programs\` pointing to `emacs.exe`.

One-liner in PowerShell:

```powershell
$shell = New-Object -ComObject WScript.Shell
$link = $shell.CreateShortcut("$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Emacs.lnk")
$link.TargetPath = "$env:USERPROFILE\scoop\apps\emacs-k\current\bin\emacs.exe"
$link.Save()
```

Or manually: `Win + R` → `shell:programs` → New Shortcut → point to `emacs.exe`.

This shortcut **only serves as an AUMID registration** — you don't need to launch Emacs from it. It can even be deleted afterward; as long as Windows has cached the AUMID association, notifications will still work.

## Appendix: Windows Notification Mechanics Worth Knowing

- The second column of `Get-StartApps` output is actually the **AUMID**, not a file path — most Win32 AUMIDs just happen to look like paths
- `HKCU\Software\Classes\AppUserModelId` is not the only place AUMIDs are stored
- The `.lnk` file itself carries the AUMID property — this is the canonical registration method for Win32 desktop applications
- Writing a registry value alone, without a corresponding `.lnk`, usually won't make notifications work

---

*Debugged against a Scoop-installed Emacs-K. All `emacs-k.json` references from the [`kiennq/scoop-misc`](https://github.com/kiennq/scoop-misc/blob/master/bucket/emacs-k.json) repository.*
