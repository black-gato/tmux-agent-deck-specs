# tmux-agent-deck Design

**Date:** 2026-05-09 (last updated: 2026-05-20)
**Status:** Current — schema v6, all milestones complete except BUG-013

## Overview

A terminal UI for managing multiple AI coding agent sessions in tmux from a single interface. Sessions are organized into nested groups. The TUI shows a split panel: a session list on the left and a live detail panel (output, pane list, notes, tags) on the right.

Key capabilities shipped:
- Multi-agent monitoring with status detection (running / waiting / idle / error / stopped)
- Context window indicator, waiting elapsed timers, fleet status bar
- Conductor workflow: designate one session per group as conductor; auto-escalate waiting workers; reply routing from conductor back to worker via `@deck-reply` blocks
- Fleet management: multi-select, bulk ops, archive/restore, tags, filter
- Session configuration: project path, tool selection, startup script, per-session tool flags (DB + UI complete; BUG-013: flags not passed to process)
- Desktop notifications with routing styles and quiet hours
- Headless / daemon mode

---

## Architecture

```
tmux-agent-deck/
├── main.go
├── cmd/
│   ├── root.go          # cobra entrypoint, launchTUI() loop, launchHeadless(), openDB()
│   ├── add.go           # `add` subcommand
│   ├── list.go          # `list` subcommand
│   ├── remove.go        # `remove` subcommand
│   ├── session.go       # `session start/stop/attach` subcommands
│   ├── group.go         # `group create/delete/move` subcommands
│   └── cmd_test.go      # integration tests via RunWith()
├── internal/
│   ├── db/
│   │   ├── db.go        # Open(), migrate(), WAL mode + busy_timeout
│   │   ├── db_test.go
│   │   ├── groups.go    # Group type + CRUD, SetGroupConductor, ErrGroupExists
│   │   ├── groups_test.go
│   │   ├── sessions.go  # Session type + CRUD, ListGroupChildSessions
│   │   └── sessions_test.go
│   ├── tmux/
│   │   ├── client.go    # Client, ClientIface, AttachSession (ctrl+q bind/restore), SendKeys/SendRawKeys, ParseBindingCommand
│   │   ├── status.go    # DetectStatus(), ParseContextPct(), stripANSI(), isClaudeTool()
│   │   └── status_test.go
│   ├── state/
│   │   ├── poller.go    # Poller: Start/Stop/PollOnce, auto-escalation, reply scanning, heartbeats
│   │   ├── poller_test.go
│   │   ├── escalate.go  # EscalationMessage() — includes worker ID and reply syntax
│   │   ├── escalate_test.go
│   │   ├── reply.go     # ParseReplyBlocks(), NewOutputSince() — pure functions
│   │   └── reply_test.go
│   ├── notify/
│   │   ├── notify.go    # Notification policy + osascript integration
│   │   └── notify_test.go
│   ├── conductordocs/
│   │   ├── conductordocs.go   # WriteBlock() — writes managed conductor role into CLAUDE.md
│   │   └── conductordocs_test.go
│   ├── ui/
│   │   ├── app.go       # Bubbletea Model, Init/Update/View, Reload(), split-panel layout
│   │   ├── app_test.go
│   │   ├── form.go      # formState, initSessionForm(), renderForm(), commitForm() — new-session bottom panel
│   │   ├── list.go      # ListItem, BuildTree(), RenderList(), RenderContextBar()
│   │   ├── list_test.go
│   │   ├── dialog.go    # dialogState, updateDialog(), commitDialog(), completePath()
│   │   └── keys.go      # KeyBindings table, actionForKey()
│   └── testutil/
│       ├── db.go        # OpenTestDB(t)
│       └── tmux.go      # FakeTmuxClient for tests
├── test/
│   └── e2e/             # e2e tests (require tmux on PATH; -tags e2e)
```

**Data flow:** `poller` reads tmux pane output on a configurable interval (`--poll`) → writes status + context % to DB → emits notification events via `notify` → `app` reads DB on tick and re-renders. Conductor reply scanning and heartbeats also run in `PollOnce`.

---

## Data Model

Schema v6. Migrations run sequentially on startup via a `metadata` table (`key=schema_version`).

