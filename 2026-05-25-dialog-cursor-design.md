# Dialog Input Cursor

**Date:** 2026-05-25

## Problem

The new-session form (`form.go`) already has full cursor support: left/right arrow movement, insert-at-position, delete-at-position, and a reverse-video cursor block via `renderWithCursor`. The two other input surfaces — `dialogState` (single-line prompts: rename, move, new-group, notes, tags, filter, send, fork) and `importState` (import form title/group fields) — always append and delete from the end. Users cannot correct a typo in the middle of a field without deleting back to it.

## Design

### 1. Refactor insert/delete helpers

`insertRune` and `deleteRune` currently take `*formField`. Change their signatures to take `(value *string, cursor *int)` so they can be called from any input site. Update the two `form.go` call sites (`updateForm`) to pass `&f.value, &f.cursor` instead of `f`.

### 2. `dialogState` cursor

Add a `cursor int` field to `dialogState`.

In `updateDialog`:
- `tea.KeyLeft` → decrement cursor (clamp to 0)
- `tea.KeyRight` → increment cursor (clamp to `len([]rune(value))`)
- `tea.KeyBackspace` → call `deleteRune(&m.dialog.value, &m.dialog.cursor)`
- rune input → call `insertRune(&m.dialog.value, &m.dialog.cursor, r)`

When `m.dialog` is initialized (all `newDialogState(...)` call sites and the direct `dialogState{...}` literals), set `cursor` to `len([]rune(value))` so the cursor starts at the end — matching current append behavior for pre-populated fields (notes, tags, filter).

In `renderDialog`, replace bare `m.dialog.value` with `renderWithCursor(m.dialog.value, m.dialog.cursor)` at every render site (the `"> " + m.dialog.value` lines in all branches).

### 3. `importState` cursor

Add `titleCursor int` and `groupCursor int` to `importState`.

In `updateImportForm`, dispatch left/right/backspace/insert to the focused field's value+cursor pair using the same refactored helpers.

When `import-form` mode is entered (in `updateImportPicker` on `tea.KeyEnter`), set both cursors to the end of their pre-populated values.

In `renderImportForm`, replace the current end-appended `formCursorStyle.Render(" ")` blocks with `renderWithCursor(value, cursor)` for the active field.

## Scope

- No new keys beyond left/right arrow
- No Home/End support
- No change to commit logic, field navigation (Tab/Up/Down), or any other behavior
- `renderWithCursor` is already independent (`(string, int) → string`) — no changes needed there

## Files Changed

| File | Change |
|------|--------|
| `internal/ui/form.go` | Refactor `insertRune`/`deleteRune` signatures; add `cursor` init in `initSessionForm` (already set); update two call sites |
| `internal/ui/dialog.go` | Add `cursor int` to `dialogState`; update `updateDialog` and `renderDialog` |
| `internal/ui/import.go` | Add `titleCursor`/`groupCursor` to `importState`; update `updateImportForm` and `renderImportForm` |

## Testing

Add/update tests covering:
- Insert at middle of field moves cursor and places character correctly
- Backspace at middle removes character to the left of cursor
- Left/right clamp at boundaries (no underflow/overflow)
- `renderWithCursor` at end-of-string renders trailing cursor block (already tested in existing suite)
- Dialog cursor initializes at end for pre-populated fields (notes, tags, filter re-open)
