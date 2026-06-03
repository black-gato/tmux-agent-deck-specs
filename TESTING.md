# Manual Test Plan — tmux-agent-deck

**Date:** 2026-06-03
**Format:** Steps + expected result. Each case is independent unless marked "requires TC-XXX."

---

## Prerequisites

```bash
go build -o tmux-agent-deck .
export AGENT_DECK_DB=/tmp/tad-test.db
rm -f /tmp/tad-test.db
```

All tests use a fresh DB unless noted. tmux must be running.

---

## Test Session Guide

Tests are grouped into **fixtures** — shared setup states that let you run multiple cases back-to-back without resetting. Run fixtures in any order; reset the DB between fixtures unless noted.

### Fixture A — Fresh DB, no tmux sessions
**Setup:** Fresh DB only (`rm -f /tmp/tad-test.db`). No tmux sessions needed.

| Test | Description |
|------|-------------|
| TC-102 | List on empty DB |
| TC-112 | Import nonexistent session rejected |

---

### Fixture B — One session in DB, no tmux session started
**Setup:**
```bash
./tmux-agent-deck add --title "Worker" --group my-sessions --tool claude --project /tmp
./tmux-agent-deck list  # note the session ID
```

| Test | Description | Order |
|------|-------------|-------|
| TC-101 | Add session | 1st |
| TC-106 | Group create | any |
| TC-107 | Group delete | after TC-106 |
| TC-103 | Remove session | last (consumes the session) |

---

### Fixture C — Session started in tmux
**Setup:** Fixture B setup, then:
```bash
./tmux-agent-deck session start <id>
# verify: tmux ls shows the session
```

| Test | Description | Order |
|------|-------------|-------|
| TC-104 | Session start | 1st |
| TC-105 | Session stop | last (kills the tmux session) |

---

### Fixture D — Untracked tmux sessions
**Setup:**
```bash
tmux new -d -s scratch-test -c /tmp
tmux new -d -s foo-a -c /tmp
tmux new -d -s foo-b -c "$HOME"
```

| Test | Description | Order |
|------|-------------|-------|
| TC-108 | Import --list | 1st |
| TC-109 | Import single session | 2nd |
| TC-110 | Import duplicate rejected | after TC-109 |
| TC-111 | Import --all | any (uses foo-a, foo-b) |
| TC-207 | TUI import picker | any |
| TC-208 | TUI import empty state | last (all sessions now tracked) |

**Teardown:** `tmux kill-session -t scratch-test; tmux kill-session -t foo-a; tmux kill-session -t foo-b`

---

### Fixture E — TUI, single session
**Setup:** Fresh DB, launch TUI:
```bash
./tmux-agent-deck
```

| Test | Description | Order |
|------|-------------|-------|
| TC-201 | Create session via TUI | 1st |
| TC-301 | Create group | any |
| TC-302 | Expand/collapse group | after TC-301 |
| TC-303 | Move session | after TC-201 and TC-301 |
| TC-204 | Rename session | after TC-201 |
| TC-205 | Archive and restore | after TC-201 |
| TC-202 | Attach and return — cursor preserved | after TC-201 |
| TC-206 | Fork session | after TC-201 (needs project path) |
| TC-603 | Help overlay | any |
| TC-203 | Delete session | last (consumes session) |

---

### Fixture F — Running claude session, status polling
**Setup:** One session tracking a live claude tmux session. Launch with short poll:
```bash
tmux new -d -s claude-worker -c /tmp
./tmux-agent-deck add --title "Claude Worker" --group my-sessions --tool claude --project /tmp --tmux-session claude-worker
./tmux-agent-deck --poll 2s
```

| Test | Description | Order |
|------|-------------|-------|
| TC-405 | No startup running flash (BUG-010) | 1st (check on launch) |
| TC-401 | Waiting detection | when claude shows `❯` prompt |
| TC-406 | Context % display | while session active |
| TC-407 | Overdue waiting indicator | wait 30s after TC-401 |
| TC-402 | Waiting → running on input | after TC-401 |
| TC-403 | Idle detection (shell session) | add a shell session, leave at `$` |
| TC-404 | Stopped detection | kill the tmux session last |

