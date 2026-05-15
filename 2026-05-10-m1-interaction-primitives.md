# M1 Interaction Primitives Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add send-to-pane (`x`), fork-session (`f`), broadcast-to-group (`b`), pane targeting (`tab`), and per-tool status heuristics to the split panel TUI.

**Architecture:** `SendKeys` is added to `ClientIface` and `Client`; a pure `interceptCtrl` helper translates bubbletea ctrl key events to tmux key names; `dialogState` gains `scope`/`scopeLabels` for broadcast; `Model` gains `activePaneIdx` for pane targeting; `DetectStatus` gains a `tool` parameter for per-tool waiting patterns.

**Tech Stack:** Go, bubbletea (TUI), lipgloss (styling), modernc SQLite, standard library only.

---

## File Structure

| File | What changes |
|---|---|
| `internal/tmux/client.go` | Add `SendKeys` to `ClientIface` and `Client`; add `paneTarget` helper |
| `internal/tmux/client_test.go` | Add `TestPaneTarget` |
| `internal/testutil/tmux.go` | Add `SentKeysCall`, `SentKeys` field, `SendKeys` stub to `FakeTmuxClient` |
| `internal/tmux/status.go` | Add `tool string` param to `DetectStatus`; add per-tool waiting patterns |
| `internal/tmux/status_test.go` | Update all existing calls; add per-tool test cases |
| `internal/state/poller.go` | Pass `s.Tool` to `DetectStatus` |
| `internal/ui/dialog.go` | Add `scope`/`scopeLabels` to `dialogState`; add `interceptCtrl`; extend `updateDialog`, `renderDialog`, `commitDialog` |
| `internal/ui/keys.go` | Add `'x'`, `'f'`, `'b'`, `tea.KeyTab` |
| `internal/ui/app.go` | Add `activePaneIdx` to `Model`; add `ActivePaneIdx()` accessor; reset in `Reload`; wire new actions in `updateNavigation`; update `renderPaneList` signature; update `renderFooter` |
| `internal/ui/app_test.go` | Add pane targeting, send-pane, fork-session, broadcast tests |

---

### Task 1: tmux client — add `SendKeys`

**Files:**
- Modify: `internal/tmux/client.go`
- Modify: `internal/tmux/client_test.go`

- [ ] **Step 1: Write the failing test for `paneTarget`**

Add to `internal/tmux/client_test.go` (file is `package tmux`, white-box):

```go
func TestPaneTarget(t *testing.T) {
	tests := []struct {
		session string
		pane    int
		want    string
	}{
		{"mysession", 0, "mysession:0"},
		{"ad-abc12345", 2, "ad-abc12345:2"},
	}
	for _, tc := range tests {
		got := paneTarget(tc.session, tc.pane)
		if got != tc.want {
			t.Errorf("paneTarget(%q, %d) = %q, want %q", tc.session, tc.pane, got, tc.want)
		}
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```
go test ./internal/tmux/... -run TestPaneTarget -v
```

Expected: compile error — `paneTarget` undefined.

- [ ] **Step 3: Add `paneTarget`, `SendKeys` to `ClientIface` and `Client`**

In `internal/tmux/client.go`, update `ClientIface` to add the new method:

```go
type ClientIface interface {
	NewSession(name, startDir, command string) error
	AttachSession(name string) error
	KillSession(name string) error
	SessionExists(name string) (bool, error)
	CapturePaneOutput(name string) (string, error)
	ListSessions() ([]string, error)
	ListPanes(session string) ([]Pane, error)
	SendKeys(session string, paneIndex int, keys string) error
}
```

Add these two functions after `ListPanes` and `parsePanesOutput`:

```go
func paneTarget(session string, paneIndex int) string {
	return fmt.Sprintf("%s:%d", session, paneIndex)
}

func (c *Client) SendKeys(session string, paneIndex int, keys string) error {
	if keys == "" {
		return nil
	}
	return runCmd("tmux", "send-keys", "-t", paneTarget(session, paneIndex), keys)
}
```

- [ ] **Step 4: Run tests**

```
go test ./internal/tmux/... -run TestPaneTarget -v
```

Expected: PASS.

Note: `go build ./...` will now fail because `FakeTmuxClient` does not implement `ClientIface`. That is expected — fixed in Task 2.

- [ ] **Step 5: Commit**

```bash
git add internal/tmux/client.go internal/tmux/client_test.go
git commit -m "feat: add SendKeys to tmux ClientIface and Client"
```

---

### Task 2: FakeTmuxClient — `SendKeys` stub

**Files:**
- Modify: `internal/testutil/tmux.go`

- [ ] **Step 1: Verify the project fails to compile**

```
go build ./...
```

Expected: compile error — `FakeTmuxClient` does not implement `tmux.ClientIface` (missing `SendKeys`).

- [ ] **Step 2: Replace `internal/testutil/tmux.go` with the updated version**

```go
package testutil

import (
	"fmt"

	"github.com/black-gato/tmux-agent-deck/internal/tmux"
)