```sql
CREATE TABLE groups (
    path                 TEXT PRIMARY KEY,  -- e.g. "work/frontend"
    name                 TEXT NOT NULL,
    default_path         TEXT NOT NULL DEFAULT '',
    default_tool         TEXT NOT NULL DEFAULT 'claude',
    conductor_session_id TEXT NOT NULL DEFAULT '',
    expanded             INTEGER NOT NULL DEFAULT 1,
    sort_order           INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE sessions (
    id             TEXT PRIMARY KEY,  -- uuid
    title          TEXT NOT NULL,
    group_path     TEXT NOT NULL DEFAULT 'my-sessions',
    tmux_session   TEXT NOT NULL DEFAULT '',  -- tma-<slug>-<8hex>
    project_path   TEXT NOT NULL,
    tool           TEXT NOT NULL DEFAULT 'claude',
    status         TEXT NOT NULL DEFAULT 'stopped',
    created_at     INTEGER NOT NULL,
    last_active    INTEGER NOT NULL DEFAULT 0,
    notes          TEXT NOT NULL DEFAULT '',
    archived       INTEGER NOT NULL DEFAULT 0,
    tags           TEXT NOT NULL DEFAULT '',
    startup_script TEXT NOT NULL DEFAULT '',
    tool_flags     TEXT NOT NULL DEFAULT ''   -- BUG-013: stored but not yet passed to process
);

CREATE TABLE metadata (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);
```

**Group nesting** is path-based (`work/frontend`). Children: `WHERE path LIKE 'work/%'`. No foreign keys.

**Session tool** values: `claude`, `claude-dangerous` (maps to `claude --dangerously-skip-permissions`), `aider`, `cursor`, `bash`, and custom strings. `bash` / `shell` launch `zsh -il`.

**Tmux session names** follow `tma-<slugified-title>-<8hex>` so sessions are identifiable in `tmux ls`.

**Reserved groups:** `my-sessions` (default), `archived` (archived sessions).

---

## TUI

Split-panel layout: session list (~35% left), detail panel (~65% right).

```
┌─ tmux-agent-deck ────────────────────────────────────────────────────────┐
│ 2 waiting  1 idle  ■■■■░░ 66%  overdue: api-refactor (4m12s)            │
├──────────────────────┬───────────────────────────────────────────────────┤
│ ▼ work               │ ● my-app                                          │
│   ▼ frontend         │ claude · work/frontend · ctx ▓▓░░ 45%            │
│  ★● my-app  45%      │ Tags: #backend                                    │
│   ○ api-refactor 4m  │ Notes: investigating auth bug                     │
│   ► backend          │ ─────────────────────────────────────             │
│                      │ Panes                                             │
│ ▼ personal           │  ▶ [0] main                                       │
│   ◐ side-project     │    [1] tests                                      │
│                      │ ─────────────────────────────────────             │
│ ▼ my-sessions        │ Output                                            │
│   ✕ old-bug-fix      │ ✻ Thinking...                                     │
│                      │ > Running tests                                   │
│ [n]ew [g]rp [?]help  │                                                   │
└──────────────────────┴───────────────────────────────────────────────────┘
```

### Status Indicators

| Symbol | Meaning |
|--------|---------|
| `●` | Running / thinking |
| `○` | Waiting for input |
| `◐` | Idle (no output change 30s+) |
| `✕` | Error / process dead |
| `—` | Stopped |

`★` prefix on a session row = group conductor. Amber color = waiting. Red = error. Bold red = waiting overdue (2m+).

### Keybindings

| Section | Key | Action |
|---------|-----|--------|
| Navigation | `j` / `↓` | Move down |
| Navigation | `k` / `↑` | Move up |
| Navigation | `Enter` | Attach or start session |
| Navigation | `Space` | Toggle group collapse / multi-select session |
| Navigation | `Tab` | Cycle active pane in detail panel |
| Editing | `n` | New session (bottom-panel form) |
| Editing | `g` | New group |
| Editing | `m` | Move session to group |
| Editing | `r` | Rename session or group |
| Editing | `d` | Delete session or group |
| Editing | `e` | Edit notes |
| Editing | `t` | Edit tags |
| Workflow | `x` | Send keys to active pane |
| Workflow | `f` | Fork session |
| Workflow | `b` | Broadcast to group |
| Workflow | `c` | Set conductor for group |
| Workflow | `C` | Escalate selected session to conductor |
| View | `v` | Toggle fullscreen output |
| View | `a` | Archive session |
| View | `A` | Toggle archived view |
| View | `/` | Filter sessions by title |
| View | `?` | Help overlay |
| View | `q` | Quit |
| View | `ctrl+c` | Quit (navigation); cancel dialog (non-send modes) |

### Session Configuration Form

Pressing `n` opens a bottom-panel form with five fields:

1. **Title** — session name
2. **Path** — project path; `Tab` cycles filesystem completions; supports `~` and `$VAR`
3. **Tool** — arrow keys cycle: `claude`, `claude-dangerous`, `aider`, `cursor`, `bash`, `custom`
4. **Flags** — free-text agent CLI flags (stored in DB; BUG-013: not yet passed to process)
5. **Script** — startup script sent to the pane after launch