---

### Fixture G — Two sessions + conductor, TUI only (no auto-escalate)
**Setup:**
```bash
tmux new -d -s conductor-session -c /tmp
tmux new -d -s worker-session -c /tmp
./tmux-agent-deck add --title "Conductor" --group my-sessions --tool claude --tmux-session conductor-session --project /tmp
./tmux-agent-deck add --title "Worker" --group my-sessions --tool claude --tmux-session worker-session --project /tmp
./tmux-agent-deck
# cursor on Conductor row, press c to designate
```

| Test | Description | Order |
|------|-------------|-------|
| TC-501 | Toggle conductor on | 1st |
| TC-601 | Search / filter | any |
| TC-602 | Multi-select bulk delete | last (use extra sessions) |
| TC-604 | Send keys | any |
| TC-605 | Broadcast | any |
| TC-503 | Manual escalate (C) | needs worker in `waiting` state |
| TC-502 | Toggle conductor off | after TC-501 |

---

### Fixture H — Auto-escalate stack
**Setup:** Fixture G sessions running, relaunch with flags:
```bash
./tmux-agent-deck --auto-escalate --conductor-heartbeat 10s --poll 2s
```
Conductor designated. Hooks installed (`./tmux-agent-deck install-hooks`).

| Test | Description | Order |
|------|-------------|-------|
| TC-504 | Auto-escalate on waiting | worker transitions to `waiting` |
| TC-505 | Reply routing | after TC-504, conductor types `@deck-reply` block |
| TC-506 | Reply not re-sent (dedup) | after TC-505, wait two ticks |
| TC-507 | Conductor heartbeat | wait 10s |
| TC-508 | --init-conductor-docs | relaunch with `--init-conductor-docs`, press `c` |
| TC-509 | hook-handler Stop event | worker Claude session completes task |
| TC-510 | hook-handler PermissionRequest | worker hits permission block |
| TC-511 | No conductor — escalation skipped | remove conductor, trigger waiting |
| TC-512 | Toggle conductor during polling | press `c` to un-toggle while poller live |

---

### Fixture I — Hooks only
**Setup:** No DB needed.
```bash
./tmux-agent-deck install-hooks
```

| Test | Description | Order |
|------|-------------|-------|
| TC-113 | install-hooks writes settings | 1st |
| TC-114 | install-hooks --uninstall | last |

---

### Fixture J — Headless mode
**Setup:** One session in DB with a tracked tmux session.

| Test | Description |
|------|-------------|
| TC-701 | Headless mode polls and exits cleanly |

---

### Fixture K — Meta-conductor, TUI only
**Setup:** Fresh DB. Launch TUI. Create at least two groups each with at least one session. No tmux sessions need to be running.
```bash
rm -f /tmp/tad-test.db
./tmux-agent-deck add --title "Meta" --group team-alpha --tool claude --project /tmp
./tmux-agent-deck add --title "Worker A" --group team-alpha --tool claude --project /tmp
./tmux-agent-deck add --title "Worker B" --group team-beta --tool claude --project /tmp
./tmux-agent-deck
```

| Test | Description | Order |
|------|-------------|-------|
| TC-801 | Pinned row — unset state | 1st |
| TC-802 | Set meta-conductor with M | 2nd |
| TC-803 | Pinned row shows assigned session title | after TC-802 |
| TC-804 | Toggle meta-conductor off with M | after TC-802 |
| TC-805 | Pinned row returns to unset state after toggle off | after TC-804 |
| TC-806 | Meta-conductor slot cleared on session delete | any (creates/deletes a session) |
| TC-807 | Help overlay includes M binding | any |

---

