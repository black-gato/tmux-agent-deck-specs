# Dialog Input Cursor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add left/right arrow cursor movement and at-position insert/delete to `dialogState` and `importState` text inputs, matching the cursor support already in `formField`.

**Architecture:** Refactor `insertRune`/`deleteRune` helpers to take `(value *string, cursor *int)` instead of `*formField`, then wire them into `dialogState` and `importState` alongside `moveCursorLeft`/`moveCursorRight`. Update render sites to use the existing `renderWithCursor` helper.

**Tech Stack:** Go, Bubbletea, Lipgloss — all already in use.

---

### Task 1: Refactor `insertRune` and `deleteRune` to take `(value *string, cursor *int)`

**Files:**
- Modify: `internal/ui/form.go:57-84` (helpers), `internal/ui/form.go:345-390` (call sites in `updateForm`)
- Test: `internal/ui/form_test.go` (existing tests must still pass — no new tests needed here)

- [ ] **Step 1: Update helper signatures in `form.go`**

Replace lines 57–84:

```go
func insertRune(value *string, cursor *int, r rune) {
	runes := []rune(*value)
	runes = append(runes[:*cursor], append([]rune{r}, runes[*cursor:]...)...)
	*value = string(runes)
	*cursor++
}

func deleteRune(value *string, cursor *int) {
	if *cursor == 0 {
		return
	}
	runes := []rune(*value)
	runes = append(runes[:*cursor-1], runes[*cursor:]...)
	*value = string(runes)
	*cursor--
}

func moveCursorLeft(value string, cursor int) int {
	if cursor > 0 {
		return cursor - 1
	}
	return cursor
}

func moveCursorRight(value string, cursor int) int {
	if cursor < len([]rune(value)) {
		return cursor + 1
	}
	return cursor
}
```

- [ ] **Step 2: Update `updateForm` call sites to match new signatures**

In `updateForm` (`form.go`), find the four places that call the old signatures and update them. Each `f` is `&m.form.fields[m.form.focusField]`:

```go
// tea.KeyLeft (was: moveCursorLeft(f))
f.cursor = moveCursorLeft(f.value, f.cursor)

// tea.KeyRight (was: moveCursorRight(f))
f.cursor = moveCursorRight(f.value, f.cursor)

// tea.KeyBackspace (was: deleteRune(f))
deleteRune(&f.value, &f.cursor)

// tea.KeySpace insert (was: insertRune(f, ' '))
insertRune(&f.value, &f.cursor, ' ')

// default rune insert (was: insertRune(f, msg.Runes[0]))
insertRune(&f.value, &f.cursor, msg.Runes[0])
```

- [ ] **Step 3: Run existing tests to verify nothing broke**

```bash
go test ./internal/ui/ -run TestTextFieldInsertAndCursor -v
go test ./internal/ui/ -run TestBackspace -v
go test ./internal/ui/ -v
```

Expected: all pass.

- [ ] **Step 4: Commit**

```bash
git add internal/ui/form.go
git commit -m "refactor(ui): decouple insertRune/deleteRune/moveCursor from formField"
```

---

### Task 2: Add cursor to `dialogState`, update input handling and render

**Files:**
- Modify: `internal/ui/dialog.go` (struct + `updateDialog` + `renderDialog`)
- Modify: `internal/ui/app.go` (pre-populated dialog init sites)
- Test: `internal/ui/form_test.go` (add dialog cursor tests — form_test.go covers all ui_test package tests)

- [ ] **Step 1: Write failing tests**

Add to `internal/ui/form_test.go`:

```go
func TestDialogCursorLeftRight(t *testing.T) {
	m, _ := openModel(t)
	// open rename dialog (pre-existing session not needed — just need mode open)
	m = sendKey(m, rune_('n'))
	m = sendKey(m, key(tea.KeyEsc))
	// open filter dialog
	m = sendKey(m, rune_('/'))
	// type "abc", move left twice, insert 'x' → "axbc"
	m = sendKey(m, rune_('a'))
	m = sendKey(m, rune_('b'))
	m = sendKey(m, rune_('c'))
	m = sendKey(m, key(tea.KeyLeft))
	m = sendKey(m, key(tea.KeyLeft))
	m = sendKey(m, rune_('x'))
	if m.DialogValue() != "axbc" {
		t.Errorf("got %q want %q", m.DialogValue(), "axbc")
	}
}

func TestDialogCursorBackspaceAtMiddle(t *testing.T) {
	m, _ := openModel(t)
	m = sendKey(m, rune_('/'))
	// type "abc", move left, backspace → removes 'b', leaves "ac"
	m = sendKey(m, rune_('a'))
	m = sendKey(m, rune_('b'))
	m = sendKey(m, rune_('c'))
	m = sendKey(m, key(tea.KeyLeft))
	m = sendKey(m, key(tea.KeyBackspace))
	if m.DialogValue() != "ac" {
		t.Errorf("got %q want %q", m.DialogValue(), "ac")
	}
}

func TestDialogCursorClampsAtBoundaries(t *testing.T) {
	m, _ := openModel(t)
	m = sendKey(m, rune_('/'))
	m = sendKey(m, rune_('a'))
	// left past start — should not panic or go negative
	m = sendKey(m, key(tea.KeyLeft))
	m = sendKey(m, key(tea.KeyLeft))
	m = sendKey(m, key(tea.KeyLeft))
	// right past end — should not panic or exceed length
	m = sendKey(m, key(tea.KeyRight))
	m = sendKey(m, key(tea.KeyRight))
	m = sendKey(m, key(tea.KeyRight))
	if m.DialogValue() != "a" {
		t.Errorf("value changed unexpectedly: %q", m.DialogValue())
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/ui/ -run "TestDialogCursor" -v
```

Expected: FAIL — `DialogValue()` method does not exist yet (or cursor has no effect).

- [ ] **Step 3: Add `cursor int` to `dialogState` and expose `DialogValue()` accessor**

In `dialog.go`, update `dialogState`:

```go
type dialogState struct {
	prompt      string
	value       string
	cursor      int
	ctrlKeys    []string
	scope       bool
	scopeLabels [2]string
	vimMode     bool
}
```

In `app.go`, add accessor near the other getters (around line 175):

```go
func (m *Model) DialogValue() string { return m.dialog.value }
```

- [ ] **Step 4: Update `updateDialog` for cursor-aware input**

In `dialog.go`, replace the `tea.KeyBackspace` and `default` cases, and add `tea.KeyLeft`/`tea.KeyRight`:

```go
case tea.KeyLeft:
	m.dialog.cursor = moveCursorLeft(m.dialog.value, m.dialog.cursor)
case tea.KeyRight:
	m.dialog.cursor = moveCursorRight(m.dialog.value, m.dialog.cursor)
case tea.KeyBackspace:
	deleteRune(&m.dialog.value, &m.dialog.cursor)
default:
	if len(msg.Runes) > 0 {
		insertRune(&m.dialog.value, &m.dialog.cursor, msg.Runes[0])
	}
```

Note: the `tea.KeyLeft`/`tea.KeyRight` cases must be added inside the outer `switch msg.Type` block, before the `case tea.KeyEsc` (i.e. within `updateDialog`, not inside the send-pane/broadcast guard at the top).

- [ ] **Step 5: Fix pre-populated dialog init sites to set cursor to end**

In `app.go`, three sites set `m.dialog.value` after calling `newDialogState`. Add a cursor assignment after each:

```go
// edit-notes (line ~321):
m.dialog = dialogState{prompt: "", value: m.items[m.cursor].Session.Notes}
m.dialog.cursor = len([]rune(m.dialog.value))

// edit-tags (line ~352):
m.dialog = newDialogState("Tags:")
m.dialog.value = m.items[m.cursor].Session.Tags
m.dialog.cursor = len([]rune(m.dialog.value))

// filter (line ~394):
m.dialog = newDialogState("Filter:")
m.dialog.value = m.searchQuery
m.dialog.cursor = len([]rune(m.dialog.value))
```

- [ ] **Step 6: Update `renderDialog` to use `renderWithCursor`**

In `dialog.go`, replace every occurrence of `m.dialog.value` in `renderDialog` with `renderWithCursor(m.dialog.value, m.dialog.cursor)`:

```go
func (m *Model) renderDialog() string {
	if m.mode == "edit-notes" {
		return "> " + renderWithCursor(m.dialog.value, m.dialog.cursor)
	}
	vimTag := ""
	if m.dialog.vimMode {
		vimTag = "  [vim]"
	}
	if m.mode == "broadcast" {
		label0 := m.dialog.scopeLabels[0]
		label1 := m.dialog.scopeLabels[1]
		if !m.dialog.scope {
			label0 = "→ " + label0
			label1 = dimStyle.Render(label1)
		} else {
			label0 = dimStyle.Render(label0)
			label1 = "→ " + label1
		}
		return fmt.Sprintf("Broadcast [%s / %s]%s:\n> %s", label0, label1, vimTag, renderWithCursor(m.dialog.value, m.dialog.cursor))
	}
	return m.dialog.prompt + vimTag + "\n> " + renderWithCursor(m.dialog.value, m.dialog.cursor)
}
```

