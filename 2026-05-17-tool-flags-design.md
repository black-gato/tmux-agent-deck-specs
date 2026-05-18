# Tool Flags Design

**Date:** 2026-05-17
**Status:** Partially implemented — free-text flags broken (BUG-013); `claude-dangerous` preset works

## Overview

Allow per-session flags to be passed to the agent tool at launch time (e.g. `--dangerously-skip-permissions`, `--model opus`, `--resume`). Flags are stored in the DB alongside the session and applied every time the session starts.

## Motivation

Users frequently need to start `claude` with flags. Without this feature they must manually type flags into the shell every time they attach to a new session, or remember to set them before attach. The `--dangerously-skip-permissions` flag in particular is needed for agentic workflows where claude would otherwise block on every file write.

## Scope

- Per-session only (not group defaults — group defaults use `default_tool` which covers the tool name; flags are session-specific)
- Free-text field: any arbitrary flags the user types
- Applied at session creation time, not on re-attach to an existing session
- A hardcoded preset (`claude-dangerous`) covers the most common case while the general mechanism is under investigation

---

## Architecture

### Schema — `internal/db/db.go`

Schema v6 migration adds `tool_flags TEXT NOT NULL DEFAULT ''` to the `sessions` table:

```go
if version == "5" {
    if _, err := conn.Exec(`ALTER TABLE sessions ADD COLUMN tool_flags TEXT NOT NULL DEFAULT ''`); err != nil {
        return err
    }
    if _, err := conn.Exec(`UPDATE metadata SET value = '6' WHERE key = 'schema_version'`); err != nil {
        return err
    }
}
```

### Session struct — `internal/db/sessions.go`

`ToolFlags string` added to `Session`. All INSERT, SELECT, and Scan call sites include `tool_flags`.

### Launch command — `internal/tmux/client.go`

`buildLaunchCommand(tool, toolFlags string) string` combines tool and flags into a single shell command string:

```go
func buildLaunchCommand(tool, toolFlags string) string {
    base := resolveLaunchCommand(tool)
    if base == "" { return "" }
    flags := strings.TrimSpace(toolFlags)
    if flags == "" { return base }
    return base + " " + flags
}
```

`resolveLaunchCommand` maps special tool names to their shell command:

```go
func resolveLaunchCommand(command string) string {
    switch strings.TrimSpace(command) {
    case "shell":
        return "zsh -il"
    case "claude-dangerous":
        return "claude --dangerously-skip-permissions"
    default:
        return command
    }
}
```

`NewSession` passes the combined string as a single positional arg to `tmux new-session`. Tmux runs it as `$SHELL -c "command"`.

### Dialog — `internal/ui/dialog.go`

The new-session flow gains a 5th step (step 3, after tool selection, before startup script):

```
Step 0: title
Step 1: project path
Step 2: tool selection (arrow keys)
Step 3: tool flags (free text, optional)  ← new
Step 4: startup script (optional)
```

`dialogState` gains `savedToolFlags string`. `advanceNewSessionStep` advances through the flags step, saving the value to `savedToolFlags`. `commitDialog` passes `ToolFlags: m.dialog.savedToolFlags` to `db.CreateSession`.

### Detail panel — `internal/ui/app.go`

When a session has `ToolFlags` set, the detail panel shows:
```
tool: claude  flags: --dangerously-skip-permissions
```

---

## Current State (post-implementation)

### What works

- `ToolFlags` is stored in the DB and round-trips correctly through all SQL queries
- The 5-step dialog captures and persists tool flags
- The `claude-dangerous` tool option (selectable in step 2) resolves to `claude --dangerously-skip-permissions` via `resolveLaunchCommand` — this path works reliably
- `claude-dangerous` is treated as a Claude-family tool by status detection, so Claude footer prompts above the status bar transition to `waiting` instead of falling through to shell-style `idle`
- Tool and flags are visible in the detail panel

### Status detection note

`claude-dangerous` stores the session `Tool` as `"claude-dangerous"`, but the running process is still Claude. `DetectStatus` must therefore use Claude's footer-aware prompt detection for `claude-*` tools, not the shell/default detector. The detector also strips ANSI CSI escape sequences from `tmux capture-pane` output before matching prompt lines so colored `❯` prompts still count as `waiting`.

