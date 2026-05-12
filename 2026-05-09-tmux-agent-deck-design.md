# tmux-agent-deck Design

**Date:** 2026-05-09  
**Status:** MVP Complete

## Overview

A terminal UI for managing multiple AI coding agent sessions in tmux from a single interface. Sessions are organized into nested groups. The shipped MVP stays focused: no remote management, watchers, or MCP management. Conductor workflows are deferred to a later milestone rather than part of the initial product surface.

---

## Architecture

```
tmux-agent-deck/
├── main.go
├── go.mod / go.sum
├── cmd/
│   ├── root.go          # cobra CLI entrypoint, openDB(), launchTUI(), RunWith()
│   ├── add.go           # `add` subcommand
│   ├── list.go          # `list` subcommand
│   ├── remove.go        # `remove` subcommand
│   ├── session.go       # `session start/stop/attach` subcommands
│   ├── group.go         # `group create/delete/move` subcommands
│   └── cmd_test.go      # integration tests via RunWith()
├── internal/
│   ├── db/
│   │   ├── db.go        # Open(), migrate()
│   │   ├── db_test.go
│   │   ├── groups.go    # Group type + CRUD functions
│   │   ├── groups_test.go
│   │   ├── sessions.go  # Session type + CRUD functions
│   │   └── sessions_test.go
│   ├── tmux/
│   │   ├── client.go    # NewClient, NewSession, Attach, Kill, Capture, Exists
│   │   ├── status.go    # DetectStatus() pure function
│   │   └── status_test.go
│   ├── state/
│   │   ├── poller.go    # Poller: Start/Stop/PollOnce, TmuxReader interface
│   │   └── poller_test.go
│   ├── ui/
│   │   ├── app.go       # bubbletea Model, Init/Update/View, Reload()
│   │   ├── app_test.go
│   │   ├── list.go      # ListItem type, BuildTree(), RenderList()
│   │   ├── list_test.go
│   │   ├── dialog.go    # dialogState, updateDialog(), commitDialog()
│   │   └── keys.go      # actionForKey() mapping
│   └── testutil/
│       ├── db.go        # OpenTestDB(t) helper
│       └── tmux.go      # fake TmuxReader for poller tests
```

**Data flow:** `poller` reads tmux pane output every ~1s → writes status to DB → `app` reads DB on tick → bubbletea re-renders list.

---

## Data Model

```sql
CREATE TABLE groups (
    path         TEXT PRIMARY KEY,  -- e.g. "work/frontend"
    name         TEXT NOT NULL,
    default_path TEXT NOT NULL DEFAULT '',
    default_tool TEXT NOT NULL DEFAULT 'claude',
    expanded     INTEGER NOT NULL DEFAULT 1,
    sort_order   INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE sessions (
    id             TEXT PRIMARY KEY,  -- uuid
    title          TEXT NOT NULL,
    group_path     TEXT NOT NULL DEFAULT 'my-sessions',
    tmux_session   TEXT NOT NULL DEFAULT '',
    project_path   TEXT NOT NULL,
    tool           TEXT NOT NULL DEFAULT 'claude',
    startup_script TEXT NOT NULL DEFAULT '',
    status         TEXT NOT NULL DEFAULT 'stopped',
    created_at     INTEGER NOT NULL,
    last_active    INTEGER NOT NULL DEFAULT 0
);
```

**Group nesting** is stored as path strings (`work/frontend`). Parent-child relationships are derived by path prefix — no foreign keys needed. Querying children of `work` is `WHERE path LIKE 'work/%'`.

Sessions inherit `default_tool` and `default_path` from their group at creation time but store their own copy — changing a group default does not affect existing sessions. Session `tool` values are launch profile IDs such as `claude`, `claude-danger`, `codex`, `codex-yolo`, and `shell`. The `shell` profile launches interactive login `zsh` (`zsh -il`).

---

## TUI

```
┌─ tmux-agent-deck ──────────────────────────────┐
│                                                  │
│  ▼ work                                          │
│    ▼ frontend                                    │
│      ● my-app          claude   running          │
│      ○ api-refactor    claude   waiting          │
│    ► backend           (collapsed)               │
│                                                  │
│  ▼ personal                                      │
│      ◐ side-project    gemini   idle             │
│                                                  │
│  ▼ my-sessions                                   │
│      ✕ old-bug-fix     claude   error            │
│                                                  │
│ [n]ew  [g]roup  [m]ove  [r]ename  [d]elete  [q]uit │
└──────────────────────────────────────────────────┘
```

### Status Indicators

| Symbol | Meaning |
|--------|---------|
| `●` | Running / thinking |
| `○` | Waiting for input |
| `◐` | Idle (no activity 30s+) |
| `✕` | Error / process dead |
| `—` | Stopped |

### Keybindings

| Key | Action |
|-----|--------|
| `Enter` | Attach to session |
| `n` | New session in current group |
| `e` | Edit notes or group defaults |
| `x` | Send keys to active pane |
| `f` | Fork session |
| `b` | Broadcast to group |
| `/` | Search output |
| `g` | New group |
| `m` | Move session to group |
| `r` | Rename session or group |
| `d` | Delete session or group |
| `Space` | Collapse/expand group |
| `q` | Quit |

### State Detection

Polled from `tmux capture-pane -t <session> -p` every ~1s:

- **waiting** — prompt visible at bottom of pane
- **running** — spinner or thinking text present
- **idle** — no state change for 30s after last `running`
- **error** — pane exited or tmux session no longer exists
- **stopped** — session was explicitly stopped via CLI or TUI

### Session Configuration

The `n` flow is multi-step:

1. Preset picker if `~/.config/tmux-agent-deck/presets.toml` exists
2. Session title
3. Project path
4. Tool selection from launch profiles
5. Optional startup script

Project paths support `~` and `$VAR` expansion. `Tab` completes directories during the project-path prompt. Tool and preset selection use arrow keys.

---

## CLI Commands

```
tmux-agent-deck                          # launch TUI
tmux-agent-deck add --title "Title" [--group "work/frontend"] [--project /path] [--tool claude] [--startup-script 'claude --resume']
tmux-agent-deck list [--json]
tmux-agent-deck remove <id|title>
tmux-agent-deck session start <id|title>
tmux-agent-deck session stop <id|title>
tmux-agent-deck session attach <id|title>
tmux-agent-deck group create <path> [--path /project] [--tool claude]
tmux-agent-deck group defaults <path> [--path /project] [--tool codex-yolo]
tmux-agent-deck group delete <name>
tmux-agent-deck group move <session> <group>
```

---

## State Storage

SQLite database at `~/.tmux-agent-deck/state.db`. Schema migrations run on startup via sequential version checks stored in a `metadata` table (`key=schema_version`).

---

## Out of Scope (Post-MVP)

- MCP attach/detach
- Skills system
- Git worktree management
- Watchers (webhook/ntfy/GitHub/Slack)
- Conductor (Telegram/Slack/Discord)
- Web UI
- Remote SSH instances
- Cost tracking
- Session fork/resume
- Profiles
