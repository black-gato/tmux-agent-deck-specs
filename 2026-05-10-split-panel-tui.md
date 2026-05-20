# Split Panel TUI Implementation Plan

**Status: Complete**

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the TUI from a single-column session list to a persistent split layout with a session detail panel showing live output, pane info, and editable notes.

**Architecture:** Left column (~35% terminal width) renders the existing group/session tree; right column (~65%) shows a detail panel for the currently selected session with header, live output tail, and notes. A `v` key toggles full-screen output; `e` enters inline notes editing. A new DB schema v2 migration adds a `notes` column. `ListPanes` is added to the tmux client and `ClientIface` to fetch per-session pane metadata.

**Tech Stack:** Go, bubbletea (TUI), lipgloss (styling), modernc SQLite, standard library only.

---

## File Structure

| File | What changes |
|------|-------------|
| `internal/db/db.go` | Add v2 migration: `ALTER TABLE sessions ADD COLUMN notes` |
| `internal/db/sessions.go` | Add `Notes string` to `Session`; update all SELECT/Scan; add `UpdateSessionNotes` |
| `internal/tmux/client.go` | Add `Pane` struct; add `ListPanes` to `ClientIface` and `Client` |
| `internal/testutil/tmux.go` | Add `Panes map[string][]Pane` field and stub `ListPanes` to `FakeTmuxClient` |
| `internal/ui/list.go` | Use the existing `width` param to truncate long names/titles |
| `internal/ui/keys.go` | Add `'v'` → `"toggle-full"` and `'e'` → `"edit-notes"` to `runeMap` |
| `internal/ui/app.go` | Add `viewFull bool`, `panes []tmux.Pane`, `output string` to `Model`; fetch panes+output in `Reload`; wire keys; add detail panel renderer; split `View` |

---

### Task 1: DB schema v2 — add notes column

**Files:**
- Modify: `internal/db/db.go`
- Modify: `internal/db/sessions.go`
- Test: `internal/db/sessions_test.go`

- [ ] **Step 1: Write the failing test for UpdateSessionNotes**

Add to `internal/db/sessions_test.go`:

```go
func TestUpdateSessionNotes(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	dbpkg.CreateSession(conn, dbpkg.Session{
		ID: "s1", Title: "a", GroupPath: "my-sessions",
		ProjectPath: "/p", Tool: "claude", Status: "stopped", CreatedAt: now,
	})

	if err := dbpkg.UpdateSessionNotes(conn, "s1", "check divergences first"); err != nil {
		t.Fatal(err)
	}
	s, err := dbpkg.GetSession(conn, "s1")
	if err != nil {
		t.Fatal(err)
	}
	if s.Notes != "check divergences first" {
		t.Errorf("notes: got %q want %q", s.Notes, "check divergences first")
	}
}

func TestSessionNotesDefaultsToEmpty(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	dbpkg.CreateSession(conn, dbpkg.Session{
		ID: "s1", Title: "a", GroupPath: "my-sessions",
		ProjectPath: "/p", Tool: "claude", Status: "stopped", CreatedAt: now,
	})
	s, err := dbpkg.GetSession(conn, "s1")
	if err != nil {
		t.Fatal(err)
	}
	if s.Notes != "" {
		t.Errorf("notes should default to empty, got %q", s.Notes)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```
go test ./internal/db/... -run TestUpdateSessionNotes -v
go test ./internal/db/... -run TestSessionNotesDefaultsToEmpty -v
```

Expected: compile error — `Session` has no `Notes` field, `UpdateSessionNotes` undefined.

- [ ] **Step 3: Add v2 migration to db.go**

Replace the `migrate` function in `internal/db/db.go`:

```go
func migrate(conn *sql.DB) error {
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
			expanded     INTEGER NOT NULL DEFAULT 1,
			sort_order   INTEGER NOT NULL DEFAULT 0
		);
		CREATE TABLE IF NOT EXISTS sessions (
			id           TEXT PRIMARY KEY,
			title        TEXT NOT NULL,
			group_path   TEXT NOT NULL DEFAULT 'my-sessions',
			tmux_session TEXT NOT NULL DEFAULT '',
			project_path TEXT NOT NULL,
			tool         TEXT NOT NULL DEFAULT 'claude',
			status       TEXT NOT NULL DEFAULT 'stopped',
			created_at   INTEGER NOT NULL,
			last_active  INTEGER NOT NULL DEFAULT 0
		);
		INSERT OR IGNORE INTO metadata (key, value) VALUES ('schema_version', '1');
		INSERT OR IGNORE INTO groups (path, name) VALUES ('my-sessions', 'my-sessions');
	`)
	if err != nil {
		return err
	}
	var version string
	conn.QueryRow(`SELECT value FROM metadata WHERE key = 'schema_version'`).Scan(&version)
	if version == "1" {
		if _, err := conn.Exec(`ALTER TABLE sessions ADD COLUMN notes TEXT NOT NULL DEFAULT ''`); err != nil {
			return err
		}
		if _, err := conn.Exec(`UPDATE metadata SET value = '2' WHERE key = 'schema_version'`); err != nil {
			return err
		}
	}
	return nil
}
```

- [ ] **Step 4: Update Session struct and all SELECT/Scan in sessions.go**

Replace the entire `internal/db/sessions.go` with:

```go
package db

import (
	"database/sql"
	"fmt"
)

type Session struct {
	ID          string
	Title       string
	GroupPath   string
	TmuxSession string
	ProjectPath string
	Tool        string
	Status      string
	CreatedAt   int64
	LastActive  int64
	Notes       string
}

func CreateSession(conn *sql.DB, s Session) error {
	_, err := conn.Exec(
		`INSERT INTO sessions (id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active)
		 VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)`,
		s.ID, s.Title, s.GroupPath, s.TmuxSession, s.ProjectPath, s.Tool, s.Status, s.CreatedAt, s.LastActive,
	)
	return err
}

func GetSession(conn *sql.DB, id string) (Session, error) {
	var s Session
	err := conn.QueryRow(
		`SELECT id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active, notes
		 FROM sessions WHERE id = ?`, id,
	).Scan(&s.ID, &s.Title, &s.GroupPath, &s.TmuxSession, &s.ProjectPath, &s.Tool, &s.Status, &s.CreatedAt, &s.LastActive, &s.Notes)
	if err != nil {
		return Session{}, fmt.Errorf("get session %q: %w", id, err)
	}
	return s, nil
}

func GetSessionByTitle(conn *sql.DB, title string) (Session, error) {
	var s Session
	err := conn.QueryRow(
		`SELECT id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active, notes
		 FROM sessions WHERE title = ? LIMIT 1`, title,
	).Scan(&s.ID, &s.Title, &s.GroupPath, &s.TmuxSession, &s.ProjectPath, &s.Tool, &s.Status, &s.CreatedAt, &s.LastActive, &s.Notes)
	if err != nil {
		return Session{}, fmt.Errorf("get session by title %q: %w", title, err)
	}
	return s, nil
}

func ListSessions(conn *sql.DB) ([]Session, error) {
	rows, err := conn.Query(
		`SELECT id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active, notes
		 FROM sessions ORDER BY created_at DESC`,
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	return scanSessions(rows)
}

func ListSessionsByGroup(conn *sql.DB, groupPath string) ([]Session, error) {
	rows, err := conn.Query(
		`SELECT id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active, notes
		 FROM sessions WHERE group_path = ? ORDER BY created_at DESC`, groupPath,
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	return scanSessions(rows)
}

func UpdateSessionStatus(conn *sql.DB, id, status string) error {
	res, err := conn.Exec(
		`UPDATE sessions SET status = ?, last_active = strftime('%s','now') WHERE id = ?`,
		status, id,
	)
	if err != nil {
		return err
	}
	n, _ := res.RowsAffected()
	if n == 0 {
		return fmt.Errorf("update status %q: %w", id, sql.ErrNoRows)
	}
	return nil
}

func UpdateSessionTmuxName(conn *sql.DB, id, tmuxSession string) error {
	res, err := conn.Exec(`UPDATE sessions SET tmux_session = ? WHERE id = ?`, tmuxSession, id)
	if err != nil {
		return err
	}
	n, _ := res.RowsAffected()
	if n == 0 {
		return fmt.Errorf("update tmux name %q: %w", id, sql.ErrNoRows)
	}
	return nil
}

func UpdateSessionNotes(conn *sql.DB, id, notes string) error {
	res, err := conn.Exec(`UPDATE sessions SET notes = ? WHERE id = ?`, notes, id)
	if err != nil {
		return err
	}
	n, _ := res.RowsAffected()
	if n == 0 {
		return fmt.Errorf("update notes %q: %w", id, sql.ErrNoRows)
	}
	return nil
}

func RenameSession(conn *sql.DB, id, newTitle string) error {
	res, err := conn.Exec(`UPDATE sessions SET title = ? WHERE id = ?`, newTitle, id)
	if err != nil {
		return err
	}
	n, _ := res.RowsAffected()
	if n == 0 {
		return fmt.Errorf("rename session %q: %w", id, sql.ErrNoRows)
	}
	return nil
}

func MoveSession(conn *sql.DB, id, groupPath string) error {
	res, err := conn.Exec(`UPDATE sessions SET group_path = ? WHERE id = ?`, groupPath, id)
	if err != nil {
		return err
	}
	n, _ := res.RowsAffected()
	if n == 0 {
		return fmt.Errorf("move session %q: %w", id, sql.ErrNoRows)
	}
	return nil
}

func DeleteSession(conn *sql.DB, id string) error {
	_, err := conn.Exec(`DELETE FROM sessions WHERE id = ?`, id)
	return err
}

func scanSessions(rows *sql.Rows) ([]Session, error) {
	sessions := []Session{}
	for rows.Next() {
		var s Session
		if err := rows.Scan(&s.ID, &s.Title, &s.GroupPath, &s.TmuxSession, &s.ProjectPath, &s.Tool, &s.Status, &s.CreatedAt, &s.LastActive, &s.Notes); err != nil {
			return nil, err
		}
		sessions = append(sessions, s)
	}
	return sessions, rows.Err()
}
```

- [ ] **Step 5: Run DB tests**

```
go test ./internal/db/... -v
```

Expected: all pass including `TestUpdateSessionNotes` and `TestSessionNotesDefaultsToEmpty`.

- [ ] **Step 6: Commit**

```bash
git add internal/db/db.go internal/db/sessions.go internal/db/sessions_test.go
git commit -m "feat: add schema v2 migration with notes column on sessions"
```

---

### Task 2: tmux — Pane type + ListPanes + ClientIface

**Files:**
- Modify: `internal/tmux/client.go`
- Create: `internal/tmux/client_test.go`

- [ ] **Step 1: Write the failing test for ListPanes**

Create `internal/tmux/client_test.go`:

```go
package tmux

import "testing"

func TestParsePanesOutput(t *testing.T) {
	tests := []struct {
		name  string
		input string
		want  []Pane
	}{
		{
			name:  "two panes",
			input: "0 claude\n1 bash\n",
			want:  []Pane{{Index: 0, Command: "claude"}, {Index: 1, Command: "bash"}},
		},
		{
			name:  "empty output",
			input: "",
			want:  []Pane{},
		},
		{
			name:  "single pane",
			input: "0 nvim\n",
			want:  []Pane{{Index: 0, Command: "nvim"}},
		},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			got := parsePanesOutput(tc.input)
			if len(got) != len(tc.want) {
				t.Fatalf("len: got %d want %d", len(got), len(tc.want))
			}
			for i, p := range got {
				if p.Index != tc.want[i].Index || p.Command != tc.want[i].Command {
					t.Errorf("[%d] got %+v want %+v", i, p, tc.want[i])
				}
			}
		})
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```
go test ./internal/tmux/... -run TestParsePanesOutput -v
```

Expected: compile error — `parsePanesOutput` undefined, `Pane` undefined.

- [ ] **Step 3: Add Pane type, ListPanes, and parsePanesOutput to client.go**

Add to `internal/tmux/client.go` after the `ClientIface` interface definition:

First, update the `ClientIface` interface to add `ListPanes`:

```go
type ClientIface interface {
	NewSession(name, startDir, command string) error
	AttachSession(name string) error
	KillSession(name string) error
	SessionExists(name string) (bool, error)
	CapturePaneOutput(name string) (string, error)
	ListSessions() ([]string, error)
	ListPanes(session string) ([]Pane, error)
}
```

Then add the `Pane` struct and implementation after the existing `ListSessions` method:

```go
type Pane struct {
	Index   int
	Command string
}

func (c *Client) ListPanes(session string) ([]Pane, error) {
	out, err := cmdOutput("tmux", "list-panes", "-t", session, "-F", "#{pane_index} #{pane_current_command}")
	if err != nil {
		if exitErr, ok := err.(*exec.ExitError); ok && exitErr.ExitCode() == 1 {
			return []Pane{}, nil
		}
		return nil, fmt.Errorf("list-panes %q: %w", session, err)
	}
	return parsePanesOutput(string(out)), nil
}

func parsePanesOutput(out string) []Pane {
	var panes []Pane
	for _, line := range strings.Split(strings.TrimSpace(out), "\n") {
		if line == "" {
			continue
		}
		var idx int
		var cmd string
		if _, err := fmt.Sscanf(line, "%d %s", &idx, &cmd); err != nil {
			continue
		}
		panes = append(panes, Pane{Index: idx, Command: cmd})
	}
	if panes == nil {
		return []Pane{}
	}
	return panes
}
```

- [ ] **Step 4: Run tests**

```
go test ./internal/tmux/... -v
```

Expected: all pass including `TestParsePanesOutput`.

- [ ] **Step 5: Commit**

```bash
git add internal/tmux/client.go internal/tmux/client_test.go
git commit -m "feat: add Pane type and ListPanes to tmux client"
```

---

### Task 3: FakeTmuxClient — ListPanes stub

**Files:**
- Modify: `internal/testutil/tmux.go`

- [ ] **Step 1: Verify the project fails to compile without the stub**

```
go build ./...
```

Expected: compile error — `FakeTmuxClient` does not implement `tmux.ClientIface` (missing `ListPanes`).

- [ ] **Step 2: Add Panes map and ListPanes stub to FakeTmuxClient**

Replace the entire `internal/testutil/tmux.go` with:

```go
package testutil

import (
	"fmt"

	"github.com/black-gato/tmux-agent-deck/internal/tmux"
)

// FakeTmuxClient implements tmux.ClientIface for tests.
// Configure Sessions to control which sessions "exist".
// Configure Panes to control per-session pane lists.
// NewSessionCalls and AttachCalls record what was called.
type FakeTmuxClient struct {
	Sessions        map[string]string    // session name → pane output
	Panes           map[string][]tmux.Pane // session name → pane list
	NewSessionCalls []NewSessionCall
	AttachCalls     []string
	KillCalls       []string
	NewSessionErr   error
	AttachErr       error
}