New sessions inherit `default_tool` and `default_path` from their group.

### State Detection

Polled from `tmux capture-pane -t <session> -p` on the configured interval (`--poll`, default 1s):

- **waiting** — agent prompt visible (`>` or `❯`) in last 8 non-empty lines; ANSI sequences stripped before match; claude-* tools use Claude-family detection
- **running** — spinner character or `Thinking` / `Running` text present, and output changed within 30s
- **idle** — no pane output change for 30s (checked before spinner heuristics, so stale output doesn't pin to running)
- **error** — pane exited or tmux session no longer exists
- **stopped** — explicitly stopped via CLI or TUI

Startup seeding: on first observation, `lastChange` is seeded from tmux `#{session_activity}` (or DB `last_active` as fallback) so pre-existing sessions don't flash `running` at launch.

Context window: `ParseContextPct` extracts the percentage from Claude's footer (e.g. `ctx:45%`) and stores it per session. Rendered as a 4-block bar (`▓▓░░`) in the list and detail panel.

---

## Conductor Workflow

One session per group can be designated as conductor (`c`). Conductors are marked with `★` in the list.

**Manual escalation (`C`):** sends an escalation message to the conductor with worker title, ID, status, notes, and filtered context lines. Includes reply syntax so the conductor knows how to respond.

**Auto-escalation (`--auto-escalate`):** poller sends escalation automatically when a worker transitions to `waiting`.

**Reply routing:** poller scans conductor pane output for `@deck-reply` blocks each poll cycle. Completed blocks are parsed and the body is sent to the target worker session. Deduplication by fingerprint (`workerID:body`) prevents replays.

**Heartbeat (`--conductor-heartbeat <duration>`):** periodically sends a digest of group worker state to the conductor. Sends "All clear" when no workers are waiting.

**CLAUDE.md init (`--init-conductor-docs`):** writes a managed conductor role block into the conductor session's `CLAUDE.md` when `c` is pressed.

Reply format (supported by conductor):
```
@deck-reply worker=<session-id>
<reply body>
@deck-end
```

---

## Notifications

**Flag:** `--notify` (opt-in)

**Routing styles** (`--notify-style`):
- `waiting` (default) — one alert per session when it goes waiting
- `conductor` — alert names the group conductor: `"<conductor>: <worker> is waiting"`
- `digest` — one combined alert per poll cycle listing all waiting workers in the group

**Quiet policy** (`--notify-quiet`, comma-separated key=value):
- `cooldown=5m` — suppress duplicate alerts within this duration
- `hours=22:00-08:00` — suppress all alerts during this time window

---

## CLI

```
tmux-agent-deck [flags]                   # launch TUI
  --poll <duration>                       # poll interval (default 1s)
  --headless                              # run poller without TUI
  --notify                                # enable desktop notifications
  --notify-style waiting|conductor|digest
  --notify-quiet "cooldown=5m,hours=22:00-08:00"
  --auto-escalate                         # auto-send escalation on waiting transition
  --conductor-heartbeat <duration>        # heartbeat digest interval (0 = disabled)
  --init-conductor-docs                   # write CLAUDE.md block when setting conductor

tmux-agent-deck add --title "Title" [--group "work/frontend"] [--project /path] [--tool claude] [--startup-script 'claude --resume']
tmux-agent-deck list [--json]
tmux-agent-deck remove <id|title>
tmux-agent-deck session start <id|title>
tmux-agent-deck session stop <id|title>
tmux-agent-deck session attach <id|title>
tmux-agent-deck group create <path> [--path /project] [--tool claude]
tmux-agent-deck group defaults <path> [--path /project] [--tool claude]
tmux-agent-deck group delete <name>
tmux-agent-deck group move <session> <group>
```

---

## State Storage

SQLite at `~/.tmux-agent-deck/state.db`. WAL mode + `busy_timeout=5000` prevent `SQLITE_BUSY` from poller / TUI contention. Single connection (`SetMaxOpenConns(1)`).

---

## Open Issues

- **BUG-013** — `tool_flags` field stored in DB and shown in detail panel but not passed as CLI arguments to the agent process. `claude-dangerous` preset is a hardcoded workaround for `--dangerously-skip-permissions`. See `docs/bugs.md` for root cause analysis.

---

## Out of Scope

- MCP attach/detach
- Skills system
- Git worktree management
- Watchers (webhook / ntfy / GitHub / Slack)
- Web UI
- Remote SSH instances
- Cost tracking
- Named presets file (`~/.config/tmux-agent-deck/presets.toml`)
