# M4 Session Configuration Implementation Plan

**Status: Complete** — all tasks implemented and verified. See `docs/superpowers/specs/2026-05-10-roadmap.md`.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the new-session dialog into a 4-step flow collecting title, project path, tool, and startup script; wire group defaults as pre-fills for path and tool; and deliver the startup script to the tmux pane 2 seconds after a new session is created.

**Architecture:** Three layers of change. (1) `internal/db` — schema v5 migration adds `startup_script TEXT` to sessions; all SQL queries updated. (2) `internal/ui/dialog.go` — `dialogState` gains step/savedTitle/savedPath/toolOptions/toolIdx; `updateDialog` advances steps 0–2 on Enter instead of committing; step 2 uses left/right arrows for tool cycling; `commitDialog` uses the multi-step fields; `updateNavigation` reads `m.groups` to pre-fill path and tool from group defaults. (3) `cmd/root.go` — reads `fm.PendingStartupScript` from the model and delivers it via `SendKeys` with a 2-second sleep before calling `AttachSession`.

**Tech Stack:** Go 1.22+, Bubbletea, Lipgloss, modernc SQLite, Cobra CLI

---

### Task 1: Schema v5 — add startup_script column

**Files:**
- Modify: `internal/db/db.go`
- Modify: `internal/db/sessions.go`
- Modify: `internal/db/db_test.go` (update schema version assertion)
- Modify: `internal/db/sessions_test.go` (add startup_script test)

- [x] **Step 1: Write the failing tests**

In `internal/db/db_test.go`, change the existing `TestOpenCreatesSchemaVersion` to expect `"5"` instead of `"4"`:

```go
func TestOpenCreatesSchemaVersion(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	var val string
	err := conn.QueryRow(`SELECT value FROM metadata WHERE key='schema_version'`).Scan(&val)
	if err != nil {
		t.Fatalf("query schema_version: %v", err)
	}
	if val != "5" {
		t.Errorf("schema_version: got %q want %q", val, "5")
	}
}
```

In `internal/db/sessions_test.go`, add:

```go
func TestSessionStartupScriptPersistedAndRetrieved(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	s := db.Session{
		ID:            "script-test-1111-2222-3333-444455556666",
		Title:         "test",
		GroupPath:     "my-sessions",
		ProjectPath:   "/tmp",
		Tool:          "claude",
		Status:        "stopped",
		CreatedAt:     1000,
		StartupScript: "claude --resume",
	}
	if err := db.CreateSession(conn, s); err != nil {
		t.Fatalf("create: %v", err)
	}
	got, err := db.GetSession(conn, s.ID)
	if err != nil {
		t.Fatalf("get: %v", err)
	}
	if got.StartupScript != "claude --resume" {
		t.Errorf("StartupScript: got %q want %q", got.StartupScript, "claude --resume")
	}
}
```

