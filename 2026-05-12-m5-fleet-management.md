# M5 Fleet Management Implementation Plan

**Status: Complete** — all tasks implemented and verified. See `docs/superpowers/specs/2026-05-10-roadmap.md`.

> This plan matches the current roadmap milestone ordering: M5 is Fleet Management.

**Goal:** Add multi-select, bulk session actions, archive/restore, and tags so the TUI can manage larger fleets without attaching to each tmux session individually.

**Architecture:** The UI model gains durable selection and archived-filter state; list rendering shows selection state and hides archived sessions by default; session persistence moves to schema v3 with `archived` and `tags` columns plus helpers for bulk updates and tag edits; search/filter parsing treats `#tag` prefixes as tag filters while preserving title search.

**Tech Stack:** Go, Bubble Tea, Lip Gloss, modernc SQLite, standard library only.

---

## File Structure

| File | What changes |
|---|---|
| `internal/db/db.go` | Migrate schema to v3 with `archived` and `tags` columns |
| `internal/db/sessions.go` | Persist `Archived`/`Tags`; add archive, bulk move, bulk lookup, and tag update helpers |
| `internal/db/sessions_test.go` | Cover schema defaults and new session helpers |
| `internal/ui/app.go` | Add selection state, archived toggle, tag/search mode, and bulk action flows |
| `internal/ui/dialog.go` | Add tag editor and bulk action dialog handling |
| `internal/ui/keys.go` | Map M3 keys: `space`, `a`, `A`, `t` |
| `internal/ui/list.go` | Render selection markers and apply archived/tag/text filters |
| `internal/ui/list_test.go` | Cover selection marks and filtering rules |
| `internal/ui/app_test.go` | Black-box tests for each user flow |

---

### Task 1: Multi-select foundation

**Files:**
- Modify: `internal/ui/app.go`
- Modify: `internal/ui/list.go`
- Modify: `internal/ui/list_test.go`
- Modify: `internal/ui/app_test.go`

- [ ] Add failing tests for `space` toggling selection on sessions, leaving groups unchanged, and clearing stale selections on reload.
- [ ] Render a selection marker in the list and a selected-count footer when one or more sessions are selected.
- [ ] Keep selection keyed by session ID so cursor movement and reloads do not lose valid selections.
- [ ] Commit: `feat: add multi-select state to session list`

### Task 2: Bulk kill and bulk move

**Files:**
- Modify: `internal/ui/app.go`
- Modify: `internal/ui/dialog.go`
- Modify: `internal/db/sessions.go`
- Modify: `internal/ui/app_test.go`

- [ ] Add failing tests for `d` killing all selected sessions and deleting them from the DB, and for `m` moving all selected sessions to a prompted group.
- [ ] Reuse existing delete/move flows so single-session behavior still works when nothing is selected.
- [ ] Apply operations to selected sessions when selection is non-empty, then clear the selection.
- [ ] Commit: `feat: add bulk kill and move actions`

### Task 3: Bulk send

**Files:**
- Modify: `internal/ui/app.go`
- Modify: `internal/ui/dialog.go`
- Modify: `internal/ui/app_test.go`

- [ ] Add failing tests for `x` sending the same input to every selected running session and ignoring stopped sessions.
- [ ] Keep existing single-session pane-target send behavior when nothing is selected.
- [ ] Send bulk input to pane `0` of each selected running session, then clear selection.
- [ ] Commit: `feat: add bulk send for selected sessions`

### Task 4: Archive and restore

**Files:**
- Modify: `internal/db/db.go`
- Modify: `internal/db/sessions.go`
- Modify: `internal/db/sessions_test.go`
- Modify: `internal/ui/app.go`
- Modify: `internal/ui/list.go`
- Modify: `internal/ui/app_test.go`

- [ ] Add failing tests for schema v3 defaults, archiving a session with `a`, hiding archived sessions by default, and toggling archived visibility with `A`.
- [ ] Archive by moving sessions into the `archived/` namespace and forcing status `stopped`.
- [ ] Restore archived sessions back to `my-sessions` with `a` while archived view is enabled.
- [ ] Commit: `feat: add archive and restore flows`

### Task 5: Tags and tag-prefixed search

**Files:**
- Modify: `internal/db/sessions.go`
- Modify: `internal/db/sessions_test.go`
- Modify: `internal/ui/app.go`
- Modify: `internal/ui/dialog.go`
- Modify: `internal/ui/list.go`
- Modify: `internal/ui/list_test.go`
- Modify: `internal/ui/app_test.go`

- [ ] Add failing tests for editing tags with `t`, persisting them, and filtering the list with `#tag` search prefixes.
- [ ] Store tags as normalized text in SQLite and render them in the detail view or list context needed by tests.
- [ ] Support plain-text title filtering and tag-prefix filtering in the same search flow.
- [ ] Commit: `feat: add session tags and tag search`

### Task 6: Final verification

**Files:**
- Modify as needed based on test results

- [ ] Run targeted package tests after each task and `go test ./...` before the final handoff.
- [ ] Fix any regressions found by local tests or the external verifier with follow-up commits.
- [ ] When the final verifier pass is green, report the completion SHA.