### Fixture L — Meta-conductor, auto-escalate stack
**Setup:** Three live tmux sessions: one meta-conductor, one group conductor, one worker. Two sessions in "team-alpha" (conductor + worker), one orphan in "team-beta" (no conductor). Launch with auto-escalate and short poll.
```bash
tmux new -d -s meta-session -c /tmp
tmux new -d -s alpha-conductor -c /tmp
tmux new -d -s alpha-worker -c /tmp
tmux new -d -s beta-orphan -c /tmp
./tmux-agent-deck add --title "Meta" --group team-alpha --tool claude --tmux-session meta-session --project /tmp
./tmux-agent-deck add --title "Alpha Conductor" --group team-alpha --tool claude --tmux-session alpha-conductor --project /tmp
./tmux-agent-deck add --title "Alpha Worker" --group team-alpha --tool claude --tmux-session alpha-worker --project /tmp
./tmux-agent-deck add --title "Beta Orphan" --group team-beta --tool claude --tmux-session beta-orphan --project /tmp
./tmux-agent-deck --auto-escalate --conductor-heartbeat 15s --poll 2s
# In TUI: cursor on "Meta", press M to designate as meta-conductor
# In TUI: cursor on "Alpha Conductor", press c to designate as group conductor for team-alpha
```

| Test | Description | Order |
|------|-------------|-------|
| TC-901 | Meta-conductor amber highlight when waiting | meta session transitions to waiting |
| TC-902 | Escalation to meta when group has no conductor (team-beta) | beta-orphan transitions to waiting |
| TC-903 | Escalation to meta when session IS the group conductor | alpha-conductor transitions to waiting |
| TC-904 | Normal escalation to group conductor (team-alpha worker) | alpha-worker transitions to waiting |
| TC-905 | Meta-conductor reply routed to worker | after TC-902 or TC-904 |
| TC-906 | Meta-conductor reply routed to group conductor | after TC-903 |
| TC-907 | Meta-conductor not escalated to itself | meta-session transitions to waiting |
| TC-908 | Deck-wide heartbeat sent to meta-conductor | wait for heartbeat interval |
| TC-909 | --init-conductor-docs writes meta-conductor block on M press | relaunch with flag, press M |

**Teardown:**
```bash
tmux kill-session -t meta-session 2>/dev/null
tmux kill-session -t alpha-conductor 2>/dev/null
tmux kill-session -t alpha-worker 2>/dev/null
tmux kill-session -t beta-orphan 2>/dev/null
```

---

## 1. CLI Commands

### TC-101: Add a session
1. Run `./tmux-agent-deck add --title "My Worker" --group my-sessions --tool claude --project /tmp`
2. Run `./tmux-agent-deck list`

**Expected:** Row with title "My Worker", group "my-sessions", tool "claude", project "/tmp" is printed.

---

### TC-102: List sessions (empty DB)
1. Run `./tmux-agent-deck list`

**Expected:** No rows printed; command exits 0.

---

### TC-103: Remove a session
1. Add a session (TC-101).
2. Note the ID from `list`.
3. Run `./tmux-agent-deck remove <id>`
4. Run `./tmux-agent-deck list`

**Expected:** Row no longer appears.

---

### TC-104: Session start
1. Add a session with `--project /tmp`.
2. Run `./tmux-agent-deck session start <id>`

**Expected:** A tmux session is created (`tmux ls` shows it). Command exits 0.

---

### TC-105: Session stop
1. TC-104 complete (session running).
2. Run `./tmux-agent-deck session stop <id>`

**Expected:** tmux session is gone (`tmux ls` no longer shows it).

---

### TC-106: Group create
1. Run `./tmux-agent-deck group create my-sessions/backend --name Backend`

**Expected:** Group appears in `./tmux-agent-deck list` output (or TUI tree).

---

### TC-107: Group delete
1. TC-106 complete, group is empty.
2. Run `./tmux-agent-deck group delete my-sessions/backend`

**Expected:** Group no longer appears.

---

### TC-108: Import --list
1. Start an untracked tmux session: `tmux new -d -s scratch-test -c /tmp`
2. Run `./tmux-agent-deck import --list`

**Expected:** "scratch-test" appears in output. Command exits 0.

---

### TC-109: Import single session
1. TC-108 complete.
2. Run `./tmux-agent-deck import scratch-test --title "Scratch" --group my-sessions`
3. Run `./tmux-agent-deck list`

