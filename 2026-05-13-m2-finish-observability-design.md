# M2 Finish Observability Design

**Date:** 2026-05-13
**Status:** Implemented

## Overview

M2 is a finish-pass on observability signals that were partially in place after M1/M6. Three features shipped: a context window percentage indicator parsed from Claude pane output, color polish on the fleet header, and overdue waiting timer color in the session list.

Output-tail search was explicitly dropped from scope — session-list filtering (M6) already covers the higher-value navigation problem and `/` was already bound.

---

## Features

### Context Window Indicator

Parses Claude's usage footer (e.g. `75% context used · /compact to reduce`) from live pane output. Rendered as a 4-block bar:

```
▓▓▓░ 75%
```

Shown in two places:
- **Session list** — inline between the wait label and the session title
- **Detail panel** — `context: ▓▓▓░ 75%` line below tags

The percentage is in-memory only (not persisted to DB). It is a live transient reading that changes every poll cycle.

### Fleet Header Color Treatment

- **Waiting count** — amber (`#214`) when > 0
- **Error count** — red (`#196`) when > 0
- **Overdue badge** (`!N`) — bold red when any session has been waiting > 30s

### Overdue Wait Label Color

Session rows with a wait label show it in amber when the session has been waiting > 30 seconds (`WaitOverdue = true` on `ListItem`).

---

## Architecture

### Data Flow

```
tmux pane output
    → ParseContextPct()          [internal/tmux/status.go]
    → Poller.setContextPct()     [internal/state/poller.go]
    → Poller.ContextPctSnapshot()
    → Model.Reload()             [internal/ui/app.go]
    → ListItem.ContextPct *int
    → RenderContextBar()         [internal/ui/list.go]
    → list row / detail panel
```

### Key Design Decisions

**Context % not persisted.** Storing it in the DB would require a schema migration and constant writes on every poll cycle for a value that resets anyway. In-memory via the poller's `contextPct map[string]*int` is sufficient.

**`ParseContextPct` is separate from `DetectStatus`.** No signature change to `DetectStatus` as part of M2. The two concerns are independent — status detection is tool-aware, context % parsing is not.

> **Post-implementation note (2026-05-15):** `DetectStatus` signature was subsequently changed by the BUG-005 fix to `DetectStatus(output string, lastChange, now time.Time, tool string)`. The `now` parameter replaced the internal `time.Since(lastChange)` call so the poller's injected clock is used consistently, and the idle check was moved before the spinner/Thinking heuristic to fix stale-output false positives. See `docs/bugs.md` BUG-005 for full details.

**Pattern matching.** Lines must contain `context used`, `context window`, or `% context` to avoid false positives from unrelated percentage values.

---

## File Changes

| File | Change |
|---|---|
| `internal/tmux/status.go` | Add `ParseContextPct(output string) *int` |
| `internal/state/poller.go` | Add `contextPct map[string]*int`, `ContextPctSnapshot()`, `setContextPct`, call in `PollOnce`, clear in `clearSessionState` |
| `internal/ui/list.go` | Add `ContextPct *int` and `WaitOverdue bool` to `ListItem`; add `RenderContextBar`; add `overdueWaitStyle`; update `RenderList` |
| `internal/ui/app.go` | Add `contextPct map[string]*int` to `Model`; populate in `Reload`; set `item.ContextPct` and `item.WaitOverdue`; add context line to `RenderDetailPanel`; color header via `headerWaitingStyle`, `headerErrorStyle`, `headerOverdueStyle` |

---

## ParseContextPct Patterns

Matched phrases (case-insensitive, per line):
- `context used` — e.g. `75% context used · /compact`
- `context window` — e.g. `context window: 50%`
- `% context` — generic fallback

Returns `nil` if no match or percentage is outside 0–100.

---

## RenderContextBar

```
filled = (pct * 4) / 100
bar = "▓" × filled + "░" × (4 - filled)
output = "{bar} {pct}%"
```

Examples: `░░░░ 0%`, `▓░░░ 25%`, `▓▓░░ 50%`, `▓▓▓░ 75%`, `▓▓▓▓ 100%`
