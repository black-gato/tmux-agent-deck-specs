# M6 Polish & Onboarding Implementation Plan

**Status: Complete** — all tasks implemented and verified. See `docs/superpowers/specs/2026-05-10-roadmap.md`.

**Goal:** Finish the roadmap by landing the polish layer that turns the app from "feature-complete" into "comfortable to live in": fast list filtering, discoverable keybindings, an onboarding empty state, configurable poll cadence, and a headless daemon mode for background alerting.

**Roadmap source:** `docs/superpowers/specs/2026-05-10-roadmap.md` M6 section.

**Architecture:** A new UI mode (`filterMode`) gives `/` live prefix filtering of the session list, reusing the existing search/tag filter pipeline from M5. A new full-screen overlay state (`helpMode`) renders a keybinding cheat sheet from a single source-of-truth table in `internal/ui/keys.go`. The empty-state branch in `app.View` renders an onboarding hint when the DB has zero sessions. `cmd/root.go` exposes `--poll` (duration) wired through `state.Poller` and `--headless` which runs the poller + notifier without launching Bubble Tea.

**Tech Stack:** Go, Bubble Tea, Lip Gloss, modernc SQLite, standard library only. No new third-party deps.

---

## File Structure

| File | What changes |
|---|---|
| `internal/ui/app.go` | Add filter mode + help overlay state; empty-state branch in `View`; wire poll interval |
| `internal/ui/list.go` | Filter predicate accepts a free-text prefix; empty groups collapse when filter active |
| `internal/ui/list_test.go` | Cover prefix filter, collapsing empty groups, restoring on clear |
| `internal/ui/keys.go` | Add `/`, `?` mappings; export keybinding table for help overlay |
| `internal/ui/dialog.go` | Filter input prompt reuses dialog input handling |
| `internal/ui/app_test.go` | Cover filter flow, help overlay toggle, empty-state rendering |
| `internal/state/poller.go` | `Poller.Interval` configurable via constructor option |
| `internal/state/poller_test.go` | Cover non-default interval |
| `cmd/root.go` | Add `--poll` and `--headless` flags; headless run loop |
| `cmd/root_test.go` (new) | Cover flag parsing and headless lifecycle |
| `README.md` | Document `/`, `?`, `--poll`, `--headless` |

---

### Task 1: Session list filter (`/`)

**Files:**
- Modify: `internal/ui/keys.go`
- Modify: `internal/ui/app.go`
- Modify: `internal/ui/dialog.go`
- Modify: `internal/ui/list.go`
- Modify: `internal/ui/list_test.go`
- Modify: `internal/ui/app_test.go`

- [ ] Failing tests: `/` opens a filter input; typed prefix narrows visible session titles; groups with zero matches collapse; `esc` clears filter; tag-prefix `#tag` filter still works in the same input.
- [ ] Implement filter state on the model; reuse `BuildTree` filter pipeline from M5.
- [ ] Persist filter across reloads until cleared.
- [ ] Commit: `feat: add session list filter`

### Task 2: Help overlay (`?`)

**Files:**
- Modify: `internal/ui/keys.go`
- Modify: `internal/ui/app.go`
- Modify: `internal/ui/app_test.go`

- [ ] Failing tests: `?` opens a full-screen overlay listing all keybindings grouped by section; `q` or `?` dismisses; underlying state preserved.
- [ ] Source the binding list from a single exported table in `keys.go` so future bindings auto-appear.
- [ ] Commit: `feat: add keyboard shortcut help overlay`

### Task 3: Empty-state onboarding

**Files:**
- Modify: `internal/ui/app.go`
- Modify: `internal/ui/app_test.go`

- [ ] Failing test: when the DB has zero sessions, the detail panel renders the quickstart hint "Press n to create your first session".
- [ ] When at least one session exists, the empty state never renders.
- [ ] Commit: `feat: add empty-state onboarding`

### Task 4: Configurable poll interval (`--poll`)

**Files:**
- Modify: `internal/state/poller.go`
- Modify: `internal/state/poller_test.go`
- Modify: `cmd/root.go`

- [ ] Failing test: `Poller` honors a non-default interval; invalid `--poll` value returns a clear error.
- [ ] Wire flag through `cmd/root.go` `launchTUI` and headless path.
- [ ] Commit: `feat: add configurable poll interval`

### Task 5: Headless / daemon mode (`--headless`)

**Files:**
- Modify: `cmd/root.go`
- Add: `cmd/root_test.go`
- Modify: `internal/state/poller.go` (if signal handling needs a hook)

- [ ] Failing tests: `--headless` runs the poller + notifier without entering Bubble Tea; SIGINT/SIGTERM stops the poller cleanly; exit code 0 on graceful shutdown.
- [ ] Reuse `notify.Config` so quiet hours / debounce remain in effect in headless mode.
- [ ] Commit: `feat: add headless mode`

### Task 6: Docs + final verification

**Files:**
- Modify: `README.md`
- Modify: `docs/superpowers/specs/2026-05-10-roadmap.md` (mark M6 complete)
- Modify: `CLAUDE.md` (refresh feature list)

- [ ] Document `/`, `?`, `--poll`, `--headless`.
- [ ] Run `go test ./...` and `go test -tags e2e ./test/e2e/...` clean.
- [ ] Commit: `docs: mark M6 polish & onboarding complete`
