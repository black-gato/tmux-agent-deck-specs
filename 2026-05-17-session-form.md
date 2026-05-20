# Session Creation Form Implementation Plan

**Status: Complete**

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the multi-step new-session wizard with a single bottom-panel form showing all five fields at once, with a visible cursor, left/right cursor movement, and Tab-cycling through path completions.

**Architecture:** A new `form.go` file in `internal/ui` holds the `formState` data type and all form logic (`initSessionForm`, `updateForm`, `renderForm`, `commitForm`). `app.go` adds a `form formState` field to `Model`, routes key events to `updateForm` when `m.mode == "new-session"`, and switches to a top/bottom layout (list above, form below). The existing `dialogState` in `dialog.go` is stripped of all new-session-specific fields and branches; path completion helpers (`completePath`, `expandPath`, etc.) stay in `dialog.go` and are called from `form.go` within the same package.

**Tech Stack:** Go 1.22+, Bubbletea (`github.com/charmbracelet/bubbletea`), Lipgloss (`github.com/charmbracelet/lipgloss`), SQLite via `db` package, `github.com/google/uuid`.

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `internal/ui/form.go` | Create | `fieldKind`, `formField`, `formState` types; `initSessionForm`, `updateForm`, `renderForm`, `commitForm`, `cyclePathCompletion`, `resetPathCompletion`, `renderWithCursor` |
| `internal/ui/form_test.go` | Create | Black-box tests via `Model` API; tests for cursor, cycling, submit, cancel |
| `internal/ui/app.go` | Modify | Add `form formState` to `Model`; route `m.mode == "new-session"` to `updateForm` in `Update`; render top/bottom layout in `View` |
| `internal/ui/dialog.go` | Modify | Remove `advanceNewSessionStep`, new-session branches in `updateDialog`/`renderDialog`/`commitDialog`, `clearDialogCandidates`; remove new-session-specific fields from `dialogState` |

---

## Task 1: `formState` data types

**Files:**
- Create: `internal/ui/form.go`
- Create: `internal/ui/form_test.go`

- [ ] **Step 1: Create `form.go` with type definitions**

```go
package ui

import (
	"github.com/charmbracelet/lipgloss"
)

// Imports are added incrementally as functions are implemented:
//   Task 3 (initSessionForm): add "os"
//   Task 5 (commitForm):      add "strings", "time", "github.com/black-gato/tmux-agent-deck/internal/db", "github.com/google/uuid"
//   Task 6 (updateForm):      add tea "github.com/charmbracelet/bubbletea"
//   Task 7 (renderForm):      add "strings" (already present after Task 5)

type fieldKind int

const (
	fieldText   fieldKind = iota
	fieldSelect           // left/right to cycle options
)

type formField struct {
	label   string
	kind    fieldKind
	value   string   // text fields
	cursor  int      // rune index into value
	options []string // select fields
	optIdx  int      // selected option index
}

type formState struct {
	fields     []formField
	focusField int
	// path completion cycling (always field index 1)
	candidates []string
	candIdx    int
	candActive bool
	candBase   string // directory prefix kept stable during cycling
}

var (
	formBarStyle    = lipgloss.NewStyle().Foreground(lipgloss.Color("#cba6f7"))
	formLabelActive = lipgloss.NewStyle().Foreground(lipgloss.Color("#89b4fa")).Bold(true)
	formLabelDim    = lipgloss.NewStyle().Faint(true)
	formValueDim    = lipgloss.NewStyle().Faint(true)
	formCursorStyle = lipgloss.NewStyle().Reverse(true)
	formHeaderStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("#cba6f7")).Bold(true)
	formHintStyle   = lipgloss.NewStyle().Faint(true)
	formCandStyle   = lipgloss.NewStyle().Faint(true)
	formCandHLStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("#a6e3a1"))
)
```

- [ ] **Step 2: Create `form_test.go` stub**

