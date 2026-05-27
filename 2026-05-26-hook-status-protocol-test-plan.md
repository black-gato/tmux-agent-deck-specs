# Manual Test Plan — Hook Status Protocol (BUG-021)

**Date:** 2026-05-26
**Feature:** File-based hook status protocol — env-var identity + atomic JSON status files + 250ms poller fast loop
**Format:** Steps + expected result. Each case is independent unless marked "requires TC-XXX."

---

## Prerequisites

```bash
go build -o tmux-agent-deck .
./tmux-agent-deck install-hooks
export AGENT_DECK_DB=/tmp/tad-hook-test.db
rm -f /tmp/tad-hook-test.db
```

tmux must be running. Claude Code must be on `$PATH` for live hook tests (TC-806 through TC-810).

---

## Fixture K — Hook status protocol

**Setup:** Fresh DB. Create and start one session through the deck.

```bash
./tmux-agent-deck add --title "Hook Worker" --group test --tool claude --project /tmp
./tmux-agent-deck list  # note the session ID (e.g. abc-123)
./tmux-agent-deck session start <id>
# verify: tmux ls shows the session
./tmux-agent-deck --poll 2s
```

| Test | Description | Order |
|------|-------------|-------|
| TC-801 | `AGENTDECK_INSTANCE_ID` injected into session | 1st (before anything else) |
| TC-802 | `UserPromptSubmit` writes `running` status file | any |
| TC-803 | `Stop` writes `waiting` status file | after TC-802 |
| TC-804 | `SessionEnd` writes `dead` status file | last (kills session) |
| TC-805 | `PreCompact` writes no file | any |
| TC-806 | TUI updates within 250ms of hook file change | requires TC-802 or TC-803 |
| TC-807 | Fresh hook status overrides pane scrape | any |
| TC-808 | Stale hook status falls back to pane scrape | any |
| TC-809 | Legacy `agent-deck` entries stripped on install | independent — any |
| TC-810 | Waiting notification fires exactly once | any |
| TC-811 | Missing `AGENTDECK_INSTANCE_ID` — hook no-ops cleanly | independent — any |

---

## TC-801: `AGENTDECK_INSTANCE_ID` injected into new session

1. Start a session through the deck (`./tmux-agent-deck session start <id>`).
2. Check the tmux environment for the session:
   ```bash
   tmux show-environment -t <tmux-session-name> AGENTDECK_INSTANCE_ID
   ```