**Expected:** Row with title "Scratch", TmuxSession="scratch-test" appears. tmux session still running.

---

### TC-110: Import duplicate rejected
1. TC-109 complete.
2. Run `./tmux-agent-deck import scratch-test` again.

**Expected:** Error message mentioning "already imported". Exit non-zero.

---

### TC-111: Import --all
1. Start two untracked sessions: `tmux new -d -s foo-a -c /tmp && tmux new -d -s foo-b -c /tmp`
2. Run `./tmux-agent-deck import --all`
3. Run `./tmux-agent-deck list`

**Expected:** Both "foo-a" and "foo-b" rows appear. Command prints a success line per session.

---

### TC-112: Import missing session rejected
1. Run `./tmux-agent-deck import nonexistent-session`

**Expected:** Error "not found". Exit non-zero.

---

### TC-113: install-hooks writes settings
1. Run `./tmux-agent-deck install-hooks`

**Expected:** Output confirms hooks written. Claude Code settings file contains `hook-handler` entries for `Stop`, `PermissionRequest`, `Notification` events.

---

### TC-114: install-hooks --uninstall
1. TC-113 complete.
2. Run `./tmux-agent-deck install-hooks --uninstall`

**Expected:** Output confirms hooks removed. Settings file no longer contains `hook-handler` entries.

---

## 2. Session Lifecycle (TUI)

### TC-201: Create session via TUI
1. Launch `./tmux-agent-deck`
2. Press `n`
3. Fill in TITLE, GROUP, TOOL fields. Press Enter.

**Expected:** New session row appears in tree. No error displayed.

---

### TC-202: Attach and return to TUI — cursor preserved
1. TC-201 complete. Cursor on the new session.
2. Press Enter to attach.
3. Inside tmux session, detach with `ctrl+q` (or `ctrl+b d` if outside tmux).

**Expected:** TUI reappears with cursor on the same session row, not at the top.

---

### TC-203: Delete session
1. TC-201 complete. Cursor on session.
2. Press `d`.

**Expected:** Session row removed from tree. tmux session stopped.

---

### TC-204: Rename session
1. TC-201 complete. Cursor on session.
2. Press `r`, type new name, press Enter.

**Expected:** Session row updates to new name.

---

### TC-205: Archive and restore
1. TC-201 complete. Cursor on session.
2. Press `a` to archive.

**Expected:** Session disappears from default view.

3. Press `A` to show archived.

**Expected:** Session reappears with archived indicator.

4. Press `a` again to restore.

**Expected:** Session moves back to normal view.

---

### TC-206: Fork session
1. TC-201 complete. Cursor on session with a project path set.
2. Press `f`.

**Expected:** New session row appears, title has "-fork" suffix (or similar). Both sessions visible in tree.

---

### TC-207: Import via TUI picker
1. Start untracked tmux session: `tmux new -d -s tui-import-test -c /tmp`
2. Launch `./tmux-agent-deck`
3. Press `I`

**Expected:** Picker opens showing "tui-import-test" with path "/tmp".

4. Press Enter to select.
5. Edit TITLE if desired. Press Tab to move to GROUP. Press Enter.

**Expected:** Picker closes, session appears in tree. Status fills in on next poll tick.

---

### TC-208: Import picker — empty state
1. Ensure all tmux sessions are already tracked.
2. Launch `./tmux-agent-deck`, press `I`.

**Expected:** Picker opens with "no untracked tmux sessions" message. Press Esc to dismiss — no error state.

---

## 3. Group Management (TUI)

### TC-301: Create group
1. Launch TUI, press `g`.
2. Enter path "my-sessions/ops", press Enter.

**Expected:** Group node appears in tree.

---

### TC-302: Expand / collapse group
1. TC-301 complete. Cursor on group node.
2. Press Enter to collapse.

**Expected:** Child items hidden.

3. Press Enter again to expand.

**Expected:** Child items visible.

---

### TC-303: Move session to different group
1. Two groups exist, one session in first group. Cursor on session.
2. Press `m`, enter target group path, press Enter.