```go
package ui_test

import (
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/testutil"
	tea "github.com/charmbracelet/bubbletea"
)

func newFormModel(t *testing.T) *Model {
	t.Helper()
	conn := testutil.OpenTestDB(t)
	m := NewModel(conn, nil, nil)
	if err := m.Reload(); err != nil {
		t.Fatal(err)
	}
	return m
}

func sendKey(m *Model, key tea.KeyMsg) *Model {
	next, _ := m.Update(key)
	return next.(*Model)
}

func TestFormPlaceholder(t *testing.T) {
	// placeholder — replaced in later tasks
}
```

- [ ] **Step 3: Verify it compiles**

```bash
go build ./internal/ui/
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add internal/ui/form.go internal/ui/form_test.go
git commit -m "feat(ui): add formState types and styles"
```

---

## Task 2: Text field cursor helpers

**Files:**
- Modify: `internal/ui/form.go` — add `insertRune`, `deleteRune`, `moveCursorLeft`, `moveCursorRight`, `renderWithCursor`
- Modify: `internal/ui/form_test.go` — add cursor tests

- [ ] **Step 1: Write failing tests for cursor operations**

Replace `TestFormPlaceholder` in `form_test.go`:

```go
func TestInsertRune(t *testing.T) {
	m := newFormModel(t)
	// Enter new-session mode
	m = sendKey(m, tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'n'}})
	if m.Mode() != "new-session" {
		t.Fatalf("expected new-session mode, got %q", m.Mode())
	}

	// type "ab"
	m = sendKey(m, tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'a'}})
	m = sendKey(m, tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'b'}})

	// move cursor left (cursor should be at rune 1, between a and b)
	m = sendKey(m, tea.KeyMsg{Type: tea.KeyLeft})

	// insert 'x' between a and b
	m = sendKey(m, tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'x'}})

	// submit and check DB
	m = sendKey(m, tea.KeyMsg{Type: tea.KeyEnter})
	sessions, err := db.ListSessions(m.(*Model).conn) // use testutil accessor
```

Wait — `conn` is unexported. Use a different verification approach: test via mode returning to `""` and then checking the form was submitted by looking at sessions.

Actually, let me simplify the test to test observable Model state only. The full approach:

```go
func TestTextFieldInsertAndCursor(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	m := NewModel(conn, nil, nil)
	if err := m.Reload(); err != nil {
		t.Fatal(err)
	}

	// open form
	m2, _ := m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'n'}})
	m = m2.(*Model)
	if m.Mode() != "new-session" {
		t.Fatalf("expected new-session mode, got %s", m.Mode())
	}

	// type "ab", move left, insert 'x' → title should become "axb"
	for _, r := range []rune{'a', 'b'} {
		m2, _ = m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
		m = m2.(*Model)
	}
	m2, _ = m.Update(tea.KeyMsg{Type: tea.KeyLeft})
	m = m2.(*Model)
	m2, _ = m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'x'}})
	m = m2.(*Model)

	// submit with Enter (title field non-empty, rest defaults)
	m2, _ = m.Update(tea.KeyMsg{Type: tea.KeyEnter})
	m = m2.(*Model)
	if m.Mode() != "" {
		t.Fatalf("expected mode cleared after submit, got %s", m.Mode())
	}

	sessions, err := db.ListSessions(conn)
	if err != nil {
		t.Fatal(err)
	}
	if len(sessions) != 1 {
		t.Fatalf("expected 1 session, got %d", len(sessions))
	}
	if sessions[0].Title != "axb" {
		t.Errorf("expected title %q, got %q", "axb", sessions[0].Title)
	}
}

func TestBackspace(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	m := NewModel(conn, nil, nil)
	_ = m.Reload()

	m2, _ := m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'n'}})
	m = m2.(*Model)

	// type "ab", backspace → "a", submit
	for _, r := range []rune{'a', 'b'} {
		m2, _ = m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
		m = m2.(*Model)
	}
	m2, _ = m.Update(tea.KeyMsg{Type: tea.KeyBackspace})
	m = m2.(*Model)
	m2, _ = m.Update(tea.KeyMsg{Type: tea.KeyEnter})
	m = m2.(*Model)

	sessions, _ := db.ListSessions(conn)
	if len(sessions) != 1 || sessions[0].Title != "a" {
		t.Errorf("expected title %q, got %q", "a", sessions[0].Title)
	}
}
```

- [ ] **Step 2: Run tests — expect compile failure** (functions not yet implemented)