type NewSessionCall struct {
	Name    string
	Dir     string
	Command string
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
```

- [ ] **Step 3: Verify the project compiles**

```
go build ./...
```

Expected: no errors.

- [ ] **Step 4: Run all tests to verify nothing regressed**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add internal/testutil/tmux.go
git commit -m "feat: add ListPanes stub to FakeTmuxClient"
```

---

### Task 4: list.go — width-aware line truncation

**Files:**
- Modify: `internal/ui/list.go`
- Test: `internal/ui/list_test.go`

- [ ] **Step 1: Write failing tests for truncation**

Add to `internal/ui/list_test.go`:

```go
func TestRenderListTruncatesLongTitleToWidth(t *testing.T) {
	groups := []db.Group{{Path: "g", Name: "g", Expanded: true}}
	sessions := []db.Session{{ID: "s1", Title: "very-long-session-title-that-should-be-cut", GroupPath: "g", Status: "running"}}
	items := ui.BuildTree(groups, sessions)

	output := ui.RenderList(items, 1, 20, 24)
	for _, line := range strings.Split(output, "\n") {
		// strip ANSI codes before measuring — compare visible rune count
		visible := stripANSI(line)
		if len([]rune(visible)) > 20 {
			t.Errorf("line exceeds width 20: %q (len %d)", visible, len([]rune(visible)))
		}
	}
}

// stripANSI removes ANSI escape sequences for length measurement in tests.
func stripANSI(s string) string {
	var result []rune
	runes := []rune(s)
	i := 0
	for i < len(runes) {
		if runes[i] == '\x1b' && i+1 < len(runes) && runes[i+1] == '[' {
			i += 2
			for i < len(runes) && runes[i] != 'm' {
				i++
			}
			i++ // skip 'm'
			continue
		}
		result = append(result, runes[i])
		i++
	}
	return string(result)
}
```

- [ ] **Step 2: Run test to verify it fails**

```
go test ./internal/ui/... -run TestRenderListTruncatesLongTitleToWidth -v
```

Expected: FAIL — long lines are not yet truncated.

- [ ] **Step 3: Add truncate helper and use it in RenderList**

Replace `internal/ui/list.go` with:

```go
package ui

import (
	"fmt"
	"strings"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/charmbracelet/lipgloss"
)

type ListItem struct {
	Kind    string // "group" or "session"
	Group   *db.Group
	Session *db.Session
	Depth   int
}

var statusSymbol = map[string]string{
	"running": "●",
	"waiting": "○",
	"idle":    "◐",
	"error":   "✕",
	"stopped": "—",
}

var (
	selectedStyle = lipgloss.NewStyle().Bold(true).Foreground(lipgloss.Color("12"))
	groupStyle    = lipgloss.NewStyle().Bold(true)
	dimStyle      = lipgloss.NewStyle().Faint(true)
)

// BuildTree returns a flattened, ordered list of groups and their sessions.
// Top-level groups are those with no "/" in their path. Nested groups are
// appended recursively directly after their parent when the parent is expanded.
func BuildTree(groups []db.Group, sessions []db.Session) []ListItem {
	sessionsByGroup := make(map[string][]db.Session)
	for _, s := range sessions {
		sessionsByGroup[s.GroupPath] = append(sessionsByGroup[s.GroupPath], s)
	}

	var items []ListItem
	for _, g := range groups {
		if strings.Contains(g.Path, "/") {
			continue // skip; appended recursively by parent
		}
		items = append(items, appendGroupItems(g, groups, sessionsByGroup, 0)...)
	}
	return items
}

func appendGroupItems(g db.Group, allGroups []db.Group, sessionsByGroup map[string][]db.Session, depth int) []ListItem {
	gc := g
	items := []ListItem{{Kind: "group", Group: &gc, Depth: depth}}
	if !g.Expanded {
		return items
	}
	for _, s := range sessionsByGroup[g.Path] {
		sc := s
		items = append(items, ListItem{Kind: "session", Session: &sc, Depth: depth + 1})
	}
	for _, child := range allGroups {
		// direct child: has exactly one more path segment than g
		prefix := g.Path + "/"
		if !strings.HasPrefix(child.Path, prefix) {
			continue
		}
		remainder := child.Path[len(prefix):]
		if strings.Contains(remainder, "/") {
			continue // grandchild or deeper — handled recursively
		}
		items = append(items, appendGroupItems(child, allGroups, sessionsByGroup, depth+1)...)
	}
	return items
}

// RenderList renders the session list into a column of the given width and height.
func RenderList(items []ListItem, cursor, width, height int) string {
	var sb strings.Builder
	sb.WriteString("SESSIONS\n")
	sb.WriteString(strings.Repeat("─", width) + "\n")

	// Viewport: show a window of items centered around cursor
	start := 0
	end := len(items)
	if height > 4 {
		viewHeight := height - 4 // reserve header (2) and footer (2) lines
		if viewHeight > 0 && len(items) > viewHeight {
			start = cursor - viewHeight/2
			if start < 0 {
				start = 0
			}
			end = start + viewHeight
			if end > len(items) {
				end = len(items)
				start = end - viewHeight
				if start < 0 {
					start = 0
				}
			}
		}
	}

	for i := start; i < end; i++ {
		item := items[i]
		indent := strings.Repeat("  ", item.Depth)
		selected := i == cursor

		var line string
		if item.Kind == "group" {
			arrow := "▼"
			if !item.Group.Expanded {
				arrow = "►"
			}
			nameMax := width - len([]rune(indent)) - 2 // arrow + space
			if nameMax < 1 {
				nameMax = 1
			}
			raw := fmt.Sprintf("%s%s %s", indent, arrow, truncate(item.Group.Name, nameMax))
			if selected {
				line = selectedStyle.Render(raw)
			} else {
				line = groupStyle.Render(raw)
			}
		} else {
			sym := statusSymbol[item.Session.Status]
			if sym == "" {
				sym = "—"
			}
			// format: indent + sym + "  " + title
			prefixLen := len([]rune(indent)) + 1 + 2 // sym(1) + spaces(2)
			titleMax := width - prefixLen
			if titleMax < 1 {
				titleMax = 1
			}
			title := truncate(item.Session.Title, titleMax)
			if selected {
				raw := fmt.Sprintf("%s%s  %s", indent, sym, title)
				line = selectedStyle.Render(raw)
			} else {
				line = fmt.Sprintf("%s%s  %s", indent, sym, title)
			}
		}
		sb.WriteString(line + "\n")
	}

	return sb.String()
}

// truncate shortens s to at most n runes, appending "…" if truncated.
func truncate(s string, n int) string {
	runes := []rune(s)
	if len(runes) <= n {
		return s
	}
	if n <= 1 {
		return "…"
	}
	return string(runes[:n-1]) + "…"
}
```

- [ ] **Step 4: Run list tests**

```
go test ./internal/ui/... -run "TestRenderList|TestBuildTree" -v
```

Expected: all pass. Note: `TestRenderListContainsSessionTitle` and `TestRenderListContainsStatusSymbol` must still pass; the header changed from `"tmux-agent-deck\n\n"` to `"SESSIONS\n..."`. If those tests check for the old header string, update them to not check for `"tmux-agent-deck"`.

If `TestRenderListContainsSessionTitle` or `TestRenderListContainsStatusSymbol` fail because they searched for `"tmux-agent-deck"`, they don't — they only check for session titles and status symbols, so they should pass.

- [ ] **Step 5: Run all tests**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add internal/ui/list.go internal/ui/list_test.go
git commit -m "feat: use width param in RenderList to truncate long names"
```

---

### Task 5: Model new fields + Reload pane/output fetching

**Files:**
- Modify: `internal/ui/app.go`
- Test: `internal/ui/app_test.go`

- [ ] **Step 1: Write failing tests for new Model fields**

Add to `internal/ui/app_test.go`:

```go
func TestReloadFetchesPanesForSelectedSession(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["ad-abc12345"] = "> "
	fake.Panes["ad-abc12345"] = []tmux.Pane{
		{Index: 0, Command: "claude"},
		{Index: 1, Command: "bash"},
	}

	db.CreateSession(conn, db.Session{
		ID:          "abc12345-0000-0000-0000-000000000000",
		Title:       "my-app",
		GroupPath:   "my-sessions",
		TmuxSession: "ad-abc12345",
		ProjectPath: "/tmp",
		Tool:        "claude",
		Status:      "running",
		CreatedAt:   1000,
	})

	m := ui.NewModel(conn, fake, nil)
	m.Reload()
	// cursor=0 is the group; move to session at index 1
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	m.Reload()

	panes := m.Panes()
	if len(panes) != 2 {
		t.Fatalf("expected 2 panes, got %d", len(panes))
	}
	if panes[0].Command != "claude" {
		t.Errorf("pane[0].Command: got %q want claude", panes[0].Command)
	}
}

func TestReloadFetchesOutputForSelectedSession(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["ad-abc12345"] = "Running tests...\n✓ 12 pass\n> "

	db.CreateSession(conn, db.Session{
		ID:          "abc12345-0000-0000-0000-000000000000",
		Title:       "my-app",
		GroupPath:   "my-sessions",
		TmuxSession: "ad-abc12345",
		ProjectPath: "/tmp",
		Tool:        "claude",
		Status:      "running",
		CreatedAt:   1000,
	})

	m := ui.NewModel(conn, fake, nil)
	m.Reload()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	m.Reload()

	if !strings.Contains(m.Output(), "12 pass") {
		t.Errorf("output missing captured pane output, got: %q", m.Output())
	}
}
```

Add import `"strings"` and `"github.com/black-gato/tmux-agent-deck/internal/tmux"` to the test file imports.

- [ ] **Step 2: Run tests to verify they fail**

```
go test ./internal/ui/... -run "TestReloadFetches" -v
```

Expected: compile error — `m.Panes()` and `m.Output()` undefined.

- [ ] **Step 3: Add viewFull, panes, output fields to Model and update Reload**

In `internal/ui/app.go`, update the `Model` struct:

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
}
```

Add accessor methods after the existing `Mode()` accessor:

```go
func (m *Model) Panes() []tmux.Pane { return m.panes }
func (m *Model) Output() string      { return m.output }
func (m *Model) ViewFull() bool      { return m.viewFull }
```

Update the `Reload()` method to fetch panes and output for the selected session. Replace the current `Reload()` with:

```go
func (m *Model) Reload() error {
	groups, err := db.ListGroups(m.conn)
	if err != nil {
		return err
	}
	sessions, err := db.ListSessions(m.conn)
	if err != nil {
		return err
	}
	m.groups = groups
	m.sessions = sessions
	m.items = BuildTree(groups, sessions)
	if m.cursor >= len(m.items) && len(m.items) > 0 {
		m.cursor = len(m.items) - 1
	}
	m.panes = nil
	m.output = ""
	if m.tmuxC != nil && m.cursor < len(m.items) && m.items[m.cursor].Kind == "session" {
		s := m.items[m.cursor].Session
		if s.TmuxSession != "" {
			if panes, err := m.tmuxC.ListPanes(s.TmuxSession); err == nil {
				m.panes = panes
			}
			if out, err := m.tmuxC.CapturePaneOutput(s.TmuxSession); err == nil {
				m.output = out
			}
		}
	}
	return nil
}
```

- [ ] **Step 4: Run tests**

```
go test ./internal/ui/... -run "TestReloadFetches" -v
```

Expected: pass.

- [ ] **Step 5: Run all tests**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add internal/ui/app.go internal/ui/app_test.go
git commit -m "feat: add panes/output fields to Model with Reload fetching"
```

---

### Task 6: keys.go — add v and e bindings

**Files:**
- Modify: `internal/ui/keys.go`

- [ ] **Step 1: Add v and e to runeMap**

Replace `internal/ui/keys.go` with:

```go
package ui

import tea "github.com/charmbracelet/bubbletea"

var keyTypeMap = map[tea.KeyType]string{
	tea.KeyUp:    "up",
	tea.KeyDown:  "down",
	tea.KeyEnter: "attach",
	tea.KeySpace: "toggle",
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

- [ ] **Step 2: Wire toggle-full and edit-notes in updateNavigation**

In `internal/ui/app.go`, add cases to the `updateNavigation` switch before `"quit"`:

```go
case "toggle-full":
    m.viewFull = !m.viewFull
case "edit-notes":
    if m.cursor < len(m.items) && m.items[m.cursor].Kind == "session" {
        m.mode = "edit-notes"
        m.dialog = newDialogState(m.items[m.cursor].Session.Notes)
    }
```

- [ ] **Step 3: Write tests for toggle-full and edit-notes key**

Add to `internal/ui/app_test.go`:

```go
func TestVTogglesFullScreen(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	m := ui.NewModel(conn, nil, nil)
	m.Reload()

	if m.ViewFull() {
		t.Fatal("viewFull should start false")
	}
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'v'}})
	if !m.ViewFull() {
		t.Error("viewFull should be true after v")
	}
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'v'}})
	if m.ViewFull() {
		t.Error("viewFull should be false after second v")
	}
}

func TestEOnSessionOpensEditNotes(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "my-app", GroupPath: "my-sessions",
		ProjectPath: "/p", Tool: "claude", Status: "stopped", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, nil, nil)
	m.Reload()
	// cursor 0 = group, move to session
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'e'}})
	if m.Mode() != "edit-notes" {
		t.Errorf("expected mode edit-notes, got %q", m.Mode())
	}
}

func TestEOnGroupHasNoEffect(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	m := ui.NewModel(conn, nil, nil)
	m.Reload()
	// cursor 0 = group
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'e'}})
	if m.Mode() != "" {
		t.Errorf("e on group should not change mode, got %q", m.Mode())
	}
}
```

- [ ] **Step 4: Run tests**

```
go test ./internal/ui/... -run "TestVToggles|TestEOn" -v
```

Expected: all pass.

- [ ] **Step 5: Run all tests**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add internal/ui/keys.go internal/ui/app.go internal/ui/app_test.go
git commit -m "feat: add v (toggle-full) and e (edit-notes) key bindings"
```

---

### Task 7: edit-notes dialog commit + cancel

**Files:**
- Modify: `internal/ui/dialog.go`
- Test: `internal/ui/app_test.go`

The `dialogState` currently stores the input in `value`. For `edit-notes`, pressing Enter should call `db.UpdateSessionNotes` with `m.dialog.value` and clear the mode. Pressing Esc should discard.

- [ ] **Step 1: Write failing tests for notes save/cancel**

Add to `internal/ui/app_test.go`:

```go
func TestEditNotesEnterSaves(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "my-app", GroupPath: "my-sessions",
		ProjectPath: "/p", Tool: "claude", Status: "stopped", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, nil, nil)
	m.Reload()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}}) // select session
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'e'}}) // open edit-notes

	for _, r := range "my note" {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
	}
	m.Update(tea.KeyMsg{Type: tea.KeyEnter}) // save

	if m.Mode() != "" {
		t.Errorf("mode should clear after Enter, got %q", m.Mode())
	}
	s, err := db.GetSession(conn, "s1")
	if err != nil {
		t.Fatal(err)
	}
	if s.Notes != "my note" {
		t.Errorf("notes: got %q want my note", s.Notes)
	}
}

