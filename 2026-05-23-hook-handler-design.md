# Hook Handler Design

**Date:** 2026-05-23
**Status:** Draft

## Overview

Replace the timer-only conductor update model with an event-driven approach using Claude Code hooks. Two new subcommands:

- `tmux-agent-deck hook-handler` — called by Claude Code on lifecycle events; resolves the current worker session, finds its conductor, and sends a lightweight real-time status message.
- `tmux-agent-deck install-hooks` — patches `~/.claude/settings.json` to register `tmux-agent-deck hook-handler` under all relevant Claude Code events. Idempotent and non-destructive.

The existing `--conductor-heartbeat` flag remains as a safety net for sessions where a hook fails to fire (crash, misconfiguration). Hooks are the primary real-time path; the heartbeat is a low-frequency backstop.

## Architecture

```
Claude Code worker session
  └─ lifecycle event fires (Stop, SessionStart, etc.)
       └─ tmux-agent-deck hook-handler (new process)
            ├─ reads event JSON from stdin
            ├─ resolves tmux session: tmux display-message -p '#S'
            ├─ opens DB (read-only): find session → find conductor
            └─ tmux send-keys lightweight message to conductor pane

Poller (existing)
  └─ --conductor-heartbeat timer (unchanged)
       └─ fires every N minutes as safety net
```

Database access from `hook-handler` is read-only — no writes. The DB is already configured with WAL mode and `busy_timeout=5000`, so concurrent hook invocations (multiple events firing in a burst) are safe: WAL allows multiple simultaneous readers without blocking.

## `hook-handler` Subcommand

### Invocation

Called by Claude Code with event JSON on stdin:

```
tmux-agent-deck hook-handler
```

Claude Code passes JSON on stdin that always includes `hook_event_name` — the name of the event that fired (e.g. `"Stop"`, `"SessionStart"`). The handler reads this field to determine event type along with any useful context (tool name, notification message, etc.). No CLI flags are needed to identify the event.

### Session Resolution

The handler runs inside the worker's tmux session. It resolves the worker by calling:

```
tmux display-message -p '#S'
```

This returns the tmux session name, which is used to look up the session record in the DB. If no matching session is found, or if no conductor is assigned to that group, the handler exits silently (no-op).

### Conductor Check

The conductor must be active (not `stopped` or `error`, and has a `TmuxSession`). If unavailable, the handler exits silently — no retry, no error. The heartbeat covers this gap.

### Supported Events

| Claude Code Event | Trigger |
|---|---|
| `SessionStart` | Claude Code process started in this session |
| `SessionEnd` | Claude Code process exited |
| `Stop` | Agent finished a task (top-level stop) |
| `UserPromptSubmit` | New prompt submitted to the agent |
| `PermissionRequest` | Agent is blocked waiting for a permission |
| `Notification` | Agent sent any notification (no matcher — all notifications forwarded) |
| `PreCompact` | Context window about to be compacted |

## Message Format

All hook messages share the `[deck]` prefix so the conductor can distinguish them from escalation messages. Everything fits on one line.

```
[deck] <worker-title> | <event> | <optional-context>
```

Examples:

```
[deck] worker-a | Stop | task complete
[deck] worker-a | SessionStart
[deck] worker-a | SessionEnd
[deck] worker-a | UserPromptSubmit
[deck] worker-a | PermissionRequest | tool: Bash
[deck] worker-a | Notification | needs input
[deck] worker-a | PreCompact
```

Context is included when the hook JSON provides it:
- `PermissionRequest`: append `tool: <tool_name>`
- `Notification`: append the notification message (truncated to 60 chars)

The message is sent with `SendKeys` and submitted with `SendRawKeys(..., "Enter")`, matching the existing escalation delivery style. No `@deck-reply` syntax — these are informational only.

## `install-hooks` Subcommand

### Behavior

Reads `~/.claude/settings.json`, adds `tmux-agent-deck hook-handler` under each supported event, and writes back atomically (temp file + rename). Creates `settings.json` if it does not exist.

Claude Code's settings format requires each event entry to be a **hook group object** — an object with a `"hooks"` array — not a bare command object. For each event, the entry added to the event array is:

```json
{
  "hooks": [
    {
      "type": "command",
      "command": "tmux-agent-deck hook-handler",
      "async": true
    }
  ]
}
```

`PermissionRequest` is synchronous (Claude Code blocks on it), so `async` is omitted for that event:

```json
{
  "hooks": [
    {
      "type": "command",
      "command": "tmux-agent-deck hook-handler"
    }
  ]
}
```

A bare `{"type": "command", ...}` object placed directly in the event array is silently ignored by Claude Code. The `"hooks"` wrapper is required.

### Idempotency

Before adding an entry, checks whether a hook with `"command": "tmux-agent-deck hook-handler"` already exists under that event. If found, skips that event. Re-running `install-hooks` is always safe.

Never removes or modifies entries for other commands (e.g. existing hooks from other tools are preserved).

### `--uninstall` Flag

Removes only entries where `"command": "tmux-agent-deck hook-handler"`. Other entries are untouched. Prints a summary of what was removed.

### Output

Prints one line per event showing what changed:

```
Stop          added
SessionStart  added
SessionEnd    added
UserPromptSubmit  added
PermissionRequest  added
Notification  added
PreCompact    added
```

Or when already installed:

```
Stop          already registered
...
```

## Heartbeat Relationship

`--conductor-heartbeat` requires no changes. It continues to fire on its timer interval regardless of hook activity. The conductor may occasionally receive both a hook update and a heartbeat digest for the same period — this is acceptable; the information is complementary.

Recommended heartbeat interval when hooks are installed: 5–10 minutes. The heartbeat's role narrows to: catching sessions that crashed, sessions where hooks were not registered, and providing a periodic fleet-wide summary.

## File Layout

```
cmd/
  hookhandler.go      # hook-handler subcommand
  installhooks.go     # install-hooks subcommand
  hookhandler_test.go # integration tests
  installhooks_test.go
internal/
  hook/
    hook.go           # HookEvent type, ParseEvent(r io.Reader)
    hook_test.go
```

The `internal/hook` package owns JSON parsing and the event type constants. `cmd/hookhandler.go` owns tmux resolution, DB lookup, and send logic.

## Testing

**`internal/hook` (pure unit tests)**
- `ParseEvent` returns correct event type and context for each supported event JSON
- Unknown event type returns a zero value without error
- Missing optional fields (tool_name, message) produce messages without context suffix

**`cmd/hookhandler.go` (integration via `testutil`)**
- Stop event → sends `[deck] <title> | Stop | task complete` to conductor pane
- PermissionRequest → sends `[deck] <title> | PermissionRequest | tool: Bash`
- Notification → sends `[deck] <title> | Notification | <message>`
- No conductor assigned → no keys sent, exits cleanly
- Conductor stopped/error → no keys sent, exits cleanly
- Tmux session not in DB → no keys sent, exits cleanly

**`cmd/installhooks.go`**
- Fresh settings.json → all events added
- Existing entries preserved, only missing events added
- `--uninstall` removes only `tmux-agent-deck hook-handler` entries
- Atomic write: settings.json not corrupted if process is interrupted

## Out of Scope for V1

- Per-event opt-out flags (e.g. `--no-hook-session-start`)
- Hook delivery confirmation or retry logic
- Conductor reply parsing triggered by hooks (reply scanning stays in the poller)
- Support for project-level `.claude/settings.json` (user-level only)
- Windows support (tmux is Unix-only)