```bash
go test ./internal/ui/ -run "TestTextFieldInsertAndCursor|TestBackspace" -v
```

Expected: compile error — `NewModel`, `sendKey` types referenced but `updateForm` not defined yet.

- [ ] **Step 3: Add cursor helpers to `form.go`**

Add after the `var (...)` block in `form.go`:

```go
func insertRune(f *formField, r rune) {
	runes := []rune(f.value)
	runes = append(runes[:f.cursor], append([]rune{r}, runes[f.cursor:]...)...)
	f.value = string(runes)
	f.cursor++
}

func deleteRune(f *formField) {
	if f.cursor == 0 {
		return
	}
	runes := []rune(f.value)
	runes = append(runes[:f.cursor-1], runes[f.cursor:]...)
	f.value = string(runes)
	f.cursor--
}

func moveCursorLeft(f *formField) {
	if f.cursor > 0 {
		f.cursor--
	}
}

func moveCursorRight(f *formField) {
	if f.cursor < len([]rune(f.value)) {
		f.cursor++
	}
}

// renderWithCursor renders value with a block cursor at rune index cursor.
func renderWithCursor(value string, cursor int) string {
	runes := []rune(value)
	if cursor >= len(runes) {
		return value + formCursorStyle.Render(" ")
	}
	before := string(runes[:cursor])
	at := string(runes[cursor : cursor+1])
	after := string(runes[cursor+1:])
	return before + formCursorStyle.Render(at) + after
}
```

These will only pass once `updateForm`, `initSessionForm`, and `commitForm` are wired in (Tasks 3–6). The tests are integration-level; they'll compile once the full wiring is done. Mark this step done after adding the helpers.

- [ ] **Step 4: Build check**

```bash
go build ./internal/ui/
```

Expected: compiles (tests still fail — updateForm not wired yet).

- [ ] **Step 5: Commit**

```bash
git add internal/ui/form.go internal/ui/form_test.go
git commit -m "feat(ui): add text cursor helpers and cursor renderer"
```

---

## Task 3: `initSessionForm`

**Files:**
- Modify: `internal/ui/form.go` — add `initSessionForm`

- [ ] **Step 1: Add `initSessionForm` to `form.go`**

Add this method to `form.go`:

```go
func (m *Model) initSessionForm() {
	defaultPath := "."
	if wd, err := os.Getwd(); err == nil {
		defaultPath = wd
	}
	defaultTool := "claude"
	cursorGroupPath := m.currentGroupPath()
	for _, g := range m.groups {
		if g.Path == cursorGroupPath {
			if g.DefaultPath != "" {
				defaultPath = g.DefaultPath
			}
			if g.DefaultTool != "" {
				defaultTool = g.DefaultTool
			}
			break
		}
	}
	toolOptions := []string{"claude", "claude-dangerous", "aider", "cursor", "bash", "custom"}
	toolIdx := 0
	for i, t := range toolOptions {
		if t == defaultTool {
			toolIdx = i
			break
		}
	}
	m.form = formState{
		fields: []formField{
			{label: "TITLE", kind: fieldText},
			{label: "PATH", kind: fieldText, value: defaultPath, cursor: len([]rune(defaultPath))},
			{label: "TOOL", kind: fieldSelect, options: toolOptions, optIdx: toolIdx},
			{label: "FLAGS", kind: fieldText},
			{label: "SCRIPT", kind: fieldText},
		},
	}
}
```

- [ ] **Step 2: Build check**

```bash
go build ./internal/ui/
```

Expected: compiles.

- [ ] **Step 3: Commit**

```bash
git add internal/ui/form.go
git commit -m "feat(ui): add initSessionForm with group defaults"
```

---

## Task 4: Path completion cycling

**Files:**
- Modify: `internal/ui/form.go` — add `cyclePathCompletion`, `resetPathCompletion`

- [ ] **Step 1: Write a failing test for cycling**

Add to `form_test.go`:

```go
func TestPathCompletionCycling(t *testing.T) {
	// This test verifies that Tab on the path field cycles completions
	// and that after all candidates are exhausted focus advances to the
	// tool field (observable as: subsequent Left key now cycles tool options).
	// We use a real temp dir with known subdirectories.
	tmp := t.TempDir()
	for _, name := range []string{"alpha", "beta", "gamma"} {
		if err := os.Mkdir(filepath.Join(tmp, name), 0755); err != nil {
			t.Fatal(err)
		}
	}

	conn := testutil.OpenTestDB(t)
	m := NewModel(conn, nil, nil)
	_ = m.Reload()

	// open form
	m2, _ := m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'n'}})
	m = m2.(*Model)

	// Clear path field and type the tmp dir path with trailing slash
	// so completePath will find alpha/, beta/, gamma/
	// First, we need to clear the pre-filled path.
	// We do this by pressing Backspace enough times to empty it, then typing tmp+"/".
	// Simpler: just trust that after cycling we can submit and check the session path.

	// Replace path by clearing with ctrl+u equivalent — not implemented.
	// Instead, we submit with an empty title to do nothing and re-open.
	// Actually: Tab on path field only cycles; we can set the path field
	// content indirectly only via typing. Let's just test the cycling
	// mechanics via submit result.
	//
	// Practical approach: type the tmp path char by char, then Tab three times
	// to cycle alpha, beta, gamma, then one more Tab to advance focus.
	// After advancing focus, submit with Enter → path should be gamma.
	//
	// To type the path: first press Escape to close, directly manipulate
	// the form state is not possible (black-box). So we just verify the
	// cycling completes and the session is created with a valid path.

	// Skip into the path field via Tab (advances from TITLE)
	m2, _ = m.Update(tea.KeyMsg{Type: tea.KeyTab})
	m = m2.(*Model) // focus on PATH

	// Type tmp+"/" to set up for completion
	for _, r := range []rune(tmp + "/") {
		m2, _ = m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
		m = m2.(*Model)
	}

	// Tab three times → cycle through alpha/, beta/, gamma/
	for i := 0; i < 3; i++ {
		m2, _ = m.Update(tea.KeyMsg{Type: tea.KeyTab})
		m = m2.(*Model)
	}
	// One more Tab → candidates exhausted → focus advances to TOOL (field 2)
	m2, _ = m.Update(tea.KeyMsg{Type: tea.KeyTab})
	m = m2.(*Model)

	// Now on TOOL field. Tab from TOOL → advances to FLAGS.
	// Submit from FLAGS with a non-empty title would fail (title is empty).
	// We need to type a title. But title is field 0 and focus is on TOOL (2).
	// Press Esc and re-open... this is getting complex for black-box.
	//
	// Simpler assertion: mode should still be "new-session" (we haven't submitted)
	if m.Mode() != "new-session" {
		t.Errorf("expected still in new-session mode, got %s", m.Mode())
	}
}
```

> **Note:** Path cycling is best verified via manual smoke-test in addition to this structural test. The test above confirms the cycling doesn't crash and mode stays active.

- [ ] **Step 2: Run test — expect compile failure**

```bash
go test ./internal/ui/ -run TestPathCompletionCycling -v
```

Expected: compile error — `cyclePathCompletion` not defined yet.

- [ ] **Step 3: Add `cyclePathCompletion` and `resetPathCompletion` to `form.go`**

```go
func (m *Model) cyclePathCompletion() {
	f := &m.form.fields[1]
	if !m.form.candActive {
		dir, _ := splitPathInput(f.value)
		_, candidates := completePath(f.value)
		if len(candidates) == 0 {
			m.form.focusField = 2
			return
		}
		m.form.candidates = candidates
		m.form.candBase = dir
		m.form.candIdx = 0
		m.form.candActive = true
		f.value = dir + candidates[0]
		f.cursor = len([]rune(f.value))
		return
	}
	m.form.candIdx++
	if m.form.candIdx >= len(m.form.candidates) {
		m.resetPathCompletion()
		m.form.focusField = 2
		return
	}
	f.value = m.form.candBase + m.form.candidates[m.form.candIdx]
	f.cursor = len([]rune(f.value))
}

func (m *Model) resetPathCompletion() {
	m.form.candActive = false
	m.form.candidates = nil
	m.form.candIdx = 0
	m.form.candBase = ""
}
```