func TestEditNotesEscDiscards(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "my-app", GroupPath: "my-sessions",
		ProjectPath: "/p", Tool: "claude", Status: "stopped", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, nil, nil)
	m.Reload()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'e'}})
	for _, r := range "discard me" {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
	}
	m.Update(tea.KeyMsg{Type: tea.KeyEsc})

	s, err := db.GetSession(conn, "s1")
	if err != nil {
		t.Fatal(err)
	}
	if s.Notes != "" {
		t.Errorf("notes should not be saved on Esc, got %q", s.Notes)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```
go test ./internal/ui/... -run "TestEditNotes" -v
```

Expected: FAIL — `edit-notes` case not handled in `commitDialog`.

- [ ] **Step 3: Add edit-notes case to newDialogState and commitDialog**

In `internal/ui/dialog.go`, update `newDialogState` to accept the pre-population value separately. Actually, `newDialogState` currently takes only a `prompt`. For `edit-notes`, we need to pre-populate `value`. Add an overload by changing the call in `app.go`:

Change this line in `app.go` (added in Task 6):
```go
m.dialog = newDialogState(m.items[m.cursor].Session.Notes)
```
to:
```go
m.dialog = dialogState{prompt: "", value: m.items[m.cursor].Session.Notes}
```

Then add the `edit-notes` case to `commitDialog` in `internal/ui/dialog.go`:

```go
case "edit-notes":
    if m.cursor < len(m.items) && m.items[m.cursor].Kind == "session" {
        s := m.items[m.cursor].Session
        if err := db.UpdateSessionNotes(m.conn, s.ID, m.dialog.value); err != nil {
            m.err = err
        }
    }
```

The full updated `commitDialog` in `internal/ui/dialog.go`:

```go
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
	}
}
```

Also fix the `renderDialog` — for `edit-notes`, we want inline rendering (no generic prompt):

```go
func (m *Model) renderDialog() string {
	if m.mode == "edit-notes" {
		return "> " + m.dialog.value
	}
	return m.dialog.prompt + "\n> " + m.dialog.value
}
```

And in `updateNavigation` in `app.go`, fix the `edit-notes` open to use `dialogState` directly:

```go
case "edit-notes":
    if m.cursor < len(m.items) && m.items[m.cursor].Kind == "session" {
        m.mode = "edit-notes"
        m.dialog = dialogState{prompt: "", value: m.items[m.cursor].Session.Notes}
    }
```

- [ ] **Step 4: Run tests**

```
go test ./internal/ui/... -run "TestEditNotes" -v
```

Expected: all pass.

- [ ] **Step 5: Run all tests**

```
go test ./...
```

Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add internal/ui/dialog.go internal/ui/app.go internal/ui/app_test.go
git commit -m "feat: wire edit-notes mode to save/discard session notes"
```

---

### Task 8: Detail panel renderer

**Files:**
- Modify: `internal/ui/app.go`
- Test: `internal/ui/app_test.go`

- [ ] **Step 1: Write tests for detail panel output**

Add to `internal/ui/app_test.go`:

```go
func TestDetailPanelShowsSessionTitle(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["ad-s1"] = "some output\n> "

	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "my-feature", GroupPath: "my-sessions",
		TmuxSession: "ad-s1", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, fake, nil)
	m.Reload()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	m.Reload()

	panel := m.RenderDetailPanel(60, 20)
	if !strings.Contains(panel, "my-feature") {
		t.Errorf("detail panel missing session title, got:\n%s", panel)
	}
}

