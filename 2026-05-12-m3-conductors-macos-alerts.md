# M3 Conductors And macOS Alerts Implementation Plan

**Status: Complete** — all tasks implemented and verified. See `docs/superpowers/specs/2026-05-10-roadmap.md`.

> This plan matches the current roadmap milestone ordering: M3 is Conductors and macOS Alerts.

**Goal:** Add group conductor ownership, conductor-targeted escalation, waiting digests, and tunable macOS notifications so waiting agents route attention to the right operator without noisy repeat alerts.

**Architecture:** Groups gain an optional conductor session reference; sessions gain helpers for resolving group peers and conductor targets; the UI adds conductor assignment plus escalation commands; the poller emits structured waiting events into a new `internal/notify` layer that applies style routing and quiet-hour/debounce policy before invoking macOS notifications.

**Tech Stack:** Go, Bubble Tea, Cobra, SQLite via modernc, tmux CLI, `osascript` for macOS notifications.

---

## File Structure

| File | What changes |
|---|---|
| `internal/db/db.go` | Migrate schema for conductor ownership if needed |
| `internal/db/groups.go` | Persist conductor session ID on groups and expose lookup/update helpers |
| `internal/db/groups_test.go` | Cover conductor assignment and lookup |
| `internal/db/sessions.go` | Add helpers for group/conductor child queries and issue-context retrieval |
| `internal/db/sessions_test.go` | Cover conductor-target and waiting-summary helpers |
| `internal/ui/keys.go` | Add `c` and `C` bindings |
| `internal/ui/app.go` | Render conductor sessions distinctly and route conductor/escalation actions |
| `internal/ui/dialog.go` | Add conductor assignment and escalate confirmation flows |
| `internal/ui/app_test.go` | Black-box tests for conductor assignment and escalation UX |
| `internal/state/poller.go` | Detect waiting transitions, digest opportunities, and notification decisions |
| `internal/state/poller_test.go` | Cover notification style routing, quiet hours, and debounce behavior |
| `internal/notify/notify.go` | New notification policy/execution package for macOS alerts |
| `internal/notify/notify_test.go` | Cover quiet-hour windows and debounce logic |
| `cmd/root.go` | Add `--notify`, `--notify-style`, and `--notify-quiet` flags and wire poller config |

---

### Task 1: Group conductor ownership

**Files:**
- Modify: `internal/db/db.go`
- Modify: `internal/db/groups.go`
- Modify: `internal/db/groups_test.go`
- Modify: `internal/ui/keys.go`
- Modify: `internal/ui/app.go`
- Modify: `internal/ui/app_test.go`

- [ ] Add failing tests for setting one session as a group conductor with `c`, replacing any prior conductor for that group, and rendering the conductor distinctly in the session list/detail view.
- [ ] Persist conductor ownership on the group record so it survives reloads and does not require tmux state.
- [ ] Keep assignment scoped to the selected session's group and ignore `c` on group rows.
- [ ] Commit: `feat: add group conductor ownership`

### Task 2: Escalate to conductor

**Files:**
- Modify: `internal/db/sessions.go`
- Modify: `internal/db/sessions_test.go`
- Modify: `internal/ui/app.go`
- Modify: `internal/ui/dialog.go`
- Modify: `internal/ui/app_test.go`

- [ ] Add failing tests for `C` on a waiting session sending a templated handoff to the group conductor with source title and current issue context.
- [ ] Build the handoff from session title, current status, notes, and recent output tail so the conductor receives actionable context.
- [ ] Keep escalation as a no-op with visible error state when no conductor exists or the conductor is not running.
- [ ] Commit: `feat: add conductor escalation flow`

### Task 3: Notification foundation and styles

**Files:**
- Add: `internal/notify/notify.go`
- Add: `internal/notify/notify_test.go`
- Modify: `internal/state/poller.go`
- Modify: `internal/state/poller_test.go`
- Modify: `cmd/root.go`
- Modify: `cmd/cmd_test.go`

- [ ] Add failing tests for `--notify` opt-in and `--notify-style` modes `waiting`, `conductor`, and `digest`.
- [ ] Create a notification package that formats macOS `osascript` calls behind a small interface so tests can stay black-box above it.
- [ ] Route waiting-session alerts according to style: direct waiting alert, conductor-targeted alert, or digest-only suppression.
- [ ] Commit: `feat: add configurable macOS notification styles`

### Task 4: Waiting summary digest

**Files:**
- Modify: `internal/db/sessions.go`
- Modify: `internal/db/sessions_test.go`
- Modify: `internal/state/poller.go`
- Modify: `internal/state/poller_test.go`

- [ ] Add failing tests for a short conductor-facing digest summarizing all waiting child sessions in a group.
- [ ] Aggregate only waiting non-conductor sessions and include concise issue context for each child.
- [ ] Emit digests only when style allows them and a conductor target exists.
- [ ] Commit: `feat: add conductor waiting digests`

### Task 5: Quiet hours and debounce

**Files:**
- Modify: `internal/notify/notify.go`
- Modify: `internal/notify/notify_test.go`
- Modify: `internal/state/poller.go`
- Modify: `internal/state/poller_test.go`
- Modify: `cmd/root.go`
- Modify: `cmd/cmd_test.go`

- [ ] Add failing tests for `--notify-quiet` windows and cooldown-based suppression of repeated alerts.
- [ ] Parse quiet-hour configuration into policy objects that can suppress notifications by local time window and by per-session/per-group debounce.
- [ ] Apply the policy consistently to waiting, conductor, and digest notification styles without changing `--notify` opt-in behavior.
- [ ] Commit: `feat: add quiet hours and alert debounce`

### Task 6: Final verification

**Files:**
- Modify as needed based on test results

- [ ] Run targeted package tests after each task and `go test ./...` before final handoff.
- [ ] Fix verifier failures with focused follow-up commits and re-notify after each one.
- [ ] When the final external verifier is green, send the completion message with the final SHA.
