# M1 Interaction Primitives Design

**Date:** 2026-05-10
**Status:** Implemented

## Overview

M1 closes the input loop: after the split panel TUI lets you *see* what every agent is doing, M1 lets you *act* without attaching. Five features: send to pane (`x`), fork session (`f`), broadcast to group (`b`), pane targeting (`tab`), and improved status heuristics.

No schema changes. Send and broadcast are fire-and-forget tmux calls. Fork reuses `db.CreateSession`.

---

## Feature Summary

| Feature | Key | Description |
|---|---|---|
| Send to pane | `x` | Inline prompt; sends typed text (including ctrl chars) to selected session's active pane |
| Fork session | `f` | Clone selected session's `ProjectPath`, `Tool`, `GroupPath`; prompt for new title; created as `stopped` |
| Broadcast to group | `b` | Send same keys to every running session in a group; Tab toggles scope (direct / include sub-groups) |
| Pane targeting | `tab` | Cycle active pane in detail panel; `x` sends to that pane specifically; resets to 0 on navigation |
| Status heuristics | — | Extend `DetectStatus` with per-tool waiting patterns (aider, copilot, raw bash) |

---

## Architecture

### File Changes

| File | What changes |
|---|---|
| `internal/tmux/client.go` | Add `SendKeys(session string, paneIndex int, keys string) error` to `ClientIface` and `Client` |
| `internal/testutil/tmux.go` | Add `SentKeys []SentKeysCall` and `SendKeys` stub to `FakeTmuxClient` |
| `internal/ui/dialog.go` | Add `scope bool`, `scopeLabels [2]string` to `dialogState`; add `interceptCtrl` helper; extend `updateDialog` and `commitDialog` for new modes; extend `renderDialog` for broadcast scope display |
| `internal/ui/keys.go` | Add `'x': "send-pane"`, `'f': "fork-session"`, `'b': "broadcast"`, `tea.KeyTab: "cycle-pane"` |
| `internal/ui/app.go` | Add `activePaneIdx int` to `Model`; wire `send-pane`, `fork-session`, `broadcast`, `cycle-pane` in `updateNavigation`; reset `activePaneIdx` in `Reload`; highlight active pane in `renderPaneList`; update `renderFooter` to include `x Send  f Fork  b Broadcast` |
| `internal/tmux/status.go` | Add `tool string` param to `DetectStatus`; add per-tool waiting patterns |
| `internal/state/poller.go` | Pass `session.Tool` to `DetectStatus` |

---

## tmux Client

### Interface methods

```go
SendKeys(session string, paneIndex int, keys string) error
SendRawKeys(session string, paneIndex int, keys string) error
```

**`SendKeys`** — sends literal text via `tmux send-keys -l -t session:paneIndex keys`. The `-l` flag ensures the argument is interpreted as literal characters, not tmux key names. No-op if `keys` is empty.

**`SendRawKeys`** — sends a tmux key name (no `-l` flag) via `tmux send-keys -t session:paneIndex keys`. Used exclusively for control sequences like `C-c`, `C-d`, `Enter`. No-op if `keys` is empty.

### `FakeTmuxClient` additions

```go
type SentKeysCall struct {
    Session    string
    PaneIndex  int
    Keys       string
}

SentKeys    []SentKeysCall   // literal text calls via SendKeys
SentRawKeys []SentKeysCall   // key name calls via SendRawKeys
```

`SendKeys` appends to `SentKeys`; `SendRawKeys` appends to `SentRawKeys`. Both return nil.

---

## Dialog Infrastructure

### `dialogState` changes

```go
type dialogState struct {
    prompt      string
    value       string
    ctrlKeys    []string    // send-pane/broadcast only: intercepted ctrl key names (e.g. "C-c")
    scope       bool        // broadcast only: false = direct group, true = include sub-groups
    scopeLabels [2]string   // broadcast only: e.g. {"this group", "all sub-groups"}
}
```

`ctrlKeys` holds tmux key names captured by `interceptCtrl`. They are dispatched via `SendRawKeys` separately from the literal text in `value`.

### `interceptCtrl` helper

Pure function in `dialog.go`:

```go
func interceptCtrl(msg tea.KeyMsg) (tmuxKey string, isCtrl bool)
```

Maps bubbletea ctrl key types to tmux key names:

| bubbletea | tmux key |
|---|---|
| `tea.KeyCtrlC` | `C-c` |
| `tea.KeyCtrlD` | `C-d` |
| `tea.KeyCtrlZ` | `C-z` |
| `tea.KeyCtrlL` | `C-l` |
| `tea.KeyCtrlU` | `C-u` |

Returns `("", false)` for unmapped keys. `Enter` and `Esc` are not intercepted — they retain their submit/cancel roles.

### `updateDialog` extensions