- [x] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/db/... -run "TestOpenCreatesSchemaVersion|TestSessionStartupScriptPersistedAndRetrieved" -v
```

Expected: `TestOpenCreatesSchemaVersion` FAIL with `got "4" want "5"`, `TestSessionStartupScriptPersistedAndRetrieved` FAIL with column does not exist or scan error.

- [x] **Step 3: Add v5 migration to db.go**

Replace the full `migrate` function in `internal/db/db.go`:

```go
func migrate(conn *sql.DB) error {
	if _, err := conn.Exec(`
		PRAGMA journal_mode=WAL;
		PRAGMA busy_timeout=5000;
	`); err != nil {
		return fmt.Errorf("pragmas: %w", err)
	}
	_, err := conn.Exec(`
		CREATE TABLE IF NOT EXISTS metadata (
			key   TEXT PRIMARY KEY,
			value TEXT NOT NULL
		);
		CREATE TABLE IF NOT EXISTS groups (
			path         TEXT PRIMARY KEY,
			name         TEXT NOT NULL,
			default_path TEXT NOT NULL DEFAULT '',
			default_tool TEXT NOT NULL DEFAULT 'claude',
			conductor_session_id TEXT NOT NULL DEFAULT '',
			expanded     INTEGER NOT NULL DEFAULT 1,
			sort_order   INTEGER NOT NULL DEFAULT 0
		);
		CREATE TABLE IF NOT EXISTS sessions (
			id             TEXT PRIMARY KEY,
			title          TEXT NOT NULL,
			group_path     TEXT NOT NULL DEFAULT 'my-sessions',
			tmux_session   TEXT NOT NULL DEFAULT '',
			project_path   TEXT NOT NULL,
			tool           TEXT NOT NULL DEFAULT 'claude',
			status         TEXT NOT NULL DEFAULT 'stopped',
			created_at     INTEGER NOT NULL,
			last_active    INTEGER NOT NULL DEFAULT 0,
			notes          TEXT NOT NULL DEFAULT '',
			archived       INTEGER NOT NULL DEFAULT 0,
			tags           TEXT NOT NULL DEFAULT '',
			startup_script TEXT NOT NULL DEFAULT ''
		);
		INSERT OR IGNORE INTO metadata (key, value) VALUES ('schema_version', '5');
		INSERT OR IGNORE INTO groups (path, name) VALUES ('my-sessions', 'my-sessions');
		INSERT OR IGNORE INTO groups (path, name) VALUES ('archived', 'archived');
	`)
	if err != nil {
		return err
	}
	var version string
	if err := conn.QueryRow(`SELECT value FROM metadata WHERE key = 'schema_version'`).Scan(&version); err != nil {
		return fmt.Errorf("read schema_version: %w", err)
	}
	if version == "1" {
		if _, err := conn.Exec(`ALTER TABLE sessions ADD COLUMN notes TEXT NOT NULL DEFAULT ''`); err != nil {
			return err
		}
		if _, err := conn.Exec(`UPDATE metadata SET value = '2' WHERE key = 'schema_version'`); err != nil {
			return err
		}
		version = "2"
	}
	if version == "2" {
		if _, err := conn.Exec(`ALTER TABLE sessions ADD COLUMN archived INTEGER NOT NULL DEFAULT 0`); err != nil {
			return err
		}
		if _, err := conn.Exec(`ALTER TABLE sessions ADD COLUMN tags TEXT NOT NULL DEFAULT ''`); err != nil {
			return err
		}
		if _, err := conn.Exec(`INSERT OR IGNORE INTO groups (path, name) VALUES ('archived', 'archived')`); err != nil {
			return err
		}
		if _, err := conn.Exec(`UPDATE metadata SET value = '3' WHERE key = 'schema_version'`); err != nil {
			return err
		}
		version = "3"
	}
	if version == "3" {
		if _, err := conn.Exec(`ALTER TABLE groups ADD COLUMN conductor_session_id TEXT NOT NULL DEFAULT ''`); err != nil {
			return err
		}
		if _, err := conn.Exec(`UPDATE metadata SET value = '4' WHERE key = 'schema_version'`); err != nil {
			return err
		}
		version = "4"
	}
	if version == "4" {
		if _, err := conn.Exec(`ALTER TABLE sessions ADD COLUMN startup_script TEXT NOT NULL DEFAULT ''`); err != nil {
			return err
		}
		if _, err := conn.Exec(`UPDATE metadata SET value = '5' WHERE key = 'schema_version'`); err != nil {
			return err
		}
	}
	return nil
}
```

- [x] **Step 4: Update Session struct and all SQL in sessions.go**

Add `StartupScript string` to the `Session` struct (after `Tags`):

```go
type Session struct {
	ID            string
	Title         string
	GroupPath     string
	TmuxSession   string
	ProjectPath   string
	Tool          string
	Status        string
	CreatedAt     int64
	LastActive    int64
	Notes         string
	Archived      bool
	Tags          string
	StartupScript string
}
```

Replace `CreateSession`:

```go
func CreateSession(conn *sql.DB, s Session) error {
	archived := 0
	if s.Archived {
		archived = 1
	}
	_, err := conn.Exec(
		`INSERT INTO sessions (id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active, notes, archived, tags, startup_script)
		 VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
		s.ID, s.Title, s.GroupPath, s.TmuxSession, s.ProjectPath, s.Tool, s.Status, s.CreatedAt, s.LastActive, s.Notes, archived, s.Tags, s.StartupScript,
	)
	return err
}
```

Replace `GetSession`:

```go
func GetSession(conn *sql.DB, id string) (Session, error) {
	var s Session
	var archived int
	err := conn.QueryRow(
		`SELECT id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active, notes, archived, tags, startup_script
		 FROM sessions WHERE id = ?`, id,
	).Scan(&s.ID, &s.Title, &s.GroupPath, &s.TmuxSession, &s.ProjectPath, &s.Tool, &s.Status, &s.CreatedAt, &s.LastActive, &s.Notes, &archived, &s.Tags, &s.StartupScript)
	if err != nil {
		return Session{}, fmt.Errorf("get session %q: %w", id, err)
	}
	s.Archived = archived == 1
	return s, nil
}
```

Replace `GetSessionByTitle`:

```go
func GetSessionByTitle(conn *sql.DB, title string) (Session, error) {
	var s Session
	var archived int
	err := conn.QueryRow(
		`SELECT id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active, notes, archived, tags, startup_script
		 FROM sessions WHERE title = ? LIMIT 1`, title,
	).Scan(&s.ID, &s.Title, &s.GroupPath, &s.TmuxSession, &s.ProjectPath, &s.Tool, &s.Status, &s.CreatedAt, &s.LastActive, &s.Notes, &archived, &s.Tags, &s.StartupScript)
	if err != nil {
		return Session{}, fmt.Errorf("get session by title %q: %w", title, err)
	}
	s.Archived = archived == 1
	return s, nil
}
```

Replace `GetGroupConductorSession`:

```go
func GetGroupConductorSession(conn *sql.DB, groupPath string) (Session, error) {
	var s Session
	var archived int
	err := conn.QueryRow(
		`SELECT s.id, s.title, s.group_path, s.tmux_session, s.project_path, s.tool, s.status, s.created_at, s.last_active, s.notes, s.archived, s.tags, s.startup_script
		 FROM sessions s
		 JOIN groups g ON g.path = s.group_path
		 WHERE g.path = ? AND g.conductor_session_id = s.id
		 LIMIT 1`,
		groupPath,
	).Scan(&s.ID, &s.Title, &s.GroupPath, &s.TmuxSession, &s.ProjectPath, &s.Tool, &s.Status, &s.CreatedAt, &s.LastActive, &s.Notes, &archived, &s.Tags, &s.StartupScript)
	if err != nil {
		return Session{}, fmt.Errorf("get group conductor %q: %w", groupPath, err)
	}
	s.Archived = archived == 1
	return s, nil
}
```

Replace `ListSessions`:

```go
func ListSessions(conn *sql.DB) ([]Session, error) {
	rows, err := conn.Query(
		`SELECT id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active, notes, archived, tags, startup_script
		 FROM sessions ORDER BY created_at DESC`,
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	return scanSessions(rows)
}
```

Replace `ListSessionsByGroup`:

```go
func ListSessionsByGroup(conn *sql.DB, groupPath string) ([]Session, error) {
	rows, err := conn.Query(
		`SELECT id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active, notes, archived, tags, startup_script
		 FROM sessions WHERE group_path = ? ORDER BY created_at DESC`, groupPath,
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	return scanSessions(rows)
}
```

Replace `ListWaitingGroupChildren`:

```go
func ListWaitingGroupChildren(conn *sql.DB, groupPath string) ([]Session, error) {
	rows, err := conn.Query(
		`SELECT s.id, s.title, s.group_path, s.tmux_session, s.project_path, s.tool, s.status, s.created_at, s.last_active, s.notes, s.archived, s.tags, s.startup_script
		 FROM sessions s
		 JOIN groups g ON g.path = s.group_path
		 WHERE s.group_path = ? AND s.status = ? AND s.id != g.conductor_session_id
		 ORDER BY s.created_at DESC`,
		groupPath, "waiting",
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	return scanSessions(rows)
}
```

Replace `scanSessions`:

```go
func scanSessions(rows *sql.Rows) ([]Session, error) {
	sessions := []Session{}
	for rows.Next() {
		var s Session
		var archived int
		if err := rows.Scan(&s.ID, &s.Title, &s.GroupPath, &s.TmuxSession, &s.ProjectPath, &s.Tool, &s.Status, &s.CreatedAt, &s.LastActive, &s.Notes, &archived, &s.Tags, &s.StartupScript); err != nil {
			return nil, err
		}
		s.Archived = archived == 1
		sessions = append(sessions, s)
	}
	return sessions, rows.Err()
}
```

- [x] **Step 5: Run tests to verify they pass**

```bash
go test ./internal/db/... -v
```

Expected: all pass. Look specifically for `TestOpenCreatesSchemaVersion` (now expects "5") and `TestSessionStartupScriptPersistedAndRetrieved`.

- [x] **Step 6: Commit**

```bash
git add internal/db/db.go internal/db/sessions.go internal/db/db_test.go internal/db/sessions_test.go
git commit -m "feat(db): add startup_script column (schema v5)"
```

---

### Task 2: Multi-step new-session dialog

**Files:**
- Modify: `internal/ui/dialog.go` — dialogState extensions, advanceNewSessionStep, renderDialog, commitDialog
- Modify: `internal/ui/app.go` — "new-session" case in updateNavigation reads group defaults, add `os` import
- Modify: `internal/ui/app_test.go` — new test for multi-step flow; update TestNewSessionDialogCreatesSession

**Design context:**
- `dialogState` gains 5 new zero-valued fields (zero-safe for all existing dialog modes)
- Step 0 = title, 1 = path, 2 = tool picker (← →), 3 = startup script
- Enter at steps 0–2 calls `advanceNewSessionStep()` and returns early (no commit, no mode clear)
- Enter at step 3 commits as normal
- Esc always cancels entirely (no step-back)
- `savedPath` is pre-filled from group default or `os.Getwd()` when the dialog opens; it becomes the initial value shown to the user at step 1
- New-session group context is resolved from the cursor: group rows use that group path, session rows use that session's `GroupPath`, and only missing/unknown rows fall back to `my-sessions`

- [x] **Step 1: Write the failing test**

In `internal/ui/app_test.go`, add this test:

```go
func TestNewSessionFlowCreatesSessionWithTool(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	m := ui.NewModel(conn, nil, nil)
	m.Reload()

	// Open new session dialog
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'n'}})
	if m.Mode() != "new-session" {
		t.Fatalf("expected new-session mode, got %q", m.Mode())
	}

	// Step 0: type title and press Enter — mode must stay "new-session"
	for _, r := range "fleet-agent" {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
	}
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})
	if m.Mode() != "new-session" {
		t.Fatalf("mode must stay new-session after step 0, got %q", m.Mode())
	}

	// Step 1: accept default path with Enter
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})
	if m.Mode() != "new-session" {
		t.Fatalf("mode must stay new-session after step 1, got %q", m.Mode())
	}

	// Step 2: cycle tool to "aider" (right arrow from "claude"), then Enter
	m.Update(tea.KeyMsg{Type: tea.KeyRight})
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})
	if m.Mode() != "new-session" {
		t.Fatalf("mode must stay new-session after step 2, got %q", m.Mode())
	}

	// Step 3: empty startup script, Enter to commit
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})
	if m.Mode() != "" {
		t.Errorf("expected mode cleared after commit, got %q", m.Mode())
	}

	sessions, err := db.ListSessions(conn)
	if err != nil {
		t.Fatal(err)
	}
	var found *db.Session
	for i := range sessions {
		if sessions[i].Title == "fleet-agent" {
			found = &sessions[i]
			break
		}
	}
	if found == nil {
		t.Fatalf("session 'fleet-agent' not created")
	}
	if found.Tool != "aider" {
		t.Errorf("Tool: got %q want %q", found.Tool, "aider")
	}
}
```

Also add a test for group defaults pre-filling:

```go
func TestNewSessionInheritsGroupDefaults(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	// Set group defaults on the default group
	if _, err := conn.Exec(`UPDATE groups SET default_path = '/opt/api', default_tool = 'aider' WHERE path = 'my-sessions'`); err != nil {
		t.Fatalf("set group defaults: %v", err)
	}
	m := ui.NewModel(conn, nil, nil)
	m.Reload()

	// Open new-session while cursor is on my-sessions group (default position)
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'n'}})
	// Step 0: title
	for _, r := range "defaults-test" {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
	}
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})
	// Step 1: accept pre-filled path (should be /opt/api from group default)
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})
	// Step 2: accept pre-selected tool (should be aider from group default)
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})
	// Step 3: no startup script
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})

	sessions, err := db.ListSessions(conn)
	if err != nil {
		t.Fatal(err)
	}
	var found *db.Session
	for i := range sessions {
		if sessions[i].Title == "defaults-test" {
			found = &sessions[i]
			break
		}
	}
	if found == nil {
		t.Fatalf("session 'defaults-test' not created")
	}
	if found.ProjectPath != "/opt/api" {
		t.Errorf("ProjectPath: got %q want %q", found.ProjectPath, "/opt/api")
	}
	if found.Tool != "aider" {
		t.Errorf("Tool: got %q want %q", found.Tool, "aider")
	}
}
```

- [x] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/ui/... -run "TestNewSessionFlowCreatesSessionWithTool|TestNewSessionInheritsGroupDefaults" -v
```

Expected: both FAIL — currently single Enter commits a session, so after step 0's Enter the mode is empty and later steps have no effect, or the session is created with the wrong tool.

- [x] **Step 3: Extend dialogState in dialog.go**

Replace the `dialogState` struct:

```go
type dialogState struct {
	prompt      string
	value       string
	ctrlKeys    []string
	scope       bool
	scopeLabels [2]string
	// multi-step new-session flow
	step        int
	savedTitle  string
	savedPath   string
	toolOptions []string
	toolIdx     int
}
```

- [x] **Step 4: Add expandPath and advanceNewSessionStep to dialog.go**

Add these imports to dialog.go (add `"os"` and `"path/filepath"`):

```go
import (
	"errors"
	"fmt"
	"os"
	"path/filepath"
	"strings"
	"time"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	tea "github.com/charmbracelet/bubbletea"
	"github.com/google/uuid"
)
```

Add `expandPath` (unexported helper, used only in commitDialog):

```go
func expandPath(p string) string {
	if p == "~" {
		if home, err := os.UserHomeDir(); err == nil {
			return home
		}
		return p
	}
	if strings.HasPrefix(p, "~/") {
		if home, err := os.UserHomeDir(); err == nil {
			return filepath.Join(home, p[2:])
		}
	}
	return os.ExpandEnv(p)
}
```

Add `advanceNewSessionStep` method on `*Model`:

```go
func (m *Model) advanceNewSessionStep() {
	switch m.dialog.step {
	case 0:
		val := strings.TrimSpace(m.dialog.value)
		if val == "" {
			return
		}
		m.dialog.savedTitle = val
		m.dialog.step = 1
		m.dialog.prompt = "Project path:"
		m.dialog.value = m.dialog.savedPath
	case 1:
		m.dialog.savedPath = m.dialog.value
		m.dialog.step = 2
		m.dialog.prompt = "Tool:"
		m.dialog.value = ""
	case 2:
		m.dialog.step = 3
		m.dialog.prompt = "Startup script (optional):"
		m.dialog.value = ""
	}
}
```

- [x] **Step 5: Update updateDialog for multi-step Enter and tool arrow keys**

In `updateDialog`, add tool selection handling right after the `send-pane`/`broadcast` block and before the `switch msg.Type`:

```go
func (m *Model) updateDialog(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	if m.mode == "send-pane" || m.mode == "broadcast" {
		if key, ok := interceptCtrl(msg); ok {
			m.dialog.ctrlKeys = append(m.dialog.ctrlKeys, key)
			return m, nil
		}
		if msg.Type == tea.KeyTab && m.mode == "broadcast" {
			m.dialog.scope = !m.dialog.scope
			return m, nil
		}
	}
	if m.mode == "new-session" && m.dialog.step == 2 {
		switch msg.Type {
		case tea.KeyLeft:
			if m.dialog.toolIdx > 0 {
				m.dialog.toolIdx--
			} else {
				m.dialog.toolIdx = len(m.dialog.toolOptions) - 1
			}
			return m, nil
		case tea.KeyRight:
			m.dialog.toolIdx = (m.dialog.toolIdx + 1) % len(m.dialog.toolOptions)
			return m, nil
		}
	}
	switch msg.Type {
	case tea.KeyEsc, tea.KeyCtrlC:
		m.mode = ""
	case tea.KeyEnter:
		if m.mode == "new-session" && m.dialog.step < 3 {
			m.advanceNewSessionStep()
			return m, nil
		}
		m.commitDialog()
		m.mode = ""
		if err := m.Reload(); err != nil {
			m.err = err
		}
		if m.navigateToGroup != "" {
			for i, item := range m.items {
				if item.Kind == "group" && item.Group.Path == m.navigateToGroup {
					m.cursor = i
					break
				}
			}
			m.navigateToGroup = ""
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
```

- [x] **Step 6: Update renderDialog for tool selection step**

Add a case for step 2 in `renderDialog`, before the final `return`:

```go
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
	if m.mode == "new-session" && m.dialog.step == 2 {
		var parts []string
		for i, t := range m.dialog.toolOptions {
			if i == m.dialog.toolIdx {
				parts = append(parts, selectedStyle.Render("["+t+"]"))
			} else {
				parts = append(parts, dimStyle.Render(t))
			}
		}
		return m.dialog.prompt + "\n← → to select:  " + strings.Join(parts, "  ")
	}
	return m.dialog.prompt + "\n> " + m.dialog.value
}
```

- [x] **Step 7: Update commitDialog "new-session" case to use multi-step fields**

Replace the `case "new-session":` block in `commitDialog`:

```go
case "new-session":
	if m.dialog.savedTitle == "" {
		return
	}
	groupPath := m.currentGroupPath()
	path := strings.TrimSpace(m.dialog.savedPath)
	if path == "" {
		path = "."
	}
	if err := db.CreateSession(m.conn, db.Session{
		ID:            uuid.New().String(),
		Title:         m.dialog.savedTitle,
		GroupPath:     groupPath,
		ProjectPath:   expandPath(path),
		Tool:          m.dialog.toolOptions[m.dialog.toolIdx],
		Status:        "stopped",
		CreatedAt:     time.Now().Unix(),
		StartupScript: strings.TrimSpace(m.dialog.value),
	}); err != nil {
		m.err = err
	}
```

- [x] **Step 8: Update "new-session" action in updateNavigation (app.go) to set up multi-step dialog**

Add `"os"` to the imports in `app.go`.

Replace the `case "new-session":` block in `updateNavigation`:

```go
case "new-session":
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
	toolOptions := []string{"claude", "aider", "cursor", "bash", "custom"}
	toolIdx := 0
	for i, t := range toolOptions {
		if t == defaultTool {
			toolIdx = i
			break
		}
	}
	m.mode = "new-session"
	m.dialog = dialogState{
		prompt:      "Session title:",
		toolOptions: toolOptions,
		toolIdx:     toolIdx,
		savedPath:   defaultPath,
	}
```

- [x] **Step 9: Update TestNewSessionDialogCreatesSession to go through all 4 steps**

The existing test presses Enter once after the title, which now only advances to step 1 — the session is no longer created at that point. Update the test:

```go
func TestNewSessionDialogCreatesSession(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	m := ui.NewModel(conn, nil, nil)
	m.Reload()

	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'n'}})
	for _, r := range "my-app" {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
	}
	m.Update(tea.KeyMsg{Type: tea.KeyEnter}) // step 0 → 1
	m.Update(tea.KeyMsg{Type: tea.KeyEnter}) // step 1 → 2
	m.Update(tea.KeyMsg{Type: tea.KeyEnter}) // step 2 → 3
	m.Update(tea.KeyMsg{Type: tea.KeyEnter}) // step 3 → commit

	sessions, err := db.ListSessions(conn)
	if err != nil {
		t.Fatal(err)
	}
	if len(sessions) != 1 {
		t.Fatalf("expected 1 session, got %d", len(sessions))
	}
	if sessions[0].Title != "my-app" {
		t.Errorf("title: got %q want my-app", sessions[0].Title)
	}
	if sessions[0].GroupPath != "my-sessions" {
		t.Errorf("group: got %q want my-sessions", sessions[0].GroupPath)
	}
}
```

- [x] **Step 10: Run all UI tests to verify they pass**

```bash
go test ./internal/ui/... -v
```

Expected: all pass. Look for the two new tests and the updated `TestNewSessionDialogCreatesSession`.

- [x] **Step 10a: Add regression coverage for session-row group context**

Add `TestNewSessionFromSelectedSessionUsesItsGroup` to cover the cursor-on-session case. The test creates a `work` group with an existing session, moves the cursor to that session row, opens `n`, completes the new-session flow, and asserts the created session has `GroupPath == "work"` instead of falling back to `my-sessions`.

- [x] **Step 11: Run full test suite**

```bash
go test ./...
```

Expected: all pass.

- [x] **Step 12: Commit**

```bash
git add internal/ui/dialog.go internal/ui/app.go internal/ui/app_test.go
git commit -m "feat(ui): multi-step new-session dialog with path, tool, and group defaults"
```

---

### Task 3: Startup script delivery

**Files:**
- Modify: `internal/ui/app.go` — add `PendingStartupScript string` to Model, change `ensureStarted` to return isNew, set PendingStartupScript in "attach" handler
- Modify: `cmd/root.go` — deliver startup script before AttachSession
- Modify: `internal/ui/app_test.go` — test PendingStartupScript is set for new sessions with a script

- [x] **Step 1: Write the failing test**

In `internal/ui/app_test.go`, add:

```go
func TestPendingStartupScriptSetForNewSession(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()

	if err := db.CreateSession(conn, db.Session{
		ID:            "abc12345-0000-0000-0000-000000000001",
		Title:         "with-script",
		GroupPath:     "my-sessions",
		ProjectPath:   "/tmp",
		Tool:          "claude",
		Status:        "stopped",
		CreatedAt:     1000,
		StartupScript: "claude --resume",
	}); err != nil {
		t.Fatalf("create: %v", err)
	}

	m := ui.NewModel(conn, fake, nil)
	m.Reload()

	moveCursorToSession(t, m, "with-script")
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})

	if m.PendingStartupScript != "claude --resume" {
		t.Errorf("PendingStartupScript: got %q want %q", m.PendingStartupScript, "claude --resume")
	}
}

