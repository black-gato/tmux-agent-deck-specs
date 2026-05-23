# Session Creation Form Design

**Date:** 2026-05-17
**Branch:** feature/session-picker
**Status:** Approved

## Summary

Replace the current multi-step wizard for creating a new session with a single bottom-panel form that shows all fields at once. Add a visible cursor in text fields and tab-cycling through path completions.

## Motivation

The existing flow requires the user to press Enter through five sequential prompts (title → path → tool → flags → script), with no way to see or go back to earlier fields without cancelling. Users also have no visual cursor indicator, making it hard to know where text will be inserted. Tab completion exists but doesn't cycle through candidates — it only expands the common prefix and shows a list.

## Design

### Layout

A bottom panel appended below the session list, separated by a horizontal rule. All five fields are visible simultaneously. The active field is highlighted with a purple left border and a slightly elevated background. Inactive fields are dimmed.

```
my-sessions
  ● worker-1   [running]
  ● worker-2   [waiting]
──────────────────────────────────
NEW SESSION
  TITLE  my-agent█
  PATH   ~/Projects/
  TOOL   claude  aider  bash
  FLAGS  ─
  SCRIPT ─
Tab · Space · Enter to create · Esc cancel
```

When the path field is cycling completions, candidates are shown as a dim row above the form and the currently-selected candidate is highlighted in the field value:

```
foo/  bar/  baz/  qux/
  PATH   ~/Projects/bar/█
  (Tab to cycle · Space to select)
```

### Fields

| # | Label | Kind | Default |
|---|-------|------|---------|
| 0 | Title | text | `""` |
| 1 | Path | text + tab-completion | cwd or group default |
| 2 | Tool | selector | group default or `claude` |
| 3 | Flags | text (optional) | `""` |
| 4 | Script | text (optional) | `""` |

### Cursor

Text fields render a block cursor at the current rune index using lipgloss reverse styling. The cursor is always visible (no blinking — terminal TUI constraint). Insertion and deletion operate at cursor position; left/right arrow keys move the cursor.

### Key Bindings

| Key | Behavior |
|-----|----------|
| Tab | **Path field:** cycle through completions one by one; when candidates exhausted, advance focus to next field. **All other fields:** advance focus to next field. |
| Down / Up | Move focus to the next / previous field. Clamped at first and last field. |
| Space | **Tool field:** advance focus. **Path field mid-cycle:** select current candidate and advance focus. **Text fields:** insert a space character. |
| Enter | Submit the form from any field. |
| Left / Right | **Text fields:** move cursor left/right. **Tool field:** cycle options left/right. |
| Backspace | Delete character before cursor. Clears completion cycle state on path field. |
| Any rune | Insert at cursor position (text fields only). Clears completion cycle state on path field. |
| Esc / Ctrl+C | Cancel, return to normal mode. |

### Path Completion Cycling

Each Tab press on the path field calls `completePath()` to get the candidate list, then advances `candIdx` to select the next candidate. The selected candidate is written into the field value. On the next Tab press, `candIdx` increments (wrapping). Typing any character or Backspace resets the cycle (`candActive = false`, `candidates = nil`, `candIdx = 0`).

When `candIdx` reaches the end of the list, the next Tab press advances focus to the Tool field (field index 2).

## Data Model

New file `internal/ui/form.go`:

```go
type fieldKind int

const (
    fieldText   fieldKind = iota
    fieldSelect
)

type formField struct {
    label   string
    kind    fieldKind
    value   string   // text fields
    cursor  int      // rune index into value
    options []string // select fields
    optIdx  int      // current selection index
}

type formState struct {
    fields     []formField
    focusField int
    // path completion cycling (always field index 1)
    candidates []string
    candIdx    int
    candActive bool
}
```

`Model` gains a `form formState` field. When `m.mode == "new-session"`, `Update` routes key messages to `updateForm()` and `View` calls `renderForm()`.

## Affected Files

| File | Change |
|------|--------|
| `internal/ui/form.go` | **New.** `formField`, `formState`, `initSessionForm()`, `updateForm()`, `renderForm()`. |
| `internal/ui/app.go` | `"new-session"` action initialises `m.form` via `initSessionForm()` and sets `m.mode = "new-session"`. `Update` and `View` route to form functions. |
| `internal/ui/dialog.go` | Remove `advanceNewSessionStep()` and the `new-session` branches in `updateDialog()` and `renderDialog()`. Helpers (`completePath`, `expandPath`, `longestCommonPrefix`, `splitPathInput`) stay in place and are called from `form.go`. |
| `internal/ui/form_test.go` | **New.** Unit tests for `updateForm`, path cycling, cursor movement, Space/Tab routing. |

## Out of Scope

- Changing any other dialog mode (rename, move, broadcast, fork, send-pane, etc.)
- Adding new fields to the session form
- Mouse support