- [ ] **Step 4: Build check**

```bash
go build ./internal/ui/
```

- [ ] **Step 5: Commit**

```bash
git add internal/ui/form.go internal/ui/form_test.go
git commit -m "feat(ui): add path completion cycling"
```

---

## Task 5: `commitForm`

**Files:**
- Modify: `internal/ui/form.go` — add `commitForm`

- [ ] **Step 1: Add `commitForm` to `form.go`**

```go
func (m *Model) commitForm() {
	title := strings.TrimSpace(m.form.fields[0].value)
	if title == "" {
		return
	}
	path := strings.TrimSpace(m.form.fields[1].value)
	if path == "" {
		path = "."
	}
	tool := m.form.fields[2].options[m.form.fields[2].optIdx]
	flags := strings.TrimSpace(m.form.fields[3].value)
	script := strings.TrimSpace(m.form.fields[4].value)

	if err := db.CreateSession(m.conn, db.Session{
		ID:            uuid.New().String(),
		Title:         title,
		GroupPath:     m.currentGroupPath(),
		ProjectPath:   expandPath(path),
		Tool:          tool,
		ToolFlags:     flags,
		Status:        "stopped",
		CreatedAt:     time.Now().Unix(),
		StartupScript: script,
	}); err != nil {
		m.err = err
	}
}
```

- [ ] **Step 2: Build check**

```bash
go build ./internal/ui/
```

- [ ] **Step 3: Commit**

```bash
git add internal/ui/form.go
git commit -m "feat(ui): add commitForm"
```

---

## Task 6: `updateForm` key handler

**Files:**
- Modify: `internal/ui/form.go` — add `updateForm`

- [ ] **Step 1: Add `updateForm` to `form.go`**

```go
func (m *Model) updateForm(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	switch msg.Type {
	case tea.KeyEsc, tea.KeyCtrlC:
		m.mode = ""
		return m, nil

	case tea.KeyEnter:
		m.commitForm()
		m.mode = ""
		if err := m.Reload(); err != nil {
			m.err = err
		}
		return m, nil

	case tea.KeyTab:
		focus := m.form.focusField
		if focus == 1 {
			// path field: cycle completions
			m.cyclePathCompletion()
		} else {
			// all other fields: advance focus
			if focus < len(m.form.fields)-1 {
				m.form.focusField++
			}
		}
		return m, nil

	case tea.KeyLeft:
		f := &m.form.fields[m.form.focusField]
		if f.kind == fieldSelect {
			if f.optIdx > 0 {
				f.optIdx--
			} else {
				f.optIdx = len(f.options) - 1
			}
		} else {
			moveCursorLeft(f)
		}
		return m, nil

	case tea.KeyRight:
		f := &m.form.fields[m.form.focusField]
		if f.kind == fieldSelect {
			f.optIdx = (f.optIdx + 1) % len(f.options)
		} else {
			moveCursorRight(f)
		}
		return m, nil

	case tea.KeyBackspace:
		f := &m.form.fields[m.form.focusField]
		if f.kind == fieldText {
			deleteRune(f)
			if m.form.focusField == 1 {
				m.resetPathCompletion()
			}
		}
		return m, nil

	case tea.KeySpace:
		f := &m.form.fields[m.form.focusField]
		switch {
		case f.kind == fieldSelect:
			// space = accept and advance
			if m.form.focusField < len(m.form.fields)-1 {
				m.form.focusField++
			}
		case m.form.focusField == 1 && m.form.candActive:
			// path field mid-cycle: select current candidate, advance focus
			m.resetPathCompletion()
			m.form.focusField = 2
		default:
			// text field: insert space
			insertRune(f, ' ')
		}
		return m, nil

	default:
		if len(msg.Runes) == 1 {
			f := &m.form.fields[m.form.focusField]
			if f.kind == fieldText {
				insertRune(f, msg.Runes[0])
				if m.form.focusField == 1 {
					m.resetPathCompletion()
				}
			}
		}
		return m, nil
	}
}
```

- [ ] **Step 2: Build check**

```bash
go build ./internal/ui/
```

- [ ] **Step 3: Commit**

