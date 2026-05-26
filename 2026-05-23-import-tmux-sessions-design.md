# Import Existing Tmux Sessions — Design

**Status:** Draft
**Date:** 2026-05-23
**Related:** [2026-05-17-session-form-design.md](2026-05-17-session-form-design.md)

## Goal

Let the deck adopt tmux sessions it didn't create — sessions started outside the deck, sessions orphaned by a DB wipe/reinstall, or sessions inherited from another machine. Today the only path into the DB is `add` / new-session form, both of which assume the deck owns the session lifecycle. Users with pre-existing tmux sessions have to kill and recreate, losing scrollback and any running agent.

The import flow lists tmux sessions present in `tmux list-sessions` but missing from the deck DB, lets the user assign a title + group, and inserts a row whose `TmuxSession` field points at the live tmux session. The poller picks it up on the next tick.

## Decisions

- **Surfaces:** TUI picker (primary) + `import` CLI subcommand (scripting / headless).
- **Eligibility:** every untracked tmux session is eligible — not just `ad-*` prefixed ones. Users may have unrelated tmux work; the picker shows everything and lets them choose.
- **User input:** minimal. TITLE (defaults to tmux name) and GROUP (defaults to the group under the TUI cursor, or `my-sessions` from CLI). Everything else is auto-detected or defaulted; the user can edit the row later via the normal flow.
- **No tmux mutation on import.** Import only inserts a DB row. The live tmux session is never renamed, killed, or modified. Detaching the import is a `remove` away.
- **Status starts as `unknown`.** The poller computes real status (running/idle/waiting) on its next tick. No special seeding.
- **Idempotency:** importing a name that already exists in the DB is an error (CLI) / hidden from the picker (TUI). No silent upsert.

## Detection

A session is "untracked" iff:

1. Its name appears in `tmux list-sessions -F "#{session_name}"` (existing `tmux.Client.ListSessions`, client.go:137).
2. No row in `sessions` has `tmux_session = <name>` (existing `GetSessionByTmuxName`, sessions.go:68).

Empty `tmux_session` strings in the DB are ignored — those rows represent deck sessions that haven't been started yet.

### Metadata read from tmux

Per session, we read:

- `#{pane_current_path}` of the active pane → `ProjectPath` default.
- `#{session_activity}` → seeds `LastActive` (already used by the poller's startup logic per BUG-010).

Both come from one call:

```
tmux display-message -p -t <name> -F "#{pane_current_path}|#{session_activity}"
```

Wrapped in a new `tmux.Client.SessionInfo(name) (SessionInfo, error)` method. We do not try to detect the running tool (claude vs gemini) — pane command names are unreliable mid-conversation, and the user can edit the Tool field later.

## CLI

New `cmd/import.go`, registered in `cmd/root.go` next to `addCmd`:

```
tmux-agent-deck import <tmux-name> [--title T] [--group G]
tmux-agent-deck import --list      # print untracked names, one per line, for scripting
tmux-agent-deck import --all       # import every untracked session with defaults
```

Defaults applied at insert time:

| Field        | Default                                                       |
|--------------|---------------------------------------------------------------|
| Title        | tmux session name                                             |
| GroupPath    | `my-sessions`                                                 |
| Tool         | group's `DefaultTool`, fallback `claude`                      |
| ProjectPath  | `SessionInfo.CurrentPath`, fallback cwd                       |
| TmuxSession  | the provided name (the linchpin field)                        |
| Status       | `unknown`                                                     |
| CreatedAt    | now                                                           |

Errors: name not present in `tmux list-sessions` → exit 1 with "not found". Name already tracked → exit 1 with "already imported". `--all` reports per-session success/failure and exits non-zero on any failure.

## TUI

### Trigger

New action `actionImport`, bound to `I` (capital) in `internal/ui/keys.go`. Lowercase `i` is reserved by vim-mode auto-detection; `I` avoids that collision and follows the existing capital-key convention for less-common ops.

### Picker dialog (`importPicker`)

When `I` is pressed:

- Call `ListUntrackedTmuxSessions(db, tmuxClient)`. If empty, show "no untracked tmux sessions" in the status line and return without opening a dialog.
- Otherwise open `importPicker`, a single-column list:

```
IMPORT TMUX SESSION
  scratch-foo     /tmp
> work-bar        /Users/.../some-repo
  ad-orphan42     /Users/.../other-repo

enter: select   esc: cancel
```

Path is right-aligned and truncated. Up/down navigates; `enter` opens the form dialog; `esc` closes.

### Form dialog (`importForm`)

A two-field form, deliberately minimal — not the 8-field new-session form:

```
IMPORT: scratch-foo
TITLE  scratch-foo
GROUP  my-sessions/agents

tab: next   enter: import   esc: cancel
```

- **TITLE** pre-filled with the tmux name. Editable. Empty is rejected (sets `formErr`, stays open).
- **GROUP** pre-filled with the group under the tree cursor when `I` was pressed, falling back to `my-sessions`. Uses the same group-chooser semantics as the new-session form (`internal/ui/form.go`). Validated against existing groups on commit; unknown group sets `formErr`.

On commit: build a `db.Session` with the defaults above plus the user's TITLE/GROUP, call `CreateSession`, then `Reload()`. The row appears in the tree immediately; the poller fills in the real status within one tick.

## Critical files

- `internal/tmux/client.go` — add `SessionInfo` struct + method; add to `ClientIface`.
- `internal/testutil/tmux.go` — extend `FakeTmuxClient` with a per-name info map.
- `internal/db/sessions.go` — add `ListUntrackedTmuxSessions`.
- `cmd/import.go` — new file, patterned on `cmd/add.go`.
- `cmd/root.go` — register `importCmd`.
- `internal/ui/keys.go` — add `actionImport` + `I` binding; update help table.
- `internal/ui/dialog.go` — add `importPicker` and `importForm` dialog states.
- `internal/ui/app.go` — route `actionImport`; render the new dialogs in `View`.

## What this does not do

- No mass deduplication for sessions whose tmux name matches an *archived* row — archived rows still count as tracked. To re-import, the user unarchives.
- No "watch for new untracked sessions" daemon. Import is on-demand.
- No detection of the agent CLI (`claude` vs `gemini`) running in the pane. The default `Tool` is taken from the group; the user adjusts later if wrong.
- No git-worktree awareness. A session imported from a worktree directory still gets that directory as `ProjectPath`; no special branch handling.

## Verification

1. **Unit tests**
   - `internal/db/sessions_test.go` — `TestListUntrackedTmuxSessions`: seed one tracked, fake tmux returns three names, expect two untracked.
   - `internal/tmux/` — parse test for `SessionInfo` pipe-delimited output.
   - `cmd/cmd_test.go` — `TestImportCommand`: `--list`, single-name import, `--all`, duplicate-name failure.
   - `internal/ui/dialog_test.go` — keystroke test: `I` opens picker → `enter` opens form → `enter` commits and calls `Reload`.

2. **Manual end-to-end**

   ```bash
   go build -o tmux-agent-deck .
   tmux new -d -s scratch-foo -c /tmp
   tmux new -d -s scratch-bar -c "$HOME"
   ./tmux-agent-deck import --list                # both names appear
   ./tmux-agent-deck import scratch-foo --title Foo
   ./tmux-agent-deck list                          # Foo present, TmuxSession=scratch-foo
   ./tmux-agent-deck                               # TUI; press I; import scratch-bar
   ```

   Confirm imported rows acquire real status within one poll interval and that `enter` (attach) connects to the live session.

3. `go test ./...` and `go vet ./...` clean.