**Expected:** Session row moves to target group.

---

## 4. Status Polling

### TC-401: Running → waiting detection (claude)
1. Create and start a claude session.
2. Launch TUI with `--poll 2s`.
3. In the session pane, type nothing — let claude reach its `❯` prompt.

**Expected:** Status column changes from `running` to `waiting` within ~2 poll ticks. Waiting timer appears.

---

### TC-402: Waiting → running on input
1. TC-401 complete (session showing `waiting`).
2. Attach to session, send any message, detach.

**Expected:** Status changes back to `running` within one poll tick. Waiting timer disappears.

---

### TC-403: Idle detection
1. Start a shell session (tool=shell), leave it at a `$` prompt with no activity.

**Expected:** Status shows `idle` after a short period of no output change.

---

### TC-404: Stopped detection
1. Create and start a session. Status shows `running` or `waiting`.
2. Kill the tmux session: `tmux kill-session -t <name>`

**Expected:** Status changes to `stopped` within one poll tick.

---

### TC-405: No startup running flash (BUG-010 regression)
1. Create a session with a known `last_active` timestamp (attach it briefly, then detach).
2. Stop the TUI and relaunch.

**Expected:** Session does not flash `running` on startup — it shows `idle` or `waiting` from the first render.

---

### TC-406: Context % display
1. Attach a running claude session and give it a large task with significant context usage.

**Expected:** Context % appears next to the session row (e.g. `42%`) and updates each poll tick.

---

### TC-407: Overdue waiting indicator
1. TC-401 complete. Wait 30+ seconds without responding to the waiting session.

**Expected:** Waiting timer turns red / overdue indicator appears.

---

## 5. Conductor Workflow

### TC-501: Toggle conductor on
1. Launch TUI. Two sessions in same group. Cursor on one.
2. Press `c`.

**Expected:** Session row gets conductor indicator (♦ or similar). `conductor: true` in right panel detail.

---

### TC-502: Toggle conductor off
1. TC-501 complete. Cursor still on the conductor session.
2. Press `c` again.

**Expected:** Conductor indicator removed. `conductor: false` in right panel detail.

---

### TC-503: Manual escalate (`C`)
1. One session designated as conductor (TC-501). A second session is `waiting`.
2. Cursor on the waiting session. Press `C`.

**Expected:** Escalation message sent to conductor's tmux pane. Conductor pane shows the message with worker ID and context.

---

### TC-504: Auto-escalate on waiting transition
1. Launch with `./tmux-agent-deck --auto-escalate --poll 2s`
2. Two sessions in same group; one designated conductor.
3. Worker session transitions to `waiting`.

**Expected:** Conductor pane receives escalation message automatically within one poll tick — without pressing `C`.

---

### TC-505: Reply routing
1. TC-504 complete. Conductor pane has escalation for worker.
2. In conductor pane type:
   ```
   @deck-reply worker=<worker-session-id>
   Try running go test ./... to see the failing test
   @deck-end
   ```
3. Wait one poll tick.

**Expected:** Worker pane receives the reply body typed into it and submitted.

---

### TC-506: Reply not re-sent (dedup)
1. TC-505 complete (reply already routed).
2. Wait two more poll ticks without sending a new reply.

**Expected:** Worker pane does not receive the same reply a second time.

---

### TC-507: Conductor heartbeat
1. Launch with `./tmux-agent-deck --auto-escalate --conductor-heartbeat 10s`
2. Conductor designated, workers running.
3. Wait 10 seconds.

**Expected:** Conductor pane receives a heartbeat digest listing worker statuses. If all clear, receives "all clear" message.

---

### TC-508: --init-conductor-docs writes CLAUDE.md
1. Conductor session has a `ProjectPath` set.
2. Launch with `./tmux-agent-deck --init-conductor-docs`
3. Press `c` on that session.

**Expected:** `<ProjectPath>/CLAUDE.md` contains the managed conductor role block between `<!-- tmux-agent-deck:conductor-role:start -->` markers.

---