**Expected:** Prints `AGENTDECK_INSTANCE_ID=<session-id>` (same UUID as the deck's DB session ID). If this is missing, all downstream hook file tests will fail — stop and investigate the `NewSession` call in `cmd/session.go`.

Repeat for a `claude-dangerous` session and a worktree session to confirm all session types inject the var.

---

## TC-802: `UserPromptSubmit` writes `running` status file

1. TC-801 complete. Session running.
2. In a second terminal, watch the hooks dir:
   ```bash
   watch -n 0.5 'ls -la ~/.tmux-agent-deck/hooks/ 2>/dev/null && echo "---" && cat ~/.tmux-agent-deck/hooks/*.json 2>/dev/null || echo "(empty)"'
   ```
3. Attach to the session and submit a prompt to Claude.

**Expected:**
- `~/.tmux-agent-deck/hooks/<session-id>.json` appears (or updates).
- File contains `"status":"running"` and `"event":"UserPromptSubmit"`.
- `ts` field is a recent Unix timestamp.

---

## TC-803: `Stop` writes `waiting` status file

1. TC-802 complete. Claude finishes responding (Stop event fires).

**Expected:**
- `~/.tmux-agent-deck/hooks/<session-id>.json` updates.
- File contains `"status":"waiting"` and `"event":"Stop"`.

---

## TC-804: `SessionEnd` writes `dead` status file

1. Session running. Inside the session, run `exit` or otherwise end the Claude session cleanly (SessionEnd event fires).

**Expected:**
- File updates to `"status":"dead"` and `"event":"SessionEnd"`.

---

## TC-805: `PreCompact` writes no file (or does not change existing file)

1. Session running with an existing status file (from TC-802 or TC-803).
2. Trigger a PreCompact event (use `--claude --max-turns 1` or approach context limit). Note the current file contents and mtime.
3. Wait for the PreCompact hook to fire.

**Expected:** The status file is **not** updated — mtime and contents unchanged. The PreCompact hook fires and exits silently with no side effects.

---

## TC-806: TUI updates within 250ms of hook file change

1. TUI running with `--poll 5s` (long poll interval so tick-driven updates are clearly delayed).
2. Submit a prompt to the worker session (UserPromptSubmit → `running` file written).

**Expected:** TUI status column updates to `running` within ~250ms — well before the 5s poll tick. Confirm by watching the TUI while submitting; the status should flip almost immediately.

---

## TC-807: Fresh hook status overrides pane scrape

1. Session currently showing `running` in TUI based on pane output (spinner visible in pane).
2. Manually write a `waiting` status file for the session to simulate a Stop event:
   ```bash
   echo '{"status":"waiting","event":"Stop","ts":'$(date +%s)'}' > ~/.tmux-agent-deck/hooks/<session-id>.json
   ```
3. Wait ~250ms for the fast loop to pick it up.

**Expected:** TUI status column changes to `waiting` even though the pane still shows a spinner. The hook file wins over pane scrape while the file is fresh (<2 minutes old).

---

## TC-808: Stale hook status falls back to pane scrape

1. TC-807 complete. Session pane shows spinner (pane scrape = `running`).
2. Overwrite the hook file with a timestamp 3 minutes in the past:
   ```bash
   echo '{"status":"waiting","event":"Stop","ts":'$(($(date +%s) - 180))'}' > ~/.tmux-agent-deck/hooks/<session-id>.json
   ```
3. Wait for the next poll tick (up to the `--poll` interval).

**Expected:** TUI status column returns to `running` (pane-scrape result), because the hook file is now stale (>2 minute FreshWindow). The stale file is ignored in `PollOnce`'s `effectiveStatus`.

---

## TC-809: Legacy `agent-deck` entries stripped on install

1. Manually inject a legacy entry into `~/.claude/settings.json`:
   ```bash
   # Edit the file to add "agent-deck hook-handler" under a Stop hooks array
   # Or: use a settings.json fixture with legacy entries
   ```
2. Run `./tmux-agent-deck install-hooks`.
3. Inspect the result:
   ```bash
   grep "hook-handler" ~/.claude/settings.json
   ```

**Expected:**
- Output of `install-hooks` prints each event as `added` or `already registered`.
- `grep` shows only `tmux-agent-deck hook-handler` entries — no `agent-deck hook-handler` lines remain.
- Our entries are not duplicated (run `install-hooks` a second time; counts stay the same).

---

## TC-810: Waiting notification fires exactly once per transition

1. Desktop notifications enabled (or log-based notification in test). Launch with `--auto-escalate --poll 2s`.
2. Worker session in `running` state.
3. Trigger Stop event (Claude finishes — fast loop writes `waiting` file, then 1s pane loop sees `waiting` via `effectiveStatus`).

**Expected:**
- Exactly **one** notification/escalation fires for the `running → waiting` transition — not two (one from the fast loop writing the DB, one from the pane loop reading it).
- Waiting for several more poll ticks does not produce additional notifications.

---

## TC-811: Missing `AGENTDECK_INSTANCE_ID` — hook no-ops cleanly

1. Start a Claude session **outside** the deck (raw `tmux new-session`) so `AGENTDECK_INSTANCE_ID` is not set.
2. With hooks installed, trigger any hook event (e.g. submit a prompt).

**Expected:**
- No file written to `~/.tmux-agent-deck/hooks/`.
- Hook handler exits 0 silently (Claude Code is not blocked or errored).
- No spurious entries appear in the deck's TUI or DB.

---

## Cleanup

```bash
rm -f /tmp/tad-hook-test.db
rm -f tmux-agent-deck
rm -f ~/.tmux-agent-deck/hooks/*.json
./tmux-agent-deck install-hooks --uninstall
```
