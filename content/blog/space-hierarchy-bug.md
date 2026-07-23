+++
title = "The Case of the Missing Sub-Room: Tracing a Matrix Hierarchy Bug Across Clients"
date = 2026-07-22
description = "Tracking down a known Continuwuity hierarchy bug — some sub-rooms vanish from rooms[] while children_state stays intact. Why Cinny is unaffected while ement.el and FluffyChat see the gap."

[taxonomies]
tags = ["matrix", "continuwuity", "bug", "spaces", "hierarchy"]
categories = ["tech"]
+++

## The Symptom

A Matrix Space contains two sub-rooms, but one of them simply doesn't show up in certain clients.

- **Cinny**: Fine — both rooms visible
- **ement.el**: One missing
- **FluffyChat**: Also one missing

Packet capture confirmed that the hierarchy API response really doesn't include the missing room — this isn't a client rendering issue, the server simply didn't return it.

<!-- more -->

## Analysis

### How the Three Clients Differ

Looking at the source code of all three clients reveals a key pattern difference:

| Client | Space room list construction |
|--------|---------------------------|
| **Cinny** | Builds the child list from local `m.space.child` state events first, then augments with hierarchy API for room summaries |
| **ement.el** | The space view reads only `rooms[]` from the hierarchy response |
| **FluffyChat** | The space page also reads the hierarchy response directly |

Cinny's code path:
1. Iterates local `m.space.child` state to get all child room IDs
2. For each child, tries to get info from already-known room data
3. Uses the hierarchy API only to fill in missing details (avatar, member count, etc.)

So even if the hierarchy API drops a room, as long as the local state has its `m.space.child` event, Cinny still displays it.

ement.el and FluffyChat, on the other hand, trust the hierarchy response wholesale — whatever is in `rooms[]` is what gets displayed. If it's missing, the room simply doesn't appear.

### Root Cause: A Continuwuity Hierarchy Bug

Looking at the hierarchy implementation in Continuwuity (a community fork of conduwuit, a Rust Matrix homeserver), there's a known issue:

> **[Issue #1789 — hierarchy only returns partial rooms](https://forgejo.ellis.link/continuwuation/continuwuity/issues/1789)**
>
> The symptoms match exactly: `children_state` contains all children, but the top-level `rooms[]` only has a subset.

The issue was still open at the time of analysis.

### Continuwuity's Hierarchy Processing

The source code shows child processing in four phases:

1. **Phase 1** — Read `m.space.child` state of the space, filter out children with empty or invalid `via`
2. **Phase 2** — For each child, attempt to resolve room info (local DB lookup first, then federation query via `via` if not local)
3. **Phase 3** — Permission/visibility checks to decide which rooms go into `rooms[]`
4. **Phase 4** — Recursive handling of sub-spaces (when `max_depth > 1`)

The bug lives in the Phase 2→Phase 3 transition: the declaration passes (so `children_state` has it), but summary resolution or visibility determination fails (so `rooms[]` doesn't).

## Summary

This is a Continuwuity server-side hierarchy bug (Issue #1789), not a client issue. The behavioral differences between the three clients are fully explained by how much they trust the hierarchy API — Cinny falls back to local state, so it's unaffected; ement.el and FluffyChat render `rooms[]` directly, so the bug is fully exposed.
