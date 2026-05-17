# Known Bugs

Tracked bugs in tmux-agent-deck. Newest first. Status: `open`, `in-progress`, `fixed`.

Current repo status as of 2026-05-17: All bugs BUG-001 through BUG-012 are fixed. No open bugs.

---

## BUG-012: broadcast and multi-select send silently skip waiting and idle sessions

**Reported:** 2026-05-17
**Status:** fixed
**Severity:** high (broadcast sends nothing in the most common real-world use case)

### Symptom

Pressing `b` (broadcast) and submitting text appears to succeed but no sessions receive the message. The same silent failure affects `x` (send-pane) when sessions are multi-selected.

### Root cause

`internal/ui/dialog.go` filtered sessions with `s.Status != "running"` in both the broadcast loop (line 443) and the send-pane multi-select loop (line 387). Sessions in `waiting` or `idle` state — the most common targets for broadcast — were silently skipped. Single-target `x` (no selection) did not have this filter and worked correctly.

### Resolution

Changed both filters from `s.Status != "running"` to `s.TmuxSession == "" || s.Status == tmux.StatusStopped || s.Status == tmux.StatusError`. Sessions in `waiting`, `idle`, or `running` state now receive broadcasts. Only sessions with no tmux backing (`stopped`/`error`) are skipped.

### Files touched

- `internal/ui/dialog.go`
- `internal/ui/app_test.go`

---

## BUG-011: fullscreen output view hides dialogs and makes the TUI appear frozen

**Reported:** 2026-05-16
**Status:** fixed
**Severity:** high (common workflows become unusable from fullscreen mode)

### Symptom

After pressing `v` to enter fullscreen output mode, opening a dialog-backed workflow such as send keys (`x`), broadcast (`b`), or new session (`n`) makes the TUI appear frozen.

Reported affected flows:

1. Press `v`, then `x` for send.
2. Press `v`, then `b` for broadcast.
3. Press `v`, then `n` for new session.

The app likely still receives input, but the active prompt is not visible and normal navigation keys no longer behave as expected.

### Suspected root cause

`internal/ui/app.go` renders fullscreen before it considers normal dialog overlays:

```go
if m.viewFull {
    sep := strings.Repeat("─", m.width)
    detail := m.RenderDetailPanel(m.width, contentH)
    return header + "\n" + sep + "\n" + detail + "\n" + footer
}
```

For non-fullscreen layout, dialog rendering happens later:

```go
if m.mode != "" && m.mode != "edit-notes" {
    rightContent = m.renderDialog()
} else {
    rightContent = m.RenderDetailPanel(rightW, contentH)
}
```

This means `x`, `b`, and `n` can set `m.mode` to `send-pane`, `broadcast`, or `new-session`, but `View()` continues to render only the fullscreen detail panel. Once `m.mode != ""`, key handling routes to `updateDialog`, so pressing `v` no longer exits fullscreen; it is treated as dialog input. That combination looks like a freeze.

### Expected behaviour

Dialog-backed workflows should remain visible and usable from fullscreen mode.

Acceptable behaviours:

1. Opening `send`, `broadcast`, or `new-session` while fullscreen should show the dialog over the fullscreen output view.
2. Or opening those workflows should automatically leave fullscreen and show the normal split layout with the dialog.
3. Esc / Ctrl-C should always make it clear that the dialog was canceled and restore usable navigation.

### Planned fix

Prefer preserving fullscreen output and rendering the active dialog as an overlay or replacement prompt area when `m.viewFull && m.mode != ""`.

Implementation options:

1. **`internal/ui/app.go` `View()`** — check for active non-help dialog mode before the fullscreen early return, and render `m.renderDialog()` visibly.
2. **`internal/ui/app.go` `updateNavigation()`** — when starting `send-pane`, `broadcast`, or `new-session`, set `m.viewFull = false` so the existing split-layout dialog path is reused.
3. Add a small regression test for each entry point, or at least a table-driven test that enters fullscreen, opens the action, and asserts the resulting view contains the expected prompt.

### Test plan

- `internal/ui/app_test.go` — fullscreen then `x` shows `Send:` in `View()`.
- `internal/ui/app_test.go` — fullscreen then `b` shows the broadcast prompt / scope selector in `View()`.
- `internal/ui/app_test.go` — fullscreen then `n` shows `Session title:` in `View()`.
- Regression guard: while a dialog is open from fullscreen, Esc exits dialog mode and the TUI remains usable.