### TC-509: hook-handler Stop event
1. TC-113 complete (hooks installed). Conductor designated for the worker's group.
2. Worker Claude session completes a task (Stop event fires).

**Expected:** Conductor pane receives `[deck] <worker title> | Stop | task complete` without waiting for a poll tick.

---

### TC-510: hook-handler PermissionRequest event
1. TC-113 complete. Worker session hits a tool permission block.

**Expected:** Conductor pane receives `[deck] <worker title> | PermissionRequest | tool: <tool name>` immediately.

---

### TC-511: No conductor — escalation silently skipped
1. Group has no conductor assigned. Worker transitions to `waiting`.
2. Launch with `--auto-escalate`.

**Expected:** No error. No escalation sent. Worker simply shows `waiting` status.

---

### TC-512: Conductor toggle during active polling
1. `--auto-escalate` running. Session is designated conductor.
2. Press `c` to un-toggle conductor while poller is live.

**Expected:** No log errors. Next `waiting` transition produces no escalation. No "can't find pane" errors in stderr.

---

## 6. TUI Navigation

### TC-601: Search / filter
1. Multiple sessions in tree. Press `/`, type part of a session title.

**Expected:** Tree filters to matching sessions only. Clearing search restores full list.

---

### TC-602: Multi-select bulk delete
1. Multiple sessions. Press Space on several to select. Press `d`.

**Expected:** All selected sessions removed. None remaining.

---

### TC-603: Help overlay
1. Press `?`.

**Expected:** Help overlay shows all key bindings including `I` (Import), `c` (Toggle conductor). Press `?` or Esc to dismiss.

---

### TC-604: Send keys (`x`)
1. Cursor on a running session. Press `x`, type a message, press Enter.

**Expected:** Message appears typed into the session's tmux pane.

---

### TC-605: Broadcast (`b`)
1. Multiple sessions in same group. Press `b`, type a message, press Enter.

**Expected:** Message sent to all sessions in the group.

---

## 7. Headless Mode

### TC-701: Headless mode polls and exits cleanly
1. Add a session with a running tmux session.
2. Run `./tmux-agent-deck --headless --poll 500ms &`
3. Wait 2 seconds, kill the process with `ctrl+c`.

**Expected:** Process exits cleanly. DB has updated status for tracked sessions.

---

## 8. Meta-Conductor

### TC-801: Pinned row — unset state
1. Launch TUI (Fixture K setup).
2. Observe the line pinned above the sessions list, before the separator.

**Expected:** Pinned row reads `M  — no meta-conductor —` in a dimmed/faint style. The sessions list renders below the separator as normal.

---

### TC-802: Set meta-conductor with M
1. Fixture K running. Cursor on any session row (e.g. "Meta").
2. Press `M`.

**Expected:** Pinned row updates immediately to show that session's title. No error or reload flicker beyond a normal re-render.

---

### TC-803: Pinned row shows assigned session title
1. TC-802 complete.
2. Observe the pinned row.

**Expected:** Pinned row shows the session's status symbol and title (e.g. `M  —  Meta`). The row is NOT dimmed.

---

### TC-804: Toggle meta-conductor off with M
1. TC-802 complete. Cursor still on the same session that is the meta-conductor.
2. Press `M` again.

**Expected:** Pinned row immediately returns to the unset (`— no meta-conductor —`) state.

---

### TC-805: Pinned row returns to unset state after toggle off
1. TC-804 complete.

**Expected:** Pinned row is dimmed and shows `— no meta-conductor —`. The previously-assigned session row has no meta-conductor indicator.

---

### TC-806: Meta-conductor slot cleared on session delete
1. Fixture K running. Designate a session as meta-conductor (press `M`). Confirm pinned row shows its title.
2. With cursor on that same session, press `d` to delete it.

**Expected:** Session is removed from the list AND pinned row reverts to `— no meta-conductor —`. No stale ID remains. Pressing `M` on a different session works correctly afterwards.

---

### TC-807: Help overlay includes M binding
1. From any TUI state, press `?`.

