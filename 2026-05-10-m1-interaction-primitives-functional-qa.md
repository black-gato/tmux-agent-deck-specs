# M1 Interaction Primitives — Functional QA Checklist

Use this checklist for real end-to-end behavior that unit/component tests do not cover well: actual tmux panes, SQLite state, terminal rendering, and interactive keyboard flows.

## Current Coverage

Automated E2E coverage now exists under `test/e2e` and runs with:

```bash
go test -timeout=140s -tags=e2e ./test/e2e
```

Covered by the E2E suite:

- send to pane, including `Ctrl+C` interception and stopped-session no-op
- pane targeting, including active-pane reset after navigation/reload
- fork session creation, including the extra startup-script confirmation step, and DB field cloning
- broadcast to direct groups, sub-groups, and session-row group roots
- shell prompt status transitions: `waiting` -> `running` -> `idle` -> `error`
- stopped-session attach and already-running attach without duplicate session creation
- narrow/wide rendering, dialog escape, and resize recovery

Still worth manual verification:

- attach detach and return behavior in a real interactive terminal after leaving tmux
- forked session start/attach independence from the source session
- real launch-profile startup behavior for `claude`, `claude-danger`, `codex`, `codex-yolo`, and `shell`
- broadcast scope indicator text while toggling between direct and sub-group modes
- long dialog editing paths: long input, repeated backspace, submit

## Setup

- [x] Build the local binary:

```bash
go build -o ./tmux-agent-deck .
```

- [x] Use a disposable test database:

```bash
export AGENT_DECK_DB="$(mktemp -t agent-deck-qa.XXXXXX.db)"
```

- [x] Create nested groups and sessions:

```bash
./tmux-agent-deck group create qa --tool shell
./tmux-agent-deck group create qa/frontend --tool shell
./tmux-agent-deck group create qa/backend --tool shell
./tmux-agent-deck add --title qa-root --group qa --tool shell --project "$PWD"
./tmux-agent-deck add --title qa-front --group qa/frontend --tool shell --project "$PWD"
./tmux-agent-deck add --title qa-back --group qa/backend --tool shell --project "$PWD"
./tmux-agent-deck add --title qa-stopped --group qa --tool shell --project "$PWD"
./tmux-agent-deck session start qa-root
./tmux-agent-deck session start qa-front
./tmux-agent-deck session start qa-back
```

- [x] Confirm tmux sessions exist:

```bash
tmux list-sessions | grep '^ad-'
```

- [ ] Record the `TmuxSession` values for each started session:

```bash
./tmux-agent-deck list --json
```

Use those names anywhere this checklist says `<qa-root-tmux>`, `<qa-front-tmux>`, or `<qa-back-tmux>`.

**Feedback:**

---

## Send To Pane (`x`)

- [ ] Launch the TUI with the disposable DB:

```bash
AGENT_DECK_DB="$AGENT_DECK_DB" ./tmux-agent-deck
```

- [ ] Select `qa-root`, press `x`, type `echo from-send-pane`, then press `Enter`.
- [ ] Attach to `qa-root` from another terminal or after exiting the TUI.
- [ ] Verify the selected pane received the literal text `echo from-send-pane`.
- [ ] Verify the TUI did not attach, quit, freeze, or move the cursor unexpectedly after committing the send dialog.
- [ ] Repeat `x`, press `Ctrl+C`, then `Enter`.
- [ ] Verify the target pane receives an interrupt key (`^C` / prompt returns), not the literal characters `C-c`.
- [ ] Press `x` on `qa-stopped`.
- [ ] Verify nothing is sent and the TUI remains usable.

**Feedback:**

---

## Pane Targeting (`Tab`)

- [ ] In the `qa-root` tmux session, create a second pane:

```bash
tmux split-window -t <qa-root-tmux>
```

- [ ] Restart or wait for the TUI to refresh while `qa-root` is selected.
- [ ] Verify the detail panel shows multiple panes.
- [ ] Press `Tab`.
- [ ] Verify the active pane indicator moves to the next pane.
- [ ] Press `x`, type `pane-target-check`, then `Enter`.
- [ ] Attach to the tmux session and verify only the active pane received `pane-target-check`.
- [ ] Move the TUI cursor to a different session and back.
- [ ] Verify active pane selection resets to pane `0` after reload/selection refresh.

**Feedback:**

---

## Fork Session (`f`)