// FakeTmuxClient implements tmux.ClientIface for tests.
// Sessions maps session name → pane output.
// Panes maps session name → pane list.
// SentKeys records all SendKeys calls.
type FakeTmuxClient struct {
	Sessions        map[string]string
	Panes           map[string][]tmux.Pane
	NewSessionCalls []NewSessionCall
	AttachCalls     []string
	KillCalls       []string
	SentKeys        []SentKeysCall
	NewSessionErr   error
	AttachErr       error
}

type NewSessionCall struct {
	Name    string
	Dir     string
	Command string
}

type SentKeysCall struct {
	Session   string
	PaneIndex int
	Keys      string
}

func NewFakeTmuxClient() *FakeTmuxClient {
	return &FakeTmuxClient{
		Sessions: make(map[string]string),
		Panes:    make(map[string][]tmux.Pane),
	}
}

func (f *FakeTmuxClient) NewSession(name, startDir, command string) error {
	f.NewSessionCalls = append(f.NewSessionCalls, NewSessionCall{name, startDir, command})
	if f.NewSessionErr != nil {
		return f.NewSessionErr
	}
	f.Sessions[name] = ""
	return nil
}

func (f *FakeTmuxClient) AttachSession(name string) error {
	f.AttachCalls = append(f.AttachCalls, name)
	return f.AttachErr
}

func (f *FakeTmuxClient) KillSession(name string) error {
	f.KillCalls = append(f.KillCalls, name)
	delete(f.Sessions, name)
	return nil
}

func (f *FakeTmuxClient) SessionExists(name string) (bool, error) {
	_, ok := f.Sessions[name]
	return ok, nil
}

func (f *FakeTmuxClient) CapturePaneOutput(name string) (string, error) {
	out, ok := f.Sessions[name]
	if !ok {
		return "", fmt.Errorf("no session %q", name)
	}
	return out, nil
}

func (f *FakeTmuxClient) ListSessions() ([]string, error) {
	names := make([]string, 0, len(f.Sessions))
	for name := range f.Sessions {
		names = append(names, name)
	}
	return names, nil
}

func (f *FakeTmuxClient) ListPanes(session string) ([]tmux.Pane, error) {
	panes, ok := f.Panes[session]
	if !ok {
		return []tmux.Pane{}, nil
	}
	return panes, nil
}

func (f *FakeTmuxClient) SendKeys(session string, paneIndex int, keys string) error {
	f.SentKeys = append(f.SentKeys, SentKeysCall{Session: session, PaneIndex: paneIndex, Keys: keys})
	return nil
}
```

- [ ] **Step 3: Verify the project compiles**

```
go build ./...
```

Expected: no errors.

- [ ] **Step 4: Run all tests**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add internal/testutil/tmux.go
git commit -m "feat: add SendKeys stub to FakeTmuxClient"
```

---

### Task 3: Status heuristics — extend `DetectStatus`

**Files:**
- Modify: `internal/tmux/status.go`
- Modify: `internal/tmux/status_test.go`
- Modify: `internal/state/poller.go`

- [ ] **Step 1: Write failing tests for per-tool patterns**

Replace `internal/tmux/status_test.go` with:

```go
package tmux_test

import (
	"testing"
	"time"

	"github.com/black-gato/tmux-agent-deck/internal/tmux"
)

func TestDetectStatusWaiting(t *testing.T) {
	for _, output := range []string{
		"Some output\n> ",
		"last line\n>",
	} {
		status := tmux.DetectStatus(output, time.Now(), "claude")
		if status != tmux.StatusWaiting {
			t.Errorf("output %q: got %q want %q", output, status, tmux.StatusWaiting)
		}
	}
}

func TestDetectStatusRunning(t *testing.T) {
	for _, output := range []string{
		"⠋ Thinking...",
		"⠙ Working...",
		"● Running",
		"Thinking about your request",
	} {
		status := tmux.DetectStatus(output, time.Now(), "claude")
		if status != tmux.StatusRunning {
			t.Errorf("output %q: got %q want running", output, status)
		}
	}
}

func TestDetectStatusIdle(t *testing.T) {
	output := "Some old output without a prompt"
	lastChange := time.Now().Add(-31 * time.Second)
	status := tmux.DetectStatus(output, lastChange, "claude")
	if status != tmux.StatusIdle {
		t.Errorf("got %q want idle", status)
	}
}

func TestDetectStatusRecentActivityIsRunning(t *testing.T) {
	output := "Some output without a prompt"
	status := tmux.DetectStatus(output, time.Now(), "claude")
	if status != tmux.StatusRunning {
		t.Errorf("got %q want running (recent activity)", status)
	}
}

func TestDetectStatusAiderWaiting(t *testing.T) {
	for _, output := range []string{
		"Some output\naider> ",
		"Some output\naider>",
	} {
		status := tmux.DetectStatus(output, time.Now(), "aider")
		if status != tmux.StatusWaiting {
			t.Errorf("aider output %q: got %q want waiting", output, status)
		}
	}
}

func TestDetectStatusAiderPromptNotMatchedForClaude(t *testing.T) {
	// "aider> " at end should NOT trigger waiting for claude tool
	output := "Some output\naider> "
	status := tmux.DetectStatus(output, time.Now(), "claude")
	if status == tmux.StatusWaiting {
		t.Errorf("aider> should not match waiting for claude tool")
	}
}

func TestDetectStatusCopilotWaiting(t *testing.T) {
	for _, output := range []string{
		"Some output\n❯ ",
		"Some output\n❯",
		"Some output\n> ",
	} {
		status := tmux.DetectStatus(output, time.Now(), "copilot")
		if status != tmux.StatusWaiting {
			t.Errorf("copilot output %q: got %q want waiting", output, status)
		}
	}
}

func TestDetectStatusBashWaiting(t *testing.T) {
	for _, output := range []string{
		"user@host:~$ ",
		"root@host:~# ",
		"Some output\n> ",
	} {
		status := tmux.DetectStatus(output, time.Now(), "")
		if status != tmux.StatusWaiting {
			t.Errorf("bash output %q: got %q want waiting", output, status)
		}
	}
}

func TestParseBindingCommand(t *testing.T) {
	tests := []struct {
		name  string
		key   string
		input string
		want  string
	}{
		{
			name:  "simple command",
			key:   "C-q",
			input: "bind-key -T root C-q display-panes",
			want:  "display-panes",
		},
		{
			name:  "multi-word command",
			key:   "C-q",
			input: "bind-key -T root C-q run-shell 'echo hi'",
			want:  "run-shell 'echo hi'",
		},
		{
			name:  "repeatable flag",
			key:   "C-q",
			input: "bind-key -rT root C-q resize-pane -D 5",
			want:  "resize-pane -D 5",
		},
		{
			name:  "empty output means no binding",
			key:   "C-q",
			input: "",
			want:  "",
		},
		{
			name:  "whitespace only",
			key:   "C-q",
			input: "   ",
			want:  "",
		},
		{
			name:  "key not present returns empty",
			key:   "C-q",
			input: "bind-key -T root x some-command",
			want:  "",
		},
		{
			name:  "single-char key still works",
			key:   "q",
			input: "bind-key -T prefix q send-keys q Enter",
			want:  "send-keys q Enter",
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := tmux.ParseBindingCommand(tt.input, tt.key)
			if got != tt.want {
				t.Errorf("ParseBindingCommand(%q, %q) = %q, want %q", tt.input, tt.key, got, tt.want)
			}
		})
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```
go test ./internal/tmux/... -v
```

Expected: compile errors (wrong number of args to `DetectStatus`) plus new test functions failing.

- [ ] **Step 3: Replace `internal/tmux/status.go`**

```go
package tmux

import (
	"strings"
	"time"
)

type Status = string

const (
	StatusRunning = "running"
	StatusWaiting = "waiting"
	StatusIdle    = "idle"
	StatusError   = "error"
	StatusStopped = "stopped"
)

var spinnerChars = []string{"⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"}

func DetectStatus(output string, lastChange time.Time, tool string) Status {
	trimmed := strings.TrimRight(output, " \t")

	switch tool {
	case "aider":
		if strings.HasSuffix(trimmed, "aider> ") || strings.HasSuffix(trimmed, "aider>") {
			return StatusWaiting
		}
	case "copilot":
		if strings.HasSuffix(trimmed, "❯ ") || strings.HasSuffix(trimmed, "❯") ||
			strings.HasSuffix(trimmed, "> ") || strings.HasSuffix(trimmed, ">") {
			return StatusWaiting
		}
	default: // "claude", "", and any other tool
		if strings.HasSuffix(trimmed, "> ") || strings.HasSuffix(trimmed, ">") ||
			strings.HasSuffix(trimmed, "$ ") || strings.HasSuffix(trimmed, "$") ||
			strings.HasSuffix(trimmed, "# ") || strings.HasSuffix(trimmed, "#") {
			return StatusWaiting
		}
	}

	tail := output
	if len(tail) > 200 {
		tail = tail[len(tail)-200:]
	}
	for _, ch := range spinnerChars {
		if strings.Contains(tail, ch) {
			return StatusRunning
		}
	}
	if strings.Contains(tail, "Thinking") || strings.Contains(tail, "Running") {
		return StatusRunning
	}

	if time.Since(lastChange) > 30*time.Second {
		return StatusIdle
	}

	return StatusRunning
}
```

- [ ] **Step 4: Fix the call site in `internal/state/poller.go`**

On line 89, change:
```go
newStatus := tmux.DetectStatus(out, lc)
```
to:
```go
newStatus := tmux.DetectStatus(out, lc, s.Tool)
```

- [ ] **Step 5: Run all tests**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add internal/tmux/status.go internal/tmux/status_test.go internal/state/poller.go
git commit -m "feat: extend DetectStatus with per-tool waiting patterns (aider, copilot, bash)"
```

---

### Task 4: `dialogState` scope fields + `interceptCtrl` helper

**Files:**
- Modify: `internal/ui/dialog.go`

No new tests in this task — `interceptCtrl` is tested indirectly in Task 7.

- [ ] **Step 1: Add `scope`/`scopeLabels` to `dialogState` and add `interceptCtrl`**

Replace `internal/ui/dialog.go` with:

```go
package ui

import (
	"fmt"
	"strings"
	"time"

	tea "github.com/charmbracelet/bubbletea"
	"github.com/google/uuid"
	"github.com/black-gato/tmux-agent-deck/internal/db"
)

const defaultGroupPath = "my-sessions"

type dialogState struct {
	prompt      string
	value       string
	scope       bool
	scopeLabels [2]string
}

func newDialogState(prompt string) dialogState {
	return dialogState{prompt: prompt}
}

// interceptCtrl maps bubbletea ctrl key events to tmux key names.
// Returns ("", false) for unmapped keys.
func interceptCtrl(msg tea.KeyMsg) (string, bool) {
	switch msg.Type {
	case tea.KeyCtrlC:
		return "C-c", true
	case tea.KeyCtrlD:
		return "C-d", true
	case tea.KeyCtrlZ:
		return "C-z", true
	case tea.KeyCtrlL:
		return "C-l", true
	case tea.KeyCtrlU:
		return "C-u", true
	}
	return "", false
}

func (m *Model) updateDialog(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	if m.mode == "send-pane" || m.mode == "broadcast" {
		if key, ok := interceptCtrl(msg); ok {
			m.dialog.value += key
			return m, nil
		}
		if msg.Type == tea.KeyTab && m.mode == "broadcast" {
			m.dialog.scope = !m.dialog.scope
			return m, nil
		}
	}
	switch msg.Type {
	case tea.KeyEsc:
		m.mode = ""
	case tea.KeyEnter:
		m.commitDialog()
		m.mode = ""
		if err := m.Reload(); err != nil {
			m.err = err
		}
	case tea.KeyBackspace:
		if len(m.dialog.value) > 0 {
			m.dialog.value = m.dialog.value[:len(m.dialog.value)-1]
		}
	default:
		if len(msg.Runes) > 0 {
			m.dialog.value += string(msg.Runes)
		}
	}
	return m, nil
}

func (m *Model) renderDialog() string {
	if m.mode == "edit-notes" {
		return "> " + m.dialog.value
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
		return fmt.Sprintf("Broadcast [%s / %s]:\n> %s", label0, label1, m.dialog.value)
	}
	return m.dialog.prompt + "\n> " + m.dialog.value
}

func (m *Model) commitDialog() {
	switch m.mode {
	case "new-session":
		val := strings.TrimSpace(m.dialog.value)
		if val == "" {
			return
		}
		groupPath := defaultGroupPath
		if m.cursor < len(m.items) && m.items[m.cursor].Kind == "group" {
			groupPath = m.items[m.cursor].Group.Path
		}
		if err := db.CreateSession(m.conn, db.Session{
			ID:          uuid.New().String(),
			Title:       val,
			GroupPath:   groupPath,
			ProjectPath: ".",
			Tool:        "claude",
			Status:      "stopped",
			CreatedAt:   time.Now().Unix(),
		}); err != nil {
			m.err = err
		}
	case "new-group":
		val := strings.TrimSpace(m.dialog.value)
		if val == "" {
			return
		}
		parts := strings.Split(val, "/")
		name := parts[len(parts)-1]
		if err := db.CreateGroup(m.conn, db.Group{
			Path:        val,
			Name:        name,
			DefaultTool: "claude",
			Expanded:    true,
		}); err != nil {
			m.err = err
		}
	case "rename":
		val := strings.TrimSpace(m.dialog.value)
		if val == "" {
			return
		}
		if m.cursor < len(m.items) {
			item := m.items[m.cursor]
			var err error
			if item.Kind == "session" {
				err = db.RenameSession(m.conn, item.Session.ID, val)
			} else if item.Kind == "group" {
				err = db.RenameGroup(m.conn, item.Group.Path, val)
			}
			if err != nil {
				m.err = err
			}
		}
	case "move":
		val := strings.TrimSpace(m.dialog.value)
		if val == "" {
			return
		}
		if m.cursor < len(m.items) && m.items[m.cursor].Kind == "session" {
			if err := db.MoveSession(m.conn, m.items[m.cursor].Session.ID, val); err != nil {
				m.err = err
			}
		}
	case "edit-notes":
		if m.cursor < len(m.items) && m.items[m.cursor].Kind == "session" {
			s := m.items[m.cursor].Session
			if err := db.UpdateSessionNotes(m.conn, s.ID, m.dialog.value); err != nil {
				m.err = err
			}
		}
	case "send-pane":
		if m.dialog.value == "" {
			return
		}
		if m.cursor < len(m.items) && m.items[m.cursor].Kind == "session" {
			s := m.items[m.cursor].Session
			if s.TmuxSession == "" {
				return
			}
			if err := m.tmuxC.SendKeys(s.TmuxSession, m.activePaneIdx, m.dialog.value); err != nil {
				m.err = err
			}
		}
	case "fork-session":
		val := strings.TrimSpace(m.dialog.value)
		if val == "" {
			return
		}
		if m.cursor < len(m.items) && m.items[m.cursor].Kind == "session" {
			s := m.items[m.cursor].Session
			if err := db.CreateSession(m.conn, db.Session{
				ID:          uuid.New().String(),
				Title:       val,
				GroupPath:   s.GroupPath,
				ProjectPath: s.ProjectPath,
				Tool:        s.Tool,
				Status:      "stopped",
				CreatedAt:   time.Now().Unix(),
			}); err != nil {
				m.err = err
			}
		}
	case "broadcast":
		if m.dialog.value == "" {
			return
		}
		if m.cursor >= len(m.items) {
			return
		}
		item := m.items[m.cursor]
		var groupPath string
		if item.Kind == "group" {
			groupPath = item.Group.Path
		} else if item.Kind == "session" {
			groupPath = item.Session.GroupPath
		} else {
			return
		}
		for _, s := range m.sessions {
			if s.Status != "running" || s.TmuxSession == "" {
				continue
			}
			inScope := s.GroupPath == groupPath
			if !inScope && m.dialog.scope {
				inScope = strings.HasPrefix(s.GroupPath, groupPath+"/")
			}
			if !inScope {
				continue
			}
			if err := m.tmuxC.SendKeys(s.TmuxSession, 0, m.dialog.value); err != nil {
				m.err = err
			}
		}
	}
}
```