**Expected:** Help overlay lists the `M` key with description "Set/clear meta-conductor" in the Workflow section.

---

### TC-901: Meta-conductor amber highlight when waiting
1. Fixture L running. Meta-conductor designated (pinned row shows "Meta").
2. Allow the meta-session tmux pane to reach its `❯` prompt (waiting state).

**Expected:** Within ~2 poll ticks the pinned row turns amber/orange. The status symbol updates to the waiting indicator.

---

### TC-902: Escalation to meta when group has no conductor (orphan session)
1. Fixture L running. No conductor assigned to team-beta.
2. Allow `beta-orphan`'s tmux session to reach `waiting` state.

**Expected:** `meta-session` tmux pane receives an escalation message beginning with "Escalation from ..." that includes beta-orphan's title and session ID. No error in stderr.

---

### TC-903: Escalation to meta when session IS the group conductor
1. Fixture L running. Alpha Conductor designated as group conductor.
2. Allow `alpha-conductor` tmux pane to reach `waiting` state.

**Expected:** `meta-session` pane receives an escalation from "Alpha Conductor". The group conductor is NOT escalated to itself.

---

### TC-904: Normal escalation to group conductor (group-alpha worker)
1. Fixture L running. Alpha Conductor designated for team-alpha. Meta-conductor designated.
2. Allow `alpha-worker` pane to reach `waiting` state.

**Expected:** `alpha-conductor` pane receives the escalation (not meta). Meta pane does NOT receive a duplicate.

---

### TC-905: Meta-conductor reply routed to worker
1. TC-902 complete. `meta-session` pane has escalation for beta-orphan.
2. In `meta-session` pane type:
   ```
   @deck-reply worker=<beta-orphan-session-id>
   Here is how to unblock yourself
   @deck-end
   ```
3. Wait one poll tick.

**Expected:** `beta-orphan` pane receives the reply body typed into it.

---

### TC-906: Meta-conductor reply routed to group conductor
1. TC-903 complete. `meta-session` has escalation for alpha-conductor.
2. In `meta-session` pane type:
   ```
   @deck-reply worker=<alpha-conductor-session-id>
   Try delegating to a worker instead
   @deck-end
   ```
3. Wait one poll tick.

**Expected:** `alpha-conductor` pane receives the reply. Reply is NOT re-sent on subsequent ticks (TC-506 dedup applies here too).

---

### TC-907: Meta-conductor not escalated to itself
1. Fixture L running. `meta-session` transitions to `waiting`.

**Expected:** No escalation is sent to `meta-session` from itself. No error in stderr. The pinned row turns amber but nothing is typed into the meta pane automatically.

---

### TC-908: Deck-wide heartbeat sent to meta-conductor
1. Fixture L running with `--conductor-heartbeat 15s`. All sessions active.
2. Wait 15 seconds.

**Expected:** `meta-session` pane receives a heartbeat message that lists group conductors and conductor-less sessions. Message includes session titles and counts (e.g. `Deck heartbeat | N groups | M conductor-less sessions`).

---

### TC-909: --init-conductor-docs writes meta-conductor block on M press
1. One session with `ProjectPath` set to a writable directory.
2. Launch with `./tmux-agent-deck --init-conductor-docs`.
3. Cursor on that session. Press `M` to designate it as meta-conductor.

**Expected:** `<ProjectPath>/CLAUDE.md` is created (or updated) with a block between `<!-- tmux-agent-deck:meta-conductor-role:start -->` and `<!-- tmux-agent-deck:meta-conductor-role:end -->` markers. Block describes the meta-conductor role, heartbeat handling, and `@deck-reply` protocol. Any pre-existing content outside the markers is preserved. Pressing `M` a second time (toggle off) does NOT remove or re-write the block.

---

## Cleanup

```bash
tmux kill-session -t scratch-test 2>/dev/null
tmux kill-session -t foo-a 2>/dev/null
tmux kill-session -t foo-b 2>/dev/null
tmux kill-session -t tui-import-test 2>/dev/null
rm -f /tmp/tad-test.db
rm -f tmux-agent-deck
```