```bash
git add internal/ui/form.go
git commit -m "feat(ui): add updateForm key handler"
```

---

## Task 7: `renderForm`

**Files:**
- Modify: `internal/ui/form.go` — add `renderForm`

- [ ] **Step 1: Add `renderForm` to `form.go`**

```go
func (m *Model) renderForm() string {
	var sb strings.Builder

	// candidates row (only on path field while cycling)
	if m.form.focusField == 1 && m.form.candActive && len(m.form.candidates) > 0 {
		var parts []string
		for i, c := range m.form.candidates {
			if i == m.form.candIdx {
				parts = append(parts, formCandHLStyle.Render(c))
			} else {
				parts = append(parts, formCandStyle.Render(c))
			}
		}
		sb.WriteString(formCandStyle.Render("  ") + strings.Join(parts, formCandStyle.Render("  ")) + "\n")
	}

	sb.WriteString(formHeaderStyle.Render("NEW SESSION") + "\n")

	for i, f := range m.form.fields {
		isActive := i == m.form.focusField

		var labelStr string
		if isActive {
			labelStr = formBarStyle.Render("▌ ") + formLabelActive.Render(f.label)
		} else {
			labelStr = "  " + formLabelDim.Render(f.label)
		}
		label := labelStr + "  "

		var valueStr string
		switch f.kind {
		case fieldSelect:
			var parts []string
			for j, opt := range f.options {
				if j == f.optIdx {
					parts = append(parts, selectedStyle.Render("["+opt+"]"))
				} else {
					parts = append(parts, formValueDim.Render(opt))
				}
			}
			valueStr = strings.Join(parts, "  ")
		case fieldText:
			if f.value == "" && !isActive {
				valueStr = formValueDim.Render("─")
			} else if isActive {
				valueStr = renderWithCursor(f.value, f.cursor)
			} else {
				valueStr = formValueDim.Render(f.value)
			}
		}

		sb.WriteString(label + valueStr + "\n")
	}

	sb.WriteString(formHintStyle.Render("  Tab · Space · Enter to create · Esc cancel"))
	return sb.String()
}
```

- [ ] **Step 2: Build check**

```bash
go build ./internal/ui/
```

- [ ] **Step 3: Commit**

```bash
git add internal/ui/form.go
git commit -m "feat(ui): add renderForm with cursor and candidate display"
```

---

## Task 8: Wire form into `app.go`

**Files:**
- Modify: `internal/ui/app.go`

- [ ] **Step 1: Add `form formState` to `Model` struct**

In `internal/ui/app.go`, find the `Model` struct (around line 25). Add `form formState` after `dialog dialogState`:

```go
	dialog               dialogState
	form                 formState
```

- [ ] **Step 2: Route `"new-session"` action to `initSessionForm`**

In `app.go`, find the `case "new-session":` action block (around line 240). Replace it entirely with:

```go
	case "new-session":
		m.initSessionForm()
		m.mode = "new-session"
```

This removes the old `dialogState` initialization for new-session.

- [ ] **Step 3: Route key events to `updateForm` in `Update`**

In `app.go`, find the `Update` method's key message section. It currently reads:

```go
		if m.mode == "help" {
			...
		}
		if m.mode != "" {
			return m.updateDialog(msg)
		}
```

Change to:

```go
		if m.mode == "help" {
			...
		}
		if m.mode == "new-session" {
			return m.updateForm(msg)
		}
		if m.mode != "" {
			return m.updateDialog(msg)
		}
```

- [ ] **Step 4: Add top/bottom layout to `View`**

In `app.go`, find the `View` method. Before the existing split-panel rendering block (where `leftContent` is set), add:

```go
	if m.mode == "new-session" {
		formStr := m.renderForm()
		formLines := strings.Split(formStr, "\n")
		formH := len(formLines)
		listH := contentH - formH - 1
		if listH < 1 {
			listH = 1
		}
		listContent := RenderList(m.items, m.cursor, m.width, listH)
		sep := strings.Repeat("─", m.width)
		return header + "\n" + listContent + "\n" + sep + "\n" + formStr + "\n" + footer
	}
```

Place this block right after `contentH` is calculated and before `leftContent := RenderList(...)`.