- [ ] **Step 2: Verify compile**

```
go build ./...
```

Expected: no errors. (`m.activePaneIdx` is referenced in `commitDialog` but not yet declared on `Model` — if this causes a compile error, add a placeholder `activePaneIdx int` field to `Model` in `app.go` now, then add the full implementation in Task 6.)

- [ ] **Step 3: Run all tests**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 4: Commit**

```bash
git add internal/ui/dialog.go
git commit -m "feat: add interceptCtrl helper and scope fields to dialogState; wire send-pane, fork-session, broadcast in commitDialog"
```

---

### Task 5: `keys.go` — add `x`, `f`, `b`, Tab

**Files:**
- Modify: `internal/ui/keys.go`

- [ ] **Step 1: Add new key mappings**

Replace `internal/ui/keys.go` with:

```go
package ui

import tea "github.com/charmbracelet/bubbletea"

var keyTypeMap = map[tea.KeyType]string{
	tea.KeyUp:    "up",
	tea.KeyDown:  "down",
	tea.KeyEnter: "attach",
	tea.KeySpace: "toggle",
	tea.KeyTab:   "cycle-pane",
}

var runeMap = map[rune]string{
	'j': "down",
	'k': "up",
	'n': "new-session",
	'g': "new-group",
	'm': "move",
	'r': "rename",
	'd': "delete",
	'q': "quit",
	'v': "toggle-full",
	'e': "edit-notes",
	'x': "send-pane",
	'f': "fork-session",
	'b': "broadcast",
}

func actionForKey(msg tea.KeyMsg) string {
	if action, ok := keyTypeMap[msg.Type]; ok {
		return action
	}
	if len(msg.Runes) == 1 {
		if action, ok := runeMap[msg.Runes[0]]; ok {
			return action
		}
	}
	return ""
}
```

- [ ] **Step 2: Run all tests**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 3: Commit**

```bash
git add internal/ui/keys.go
git commit -m "feat: add x, f, b, Tab key bindings"
```

---

### Task 6: Pane targeting — `activePaneIdx`

**Files:**
- Modify: `internal/ui/app.go`
- Modify: `internal/ui/app_test.go`

- [ ] **Step 1: Write failing tests**

Add to `internal/ui/app_test.go`:

```go
func TestCyclePaneAdvancesIndex(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["ad-s1"] = "> "
	fake.Panes["ad-s1"] = []tmux.Pane{
		{Index: 0, Command: "claude"},
		{Index: 1, Command: "bash"},
		{Index: 2, Command: "nvim"},
	}
	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "a", GroupPath: "my-sessions",
		TmuxSession: "ad-s1", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, fake, nil)
	m.Reload()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	m.Reload()

	if m.ActivePaneIdx() != 0 {
		t.Fatalf("expected 0 initially, got %d", m.ActivePaneIdx())
	}
	m.Update(tea.KeyMsg{Type: tea.KeyTab})
	if m.ActivePaneIdx() != 1 {
		t.Errorf("expected 1 after first Tab, got %d", m.ActivePaneIdx())
	}
	m.Update(tea.KeyMsg{Type: tea.KeyTab})
	if m.ActivePaneIdx() != 2 {
		t.Errorf("expected 2 after second Tab, got %d", m.ActivePaneIdx())
	}
	m.Update(tea.KeyMsg{Type: tea.KeyTab})
	if m.ActivePaneIdx() != 0 {
		t.Errorf("expected wrap to 0 after third Tab, got %d", m.ActivePaneIdx())
	}
}

func TestCyclePaneResetsOnReload(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["ad-s1"] = "> "
	fake.Panes["ad-s1"] = []tmux.Pane{
		{Index: 0, Command: "claude"},
		{Index: 1, Command: "bash"},
	}
	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "a", GroupPath: "my-sessions",
		TmuxSession: "ad-s1", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, fake, nil)
	m.Reload()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	m.Reload()
	m.Update(tea.KeyMsg{Type: tea.KeyTab})
	if m.ActivePaneIdx() != 1 {
		t.Fatalf("expected 1 after Tab, got %d", m.ActivePaneIdx())
	}
	m.Reload()
	if m.ActivePaneIdx() != 0 {
		t.Errorf("expected reset to 0 after Reload, got %d", m.ActivePaneIdx())
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```
go test ./internal/ui/... -run "TestCyclePane" -v
```

Expected: compile error — `m.ActivePaneIdx()` undefined.

- [ ] **Step 3: Add `activePaneIdx` to `Model`, accessor, cycle-pane action, reset in `Reload`, update `renderPaneList`**

In `internal/ui/app.go`:

Add `activePaneIdx int` field to `Model`:

```go
type Model struct {
	conn          *sql.DB
	tmuxC         tmux.ClientIface
	poller        *state.Poller
	groups        []db.Group
	sessions      []db.Session
	items         []ListItem
	cursor        int
	width         int
	height        int
	mode          string
	dialog        dialogState
	err           error
	PendingAttach string
	viewFull      bool
	panes         []tmux.Pane
	output        string
	activePaneIdx int
}
```

Add accessor after `ViewFull()`:

```go
func (m *Model) ActivePaneIdx() int { return m.activePaneIdx }
```

In `Reload()`, add reset after `m.panes = nil`:

```go
m.activePaneIdx = 0
```

Add `"cycle-pane"` case to `updateNavigation` before `"quit"`:

```go
case "cycle-pane":
	if len(m.panes) > 0 {
		m.activePaneIdx = (m.activePaneIdx + 1) % len(m.panes)
	}
