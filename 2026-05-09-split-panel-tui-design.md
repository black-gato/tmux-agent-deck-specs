# Split Panel TUI Design

**Date:** 2026-05-09
**Status:** Implemented

## Overview

Redesign the TUI from a single-column session list to a persistent split layout: session list on the left (~35% width) and a session detail panel on the right (~65% width). The detail panel shows the selected session's name, group, pane programs, live output, and editable notes. A key toggle expands the output to full screen.

---

## Layout

```
 Agent Deck  ● 2 running  ○ 3 waiting  ◐ 1 idle
─────────────────┬──────────────────────────────────────
 SESSIONS        │  SESSION ──────────────────────────────
 ──────────────  │   update-design-spec-divergences  ○ waiting
 ▼ My Sessions   │   group: My Sessions
   ○ mama        │   [0] claude  [1] bash  [2] nvim
   ● api-work    │
   ○ update-desi │  OUTPUT ───────────────────────────────
   ◐ notes       │   Analyzing changes to client.go...
                 │   Running tests...
 ▼ Research      │   ✓ 12 tests pass
   ◐ side-proj   │   >
                 │
                 │  NOTES ────────────────────────────────
                 │   Implement sub-agent approach for spec
                 │   review. Check divergences doc first.
                 │   e edit
 Enter Attach  v Output  e Notes  x Send  n New  q Quit
```

### Column widths

Left column width is fixed at 35% of terminal width (floored to nearest character). Right column fills the remainder. Both columns receive `tea.WindowSizeMsg` updates.

### Full-screen output (`v`)

When `m.viewFull` is true, the left column is hidden and the right panel fills the full terminal width. Pressing `v` again sets `m.viewFull = false`. All other keys still work in full-screen mode (attach, notes, quit, etc.).

---

## Detail Panel Sections

### Session header

Three lines:
1. Session title + status indicator
2. `group: <group_path>`
3. Pane list: `[0] <command>  [1] <command>  ...` — active pane (index 0 of `list-panes` output) shown in a distinct style; inactive panes dimmed

### Output

Fills all vertical space between the session header and the notes section. Height = terminal height − header rows − notes rows − footer row. The existing `CapturePaneOutput` result (polled every ~1s by the poller) is split on newlines and the last N lines are shown where N equals the available output height. No scrolling — always shows the tail.

### Notes

Fixed at 4 lines: 3 lines of note text (truncated/wrapped to panel width) + 1 hint line (`e edit`). When no notes exist, shows `No notes` as a placeholder.

---

## Notes Editing

Pressing `e` sets `mode = "edit-notes"`. In this mode:

- The notes section renders an inline text input using the existing `dialogState.input` field
- The input is pre-populated with the session's current notes
- Characters are appended/deleted using the same key handling as existing dialog modes
- `Enter` saves the notes to the DB and clears the mode
- `Esc` cancels without saving

The edit input is rendered in-place within the notes section of the detail panel — not as a popup overlay.

---

## Pane List

A new method on `Client` fetches pane info for the selected session each tick:

```go
type Pane struct {
    Index   int
    Command string
}

func (c *Client) ListPanes(session string) ([]Pane, error)
```

Calls `tmux list-panes -t <session> -F "#{pane_index} #{pane_current_command}"`. Returns an empty slice (no error) when the session has no panes or does not exist. `ListPanes` is added to `ClientIface`.

Pane info is fetched in `Reload()` for the currently selected session only. Result stored on `Model` as `[]tmux.Pane`.

---

## DB Change

New column on `sessions`:

```sql
ALTER TABLE sessions ADD COLUMN notes TEXT NOT NULL DEFAULT '';
```

Applied as schema version 2 in `migrate()`. The `Session` struct gains a `Notes string` field. `UpdateSessionNotes(conn, id, notes string) error` is added to `internal/db/sessions.go`.

---

## Keybindings

| Key | Action |
|-----|--------|
| `v` | Toggle full-screen output view |
| `e` | Edit notes for selected session (when a session is selected) |
| All existing keys | Unchanged |

`e` has no effect when a group row is selected (notes are per-session only).

---

## Files Changed

| File | Change |
|------|--------|
| `internal/db/sessions.go` | Add `Notes` field, `UpdateSessionNotes()`, schema v2 migration |
| `internal/tmux/client.go` | Add `Pane` type and `ListPanes()` method |
| `internal/tmux/client.go` (ClientIface) | Add `ListPanes` to interface |
| `internal/testutil/tmux.go` | Add stub `ListPanes` to fake client |
| `internal/ui/app.go` | Add `viewFull bool`, `panes []tmux.Pane`, detail panel rendering, `edit-notes` mode |
| `internal/ui/list.go` | Accept width param so left column renders at correct narrowed width |

---

## Out of Scope

- Scrolling through output history (output is always the live tail)
- Per-group notes
- Notes on the detail panel for groups (groups are not selectable in the detail panel)
- Pane switching (clicking a pane to view its output — always shows pane 0)
