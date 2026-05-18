# Auto-Escalation Design

**Date:** 2026-05-16
**Status:** Implemented

## Overview

When a worker session transitions to `waiting` and its group has a conductor set, automatically send the escalation message to the conductor's tmux pane — the same message the user sends manually with `C`. Opt-in via `--auto-escalate` flag.

## Motivation

The manual `C` workflow requires the user to notice a waiting session and press the key. Auto-escalation lets a conductor agent manage workers autonomously: it receives context the moment a worker blocks and can respond without human intervention.

## Scope

- Fires once per `→ waiting` transition (not on repeated poll cycles while already waiting)
- Opt-in: only active when `--auto-escalate` is passed at startup
- No new UI, no new DB columns, no changes to the escalation message format

## Architecture

### `internal/state/poller.go`

Add `TmuxSender` interface:

```go
type TmuxSender interface {
    SendKeys(session string, pane int, keys string) error
}
```

Add optional `sender TmuxSender` field to `Poller`. Add constructor `NewWithSender(conn, tc, notifier, sender)` that wires it in. When `sender` is nil the feature is disabled.

Add package-level pure function:

```go
func escalationMessage(session db.Session, lastOutput string) string
```

Produces:
```
Escalation from <title>
Status: waiting
Notes: <notes>           // omitted if empty
Current issue context:
<last 3 lines of output> // omitted if blank
```

This is the same format as the manual `C` escalation in `ui/app.go`. The existing `ui/app.go` implementation is left in place — it reads from TUI state and is used by the interactive key handler. The poller version takes raw strings so it has no UI dependency.

Add method `autoEscalate(session db.Session)` called from `PollOnce` at the existing `→ waiting` transition point, alongside `notifyWaiting`.

### Data Flow

```
PollOnce()
  └─ session transitions to waiting
       ├─ notifyWaiting(s)          ← existing, unchanged
       └─ autoEscalate(s)           ← new
            ├─ sender == nil → return
            ├─ db.GetGroupConductorSession(groupPath) → conductor
            ├─ no conductor or same session → return
            ├─ conductor not running → log and return
            └─ sender.SendKeys(conductor.TmuxSession, 0, escalationMessage(s, lastOutput[s.ID]))
```

`lastOutput[s.ID]` is already populated by `observeOutput` at the time of the transition — no extra tmux capture needed.

### `cmd/root.go`

Add `--auto-escalate` boolean flag. When true, pass the `tmux.Client` (already implements `TmuxSender` via its existing `SendKeys` method) to the new constructor.

### `testutil/tmux.go`

Add `SendKeys` recording to `FakeTmuxClient`:

```go
type SentKeys struct {
    Session string
    Pane    int
    Keys    string
}
SentKeys []SentKeys
```

## Error Handling

Errors from `autoEscalate` are logged (`log.Printf`) and discarded — same pattern as `notifyWaiting`. The poller is a background goroutine; delivery failures cannot be surfaced to the user meaningfully. If the conductor session disappears between cycles, `GetGroupConductorSession` returns an error or empty session and the method returns silently.

## Testing

**`internal/state/poller_test.go`** — new cases:
- Auto-escalation fires on `→ waiting` transition when sender is set and conductor exists
- Does not fire on subsequent polls while session remains waiting (once-per-transition)
- Does not fire when sender is nil
- Does not fire when no conductor is set for the group
- Does not fire when the waiting session is itself the conductor
- Does not fire when conductor status is not running

**`internal/state/escalation_message_test.go`** (or added to poller_test.go) — table-driven:
- Notes present and absent
- Output present and absent
- Trims blank output lines correctly

---

## Post-Implementation Notes

**Implemented 2026-05-16.** The design was followed with two deviations:

1. **`NewWithSender` constructor not used.** Instead a `SetSender(s TmuxSender)` method was added to `Poller`. The CLI calls `poller.SetSender(tc)` when `--auto-escalate` is true. This keeps the constructor signatures consistent with `NewWithNotifier` / `NewWithClock` pattern and avoids a combinatorial explosion of constructors.

2. **Conductor availability check expanded.** The design said "conductor not running → log and return." The implementation also skips conductors in `waiting` state (not just non-running ones) to avoid interrupting a conductor that is itself waiting for input. This was later relaxed: conductors in `waiting` or `idle` state are allowed to receive escalations (only `stopped` and `error` are skipped), since an idle/waiting conductor can still receive keys via SendKeys.

3. **Escalation sent as a single-line message.** The escalation message is passed to `SendKeys` as one call rather than line-by-line, to avoid tmux interpreting embedded newlines as separate key events.

4. **E2E test added** in `test/e2e/` that spins up real tmux sessions and verifies the full auto-escalation path end-to-end.

---

## Iteration 2 — 2026-05-17 Improvements

### Escalation submission fix

The original implementation sent the escalation message via `SendKeys` (which uses `tmux send-keys -l`) but never submitted it — the conductor received the text typed into its prompt but the agent did not execute it. Fixed by adding `SendRawKeys(session, 0, "Enter")` after `SendKeys`, matching the pattern used by the manual `C` escalation in `ui/app.go`.

`TmuxSender` interface updated to include `SendRawKeys`:

```go
type TmuxSender interface {
    SendKeys(session string, pane int, keys string) error
    SendRawKeys(session string, pane int, keys string) error
}
```

### Context line filtering

`escalationMessage` previously included the last 3 raw lines of pane output as context. Terminal UI chrome (prompt lines, status bar, separator characters) was polluting the context sent to the conductor.

`tailLines` replaced with `contextLines` + `isContextLine` filter:

- Scans backward through output, collecting up to 5 non-empty lines
- Skips: bare `>` or `❯` prompt lines, lines containing `-- INSERT --`, lines matching the Claude status bar pattern (`ctx:` and `@`)
- Skips: lines that are entirely separator characters (`─━═-`)
- Reverses the collected lines to restore chronological order

Context window expanded from 3 to 5 lines to compensate for filtered noise.

### Claude prompt detection improvement (`internal/tmux/status.go`)

`DetectStatus` for the `claude` tool previously checked only the last line for a `>` prompt. Claude's terminal output often has trailing empty lines or status bar lines after the actual prompt, causing the prompt to be missed.

Fixed by scanning the most recent 8 non-empty lines via `recentNonEmptyLines`:

```go
case isClaudeTool(tool):
    for _, line := range recentNonEmptyLines(trimmed, 8) {
        if isAgentPromptLine(line) {
            return StatusWaiting
        }
    }
```

`isAgentPromptLine` matches `">"` and `"❯"` exactly (trimmed), so partial matches are not false positives.

Also added `lastNonEmptyLine` helper used by other tool cases to avoid being fooled by trailing whitespace/newlines in pane output.

---

## Iteration 3 — 2026-05-18 Status Detection Hardening

Claude prompt detection was hardened after a real `claude-dangerous` session showed a visible `❯` prompt but still transitioned to `idle`.

Two additional cases are now covered:

- ANSI-wrapped prompts from `tmux capture-pane`, such as `\x1b[32m❯\x1b[0m`, are normalized by stripping ANSI CSI escape sequences before status matching.
- Claude-family tool names (`claude` and `claude-*`) use the Claude footer-aware detector. This includes the `claude-dangerous` preset, which previously fell through to shell-style detection and missed prompts above Claude's footer.

Regression tests:

- `TestDetectStatusClaudePromptWithANSI`
- `TestDetectStatusClaudePresetPromptAboveStatusFooter`