func TestPendingStartupScriptNotSetForExistingSession(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	// Pre-create the tmux session so ensureStarted returns isNew=false
	fake.Sessions["ad-abc12345"] = ""

	if err := db.CreateSession(conn, db.Session{
		ID:            "abc12345-0000-0000-0000-000000000002",
		Title:         "already-running",
		GroupPath:     "my-sessions",
		TmuxSession:   "ad-abc12345",
		ProjectPath:   "/tmp",
		Tool:          "claude",
		Status:        "running",
		CreatedAt:     1000,
		StartupScript: "claude --resume",
	}); err != nil {
		t.Fatalf("create: %v", err)
	}

	m := ui.NewModel(conn, fake, nil)
	m.Reload()

	moveCursorToSession(t, m, "already-running")
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})

	if m.PendingStartupScript != "" {
		t.Errorf("PendingStartupScript should be empty for existing session, got %q", m.PendingStartupScript)
	}
}
```

- [x] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/ui/... -run "TestPendingStartupScriptSetForNewSession|TestPendingStartupScriptNotSetForExistingSession" -v
```

Expected: both FAIL — `PendingStartupScript` field does not exist on Model.

- [x] **Step 3: Add PendingStartupScript to Model and update ensureStarted in app.go**

Add `PendingStartupScript string` to the `Model` struct (alongside `PendingAttach`):