func TestDetailPanelShowsNotes(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["ad-s1"] = "> "

	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "my-feature", GroupPath: "my-sessions",
		TmuxSession: "ad-s1", ProjectPath: "/p", Tool: "claude",
		Status: "waiting", CreatedAt: 1000,
	})
	db.UpdateSessionNotes(conn, "s1", "check divergences first")

	m := ui.NewModel(conn, fake, nil)
	m.Reload()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	m.Reload()

	panel := m.RenderDetailPanel(60, 20)
	if !strings.Contains(panel, "check divergences first") {
		t.Errorf("detail panel missing notes, got:\n%s", panel)
	}
}

func TestDetailPanelShowsPaneList(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["ad-s1"] = "> "
	fake.Panes["ad-s1"] = []tmux.Pane{{Index: 0, Command: "claude"}, {Index: 1, Command: "bash"}}

	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "my-feature", GroupPath: "my-sessions",
		TmuxSession: "ad-s1", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, fake, nil)
	m.Reload()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	m.Reload()

	panel := m.RenderDetailPanel(60, 20)
	if !strings.Contains(panel, "[0] claude") {
		t.Errorf("detail panel missing pane list, got:\n%s", panel)
	}
	if !strings.Contains(panel, "[1] bash") {
		t.Errorf("detail panel missing pane [1], got:\n%s", panel)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```
go test ./internal/ui/... -run "TestDetailPanel" -v
```

Expected: compile error — `m.RenderDetailPanel` undefined.

- [ ] **Step 3: Implement RenderDetailPanel in app.go**

Add the following methods to `internal/ui/app.go`. Add them after the `ensureStarted` method:

```go
// RenderDetailPanel renders the right panel for the selected session.
// w = available width, h = available height (excluding footer and app-header rows).
func (m *Model) RenderDetailPanel(w, h int) string {
	if m.cursor >= len(m.items) || m.items[m.cursor].Kind != "session" {
		return ""
	}
	s := m.items[m.cursor].Session

	var lines []string

	// Section header
	lines = append(lines, sectionHeader("SESSION", w))

	// Session title + status
	sym := statusSymbol[s.Status]
	if sym == "" {
		sym = "—"
	}
	lines = append(lines, fmt.Sprintf(" %s  %s", s.Title, sym))

	// Group
	lines = append(lines, fmt.Sprintf(" group: %s", s.GroupPath))

	// Pane list
	paneStr := renderPaneList(m.panes)
	lines = append(lines, " "+paneStr)

	// OUTPUT section
	// notes section takes 4 lines: header + 3 text/hint
	// session header takes 4 lines: section-header + title+sym + group + panes
	const sessionHeaderLines = 4
	const notesLines = 4 // NOTES header + 3 lines
	outputH := h - sessionHeaderLines - notesLines - 1 // -1 for OUTPUT header line
	if outputH < 0 {
		outputH = 0
	}
	lines = append(lines, sectionHeader("OUTPUT", w))
	outputTail := tailLines(m.output, outputH)
	for _, ol := range outputTail {
		lines = append(lines, " "+truncate(ol, w-1))
	}
	// pad to fill outputH
	for i := len(outputTail); i < outputH; i++ {
		lines = append(lines, "")
	}

	// NOTES section
	lines = append(lines, sectionHeader("NOTES", w))
	var noteText string
	if s.Notes != "" {
		noteText = s.Notes
	} else {
		noteText = "No notes"
	}
	noteRunes := []rune(noteText)
	for row := 0; row < 3; row++ {
		start := row * (w - 1)
		if start >= len(noteRunes) {
			lines = append(lines, "")
			continue
		}
		end := start + (w - 1)
		if end > len(noteRunes) {
			end = len(noteRunes)
		}
		lines = append(lines, " "+string(noteRunes[start:end]))
	}
	if m.mode == "edit-notes" {
		lines = append(lines, " > "+m.dialog.value)
	} else {
		lines = append(lines, " e edit")
	}

	return strings.Join(lines, "\n")
}

func sectionHeader(title, width int) string {
	suffix := strings.Repeat("─", width-len(title)-2)
	return title + " " + suffix
}

func renderPaneList(panes []tmux.Pane) string {
	if len(panes) == 0 {
		return ""
	}
	var parts []string
	for i, p := range panes {
		if i == 0 {
			parts = append(parts, fmt.Sprintf("[%d] %s", p.Index, p.Command))
		} else {
			parts = append(parts, dimStyle.Render(fmt.Sprintf("[%d] %s", p.Index, p.Command)))
		}
	}
	return strings.Join(parts, "  ")
}

func tailLines(output string, n int) []string {
	if n <= 0 || output == "" {
		return nil
	}
	all := strings.Split(output, "\n")
	if len(all) > 0 && all[len(all)-1] == "" {
		all = all[:len(all)-1]
	}
	if len(all) <= n {
		return all
	}
	return all[len(all)-n:]
}
```

Fix the `sectionHeader` signature — `title` should be `string` not `int`:

```go
func sectionHeader(title string, width int) string {
	dashes := width - len([]rune(title)) - 2
	if dashes < 0 {
		dashes = 0
	}
	return title + " " + strings.Repeat("─", dashes)
}
```

- [ ] **Step 4: Run detail panel tests**

```
go test ./internal/ui/... -run "TestDetailPanel" -v
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
git commit -m "feat: add RenderDetailPanel with session header, output tail, and notes"
```

---

### Task 9: Split View() rendering

**Files:**
- Modify: `internal/ui/app.go`
- Test: `internal/ui/app_test.go`

- [ ] **Step 1: Write tests for split view**

Add to `internal/ui/app_test.go`:

```go
func TestViewRendersSplitLayout(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	m := ui.NewModel(conn, nil, nil)
	m.Reload()
	// Set terminal size
	m.Update(tea.WindowSizeMsg{Width: 120, Height: 30})

	view := m.View()
	// The split layout uses │ as divider
	if !strings.Contains(view, "│") {
		t.Errorf("split layout view should contain │ divider, got:\n%s", view)
	}
	if !strings.Contains(view, "SESSIONS") {
		t.Errorf("split layout view should contain SESSIONS header, got:\n%s", view)
	}
}

func TestViewFullScreenHidesLeftColumn(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["ad-s1"] = "output\n> "
	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "my-feature", GroupPath: "my-sessions",
		TmuxSession: "ad-s1", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, fake, nil)
	m.Reload()
	m.Update(tea.WindowSizeMsg{Width: 120, Height: 30})
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'v'}})
	m.Reload()

	view := m.View()
	if strings.Contains(view, "SESSIONS") {
		t.Errorf("full-screen view should not show SESSIONS column, got:\n%s", view)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```
go test ./internal/ui/... -run "TestViewRenders|TestViewFull" -v
```

Expected: FAIL — `View()` still renders the old single-column list.

- [ ] **Step 3: Implement split View() in app.go**

Replace the `View()` method in `internal/ui/app.go`:

```go
func (m *Model) View() string {
	if m.err != nil {
		return "error: " + m.err.Error()
	}

	leftW := int(float64(m.width) * 0.35)
	if leftW < 10 {
		leftW = 10
	}
	rightW := m.width - leftW - 1 // 1 for the │ divider
	if rightW < 10 {
		rightW = 10
	}

	// rows available for content: total height minus app-header(1) + separator(1) + footer(1)
	contentH := m.height - 3
	if contentH < 1 {
		contentH = 1
	}

	// Build app header (full width)
	header := m.renderAppHeader()

	if m.viewFull {
		detail := m.RenderDetailPanel(m.width, contentH)
		footer := renderFooter(m.width)
		return header + "\n" + strings.Repeat("─", m.width) + "\n" + detail + "\n" + footer
	}

	// Render left and right columns
	leftContent := RenderList(m.items, m.cursor, leftW, contentH)
	var rightContent string
	if m.mode == "edit-notes" {
		rightContent = m.RenderDetailPanel(rightW, contentH)
	} else if m.mode != "" {
		rightContent = m.renderDialog()
	} else {
		rightContent = m.RenderDetailPanel(rightW, contentH)
	}

	// Build separator line: left side ─ + ┬ + right side ─
	sep := strings.Repeat("─", leftW) + "┬" + strings.Repeat("─", rightW)

	// Merge left and right columns line-by-line with │ divider
	leftLines := strings.Split(leftContent, "\n")
	rightLines := strings.Split(rightContent, "\n")
	maxLines := contentH
	var merged []string
	for i := 0; i < maxLines; i++ {
		left := ""
		if i < len(leftLines) {
			left = leftLines[i]
		}
		right := ""
		if i < len(rightLines) {
			right = rightLines[i]
		}
		// Pad left to leftW visible chars (strip ANSI for measuring is complex;
		// pad based on rune count of raw string as an approximation)
		left = padRight(left, leftW)
		merged = append(merged, left+"│"+right)
	}

	footer := renderFooter(m.width)
	return header + "\n" + sep + "\n" + strings.Join(merged, "\n") + "\n" + footer
}

func (m *Model) renderAppHeader() string {
	var running, waiting, idle int
	for _, s := range m.sessions {
		switch s.Status {
		case "running":
			running++
		case "waiting":
			waiting++
		case "idle":
			idle++
		}
	}
	return fmt.Sprintf(" Agent Deck  ● %d running  ○ %d waiting  ◐ %d idle", running, waiting, idle)
}

func renderFooter(width int) string {
	return " Enter Attach  v Expand output  e Notes  n New  g Group  d Delete  q Quit"
}

// padRight pads s with spaces on the right to reach width visible characters.
// It uses rune count as a proxy for visible width (ANSI-free content only).
func padRight(s string, width int) string {
	runeLen := len([]rune(s))
	if runeLen >= width {
		return s
	}
	return s + strings.Repeat(" ", width-runeLen)
}
```

- [ ] **Step 4: Run split view tests**

```
go test ./internal/ui/... -run "TestViewRenders|TestViewFull" -v
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
git commit -m "feat: implement split panel View with app header, divider, and full-screen toggle"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|-----------------|------|
| Left column 35%, right 65% | Task 9 `View()` — `leftW = int(float64(m.width) * 0.35)` |
| Full-screen `v` toggle | Task 6 keys, Task 9 `viewFull` branch |
| Session header: title+status, group, panes | Task 8 `RenderDetailPanel` lines 2-4 |
| Output: tail of `CapturePaneOutput` | Task 5 `Reload` + Task 8 `tailLines` |
| Output fills between header and notes | Task 8 `outputH` calculation |
| Notes: 3 lines text + 1 hint | Task 8 notes rendering loop |
| `e` enters `edit-notes` mode | Task 6 + Task 7 |
| `edit-notes` pre-populated with current notes | Task 7 `dialogState{value: s.Notes}` |
| Enter saves, Esc discards | Task 7 `commitDialog` + existing `updateDialog` Esc |
| `notes TEXT NOT NULL DEFAULT ''` migration | Task 1 v2 migration |
| `UpdateSessionNotes` | Task 1 `sessions.go` |
| `Pane` struct + `ListPanes` on `ClientIface` + `Client` | Task 2 |
| `ListPanes` stub on `FakeTmuxClient` | Task 3 |
| `e` has no effect on groups | Task 6 guard `items[cursor].Kind == "session"` |
| `list.go` accepts width, truncates | Task 4 |

All spec requirements covered.

**Placeholder scan:** No TBDs, no "implement later", no references to undefined types. All code in steps references types defined in the same or earlier tasks.

**Type consistency:**
- `tmux.Pane` defined in Task 2, referenced in Task 3 (`FakeTmuxClient.Panes`), Task 5 (`m.panes []tmux.Pane`), Task 8 (`renderPaneList`)
- `db.UpdateSessionNotes` defined in Task 1, called in Task 7
- `RenderDetailPanel(w, h int) string` defined in Task 8, tested in Task 8 tests and called in Task 9 `View()`
- `RenderList(items, cursor, width, height int)` signature unchanged from Task 4
- `sectionHeader(title string, width int) string` defined in Task 8 and used in Task 8 only
- `tailLines`, `renderPaneList`, `padRight` defined in Task 8/9, used in same tasks