For modes `"send-pane"` and `"broadcast"`:
- Ctrl keys → `interceptCtrl` → if matched, append tmux key name to `dialog.ctrlKeys` (not `dialog.value`)
- Tab in `"broadcast"` mode → toggle `dialog.scope`
- Enter → submit; Esc and `ctrl+c` → cancel (clear `m.mode`); Backspace → existing behavior
- `ctrl+c` in `"send-pane"`/`"broadcast"` is intercepted before the Esc/cancel check, so it appends to `ctrlKeys` rather than cancelling the dialog

### `renderDialog` broadcast display

```
Broadcast [→ this group / all sub-groups]:
> C-c_
```

Active scope shown with `→`. Non-active scope is dim.

### `commitDialog` new cases

**`"send-pane"`:** Dispatches via `sendToPane(session, paneIndex)`. No-op if both `dialog.value` and `dialog.ctrlKeys` are empty. `sendToPane` calls `SendKeys` with the literal text (if non-empty) then `SendRawKeys` for each ctrl key in order.

**`"fork-session"`:** Creates a new `db.Session` copying `ProjectPath`, `Tool`, `GroupPath` from the current session. Title is `m.dialog.value` (trimmed). Status is `"stopped"`. `CreatedAt` is `time.Now().Unix()`. `ID` is `uuid.New().String()`.

**`"broadcast"`:** Resolves the target group: if cursor is on a group, use `group.Path`; if cursor is on a session, use `session.GroupPath`. Opens with `dialogState{scopeLabels: [2]string{"this group", "all sub-groups"}}`. Collects sessions from `m.sessions` matching the group scope, filters to `status == "running"`, calls `sendToPane` for each. No-op if both `dialog.value` and `dialog.ctrlKeys` are empty. Scope logic:
- `scope=false`: sessions where `GroupPath == group.Path`
- `scope=true`: sessions where `GroupPath == group.Path` or `strings.HasPrefix(GroupPath, group.Path+"/")`

---

## Pane Targeting

`Model` gains `activePaneIdx int`. 

`Reload` resets `activePaneIdx = 0` on every call (cursor change or tick).

`"cycle-pane"` action in `updateNavigation`:
```go
case "cycle-pane":
    if len(m.panes) > 0 {
        m.activePaneIdx = (m.activePaneIdx + 1) % len(m.panes)
    }
```

`renderPaneList` highlights the entry at `activePaneIdx` with `selectedStyle`; all others use `dimStyle`.

New accessor: `func (m *Model) ActivePaneIdx() int { return m.activePaneIdx }`.

---

## Status Heuristics

### `DetectStatus` signature change

```go
func DetectStatus(output string, lastChange time.Time, tool string) Status
```

### Per-tool waiting patterns

Checked before the existing fallback:

| Tool | Waiting pattern |
|---|---|
| `"claude"` | ends with `> ` or `>` |
| `"aider"` | ends with `aider> ` |
| `"copilot"` | ends with `❯ ` or `> ` |
| `""` / other | ends with `$ `, `# `, or `> ` |

Running patterns (spinners, `"Thinking"`, `"Running"`) remain tool-agnostic.

### `poller.go` change

`PollOnce` already reads `session.Tool` from the DB row. Pass it as the third argument to `DetectStatus`.

---

## Keys

```go
// keyTypeMap additions
tea.KeyTab: "cycle-pane",

// runeMap additions
'x': "send-pane",
'f': "fork-session",
'b': "broadcast",
```

`"cycle-pane"` is only processed in navigation mode. In dialog mode, Tab is handled directly inside `updateDialog`.

---

## Testing

### `internal/tmux/client_test.go`
- `TestSendKeysTargetFormat` — verifies `-t session:paneIndex` formatting

### `internal/ui/dialog_test.go` (new)
- `TestInterceptCtrl` — table-driven: all mapped ctrl keys return correct tmux name; unmapped keys return `("", false)`

### `internal/ui/app_test.go` additions
- `TestSendPaneCallsSendKeys` — types `C-c`, Enter; asserts `fake.SentKeys[0]` has correct session + pane index
- `TestSendPaneNoOpWithoutTmuxSession` — no `TmuxSession`, no call, no error
- `TestForkSessionClonesFields` — fork; assert new DB row has matching `ProjectPath`/`Tool`/`GroupPath` and status `"stopped"`
- `TestBroadcastDirectGroup` — two sessions in group + one in sub-group; `scope=false`; only two receive `SendKeys`
- `TestBroadcastIncludesSubGroups` — same setup; `scope=true`; all three receive it
- `TestBroadcastSkipsNonRunning` — running + stopped sessions; only running receive it
- `TestCyclePaneAdvancesIndex` — Tab three times over three panes; index wraps to 0
- `TestCyclePaneResetsOnReload` — set index to 1, Reload, assert index is 0

### `internal/tmux/status_test.go` additions
- Table-driven rows for each new tool/pattern combination