- [ ] Select `qa-front`, press `f`, type `qa-front-fork`, then press `Enter`.
- [ ] Press `Enter` again to accept the default empty startup script for the fork.
- [ ] Verify a new session row appears named `qa-front-fork`.
- [ ] Verify it appears in the same group as `qa-front`.
- [ ] Run:

```bash
./tmux-agent-deck list --json
```

- [ ] Verify `qa-front-fork` has the same group, project path, tool, and startup script as `qa-front`.
- [ ] Verify `qa-front-fork` is stopped and does not automatically create a tmux session.
- [ ] Start and attach the fork:

```bash
./tmux-agent-deck session start qa-front-fork
./tmux-agent-deck session attach qa-front-fork
```

- [ ] Verify the fork starts independently from the original session.

**Feedback:**

---

## Broadcast To Direct Group (`b`)

- [ ] Make `qa-root` actively running by attaching to it or using tmux directly to run a long command:

```bash
tmux send-keys -t <qa-root-tmux> 'sleep 60' Enter
```

- [ ] Leave `qa-stopped` stopped.
- [ ] In the TUI, select the `qa` group.
- [ ] Press `b`, leave scope on `this group`, type `direct-broadcast`, then press `Enter`.
- [ ] Verify running direct children of `qa` receive `direct-broadcast`.
- [ ] Verify sessions in `qa/frontend` and `qa/backend` do not receive it.
- [ ] Verify stopped sessions do not receive it.

**Feedback:**

---

## Broadcast To Sub-Groups (`b`, `Tab`)

- [ ] Make `qa-root`, `qa-front`, and `qa-back` actively running:

```bash
tmux send-keys -t <qa-root-tmux> C-c 'sleep 60' Enter
tmux send-keys -t <qa-front-tmux> 'sleep 60' Enter
tmux send-keys -t <qa-back-tmux> 'sleep 60' Enter
```

- [ ] In the TUI, select the `qa` group.
- [ ] Press `b`.
- [ ] Press `Tab`.
- [ ] Verify the scope indicator changes from `this group` to `all sub-groups`.
- [ ] Type `subtree-broadcast`, then press `Enter`.
- [ ] Verify running sessions in `qa`, `qa/frontend`, and `qa/backend` receive `subtree-broadcast`.
- [ ] Verify `qa-stopped` still does not receive it.
- [ ] Repeat from a session row instead of a group row.
- [ ] Verify broadcasting from a session row uses that session's group as the broadcast root.

**Feedback:**

---

## Status Heuristics

- [ ] Start one `shell` session and leave it at a `zsh` prompt.
- [ ] Verify it becomes `waiting`.
- [ ] Run `sleep 60` inside that pane.
- [ ] Verify it becomes `running`.
- [ ] Stop typing and let a session sit with unchanged output.
- [ ] Verify it eventually becomes `idle` rather than staying `running`.
- [ ] Kill one managed tmux session externally:

```bash
tmux kill-session -t <tmux-session-name>
```

- [ ] Verify the TUI handles the missing/dead tmux session without crashing.
- [ ] If available locally, repeat with `claude`, `claude-danger`, `codex`, and `codex-yolo` sessions and verify their prompt states are classified as waiting when they return to their ready prompt.

**Feedback:**

---

## Attach / Return Flow

- [ ] Select a stopped session and press `Enter`.
- [ ] Verify the TUI starts the tmux session and attaches to it.
- [ ] Detach from tmux.
- [ ] Verify control returns cleanly to the TUI loop or exits in the expected way for the current attach flow.
- [ ] Select an already-started session and press `Enter`.
- [ ] Verify it attaches to the existing tmux session instead of creating a duplicate.

**Feedback:**

---

## Terminal Rendering And Recovery

- [ ] Run the TUI at roughly 80 columns wide.
- [ ] Verify send, fork, and broadcast dialogs fit without corrupting the divider/footer.
- [ ] Resize to a wide terminal.
- [ ] Verify the split panel redraws cleanly after resize.
- [ ] Open a dialog, press `Esc`.
- [ ] Verify the dialog closes and normal navigation resumes.
- [ ] Open a dialog, type a long value, backspace several characters, then submit.
- [ ] Verify text editing works and the TUI remains responsive.

**Feedback:**

---

## Cleanup

- [ ] Stop test sessions:

```bash
./tmux-agent-deck session stop qa-root
./tmux-agent-deck session stop qa-front
./tmux-agent-deck session stop qa-back
./tmux-agent-deck session stop qa-front-fork
```

- [ ] Remove the disposable database:

```bash
rm -f "$AGENT_DECK_DB"
unset AGENT_DECK_DB
```

**Feedback:**
