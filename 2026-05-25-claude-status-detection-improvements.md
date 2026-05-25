# Claude Status Detection Improvements

**Date:** 2026-05-25

## Problem

Three related detection gaps in `DetectStatus` for Claude Code sessions:

1. **False waiting during active generation.** Claude Code always renders a `❯` input box at the bottom of the pane, even while the model is actively generating a response. The previous logic matched bare `>` or `❯` as "waiting" before checking any running indicators, causing sessions to flash "waiting" mid-run.

2. **❯ prefix not matched.** Claude Code renders `❯` followed by display text (e.g. `❯ Stop the escalation recaps`). The old `isAgentPromptLine` did an exact match (`line == "❯"`), so the prompt-with-text variant was never detected as waiting.

3. **No idle state for ❯ prompt.** A session at the `❯` input prompt with no pane activity for 60+ seconds is effectively idle (the user or another agent is not responding). There was no path from waiting to idle.

## Solution

### Running indicators checked first

Introduce `isClaudeRunningLine(line string) bool` that returns true for any of:

- `✢` (active spinner, Unicode U+2722) — Claude's in-progress token generation spinner
- `· ↓` — appears in the token-stream status line during generation
- `"thinking with"` — appears when extended thinking is active

When pane activity is recent (`now - lastChange ≤ 30s`), scan the 8 most-recent non-empty lines for a running indicator **before** checking for the ❯ prompt. If one is found, return `StatusRunning` immediately. The 30s guard prevents a frozen ✢ line (stale pane) from pinning the session as running indefinitely.

`✻` (completed-step marker) is intentionally excluded — it indicates a finished step, not active generation.

### ❯ prefix match

Change `isAgentPromptLine` from exact-equals to `strings.HasPrefix(line, "❯")` so both the bare `❯` and `❯ <display text>` forms are matched.

### Idle after 60 seconds at ❯ prompt

When the ❯ prompt is detected but `now - lastChange ≥ 60s`, return `StatusIdle` instead of `StatusWaiting`. This models the real state: the agent finished and no one has responded for a minute.

## Behavioral Summary

| Pane state | lastChange age | Result |
|-----------|---------------|--------|
| ✢ / `· ↓` / `thinking with` visible | ≤ 30s | `running` |
| ✢ visible | > 30s | falls through to ❯ check |
| `❯` (bare or with text) | < 60s | `waiting` |
| `❯` (bare or with text) | ≥ 60s | `idle` |
| Neither | any | existing fallback logic |

## Additional Changes (same commit)

### Restore cursor after attach

When the user attaches to a session and returns to the TUI (`ctrl+q` / tmux detach), the cursor was previously reset to position 0 (top of list). The deck now records `PendingAttachSessionID` before exiting, passes it through `launchTUI`'s re-launch loop as `restoreSessionID`, and `Reload()` scans the new item list to restore the cursor to that session ID.

### Conductor toggle

Pressing `c` on the session that is already the group conductor now clears the conductor assignment (sets it to `""`). Previously `c` was a one-way set, so the only way to un-set a conductor was to assign a different session or edit the DB directly. The key description in the help table updated from "Set conductor" to "Toggle conductor".