```go
type Model struct {
	// ... existing fields ...
	PendingAttach        string
	PendingStartupScript string
	// ... rest of fields ...
}
```

Change `ensureStarted` to return `(string, bool, error)` where the bool is `isNew`:

```go
func (m *Model) ensureStarted(s *db.Session) (string, bool, error) {
	if s.TmuxSession != "" {
		exists, err := m.tmuxC.SessionExists(s.TmuxSession)
		if err == nil && exists {
			return s.TmuxSession, false, nil
		}
	}
	tmuxName := fmt.Sprintf("ad-%s", s.ID[:8])
	if err := m.tmuxC.NewSession(tmuxName, s.ProjectPath, s.Tool); err != nil {
		return "", false, fmt.Errorf("start session: %w", err)
	}
	_ = db.UpdateSessionTmuxName(m.conn, s.ID, tmuxName)
	_ = db.UpdateSessionStatus(m.conn, s.ID, "waiting")
	return tmuxName, true, nil
}
```

Update the `"attach"` case in `updateNavigation` to use the new signature and set `PendingStartupScript`:

```go
case "attach":
	if m.cursor < len(m.items) {
		item := m.items[m.cursor]
		if item.Kind == "session" {
			tmuxName, isNew, err := m.ensureStarted(item.Session)
			if err != nil {
				m.err = err
				break
			}
			m.PendingAttach = tmuxName
			if isNew && item.Session.StartupScript != "" {
				m.PendingStartupScript = item.Session.StartupScript
			}
			if m.poller != nil {
				m.poller.Stop()
			}
			return m, tea.Quit
		}
	}
```