```

Update `renderPaneList` signature to accept `activeIdx` and highlight the active pane:

```go
func renderPaneList(panes []tmux.Pane, activeIdx int) string {
	if len(panes) == 0 {
		return ""
	}
	var parts []string
	for i, p := range panes {
		entry := fmt.Sprintf("[%d] %s", p.Index, p.Command)
		if i == activeIdx {
			parts = append(parts, selectedStyle.Render(entry))
		} else {
			parts = append(parts, dimStyle.Render(entry))
		}
	}
	return strings.Join(parts, "  ")
}
```

Update the call site in `RenderDetailPanel` to pass `m.activePaneIdx`:

```go
lines = append(lines, " "+renderPaneList(m.panes, m.activePaneIdx))
```

Also add `"send-pane"`, `"fork-session"`, `"broadcast"` stubs to `updateNavigation` (they open the dialog — implementations already in `commitDialog` from Task 4):

```go
case "send-pane":
	if m.cursor < len(m.items) && m.items[m.cursor].Kind == "session" {
		m.mode = "send-pane"
		m.dialog = newDialogState("Send:")
	}
case "fork-session":
	if m.cursor < len(m.items) && m.items[m.cursor].Kind == "session" {
		m.mode = "fork-session"
		m.dialog = newDialogState("Fork title:")
	}
case "broadcast":
	if m.cursor < len(m.items) {
		m.mode = "broadcast"
		m.dialog = dialogState{scopeLabels: [2]string{"this group", "all sub-groups"}}
	}
```

- [ ] **Step 4: Run pane targeting tests**

```
go test ./internal/ui/... -run "TestCyclePane" -v
```

Expected: all pass.

- [ ] **Step 5: Run all tests**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add internal/ui/app.go internal/ui/app_test.go
git commit -m "feat: add activePaneIdx pane targeting with Tab cycling and Reload reset"
```

---

### Task 7: send-pane mode — tests

**Files:**
- Modify: `internal/ui/app_test.go`

The implementation is already wired from Tasks 4 and 6. This task adds tests to verify observable behavior including ctrl key interception.

- [ ] **Step 1: Add send-pane tests**

Add to `internal/ui/app_test.go`:

```go
func TestSendPaneCallsSendKeys(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["ad-s1"] = "> "
	fake.Panes["ad-s1"] = []tmux.Pane{{Index: 0, Command: "claude"}}

	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "my-app", GroupPath: "my-sessions",
		TmuxSession: "ad-s1", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, fake, nil)
	m.Reload()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	m.Reload()

	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'x'}})
	for _, r := range "hello" {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
	}
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})

	if len(fake.SentKeys) != 1 {
		t.Fatalf("expected 1 SendKeys call, got %d", len(fake.SentKeys))
	}
	if fake.SentKeys[0].Session != "ad-s1" {
		t.Errorf("session: got %q want ad-s1", fake.SentKeys[0].Session)
	}
	if fake.SentKeys[0].Keys != "hello" {
		t.Errorf("keys: got %q want hello", fake.SentKeys[0].Keys)
	}
	if fake.SentKeys[0].PaneIndex != 0 {
		t.Errorf("pane: got %d want 0", fake.SentKeys[0].PaneIndex)
	}
}

func TestSendPaneCtrlCharSent(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["ad-s1"] = "> "

	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "my-app", GroupPath: "my-sessions",
		TmuxSession: "ad-s1", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, fake, nil)
	m.Reload()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	m.Reload()

	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'x'}})
	m.Update(tea.KeyMsg{Type: tea.KeyCtrlC})
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})

	if len(fake.SentKeys) != 1 {
		t.Fatalf("expected 1 SendKeys call, got %d", len(fake.SentKeys))
	}
	if fake.SentKeys[0].Keys != "C-c" {
		t.Errorf("keys: got %q want C-c", fake.SentKeys[0].Keys)
	}
}

func TestSendPaneNoOpWithoutTmuxSession(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()

	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "my-app", GroupPath: "my-sessions",
		TmuxSession: "", ProjectPath: "/p", Tool: "claude",
		Status: "stopped", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, fake, nil)
	m.Reload()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	m.Reload()

	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'x'}})
	for _, r := range "hello" {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
	}
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})

	if len(fake.SentKeys) != 0 {
		t.Errorf("expected no SendKeys calls, got %d", len(fake.SentKeys))
	}
}
```