### What is broken — BUG-013

Free-text `ToolFlags` values entered in step 3 are not reliably applied to the agent process. Two approaches were tried:

**Approach 1 — combined string to `tmux new-session`**

`buildLaunchCommand("claude", "--flags")` returns `"claude --flags"` passed as a single positional arg to `tmux new-session`. Tmux should run `$SHELL -c "claude --flags"`, which the shell parses to pass `--flags` as a separate argv element to claude. In practice the agent started without the flags. Root cause not confirmed.

**Approach 2 — bare shell + `send-keys -l`**

`ensureStarted` started a bare zsh session (no tool). `PendingStartupScript = "claude --flags\n"` was sent via `tmux send-keys -l` before attach. The literal LF byte (0x0A) was intended to execute the command, but `tmux send-keys -l` sends characters literally and a bare LF does not reliably trigger execution in all PTY/shell configurations.

**Current code** uses Approach 1 (the combined string), since the bare-shell approach is also unreliable and more complex. The `claude-dangerous` preset bypasses the issue entirely by putting the full command in `resolveLaunchCommand`.

### Workaround

Select `claude-dangerous` as the tool (step 2) instead of `claude`. This maps directly to `claude --dangerously-skip-permissions` without going through the flags field.

---

## Testing

- `internal/db/sessions_test.go` — `TestToolFlagsRoundTrip`: creates session with `ToolFlags: "--model opus"`, reads back, verifies round-trip
- `internal/db/db_test.go` — `TestOpenCreatesSchemaVersion` expects `"6"`
- `internal/tmux/client_test.go` — `TestBuildLaunchCommand`: 6 cases covering empty flags, with flags, multiple flags, shell tool with flags, empty tool, trimmed flags
- `internal/tmux/client_test.go` — `TestResolveLaunchCommand`: includes `claude-dangerous → "claude --dangerously-skip-permissions"`
- `internal/tmux/status_test.go` — `TestDetectStatusClaudePresetPromptAboveStatusFooter`: `claude-dangerous` uses Claude prompt detection above the footer
- `internal/tmux/status_test.go` — `TestDetectStatusClaudePromptWithANSI`: ANSI-wrapped `❯` prompts are detected as `waiting`
- `internal/ui/app_test.go` — `TestNewSessionDialogPersistsToolFlags`: dialog flow with flags, verifies DB persistence
- `internal/ui/app_test.go` — `TestToolFlagsPassedToNewSession`: flags in DB → NewSession receives correct tool+toolFlags

---

## Files Touched

| File | Change |
|------|--------|
| `internal/db/db.go` | Schema v6 migration, updated CREATE TABLE |
| `internal/db/sessions.go` | `ToolFlags` field, all INSERT/SELECT/Scan sites |
| `internal/tmux/client.go` | `NewSession` 4-arg signature, `buildLaunchCommand`, `claude-dangerous` in `resolveLaunchCommand` |
| `internal/tmux/client_test.go` | `TestBuildLaunchCommand`, `TestResolveLaunchCommand` |
| `internal/tmux/status.go` | Claude-family status detection for `claude-*`, ANSI stripping before prompt matching |
| `internal/tmux/status_test.go` | Regression tests for `claude-dangerous` footer prompts and ANSI-wrapped prompts |
| `internal/testutil/tmux.go` | `FakeTmuxClient.NewSession` 4-arg signature, `NewSessionCall` struct |
| `internal/ui/dialog.go` | `savedToolFlags`, step 3 in `advanceNewSessionStep`, step guard, `commitDialog` |
| `internal/ui/app.go` | `ensureStarted`, detail panel tool/flags line |
| `internal/ui/app_test.go` | New tests, updated existing tests for 5-step dialog |

---

## Open Issues

- **BUG-013** — free-text tool flags not applied at launch. See `docs/bugs.md` for full investigation history and planned fix paths.
- **Session presets** — deferred from M4; `claude-dangerous` is a minimal stand-in. A proper preset system (`~/.config/tmux-agent-deck/presets.toml`) remains on the roadmap.