### Files likely touched

- `internal/ui/app.go`
- `internal/ui/app_test.go`

---

## BUG-010: existing tmux sessions show `running` on app startup even when inactive for days

**Reported:** 2026-05-16
**Status:** fixed
**Severity:** medium (status view is misleading at launch; old inactive work appears active)

### Symptom

When `tmux-agent-deck` starts, existing sessions can all display as `running` even though they have not produced output or been active for days.

This is visible immediately after launching the TUI. Sessions that should appear `idle` get treated as recently active until the poller observes enough samples in the current process.

### Relationship to BUG-005

This is related to, but distinct from, BUG-005.

BUG-005 fixed stale output detection after the poller has observed a session and can compare pane output across polls. This startup case still fails because the poller has no in-memory baseline when the app first launches.

### Root cause

`internal/state/poller.go` stores pane activity in memory only:

```go
lastChange map[string]time.Time
lastOutput map[string]string
```

On first observation after startup, `observeOutput` treats the captured pane output as newly changed:

```go
if !hadPrev || prev != out {
    p.lastOutput[id] = out
    p.lastChange[id] = now
}
```

That means every pre-existing tmux session gets `lastChange = now` on the first poll, regardless of whether the actual tmux pane has been silent for minutes, hours, or days. `DetectStatus` then sees `now.Sub(lastChange) <= 30s` and returns `running` unless the output matches a waiting prompt.

The persisted `sessions.last_active` value is not used to seed the poller's `lastChange`, and the tmux client does not currently expose tmux's own activity timestamp, such as `#{session_activity}`.

### Expected behaviour

When the app starts, status detection for existing tmux sessions should reflect real inactivity age instead of treating startup as fresh activity.

Examples:

1. A session whose pane output has not changed for days should appear `idle` on the first poll.
2. A session currently producing output should remain `running`.
3. A session sitting at a prompt should still appear `waiting`.

### Planned fix

1. Seed `lastChange` for first-observed sessions from a durable activity source instead of `now`.
2. Preferred source: tmux's session or pane activity timestamp, e.g. `tmux display-message -p -t <session> '#{session_activity}'`.
3. Fallback source: the DB `last_active` value if tmux activity cannot be read.
4. Keep the existing output-comparison logic for subsequent polls so BUG-005 stays fixed.

### Test plan

- `internal/state/poller_test.go` — first poll for a session with an old activity timestamp and stale non-prompt output marks it `idle`.
- `internal/state/poller_test.go` — first poll for a session with recent activity stays `running`.
- `internal/state/poller_test.go` — first poll for an old session at a waiting prompt still marks it `waiting`.
- Existing BUG-005 tests continue to pass.

### Files touched

- `internal/state/poller.go`
- `internal/state/poller_test.go`
- `internal/tmux/client.go`
- `internal/tmux/client_test.go`

---

## BUG-009: deleting a session can fail with raw `exit status 1` when its tmux session is already gone

**Reported:** 2026-05-16
**Status:** fixed
**Severity:** medium (session cannot be removed from the TUI; raw infrastructure error leaks to the user)

### Symptom

A newly created session can appear fine in the TUI, but deleting it fails with:

```text
error: exit status 1

Press q or ctrl+c to quit
```

Example report: create a session titled `",,,,fsd"`, then try to delete it. The title is likely incidental; the user-visible failure is that delete surfaces a raw tmux exit code instead of removing the session or explaining what went wrong.

### Root cause

`internal/ui/app.go:736` in `deleteSession` treats any tmux kill failure as fatal:

```go
if session.TmuxSession != "" && m.tmuxC != nil {
    if err := m.tmuxC.KillSession(session.TmuxSession); err != nil {
        return err
    }
}
return db.DeleteSession(m.conn, session.ID)
```

`internal/tmux/client.go:59` `KillSession` directly returns `runCmd("tmux", "kill-session", "-t", name)`. If tmux exits with status 1 because the session no longer exists, that raw process error bubbles into the UI unchanged, and the DB delete never runs.

### Expected behaviour

Deleting a session should succeed even if its backing tmux session is already gone.

Acceptable behaviours:

1. Treat tmux exit code 1 / "can't find session" as an already-deleted no-op and still remove the DB row.
2. If deletion truly fails, show a user-facing message that names the tmux session and explains the problem instead of `exit status 1`.

### Planned fix

1. **`internal/tmux/client.go`** — make `KillSession` mirror `SessionExists`: if `tmux kill-session -t <name>` exits 1 because the session does not exist, treat that as success.
2. **`internal/ui/app.go`** — keep the DB delete path running when the tmux session is already absent.
3. Consider wrapping unexpected tmux errors with session context, e.g. `delete tmux session %q: %w`, so the UI error is actionable if a real failure occurs.

### Test plan

- `internal/tmux/client_test.go` — killing a missing session is treated as success.
- `internal/ui/app_test.go` — deleting a session whose `tmux_session` is recorded in the DB but absent in tmux still removes the DB row and does not set `m.err`.
- Regression guard: deleting a live tmux-backed session still kills the tmux session and removes the DB row.

### Files touched

- `internal/tmux/client.go`
- `internal/tmux/client_test.go`
- `internal/ui/app.go`
- `internal/ui/app_test.go`

---

## BUG-008: tmux session names use opaque `ad-` prefix instead of readable `tma-<name>-<id>` convention

**Reported:** 2026-05-14
**Status:** fixed
**Severity:** low (UX; tmux sessions are hard to identify outside the TUI)

### Symptom

When the TUI starts a tmux session it is named `ad-<first 8 chars of UUID>`, e.g. `ad-3f7a1c2b`. Inside the TUI this is fine — sessions are identified by their title. But if the user lists tmux sessions directly (`tmux ls`) or attaches from outside the TUI, the names are opaque and impossible to associate with the session title without cross-referencing the DB.

### Root cause

`internal/ui/app.go:901` in `ensureStarted`:

```go
tmuxName := fmt.Sprintf("ad-%s", s.ID[:8])
```

The name is derived solely from the internal UUID, discarding the human-readable title entirely.

### Expected behaviour

Tmux session names should follow the convention `tma-<session-name>-<random-id>` so that:

1. The `tma-` prefix makes all managed sessions identifiable as belonging to tmux-agent-deck.
2. The session title (slugified) is visible in `tmux ls` output.
3. The random suffix ensures uniqueness even when two sessions share the same title — no collision handling needed.

Example: a session titled `"api worker"` → `tma-api-worker-3f7a1c2b`.

### Planned fix

1. **`internal/ui/app.go` `ensureStarted`** — replace the name derivation:
   ```go
   tmuxName := fmt.Sprintf("tma-%s-%s", slugify(s.Title), s.ID[:8])
   ```
2. Add a `slugify` helper that lowercases, replaces non-alphanumeric runs with `-`, and trims leading/trailing dashes. Keep it to a reasonable max length (e.g. 40 chars for the title portion) to stay within tmux session name limits.
3. No DB migration needed — `tmux_session` is set at start time, not stored in a unique-constrained column.

### Test plan

- `internal/ui/app_test.go` — `ensureStarted` on a new session produces a tmux name matching `tma-<slug>-<8hex>`.
- `internal/ui/app_test.go` — two sessions with the same title each get distinct tmux names (different ID suffix).
- Existing `ensureStarted` tests for the "already running" path continue to pass.

### Files touched

- `internal/ui/app.go`
- `internal/ui/app_test.go`

---

## BUG-007: project path field in new-session dialog has no tab completion

**Reported:** 2026-05-14
**Status:** fixed
**Severity:** low (UX paper cut; users can still paste or type the full path)

### Symptom

In the new-session flow (`n`), step 2 prompts for `Project path:`. Pressing Tab in that field does nothing — no path is completed, no candidate list is shown, no feedback at all. Shell users expect Tab to expand partial paths against the filesystem.

### Reproduction

1. Press `n` to start a new session.
2. Enter a title, press Enter to advance to the project path step.
3. Type a partial path such as `~/Pro` and press Tab.
4. Observe that nothing happens; `~/Pro` stays unchanged.

### Root cause

`internal/ui/dialog.go:90` `updateDialog` has no branch for `tea.KeyTab` when `mode == "new-session" && m.dialog.step == 1`. Tab handling only exists for:

- `broadcast` mode — toggles the scope flag (line 96).
- `new-session` step 2 (tool selection) — left/right arrows cycle options, Tab is unhandled.

For the path step, `tea.KeyTab` arrives with no `msg.Runes`, falls through to the `default` branch at line 141, and contributes nothing to `m.dialog.value`. The keypress is silently dropped.

### Expected behaviour

In the project path field:

- Single Tab with one matching filesystem entry — complete `m.dialog.value` to the match, appending `/` if it resolves to a directory.
- Single Tab with multiple matches — extend `m.dialog.value` to the longest common prefix.
- Second Tab with multiple matches — render the candidate list above the prompt.
- No matches — no-op.

`~` and `$VAR` should be expanded the same way `expandPath` already does at commit time.

### Planned fix

1. **`internal/ui/dialog.go`** — branch in `updateDialog` for the path step on `tea.KeyTab`. Split current value into directory + prefix, expand `~` / env vars, call `os.ReadDir` on the directory, filter entries by prefix.
2. New helper (likely `completePath`) returns the completed value and an optional candidate list; pure function so it can be unit-tested without a tmux client.
3. `dialogState` gains a `candidates []string` field rendered by `renderDialog` when populated.
4. Consider applying the same machinery to the `move` and `new-group` dialogs, but completing against in-DB group paths rather than the filesystem — out of scope for this bug; track separately if desired.

### Test plan

- `internal/ui/dialog_test.go` — tab on `~/` in a temp HOME with a known directory layout expands to the longest common prefix; second tab populates `candidates`.
- `internal/ui/dialog_test.go` — tab in non-path dialogs (e.g. rename, edit-notes) remains a no-op (regression guard).
- Existing new-session tests continue to pass; commit path still uses `expandPath`.

### Files touched

- `internal/ui/dialog.go`
- `internal/ui/dialog_test.go` (new or extend existing)

---

## BUG-006: `--notify-style` and `--notify-quiet` flags have no usage documentation

**Reported:** 2026-05-14
**Status:** fixed
**Severity:** low (discoverability; no functional impact)

### Symptom

Running `tmux-agent-deck --help` shows:

```
--notify-style string   Notification style: waiting, conductor, digest (default "waiting")
--notify-quiet string   Quiet hours / cooldown policy
```

Neither flag explains what the values do or what format `--notify-quiet` expects. A user who has never read the source code cannot tell what `conductor` or `digest` mean, or how to write a valid quiet policy string.

### Root cause

`cmd/root.go:167-168` registers both flags with single-line usage strings:

```go
rootCmd.PersistentFlags().StringVar(&notifyStyle, "notify-style", "waiting", "Notification style: waiting, conductor, digest")
rootCmd.PersistentFlags().StringVar(&notifyQuiet, "notify-quiet", "", "Quiet hours / cooldown policy")
```

Cobra renders the usage string verbatim in `--help` output. There is no secondary long-form help, man page, or docs site that covers these flags.

### Expected behaviour

`--help` should explain each `--notify-style` value and show a valid `--notify-quiet` format example, e.g.:

```
--notify-style string   Notification routing style:
                          waiting    alert per session the moment it goes waiting (default)
                          conductor  alert names the group conductor: "lead: worker is waiting"
                          digest     one combined alert per poll cycle listing all waiting workers in the group
--notify-quiet string   Quiet hours and cooldown policy (comma-separated key=value):
                          cooldown=5m          suppress duplicate alerts within this duration
                          hours=22:00-08:00    suppress all alerts during this time window
                          example: --notify-quiet "cooldown=5m,hours=22:00-08:00"
```

### Fix

Expand the usage strings in `cmd/root.go` `init()` to multi-line backtick strings. Cobra passes them through unchanged, so newlines and indentation render correctly in `--help`.

### Files touched

- `cmd/root.go`

---

## BUG-005: idle detection uses stale output heuristics, so inactive sessions can remain `running`

**Reported:** 2026-05-14
**Status:** fixed
**Severity:** medium (misleading status; weakens observability and waiting triage)

### Symptom

A tmux session can sit untouched for well over 30 seconds and still display as `running` instead of `idle`.

Common example:

1. The pane output contains text like `Thinking...`, `Running tests...`, or an old spinner glyph.
2. The underlying process stops producing new output.
3. Even after 30+ seconds, the session remains `running`.