- [x] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/ui/... -run "TestPendingStartupScriptSetForNewSession|TestPendingStartupScriptNotSetForExistingSession" -v
```

Expected: both pass.

- [x] **Step 5: Deliver startup script in launchTUI (cmd/root.go)**

In `launchTUI`, update the post-TUI block to deliver the startup script before attaching:

```go
func launchTUI(conn *sql.DB, tc tmux.ClientIface) error {
	for {
		poller := state.NewWithNotifierInterval(conn, tc, notify.New(notify.Config{
			Enabled: notifyEnabled,
			Style:   notify.Style(notifyStyle),
			Quiet:   notifyQuiet,
		}), pollInterval)
		poller.Start()

		m := ui.NewModel(conn, tc, poller)
		p := tea.NewProgram(m, tea.WithAltScreen())
		finalModel, err := p.Run()
		if err != nil {
			return err
		}
		fm, ok := finalModel.(*ui.Model)
		if !ok || fm.PendingAttach == "" {
			return nil
		}
		exists, _ := tc.SessionExists(fm.PendingAttach)
		if !exists {
			return fmt.Errorf("tmux session %q exited before attach", fm.PendingAttach)
		}
		if fm.PendingStartupScript != "" {
			time.Sleep(2 * time.Second)
			_ = tc.SendKeys(fm.PendingAttach, 0, fm.PendingStartupScript)
		}
		if err := tc.AttachSession(fm.PendingAttach); err != nil {
			return err
		}
	}
}
```

- [x] **Step 6: Run full test suite**

```bash
go test ./...
```

Expected: all pass.

- [x] **Step 7: Commit**

```bash
git add internal/ui/app.go internal/ui/app_test.go cmd/root.go
git commit -m "feat: deliver startup script 2s after new session creation"
```

---

## Self-Review

**Spec coverage check:**

| Feature | Task |
|---|---|
| Project path picker (`n` flow, cwd pre-fill, `~`/`$VAR` expansion) | Task 2 |
| Tool selection (`n` flow, arrow key cycle) | Task 2 |
| Group defaults (wire `default_tool`/`default_path` into pre-fills) | Task 2 |
| Startup script column (schema v5) | Task 1 |
| Startup script input in `n` flow | Task 2 |
| Startup script delivery (2s delay, SendKeys) | Task 3 |

All spec features covered. Session presets explicitly deferred per spec.

**Placeholder scan:** None found. Every step contains complete code.

**Type consistency check:**
- `dialogState.savedPath` — set in Task 2 Step 3, read in Steps 4/7 ✓
- `dialogState.toolOptions`/`toolIdx` — set in Step 8, read in Steps 4/5/6/7 ✓
- `Session.StartupScript` — defined in Task 1 Step 4, used in Task 2 Step 7 and Task 3 Step 3 ✓
- `ensureStarted` returns `(string, bool, error)` — changed in Task 3 Step 3, only called in "attach" case in the same function ✓
- `PendingStartupScript` — added to Model in Task 3 Step 3, read in `cmd/root.go` Task 3 Step 5 ✓