- [ ] **Step 2: Run tests**

```
go test ./internal/ui/... -run "TestSendPane" -v
```

Expected: all pass.

- [ ] **Step 3: Run all tests**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 4: Commit**

```bash
git add internal/ui/app_test.go
git commit -m "test: add send-pane tests including ctrl char interception"
```

---

### Task 8: fork-session mode — tests

**Files:**
- Modify: `internal/ui/app_test.go`

- [ ] **Step 1: Add fork-session test**

Add to `internal/ui/app_test.go`:

```go
func TestForkSessionClonesFields(t *testing.T) {
	conn := testutil.OpenTestDB(t)

	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "original", GroupPath: "my-sessions",
		TmuxSession: "ad-s1", ProjectPath: "/my/project", Tool: "aider",
		Status: "running", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, nil, nil)
	m.Reload()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})

	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'f'}})
	for _, r := range "forked" {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
	}
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})

	sessions, err := db.ListSessions(conn)
	if err != nil {
		t.Fatal(err)
	}
	var forked *db.Session
	for i := range sessions {
		if sessions[i].Title == "forked" {
			forked = &sessions[i]
		}
	}
	if forked == nil {
		t.Fatal("forked session not found in DB")
	}
	if forked.ProjectPath != "/my/project" {
		t.Errorf("ProjectPath: got %q want /my/project", forked.ProjectPath)
	}
	if forked.Tool != "aider" {
		t.Errorf("Tool: got %q want aider", forked.Tool)
	}
	if forked.GroupPath != "my-sessions" {
		t.Errorf("GroupPath: got %q want my-sessions", forked.GroupPath)
	}
	if forked.Status != "stopped" {
		t.Errorf("Status: got %q want stopped", forked.Status)
	}
}
```

- [ ] **Step 2: Run test**

```
go test ./internal/ui/... -run TestForkSessionClonesFields -v
```

Expected: PASS.

- [ ] **Step 3: Run all tests**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 4: Commit**

```bash
git add internal/ui/app_test.go
git commit -m "test: add fork-session clone fields test"
```

---

### Task 9: broadcast mode — tests

**Files:**
- Modify: `internal/ui/app_test.go`

- [ ] **Step 1: Add broadcast tests**

Add to `internal/ui/app_test.go`:

```go
func TestBroadcastDirectGroup(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()

	db.CreateGroup(conn, db.Group{Path: "my-sessions/sub", Name: "sub", DefaultTool: "claude", Expanded: true})

	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "a", GroupPath: "my-sessions",
		TmuxSession: "ad-s1", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: 1000,
	})
	fake.Sessions["ad-s1"] = "> "
	db.CreateSession(conn, db.Session{
		ID: "s2", Title: "b", GroupPath: "my-sessions",
		TmuxSession: "ad-s2", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: 1001,
	})
	fake.Sessions["ad-s2"] = "> "
	db.CreateSession(conn, db.Session{
		ID: "s3", Title: "c", GroupPath: "my-sessions/sub",
		TmuxSession: "ad-s3", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: 1002,
	})
	fake.Sessions["ad-s3"] = "> "

	m := ui.NewModel(conn, fake, nil)
	m.Reload()
	// cursor=0 is the "my-sessions" group

	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'b'}})
	// scope=false by default (this group only)
	for _, r := range "ping" {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
	}
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})

	if len(fake.SentKeys) != 2 {
		t.Fatalf("expected 2 SendKeys calls (direct group only), got %d", len(fake.SentKeys))
	}
	for _, sk := range fake.SentKeys {
		if sk.Session == "ad-s3" {
			t.Errorf("sub-group session ad-s3 should not receive direct-group broadcast")
		}
	}
}

func TestBroadcastIncludesSubGroups(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()

	db.CreateGroup(conn, db.Group{Path: "my-sessions/sub", Name: "sub", DefaultTool: "claude", Expanded: true})

	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "a", GroupPath: "my-sessions",
		TmuxSession: "ad-s1", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: 1000,
	})
	fake.Sessions["ad-s1"] = "> "
	db.CreateSession(conn, db.Session{
		ID: "s2", Title: "b", GroupPath: "my-sessions/sub",
		TmuxSession: "ad-s2", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: 1001,
	})
	fake.Sessions["ad-s2"] = "> "

	m := ui.NewModel(conn, fake, nil)
	m.Reload()

	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'b'}})
	m.Update(tea.KeyMsg{Type: tea.KeyTab}) // toggle to include sub-groups
	for _, r := range "ping" {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
	}
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})

	if len(fake.SentKeys) != 2 {
		t.Fatalf("expected 2 SendKeys calls (group + sub-group), got %d", len(fake.SentKeys))
	}
	sent := map[string]bool{}
	for _, sk := range fake.SentKeys {
		sent[sk.Session] = true
	}
	if !sent["ad-s1"] {
		t.Error("ad-s1 (direct group) should receive broadcast")
	}
	if !sent["ad-s2"] {
		t.Error("ad-s2 (sub-group) should receive broadcast")
	}
}

func TestBroadcastSkipsNonRunning(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()

	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "a", GroupPath: "my-sessions",
		TmuxSession: "ad-s1", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: 1000,
	})
	fake.Sessions["ad-s1"] = "> "
	db.CreateSession(conn, db.Session{
		ID: "s2", Title: "b", GroupPath: "my-sessions",
		TmuxSession: "ad-s2", ProjectPath: "/p", Tool: "claude",
		Status: "stopped", CreatedAt: 1001,
	})

	m := ui.NewModel(conn, fake, nil)
	m.Reload()

	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'b'}})
	for _, r := range "ping" {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
	}
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})

	if len(fake.SentKeys) != 1 {
		t.Fatalf("expected 1 SendKeys call (running only), got %d", len(fake.SentKeys))
	}
	if fake.SentKeys[0].Session != "ad-s1" {
		t.Errorf("expected ad-s1, got %q", fake.SentKeys[0].Session)
	}
}
```