### Root cause

Status detection in [internal/tmux/status.go](/Users/anthonymirville/Projects/tmux-agent-deck/.worktrees/feature-feedback/internal/tmux/status.go:56) is heuristic:

1. If the current pane tail contains spinner characters or the substrings `Thinking` / `Running`, it returns `running` immediately.
2. The idle fallback only triggers when none of those markers are present and `time.Since(lastChange) > 30*time.Second`.

But `lastChange` in [internal/state/poller.go](/Users/anthonymirville/Projects/tmux-agent-deck/.worktrees/feature-feedback/internal/state/poller.go:145) is not the time of the last pane-output change. It is only updated when the derived status changes:

```go
if newStatus != s.Status {
    p.setLastChange(s.ID, now)
    ...
}
```

As a result, stale pane text that still looks "running-ish" can pin the session in `running` indefinitely, and the 30-second rule is not a true inactivity timer.

### Resolution

`lastChange` now tracks the time the pane output last *changed* rather than the time of the last status transition.

1. **`internal/state/poller.go`** — new `lastOutput map[string]string` stores the previous capture per session. `observeOutput` replaces `lastObservedChange` / `setLastChange`: on each poll it compares the new capture to the prior one and only advances `lastChange` when they differ. The stale "update lastChange on status transition" call after `UpdateSessionStatus` is removed.
2. **`internal/tmux/status.go`** — `DetectStatus` takes an extra `now time.Time` parameter and computes `now.Sub(lastChange)` (the poller's injected clock) instead of `time.Since(lastChange)`. The idle check is moved *before* the spinner / `Thinking` / `Running` heuristics, so stale running markers on a silent pane resolve to `idle` after 30s.
3. Waiting prompt detection is unchanged — a live prompt still wins regardless of how long the pane has been silent.

### Test plan

- `internal/tmux/status_test.go` `TestDetectStatusStaleRunningMarkerBecomesIdle` — stale spinner / `Thinking` / `Running` text with a 31s-old `lastChange` returns `idle`.
- `internal/state/poller_test.go` `TestPollerMarksSessionIdleWhenOutputUnchanged` — two polls 31s apart with identical "⠋ Thinking..." output transitions status to `idle`.
- `internal/state/poller_test.go` `TestPollerKeepsRunningWhenOutputKeepsChanging` — two polls 31s apart with changing spinner output keep status at `running`.
- All existing `DetectStatus` call sites updated to the new 4-arg signature; `go test ./...` passes.

### Files touched

- `internal/state/poller.go`
- `internal/state/poller_test.go`
- `internal/tmux/status.go`
- `internal/tmux/status_test.go`

---

## BUG-004: pressing `C` on a non-waiting session surfaces an error instead of no-op

**Reported:** 2026-05-13
**Status:** fixed
**Severity:** low (jarring UX; no data loss)

### Symptom

Pressing `C` (escalate to conductor) on any session that is not in the `waiting` state produces:

```
error: session "hell" is not waiting
```

This strands the user on the error screen (compounded by BUG-001).

### Root cause

`internal/ui/app.go:742` in `escalateSelectedSession`:

```go
if session.Status != tmux.StatusWaiting {
    return fmt.Errorf("session %q is not waiting", session.Title)
}
```

The escalate action only makes sense for a waiting session, but instead of silently doing nothing (the pattern used by other guarded actions), it returns an error that propagates to `m.err`.

### Resolution

`internal/ui/app.go` now uses the guarded no-op:

```go
if session.Status != tmux.StatusWaiting {
    return nil
}
```

The non-waiting path no longer surfaces an error. Escalation still errors for genuinely invalid states such as "already the conductor" or "conductor not running".

### Test plan

- `internal/ui/app_test.go` — pressing `C` on a running/idle/stopped session sets neither `m.err` nor `m.mode`; model state is unchanged.
- Existing escalation tests continue to pass for the waiting-session path.

### Files touched

- `internal/ui/app.go`
- `internal/ui/app_test.go`

---

## BUG-003: send-pane and broadcast send Enter after text; `-l` flag missing from send-keys

**Reported:** 2026-05-13
**Status:** fixed
**Severity:** medium (send behavior is unreliable; Enter fires commands unintentionally)

### Symptom

Using `x` (send-pane) or `b` (broadcast), the text typed in the dialog is submitted to the target session with a trailing Enter — the AI agent executes the command immediately rather than just seeing text typed into the prompt.

### Root cause

`internal/tmux/client.go:114` calls:

```go
runCmd("tmux", "send-keys", "-t", paneTarget(session, paneIndex), keys)
```

Two problems:

1. **Missing `-l` flag.** Without `-l`, `tmux send-keys` interprets the argument as tmux key names, not literal text. For a multi-character string that isn't a known key name, tmux falls back to character-by-character dispatch — but the behavior is version-dependent and can produce unexpected results, including spurious Enter keypresses on some tmux versions.

2. **`interceptCtrl` intentionally exploits no-`-l` mode.** `internal/ui/dialog.go:26-40` encodes Ctrl keys as strings like `"C-c"`, `"C-d"`, `"C-z"`, `"C-l"`, `"C-u"` and appends them to `m.dialog.value`. These strings are only interpreted as control characters by `tmux send-keys` when the `-l` flag is absent. Adding `-l` would fix the spurious-Enter problem but break control-key forwarding.

The two concerns are currently conflated in a single `dialog.value` string and a single `SendKeys` call, which makes it impossible to fix one without breaking the other.

### Reproduction

1. Start a running tmux session with a Claude Code agent at its `>` prompt.
2. Press `x`, type any text (e.g. `hello`), press Enter to confirm.
3. Observe that the agent's session receives `hello` AND executes it (Enter was sent).

### Resolution

The shipped fix separates literal text from control-key sequences at the send layer:

1. **`internal/tmux/client.go`** sends literal text with `tmux send-keys -l`.
2. **`internal/ui/dialog.go`** stores intercepted ctrl sequences separately in `dialogState.ctrlKeys`.
3. **`internal/tmux/client.go` / `internal/testutil/tmux.go`** use `SendRawKeys` for tmux key names such as `C-c`.

Enter is no longer appended implicitly. Submission now only happens when the user explicitly sends a control key for it.

### Test plan

- `internal/tmux` — `SendKeys` with literal text containing a space sends the text without Enter; a ctrl key `"C-c"` is sent as Ctrl+C.
- `internal/ui/app_test.go` — in send-pane dialog, `interceptCtrl` stores ctrl keys separately and committed sends split literal text from raw ctrl sequences.
- Regression: broadcast ctrl key test (sending `C-c` to broadcast still works).

### Files touched

- `internal/tmux/client.go`
- `internal/tmux/client_test.go` (or add)
- `internal/testutil/tmux.go`
- `internal/ui/dialog.go`
- `internal/ui/app_test.go`

---

## BUG-002: duplicate group path surfaces raw SQLite UNIQUE constraint error

**Reported:** 2026-05-13
**Status:** fixed
**Severity:** medium (poor UX; combines with BUG-001 to trap the user on an unreadable error screen)

### Symptom

Creating a group with a path that already exists produces:

```
constraint failed: UNIQUE constraint failed: groups.path (1555)
```

In the TUI this is rendered raw via `m.err` and (because of BUG-001) leaves the user on a dead-end error screen. In the CLI it's wrapped as `create group: <raw>`.

### Reproduction

TUI:
1. Press `g`, type `work`, Enter — group created.
2. Press `g`, type `work` again, Enter — raw SQLite error appears.

CLI:
```
tmux-agent-deck group create work
tmux-agent-deck group create work   # → error
```

### Root cause

`internal/db/groups.go:19-30` `CreateGroup` does a plain `INSERT INTO groups (path, ...)` and returns whatever SQLite emits. `path` is the primary key, so a second insert with the same path raises `SQLITE_CONSTRAINT_PRIMARYKEY` (1555). No caller translates this into a friendly message.

Same shape of issue could surface for any other `INSERT` against a unique/PK column (sessions: `id` UUID — collision astronomically unlikely, so not a practical concern; session titles are *not* unique).

Note: the bug title in the original report said "same name" but the actual constraint is on `path`. `name` is not unique. Two groups under different parents *can* share a leaf name (`work/frontend` and `personal/frontend` are fine); the collision happens when the full path matches.

### Resolution

The current implementation translates the constraint violation into a typed error at the data layer and avoids exposing raw SQL text:

1. **`internal/db/groups.go`** returns `ErrGroupExists` for duplicate `groups.path`.
2. **`cmd/group.go`** formats that as `group "X" already exists`.
3. **`internal/ui/dialog.go`** treats a duplicate create as "navigate to the existing group" instead of surfacing a raw SQLite error.

### Test plan

- `internal/db/groups_test.go` — creating a group with an existing path returns an error that satisfies `errors.Is(err, db.ErrGroupExists)`.
- `cmd/cmd_test.go` — `group create work` twice via `RunWith`; second run's stderr/stdout contains `already exists` and not `UNIQUE constraint`.
- `internal/ui/app_test.go` — duplicate group creation leaves the TUI focused on the existing group instead of creating a second entry.

### Files touched

- `internal/db/groups.go`
- `internal/db/groups_test.go`
- `cmd/group.go`
- `cmd/cmd_test.go`
- `internal/ui/dialog.go`
- `internal/ui/app_test.go`

---

## BUG-001: ctrl+c does not quit the TUI; error screen is a dead end

**Reported:** 2026-05-13
**Status:** fixed
**Severity:** medium (UX dead end; `q` still works in navigation mode)

### Symptom

When an error arises in the TUI, the error screen appears stuck — ctrl+c does nothing. The user has no visible way to quit.

### Reproduction

1. Trigger any code path that sets `m.err` (e.g. a DB failure during `Reload`, or an error from `commitDialog` that leaves the user in navigation mode).
2. The view renders only `error: <message>` with no footer or hint.
3. Press ctrl+c — nothing happens.
4. `q` does still quit (from navigation mode) but the user has no way to know that.

### Root cause

Two compounding problems:

1. **`internal/ui/keys.go` `keyTypeMap`** does not map `tea.KeyCtrlC` to any action. `actionForKey` returns `""` for ctrl+c, so `updateNavigation`'s switch falls through and the keypress is silently dropped.
2. `tea.WithAltScreen()` (`cmd/root.go:130`) puts the terminal in raw mode, so ctrl+c is delivered as a `KeyCtrlC` byte, **not** SIGINT. The parent `signal.NotifyContext` in `Execute()` and bubbletea's signal handler never see it.
3. **`internal/ui/app.go` `View()`** returns only `"error: " + m.err.Error()` when `m.err != nil` — no footer, no exit instructions. The user sees a bare error and instinctively reaches for ctrl+c.
4. In non-send dialogs (`new-session`, `rename`, `move`, `edit-notes`, `edit-tags`, `fork-session`, `new-group`, `filter`), ctrl+c is also silently dropped — only `Esc` cancels them. `send-pane` and `broadcast` intentionally intercept ctrl+c into the dialog buffer (`internal/ui/dialog.go:26-40`); that behavior is correct and must be preserved so users can send `^C` to tmux panes.

### Resolution

The current implementation applies the scoped fix:

1. **`internal/ui/keys.go`** — add `tea.KeyCtrlC: "quit"` to `keyTypeMap`. Makes ctrl+c quit globally from navigation mode.
2. **`internal/ui/dialog.go` `updateDialog`** — for dialog modes *other than* `send-pane` and `broadcast`, treat `tea.KeyCtrlC` as cancel (equivalent to `Esc`: clear `m.mode`, do not commit). The existing `interceptCtrl` path in `send-pane`/`broadcast` runs first and is unaffected.
3. **`internal/ui/app.go` `View()`** — when `m.err != nil`, append `"\n\nPress q or ctrl+c to quit"` so the error screen is no longer a dead end.

### Test plan

- Failing test: in navigation mode, send `tea.KeyMsg{Type: tea.KeyCtrlC}` and assert the returned command is `tea.Quit`.
- Failing test: in `new-session` dialog, send ctrl+c and assert `m.mode == ""` and dialog state is cleared, with no session created.
- Regression test: in `send-pane` dialog, send ctrl+c and assert the dialog buffer contains the literal `"C-c"` (existing intercept preserved).
- Failing test: when `m.err != nil`, `View()` output contains the substring `"Press q or ctrl+c to quit"`.

### Files touched

- `internal/ui/keys.go`
- `internal/ui/dialog.go`
- `internal/ui/app.go`
- `internal/ui/app_test.go`
- `internal/ui/dialog_test.go` (new or extend existing)