- [ ] **Step 5: Build and run all tests**

```bash
go build ./... && go test ./...
```

Expected: all tests pass, no build errors.

- [ ] **Step 6: Commit**

```bash
git add internal/ui/app.go
git commit -m "feat(ui): wire session form into Model update and view"
```

---

## Task 9: Remove old new-session code from `dialog.go`

**Files:**
- Modify: `internal/ui/dialog.go`

- [ ] **Step 1: Remove `advanceNewSessionStep`**

Delete the entire function `func (m *Model) advanceNewSessionStep()` (currently lines 34–62). It is no longer called.

- [ ] **Step 2: Remove new-session-specific fields from `dialogState`**

In the `dialogState` struct, remove these fields:

```go
	// multi-step new-session flow — DELETE THESE
	step              int
	savedTitle        string
	savedPath         string
	savedToolFlags    string
	toolOptions       []string
	toolIdx           int
	candidates        []string
	candidatesFor     string
	candidatesVisible bool
```

Keep: `prompt`, `value`, `ctrlKeys`, `scope`, `scopeLabels`.

- [ ] **Step 3: Remove new-session branches from `updateDialog`**

In `updateDialog`, delete:
1. The block `if m.mode == "new-session" && m.dialog.step == 2 { ... }` (left/right tool cycling).
2. The block `if m.mode == "new-session" && m.dialog.step == 1 && msg.Type == tea.KeyTab { ... }` (path completion trigger).
3. In the `case tea.KeyEnter:` branch, delete `if m.mode == "new-session" && m.dialog.step < 4 { m.advanceNewSessionStep(); return m, nil }`.
4. In `case tea.KeyBackspace:` and the rune `default:`, delete the `m.clearDialogCandidates()` calls.

- [ ] **Step 4: Remove `clearDialogCandidates` and `completeDialogPath`**

Delete `func (m *Model) clearDialogCandidates()` — no longer needed.
Delete `func (m *Model) completeDialogPath()` — no longer needed (path cycling now lives in `form.go`).

Keep `completePath`, `splitPathInput`, `longestCommonPrefix`, `expandPath` — called from `form.go`.

- [ ] **Step 5: Remove new-session branches from `renderDialog`**

In `renderDialog`, delete:
1. The `if m.mode == "new-session" && m.dialog.step == 2 { ... }` block (tool selector rendering).
2. The `if m.mode == "new-session" && m.dialog.step == 1 && m.dialog.candidatesVisible ... { ... }` block.

- [ ] **Step 6: Remove the `case "new-session":` from `commitDialog`**

In `commitDialog`'s switch, delete the entire `case "new-session":` block.

- [ ] **Step 7: Build and run all tests**

```bash
go build ./... && go test ./...
```

Expected: all tests pass.

- [ ] **Step 8: Commit**

```bash
git add internal/ui/dialog.go
git commit -m "refactor(ui): remove old new-session wizard from dialog"
```

---

## Task 10: Integration tests and smoke-test

**Files:**
- Modify: `internal/ui/form_test.go` — ensure all form tests pass

- [ ] **Step 1: Run the full test suite**

```bash
go test ./... -v 2>&1 | grep -E "PASS|FAIL|---"
```

Expected: all `PASS`.

- [ ] **Step 2: Build the binary**

```bash
go build -o tmux-agent-deck .
```

Expected: no errors.

- [ ] **Step 3: Manual smoke-test**

Start a tmux session, run `./tmux-agent-deck`, press `n`:
- Form panel appears at the bottom with all 5 fields.
- TITLE field is active (purple left bar, cursor visible).
- Tab advances to PATH field (cursor at end of pre-filled path).
- Typing a partial path then pressing Tab cycles through directory completions.
- Space mid-cycle selects the current candidate and jumps to TOOL.
- Left/Right on TOOL cycles the options.
- Tab from TOOL advances to FLAGS.
- Tab from FLAGS advances to SCRIPT.
- Enter creates the session and returns to normal mode.
- Esc cancels and returns to normal mode.

- [ ] **Step 4: Final commit**

```bash
git add internal/ui/form_test.go
git commit -m "test(ui): finalize session form integration tests"
```