- [ ] **Step 2: Run tests**

```
go test ./internal/ui/... -run "TestBroadcast" -v
```

Expected: all pass.

- [ ] **Step 3: Run all tests**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 4: Commit**

```bash
git add internal/ui/app_test.go
git commit -m "test: add broadcast scope, sub-group, and non-running session tests"
```

---

### Task 10: Update footer + final verification

**Files:**
- Modify: `internal/ui/app.go`

- [ ] **Step 1: Update `renderFooter`**

In `internal/ui/app.go`, replace:

```go
func renderFooter() string {
	return " Enter Attach  v Expand output  e Notes  n New  g Group  d Delete  q Quit"
}
```

with:

```go
func renderFooter() string {
	return " Enter Attach  x Send  f Fork  b Broadcast  v Output  e Notes  n New  d Delete  q Quit"
}
```

- [ ] **Step 2: Run all tests**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 3: Build**

```
go build -o tmux-agent-deck .
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add internal/ui/app.go
git commit -m "feat: update footer with x, f, b key hints"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| `SendKeys(session, paneIndex, keys)` on `ClientIface` + `Client` | Task 1 |
| `FakeTmuxClient.SentKeys` + `SendKeys` stub | Task 2 |
| `DetectStatus` `tool` param + aider/copilot/bash patterns | Task 3 |
| `poller.go` passes `s.Tool` to `DetectStatus` | Task 3 |
| `interceptCtrl` pure helper | Task 4 |
| `scope`/`scopeLabels` on `dialogState` | Task 4 |
| `updateDialog` ctrl-intercept branch for send-pane/broadcast | Task 4 |
| Tab toggles `dialog.scope` in broadcast mode | Task 4 |
| `renderDialog` broadcast scope display with `→` | Task 4 |
| `commitDialog` send-pane case | Task 4 |
| `commitDialog` fork-session case | Task 4 |
| `commitDialog` broadcast case (scope logic, running filter) | Task 4 |
| `'x'`, `'f'`, `'b'`, `tea.KeyTab` keys | Task 5 |
| `activePaneIdx` on `Model`, accessor, reset in `Reload` | Task 6 |
| `cycle-pane` action in `updateNavigation` | Task 6 |
| `renderPaneList` highlights active pane | Task 6 |
| `updateNavigation` opens send-pane/fork-session/broadcast modes | Task 6 |
| `TestSendPaneCallsSendKeys`, `TestSendPaneCtrlCharSent`, `TestSendPaneNoOpWithoutTmuxSession` | Task 7 |
| `TestForkSessionClonesFields` | Task 8 |
| `TestBroadcastDirectGroup`, `TestBroadcastIncludesSubGroups`, `TestBroadcastSkipsNonRunning` | Task 9 |
| `TestCyclePaneAdvancesIndex`, `TestCyclePaneResetsOnReload` | Task 6 |
| Footer updated with x/f/b hints | Task 10 |

All spec requirements covered.

**Placeholder scan:** No TBDs, no incomplete sections, no "similar to Task N" references. All code blocks are complete.

**Type consistency:**
- `SentKeysCall{Session, PaneIndex, Keys}` defined Task 2, used in Task 7 tests (`fake.SentKeys[0].Session`, `.PaneIndex`, `.Keys`)
- `tmux.ClientIface.SendKeys(session string, paneIndex int, keys string) error` defined Task 1, implemented on `Client` Task 1, stubbed on `FakeTmuxClient` Task 2, called in `commitDialog` Task 4
- `dialogState.scope bool`, `dialogState.scopeLabels [2]string` added Task 4, used in `renderDialog` Task 4 and broadcast dialog init Task 6
- `m.activePaneIdx int` added Task 6, used in `commitDialog` send-pane case Task 4 (step 2 note handles compile order)
- `renderPaneList(panes []tmux.Pane, activeIdx int)` signature changed Task 6, call site updated same task
- `DetectStatus(output, lastChange, tool)` 3-arg signature Task 3, all call sites updated same task
