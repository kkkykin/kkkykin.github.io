+++
title = "Debugging Ement.el: Why @mentions Don't Render and TAB Completion Only Works Without @"
date = 2026-07-23
description = "A quick debug session into an Emacs Matrix client: the formatted_body gap in message sending, and a note on how member completion interacts with the @ sigil."

[taxonomies]
tags = ["emacs", "matrix", "ement", "debugging"]
categories = ["tech"]
+++

## The Setup

I've been poking at [ement.el](https://github.com/alphapapa/ement.el), an Emacs Matrix client, and noticed two odd behaviors:

1. Typing `@username` in a message and sending it — the mention arrives as plain text, not clickable. But replies work fine, the `@` link in the reply header is perfectly clickable.
2. TAB completion for usernames works when you type the name *without* a leading `@`. Once you add the `@`, completion either shows "No match" or gives you a bare MXID instead of a display name.

Tracing the code paths cleared things up quickly.

<!-- more -->

## @mentions Don't Render

### The Matrix Protocol Context

In Matrix, a "clickable mention" isn't magic — it's just an `<a>` tag in the message's `formatted_body` (HTML):

```html
<a href="https://matrix.to/#/@user:server">Display Name</a>
```

The plain-text `body` can say `@user:server` all day, but that's just text. It's only when the `formatted_body` contains that link that the client renders it as a clickable pill.

### The Code Path

Ement's interactive send command (`ement-room-send-message`) calls into `ement-send-message` with **only `:body`**, no `:formatted-body`:

```elisp
;; ement-room.el:2141
(list ement-room ement-session :body body)
```

Inside `ement-send-message` (`ement-lib.el:1133`), the mention-linkification step is **gated** on `formatted-body` being non-nil:

```elisp
(formatted-body (when formatted-body              ; ← nil → entire block skipped
                  (ement--format-body-mentions formatted-body room)))
```

Since it's nil, `ement--format-body-mentions` — the function that actually converts `@name` into `<a href="...">` links — is never called. The message goes out with `body` only, no `formatted_body`, so the receiving client just sees plain text.

The `ement-room-send-message-filter` variable defaults to `nil` as well, so the Org-mode filter path (`ement-room-send-org-filter`) that *would* call `ement--format-body-mentions` is also inactive out of the box.

### Why Replies Are Fine

The reply path calls `ement--add-reply` (`ement-lib.el:1144-1146`) which **unconditionally** assembles an `<mx-reply>` HTML block with hardcoded `matrix.to` links:

```elisp
;; ement-lib.el:1190-1199
"<mx-reply><blockquote>
   <a href=\"https://matrix.to/#/%s/%s\">In reply to</a>
   <a href=\"https://matrix.to/#/%s\">%s</a> ..."
```

This is appended to `formatted_body` regardless of whether it was nil. So replies always get a clickable link to the person being replied to — no configuration needed.

### The Workaround

Setting `ement-room-send-message-filter` to `ement-room-send-org-filter` makes it work: the Org filter processes the body through HTML export and `ement--format-body-mentions` before sending.

---

## A Note on TAB Completion Behavior

### Why It Works Without @

The completion function `ement-room--complete-members-at-point` computes the region start by searching backward for the nearest word boundary, then skipping forward over whitespace. For input ` alice`, the start lands on `a`, so the completion string is the bare word `alice`.

The candidate table returns two types of values:

```elisp
collect id     ; "@alice:matrix.org"    — starts with @
collect name   ; "Alice Cooper"         — no @ prefix
```

Default prefix completion requires candidates to start with the input string. `alice` → prefix matches `Alice Cooper` ✓, so it works.

### What Happens With @

With input ` @alice`, the region computation includes the `@` — the completion string is `@alice`. Candidate matching now looks like:

- `Alice Cooper` ❌ — no leading `@`
- `@alice:matrix.org` ⚠️ — only if the localpart happens to match

When display name ≠ localpart (say display name `Alice Cooper`, localpart `acooper`), typing `@Alice` matches neither: `@Alice` doesn't match `@acooper:...` and doesn't match `Alice Cooper` (because the input includes `@`). Result: "No match".

### Contrast: Room Completion

Room completion (`ement-room--complete-rooms-at-point`) handles this correctly. Its regex explicitly consumes the `#`/`!` sigil, and its candidate table returns aliases/IDs that *also* start with `#`/`!`. Consistent on both sides → works perfectly.

### Quick Note on a Fix

If you wanted `@name` to complete too, adding `(concat "@" name)` to the candidate table would do it — `@Alice` would match `@Alice Cooper`, while `@alice` (localpart search) would still match `@alice:matrix.org`.

---

## Takeaways

- **Gated functionality you can't reach isn't dead code — it's a missing default.** `ement--format-body-mentions` existed and worked correctly, but was behind a nil-guard the normal send path could never satisfy.
- **The smallest diffs fix the most confusing UX.** Sometimes a fix is just a one-liner.

---

*Discovered while debugging ement.el against a self-hosted Matrix homeserver. All code references are from commit `790aee7`.*