- [ ] **Step 7: Run tests**

```bash
go test ./internal/ui/ -run "TestDialogCursor" -v
go test ./internal/ui/ -v
```

Expected: all pass.

- [ ] **Step 8: Commit**

```bash
git add internal/ui/dialog.go internal/ui/app.go internal/ui/form_test.go
git commit -m "feat(ui): add cursor to dialogState text inputs"
```

---

### Task 3: Add cursors to `importState`, update input handling and render

**Files:**
- Modify: `internal/ui/import.go` (struct + `updateImportPicker` + `updateImportForm` + `renderImportForm`)
- Test: `internal/ui/import_test.go` (add import cursor tests)

- [ ] **Step 1: Write failing tests**

Add to `internal/ui/import_test.go`:

```go
func TestImportFormCursorLeftRight(t *testing.T) {
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["mysess"] = ""
	fake.SessionInfos["mysess"] = tmux.SessionInfo{Name: "mysess", CurrentPath: "/tmp"}

	m, _ := openModelWithTmux(t, fake)
	m = sendKey(m, rune_('I'))
	m = sendKey(m, key(tea.KeyEnter)) // enter form; title pre-filled as "mysess"

	// title field is focused (focus=0); move left twice, insert 'X' → "myXsess"[0:2]+'X'+"sess"
	// "mysess" has 6 runes; left×2 = cursor at 4, insert 'X' → "mysesXs"... wait:
	// "mysess": m=0,y=1,s=2,e=3,s=4,s=5 → cursor starts at 6 (end)
	// left×2 → cursor=4; insert 'X' → "myseXss"
	m = sendKey(m, key(tea.KeyLeft))
	m = sendKey(m, key(tea.KeyLeft))
	m = sendKey(m, rune_('X'))
	if m.ImportFormTitle() != "myseXss" {
		t.Errorf("got %q want %q", m.ImportFormTitle(), "myseXss")
	}
}

func TestImportFormCursorBackspaceAtMiddle(t *testing.T) {
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["abc"] = ""
	fake.SessionInfos["abc"] = tmux.SessionInfo{Name: "abc", CurrentPath: "/tmp"}

	m, _ := openModelWithTmux(t, fake)
	m = sendKey(m, rune_('I'))
	m = sendKey(m, key(tea.KeyEnter)) // title = "abc", cursor at end (3)

	// left once → cursor=2; backspace removes 'b' → "ac"
	m = sendKey(m, key(tea.KeyLeft))
	m = sendKey(m, key(tea.KeyBackspace))
	if m.ImportFormTitle() != "ac" {
		t.Errorf("got %q want %q", m.ImportFormTitle(), "ac")
	}
}

func TestImportFormCursorClampsAtBoundaries(t *testing.T) {
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["z"] = ""
	fake.SessionInfos["z"] = tmux.SessionInfo{Name: "z", CurrentPath: "/tmp"}

	m, _ := openModelWithTmux(t, fake)
	m = sendKey(m, rune_('I'))
	m = sendKey(m, key(tea.KeyEnter)) // title = "z"

	// left past start, right past end — no panic, value unchanged
	m = sendKey(m, key(tea.KeyLeft))
	m = sendKey(m, key(tea.KeyLeft))
	m = sendKey(m, key(tea.KeyRight))
	m = sendKey(m, key(tea.KeyRight))
	m = sendKey(m, key(tea.KeyRight))
	if m.ImportFormTitle() != "z" {
		t.Errorf("value changed unexpectedly: %q", m.ImportFormTitle())
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/ui/ -run "TestImportFormCursor" -v
```

Expected: FAIL — cursor has no effect on import form.

- [ ] **Step 3: Add `titleCursor`/`groupCursor` to `importState`**

In `import.go`, update `importState`:

```go
type importState struct {
	candidates  []importCandidate
	cursor      int
	selected    string
	title       string
	titleCursor int
	group       string
	groupCursor int
	formErr     string
	focus       int // 0 = title, 1 = group
}
```

- [ ] **Step 4: Initialize cursors to end of pre-populated values in `updateImportPicker`**

In `updateImportPicker`, update the `tea.KeyEnter` case where `import-form` mode is entered:

```go
case tea.KeyEnter:
	if len(m.imp.candidates) == 0 {
		m.mode = ""
		m.imp = importState{}
		return m, nil
	}
	c := m.imp.candidates[m.imp.cursor]
	m.imp.selected = c.Name
	m.imp.title = c.Name
	m.imp.titleCursor = len([]rune(c.Name))
	m.imp.group = m.currentGroupPath()
	if m.imp.group == "" {
		m.imp.group = defaultGroupPath
	}
	m.imp.groupCursor = len([]rune(m.imp.group))
	m.imp.formErr = ""
	m.imp.focus = 0
	m.mode = "import-form"
```

- [ ] **Step 5: Update `updateImportForm` for cursor-aware input**

Replace the `tea.KeyBackspace`, `tea.KeySpace`, and `default` cases, and add `tea.KeyLeft`/`tea.KeyRight`:

```go
func (m *Model) updateImportForm(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	if msg.Type != tea.KeyEnter {
		m.imp.formErr = ""
	}
	switch msg.Type {
	case tea.KeyEsc:
		m.mode = ""
		m.imp = importState{}
	case tea.KeyTab:
		m.imp.focus = 1 - m.imp.focus
	case tea.KeyLeft:
		if m.imp.focus == 0 {
			m.imp.titleCursor = moveCursorLeft(m.imp.title, m.imp.titleCursor)
		} else {
			m.imp.groupCursor = moveCursorLeft(m.imp.group, m.imp.groupCursor)
		}
	case tea.KeyRight:
		if m.imp.focus == 0 {
			m.imp.titleCursor = moveCursorRight(m.imp.title, m.imp.titleCursor)
		} else {
			m.imp.groupCursor = moveCursorRight(m.imp.group, m.imp.groupCursor)
		}
	case tea.KeyBackspace:
		if m.imp.focus == 0 {
			deleteRune(&m.imp.title, &m.imp.titleCursor)
		} else {
			deleteRune(&m.imp.group, &m.imp.groupCursor)
		}
	case tea.KeySpace:
		if m.imp.focus == 0 {
			insertRune(&m.imp.title, &m.imp.titleCursor, ' ')
		} else {
			insertRune(&m.imp.group, &m.imp.groupCursor, ' ')
		}
	case tea.KeyEnter:
		if _, err := db.ImportSession(m.conn, m.tmuxC, db.ImportRequest{
			TmuxName:  m.imp.selected,
			Title:     m.imp.title,
			GroupPath: m.imp.group,
		}); err != nil {
			m.imp.formErr = err.Error()
			return m, nil
		}
		m.mode = ""
		m.imp = importState{}
		if err := m.Reload(); err != nil {
			m.err = err
		}
	default:
		if len(msg.Runes) > 0 {
			if m.imp.focus == 0 {
				insertRune(&m.imp.title, &m.imp.titleCursor, msg.Runes[0])
			} else {
				insertRune(&m.imp.group, &m.imp.groupCursor, msg.Runes[0])
			}
		}
	}
	return m, nil
}
```

- [ ] **Step 6: Update `renderImportForm` to use `renderWithCursor`**

Replace the two `formCursorStyle.Render(" ")` blocks:

```go
func (m *Model) renderImportForm() string {
	var b strings.Builder
	b.WriteString(formHeaderStyle.Render("IMPORT: " + m.imp.selected))
	b.WriteString("\n\n")

	titleLine := "  TITLE  " + m.imp.title
	groupLine := "  GROUP  " + m.imp.group
	if m.imp.focus == 0 {
		titleLine = formLabelActive.Render("> TITLE  ") + renderWithCursor(m.imp.title, m.imp.titleCursor)
	}
	if m.imp.focus == 1 {
		groupLine = formLabelActive.Render("> GROUP  ") + renderWithCursor(m.imp.group, m.imp.groupCursor)
	}
	b.WriteString(titleLine + "\n")
	b.WriteString(groupLine + "\n\n")
	b.WriteString(formHintStyle.Render("tab: next   enter: import   esc: cancel"))
	if m.imp.formErr != "" {
		b.WriteString("\n")
		b.WriteString(formErrStyle.Render(m.imp.formErr))
	}
	return b.String()
}
```

- [ ] **Step 7: Run tests**

```bash
go test ./internal/ui/ -run "TestImportFormCursor" -v
go test ./internal/ui/ -v
```

Expected: all pass.

- [ ] **Step 8: Commit**

```bash
git add internal/ui/import.go internal/ui/import_test.go
git commit -m "feat(ui): add cursor to importState text inputs"
```

---

### Task 4: Final verification and push

- [ ] **Step 1: Run full test suite**

```bash
go test ./... -count=1
```

Expected: all packages pass.

- [ ] **Step 2: Vet**

```bash
go vet ./...
```

Expected: no output.

- [ ] **Step 3: Push**

```bash
git push
```
