# tmux-agent-deck MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a terminal UI for managing multiple AI coding agent sessions in tmux, organized into nested groups with SQLite state storage.

**Architecture:** cobra CLI launches either a bubbletea TUI (no args) or subcommands; SQLite at `~/.tmux-agent-deck/state.db` stores groups and sessions; a background poller reads tmux pane output every ~1s to update session status; the TUI reads DB state on a tick and re-renders.

**Tech Stack:** Go 1.22+, cobra, bubbletea + lipgloss + bubbles/textinput, modernc.org/sqlite (pure Go, no CGO), google/uuid

---

## File Map

```
tmux-agent-deck/
├── main.go                            # entrypoint — cobra Execute()
├── go.mod / go.sum
├── cmd/
│   ├── root.go                        # root command, openDB(), launchTUI(), RunWith()
│   ├── add.go                         # `add` subcommand
│   ├── list.go                        # `list` subcommand
│   ├── remove.go                      # `remove` subcommand
│   ├── session.go                     # `session start/stop/attach` subcommands
│   ├── group.go                       # `group create/delete/move` subcommands
│   └── cmd_test.go                    # integration tests via RunWith()
├── internal/
│   ├── db/
│   │   ├── db.go                      # Open(), migrate()
│   │   ├── db_test.go
│   │   ├── groups.go                  # Group type + CRUD functions
│   │   ├── groups_test.go
│   │   ├── sessions.go                # Session type + CRUD functions
│   │   └── sessions_test.go
│   ├── tmux/
│   │   ├── client.go                  # NewClient, NewSession, Attach, Kill, Capture, Exists
│   │   ├── status.go                  # DetectStatus() pure function
│   │   └── status_test.go
│   ├── state/
│   │   ├── poller.go                  # Poller: Start/Stop/PollOnce, TmuxReader interface
│   │   └── poller_test.go
│   ├── ui/
│   │   ├── app.go                     # bubbletea Model, Init/Update/View, Reload()
│   │   ├── app_test.go
│   │   ├── list.go                    # ListItem type, BuildTree(), RenderList()
│   │   ├── list_test.go
│   │   ├── dialog.go                  # dialogState, updateDialog(), commitDialog()
│   │   └── keys.go                    # actionForKey() mapping
│   └── testutil/
│       └── db.go                      # OpenTestDB(t) helper
```

---

### Task 1: Go module scaffold

**Files:**
- Create: `go.mod`
- Create: `main.go`
- Create: `cmd/root.go`

- [ ] **Step 1: Initialize the Go module**

```bash
cd ~/Projects/tmux-agent-deck
go mod init github.com/black-gato/tmux-agent-deck
```

Expected: `go.mod` created with `module github.com/black-gato/tmux-agent-deck` and `go 1.22`

- [ ] **Step 2: Add dependencies**

```bash
cd ~/Projects/tmux-agent-deck
go get github.com/spf13/cobra@latest
go get github.com/charmbracelet/bubbletea@latest
go get github.com/charmbracelet/lipgloss@latest
go get github.com/charmbracelet/bubbles@latest
go get modernc.org/sqlite@latest
go get github.com/google/uuid@latest
```

- [ ] **Step 3: Write cmd/root.go**

```go
package cmd

import (
	"github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
	Use:   "tmux-agent-deck",
	Short: "Manage AI coding agent sessions in tmux",
}

func Execute() error {
	return rootCmd.Execute()
}
```

- [ ] **Step 4: Write main.go**

```go
package main

import (
	"fmt"
	"os"

	"github.com/black-gato/tmux-agent-deck/cmd"
)

func main() {
	if err := cmd.Execute(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

- [ ] **Step 5: Verify it builds**

```bash
cd ~/Projects/tmux-agent-deck
go build ./...
```

Expected: no errors

- [ ] **Step 6: Commit**

```bash
git -C ~/Projects/tmux-agent-deck add go.mod go.sum main.go cmd/root.go
git -C ~/Projects/tmux-agent-deck commit -m "feat: scaffold Go module and cobra root command"
```

---

### Task 2: DB — open and migrate

**Files:**
- Create: `internal/db/db.go`
- Create: `internal/db/db_test.go`
- Create: `internal/testutil/db.go`

- [ ] **Step 1: Write the test helper**

```go
// internal/testutil/db.go
package testutil

import (
	"database/sql"
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/db"
)

func OpenTestDB(t *testing.T) *sql.DB {
	t.Helper()
	conn, err := db.Open(":memory:")
	if err != nil {
		t.Fatalf("open test db: %v", err)
	}
	t.Cleanup(func() { conn.Close() })
	return conn
}
```

- [ ] **Step 2: Write failing test**

```go
// internal/db/db_test.go
package db_test

import (
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/testutil"
)

func TestOpenCreatesSchema(t *testing.T) {
	conn := testutil.OpenTestDB(t)

	for _, table := range []string{"groups", "sessions", "metadata"} {
		var n int
		err := conn.QueryRow(
			`SELECT count(*) FROM sqlite_master WHERE type='table' AND name=?`, table,
		).Scan(&n)
		if err != nil {
			t.Fatalf("check table %q: %v", table, err)
		}
		if n != 1 {
			t.Errorf("table %q not created", table)
		}
	}
}

func TestOpenCreatesMySessions(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	var n int
	conn.QueryRow(`SELECT count(*) FROM groups WHERE path='my-sessions'`).Scan(&n)
	if n != 1 {
		t.Errorf("my-sessions group not seeded")
	}
}
```

- [ ] **Step 3: Run tests — expect failure**

```bash
cd ~/Projects/tmux-agent-deck
go test ./internal/db/... -run TestOpen -v
```

Expected: FAIL — `db` package not found

- [ ] **Step 4: Write internal/db/db.go**

```go
package db

import (
	"database/sql"
	"fmt"

	_ "modernc.org/sqlite"
)

func Open(path string) (*sql.DB, error) {
	conn, err := sql.Open("sqlite", path)
	if err != nil {
		return nil, fmt.Errorf("open sqlite: %w", err)
	}
	if err := migrate(conn); err != nil {
		conn.Close()
		return nil, fmt.Errorf("migrate: %w", err)
	}
	return conn, nil
}

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
	return err
}
```

- [ ] **Step 5: Run tests — expect pass**

```bash
go test ./internal/db/... -run TestOpen -v
```

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C ~/Projects/tmux-agent-deck add internal/db/db.go internal/db/db_test.go internal/testutil/db.go
git -C ~/Projects/tmux-agent-deck commit -m "feat: add SQLite open and schema migration"
```

---

### Task 3: DB — Groups CRUD

**Files:**
- Create: `internal/db/groups.go`
- Create: `internal/db/groups_test.go`

- [ ] **Step 1: Write failing tests**

```go
// internal/db/groups_test.go
package db_test

import (
	"testing"

	dbpkg "github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/testutil"
)

func TestGroupCreateAndGet(t *testing.T) {
	conn := testutil.OpenTestDB(t)

	g := dbpkg.Group{
		Path:        "work/frontend",
		Name:        "frontend",
		DefaultPath: "/home/user/projects",
		DefaultTool: "claude",
		Expanded:    true,
	}
	if err := dbpkg.CreateGroup(conn, g); err != nil {
		t.Fatalf("create: %v", err)
	}

	got, err := dbpkg.GetGroup(conn, "work/frontend")
	if err != nil {
		t.Fatalf("get: %v", err)
	}
	if got.Name != "frontend" {
		t.Errorf("name: got %q want %q", got.Name, "frontend")
	}
	if got.DefaultTool != "claude" {
		t.Errorf("tool: got %q want %q", got.DefaultTool, "claude")
	}
}

func TestListGroups(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	// my-sessions already created by migration
	dbpkg.CreateGroup(conn, dbpkg.Group{Path: "work", Name: "work"})
	dbpkg.CreateGroup(conn, dbpkg.Group{Path: "work/frontend", Name: "frontend"})

	groups, err := dbpkg.ListGroups(conn)
	if err != nil {
		t.Fatal(err)
	}
	if len(groups) < 3 {
		t.Errorf("expected at least 3 groups, got %d", len(groups))
	}
}

func TestChildGroups(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	dbpkg.CreateGroup(conn, dbpkg.Group{Path: "work", Name: "work"})
	dbpkg.CreateGroup(conn, dbpkg.Group{Path: "work/frontend", Name: "frontend"})
	dbpkg.CreateGroup(conn, dbpkg.Group{Path: "work/backend", Name: "backend"})
	dbpkg.CreateGroup(conn, dbpkg.Group{Path: "personal", Name: "personal"})

	children, err := dbpkg.ChildGroups(conn, "work")
	if err != nil {
		t.Fatal(err)
	}
	if len(children) != 2 {
		t.Errorf("expected 2 children, got %d", len(children))
	}
}

func TestUpdateGroupExpanded(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	dbpkg.CreateGroup(conn, dbpkg.Group{Path: "work", Name: "work", Expanded: true})

	if err := dbpkg.SetGroupExpanded(conn, "work", false); err != nil {
		t.Fatal(err)
	}
	g, _ := dbpkg.GetGroup(conn, "work")
	if g.Expanded {
		t.Errorf("expected expanded=false")
	}
}

func TestDeleteGroup(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	dbpkg.CreateGroup(conn, dbpkg.Group{Path: "work", Name: "work"})
	if err := dbpkg.DeleteGroup(conn, "work"); err != nil {
		t.Fatal(err)
	}
	_, err := dbpkg.GetGroup(conn, "work")
	if err == nil {
		t.Errorf("expected error after delete")
	}
}
```

- [ ] **Step 2: Run tests — expect failure**

```bash
go test ./internal/db/... -run TestGroup -v
```

Expected: compile error — Group type and functions not defined

- [ ] **Step 3: Write internal/db/groups.go**

```go
package db

import (
	"database/sql"
	"fmt"
)

type Group struct {
	Path        string
	Name        string
	DefaultPath string
	DefaultTool string
	Expanded    bool
	SortOrder   int
}

func CreateGroup(conn *sql.DB, g Group) error {
	expanded := 0
	if g.Expanded {
		expanded = 1
	}
	_, err := conn.Exec(
		`INSERT INTO groups (path, name, default_path, default_tool, expanded, sort_order)
		 VALUES (?, ?, ?, ?, ?, ?)`,
		g.Path, g.Name, g.DefaultPath, g.DefaultTool, expanded, g.SortOrder,
	)
	return err
}

func GetGroup(conn *sql.DB, path string) (Group, error) {
	var g Group
	var expanded int
	err := conn.QueryRow(
		`SELECT path, name, default_path, default_tool, expanded, sort_order
		 FROM groups WHERE path = ?`, path,
	).Scan(&g.Path, &g.Name, &g.DefaultPath, &g.DefaultTool, &expanded, &g.SortOrder)
	if err != nil {
		return Group{}, fmt.Errorf("get group %q: %w", path, err)
	}
	g.Expanded = expanded == 1
	return g, nil
}

func ListGroups(conn *sql.DB) ([]Group, error) {
	rows, err := conn.Query(
		`SELECT path, name, default_path, default_tool, expanded, sort_order
		 FROM groups ORDER BY sort_order, path`,
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	return scanGroups(rows)
}

// ChildGroups returns direct children of parentPath (one level deep only).
func ChildGroups(conn *sql.DB, parentPath string) ([]Group, error) {
	prefix := parentPath + "/%"
	deeperPrefix := parentPath + "/%/%"
	rows, err := conn.Query(
		`SELECT path, name, default_path, default_tool, expanded, sort_order
		 FROM groups WHERE path LIKE ? AND path NOT LIKE ?
		 ORDER BY sort_order, path`,
		prefix, deeperPrefix,
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	return scanGroups(rows)
}

func SetGroupExpanded(conn *sql.DB, path string, expanded bool) error {
	v := 0
	if expanded {
		v = 1
	}
	_, err := conn.Exec(`UPDATE groups SET expanded = ? WHERE path = ?`, v, path)
	return err
}

func RenameGroup(conn *sql.DB, path, newName string) error {
	_, err := conn.Exec(`UPDATE groups SET name = ? WHERE path = ?`, newName, path)
	return err
}

func DeleteGroup(conn *sql.DB, path string) error {
	_, err := conn.Exec(`DELETE FROM groups WHERE path = ?`, path)
	return err
}

func scanGroups(rows *sql.Rows) ([]Group, error) {
	var groups []Group
	for rows.Next() {
		var g Group
		var expanded int
		if err := rows.Scan(&g.Path, &g.Name, &g.DefaultPath, &g.DefaultTool, &expanded, &g.SortOrder); err != nil {
			return nil, err
		}
		g.Expanded = expanded == 1
		groups = append(groups, g)
	}
	return groups, rows.Err()
}
```

- [ ] **Step 4: Run tests — expect pass**

```bash
go test ./internal/db/... -run TestGroup -v
```

Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C ~/Projects/tmux-agent-deck add internal/db/groups.go internal/db/groups_test.go
git -C ~/Projects/tmux-agent-deck commit -m "feat: add groups CRUD"
```

---

### Task 4: DB — Sessions CRUD

**Files:**
- Create: `internal/db/sessions.go`
- Create: `internal/db/sessions_test.go`

- [ ] **Step 1: Write failing tests**

```go
// internal/db/sessions_test.go
package db_test

import (
	"testing"
	"time"

	dbpkg "github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/testutil"
)

func TestSessionCreateAndGet(t *testing.T) {
	conn := testutil.OpenTestDB(t)

	s := dbpkg.Session{
		ID:          "abc-123",
		Title:       "my-app",
		GroupPath:   "my-sessions",
		TmuxSession: "tmux-abc-123",
		ProjectPath: "/home/user/projects/my-app",
		Tool:        "claude",
		Status:      "stopped",
		CreatedAt:   time.Now().Unix(),
	}
	if err := dbpkg.CreateSession(conn, s); err != nil {
		t.Fatalf("create: %v", err)
	}

	got, err := dbpkg.GetSession(conn, "abc-123")
	if err != nil {
		t.Fatalf("get: %v", err)
	}
	if got.Title != "my-app" {
		t.Errorf("title: got %q want %q", got.Title, "my-app")
	}
	if got.ProjectPath != "/home/user/projects/my-app" {
		t.Errorf("project_path mismatch")
	}
}

func TestListSessionsByGroup(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	dbpkg.CreateSession(conn, dbpkg.Session{ID: "s1", Title: "a", GroupPath: "my-sessions", ProjectPath: "/p", Tool: "claude", Status: "stopped", CreatedAt: now})
	dbpkg.CreateSession(conn, dbpkg.Session{ID: "s2", Title: "b", GroupPath: "my-sessions", ProjectPath: "/p", Tool: "claude", Status: "stopped", CreatedAt: now})
	dbpkg.CreateSession(conn, dbpkg.Session{ID: "s3", Title: "c", GroupPath: "work", ProjectPath: "/p", Tool: "claude", Status: "stopped", CreatedAt: now})

	sessions, err := dbpkg.ListSessionsByGroup(conn, "my-sessions")
	if err != nil {
		t.Fatal(err)
	}
	if len(sessions) != 2 {
		t.Errorf("expected 2, got %d", len(sessions))
	}
}

func TestUpdateSessionStatus(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	dbpkg.CreateSession(conn, dbpkg.Session{ID: "s1", Title: "a", GroupPath: "my-sessions", ProjectPath: "/p", Tool: "claude", Status: "stopped", CreatedAt: now})

	if err := dbpkg.UpdateSessionStatus(conn, "s1", "running"); err != nil {
		t.Fatal(err)
	}
	s, _ := dbpkg.GetSession(conn, "s1")
	if s.Status != "running" {
		t.Errorf("status: got %q want running", s.Status)
	}
}

func TestMoveSession(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	dbpkg.CreateGroup(conn, dbpkg.Group{Path: "work", Name: "work"})
	dbpkg.CreateSession(conn, dbpkg.Session{ID: "s1", Title: "a", GroupPath: "my-sessions", ProjectPath: "/p", Tool: "claude", Status: "stopped", CreatedAt: now})

	if err := dbpkg.MoveSession(conn, "s1", "work"); err != nil {
		t.Fatal(err)
	}
	s, _ := dbpkg.GetSession(conn, "s1")
	if s.GroupPath != "work" {
		t.Errorf("group_path: got %q want work", s.GroupPath)
	}
}

func TestDeleteSession(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	dbpkg.CreateSession(conn, dbpkg.Session{ID: "s1", Title: "a", GroupPath: "my-sessions", ProjectPath: "/p", Tool: "claude", Status: "stopped", CreatedAt: now})

	if err := dbpkg.DeleteSession(conn, "s1"); err != nil {
		t.Fatal(err)
	}
	_, err := dbpkg.GetSession(conn, "s1")
	if err == nil {
		t.Errorf("expected error after delete")
	}
}
```

- [ ] **Step 2: Run tests — expect failure**

```bash
go test ./internal/db/... -run TestSession -v
```

Expected: compile error — Session type and functions not defined

- [ ] **Step 3: Write internal/db/sessions.go**

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
		`SELECT id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active
		 FROM sessions WHERE id = ?`, id,
	).Scan(&s.ID, &s.Title, &s.GroupPath, &s.TmuxSession, &s.ProjectPath, &s.Tool, &s.Status, &s.CreatedAt, &s.LastActive)
	if err != nil {
		return Session{}, fmt.Errorf("get session %q: %w", id, err)
	}
	return s, nil
}

func GetSessionByTitle(conn *sql.DB, title string) (Session, error) {
	var s Session
	err := conn.QueryRow(
		`SELECT id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active
		 FROM sessions WHERE title = ? LIMIT 1`, title,
	).Scan(&s.ID, &s.Title, &s.GroupPath, &s.TmuxSession, &s.ProjectPath, &s.Tool, &s.Status, &s.CreatedAt, &s.LastActive)
	if err != nil {
		return Session{}, fmt.Errorf("get session by title %q: %w", title, err)
	}
	return s, nil
}

func ListSessions(conn *sql.DB) ([]Session, error) {
	rows, err := conn.Query(
		`SELECT id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active
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
		`SELECT id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active
		 FROM sessions WHERE group_path = ? ORDER BY created_at DESC`, groupPath,
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	return scanSessions(rows)
}

func UpdateSessionStatus(conn *sql.DB, id, status string) error {
	_, err := conn.Exec(
		`UPDATE sessions SET status = ?, last_active = strftime('%s','now') WHERE id = ?`,
		status, id,
	)
	return err
}

func UpdateSessionTmuxName(conn *sql.DB, id, tmuxSession string) error {
	_, err := conn.Exec(`UPDATE sessions SET tmux_session = ? WHERE id = ?`, tmuxSession, id)
	return err
}

func RenameSession(conn *sql.DB, id, newTitle string) error {
	_, err := conn.Exec(`UPDATE sessions SET title = ? WHERE id = ?`, newTitle, id)
	return err
}

func MoveSession(conn *sql.DB, id, groupPath string) error {
	_, err := conn.Exec(`UPDATE sessions SET group_path = ? WHERE id = ?`, groupPath, id)
	return err
}

func DeleteSession(conn *sql.DB, id string) error {
	_, err := conn.Exec(`DELETE FROM sessions WHERE id = ?`, id)
	return err
}

func scanSessions(rows *sql.Rows) ([]Session, error) {
	var sessions []Session
	for rows.Next() {
		var s Session
		if err := rows.Scan(&s.ID, &s.Title, &s.GroupPath, &s.TmuxSession, &s.ProjectPath, &s.Tool, &s.Status, &s.CreatedAt, &s.LastActive); err != nil {
			return nil, err
		}
		sessions = append(sessions, s)
	}
	return sessions, rows.Err()
}
```

- [ ] **Step 4: Run tests — expect pass**

```bash
go test ./internal/db/... -v
```

Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C ~/Projects/tmux-agent-deck add internal/db/sessions.go internal/db/sessions_test.go
git -C ~/Projects/tmux-agent-deck commit -m "feat: add sessions CRUD"
```

---

### Task 5: tmux — state detection (pure function)

**Files:**
- Create: `internal/tmux/status.go`
- Create: `internal/tmux/status_test.go`

State detection is a pure function: given pane output text and the time of last change, return a status string. Fully testable without a live tmux session.

- [ ] **Step 1: Write failing tests**

```go
// internal/tmux/status_test.go
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
		status := tmux.DetectStatus(output, time.Now())
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
		status := tmux.DetectStatus(output, time.Now())
		if status != tmux.StatusRunning {
			t.Errorf("output %q: got %q want running", output, status)
		}
	}
}

func TestDetectStatusIdle(t *testing.T) {
	// No prompt, no spinner, no activity for >30s
	output := "Some old output without a prompt"
	lastChange := time.Now().Add(-31 * time.Second)
	status := tmux.DetectStatus(output, lastChange)
	if status != tmux.StatusIdle {
		t.Errorf("got %q want idle", status)
	}
}

func TestDetectStatusRecentActivityIsRunning(t *testing.T) {
	output := "Some output without a prompt"
	status := tmux.DetectStatus(output, time.Now())
	if status != tmux.StatusRunning {
		t.Errorf("got %q want running (recent activity)", status)
	}
}
```

- [ ] **Step 2: Run tests — expect failure**

```bash
go test ./internal/tmux/... -run TestDetectStatus -v
```

Expected: compile error — package not found

- [ ] **Step 3: Write internal/tmux/status.go**

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

func DetectStatus(output string, lastChange time.Time) Status {
	trimmed := strings.TrimRight(output, " \t")

	// waiting: Claude prompt visible at end of pane
	if strings.HasSuffix(trimmed, "> ") || strings.HasSuffix(trimmed, ">") {
		return StatusWaiting
	}

	// running: spinner or thinking text in tail of output
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

	// idle: nothing recognizable, and no change for >30s
	if time.Since(lastChange) > 30*time.Second {
		return StatusIdle
	}

	return StatusRunning
}
```

- [ ] **Step 4: Run tests — expect pass**

```bash
go test ./internal/tmux/... -run TestDetectStatus -v
```

Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C ~/Projects/tmux-agent-deck add internal/tmux/status.go internal/tmux/status_test.go
git -C ~/Projects/tmux-agent-deck commit -m "feat: add tmux status detection from pane output"
```

---

### Task 6: tmux — client (exec-based)

**Files:**
- Create: `internal/tmux/client.go`

The client wraps `tmux` CLI calls via `os/exec`. These require a live tmux daemon to test, so we verify compilation only here; integration is exercised via the CLI in Task 11.

- [ ] **Step 1: Write internal/tmux/client.go**

```go
package tmux

import (
	"bytes"
	"fmt"
	"os/exec"
	"strings"
)

type Client struct{}

func NewClient() *Client {
	return &Client{}
}

func (c *Client) NewSession(name, startDir, command string) error {
	args := []string{"new-session", "-d", "-s", name, "-c", startDir}
	if command != "" {
		args = append(args, command)
	}
	return runCmd("tmux", args...)
}

func (c *Client) AttachSession(name string) error {
	return runCmd("tmux", "attach-session", "-t", name)
}

func (c *Client) KillSession(name string) error {
	return runCmd("tmux", "kill-session", "-t", name)
}

func (c *Client) SessionExists(name string) (bool, error) {
	err := runCmd("tmux", "has-session", "-t", name)
	if err == nil {
		return true, nil
	}
	if exitErr, ok := err.(*exec.ExitError); ok && exitErr.ExitCode() == 1 {
		return false, nil
	}
	return false, err
}

func (c *Client) CapturePaneOutput(name string) (string, error) {
	out, err := cmdOutput("tmux", "capture-pane", "-t", name, "-p")
	if err != nil {
		return "", fmt.Errorf("capture-pane %q: %w", name, err)
	}
	return string(out), nil
}

func (c *Client) ListSessions() ([]string, error) {
	out, err := cmdOutput("tmux", "list-sessions", "-F", "#{session_name}")
	if err != nil {
		// tmux exits 1 when there are no sessions — treat as empty
		return nil, nil
	}
	var names []string
	for _, line := range strings.Split(strings.TrimSpace(string(out)), "\n") {
		if line != "" {
			names = append(names, line)
		}
	}
	return names, nil
}

func runCmd(name string, args ...string) error {
	return exec.Command(name, args...).Run()
}

func cmdOutput(name string, args ...string) ([]byte, error) {
	var buf bytes.Buffer
	cmd := exec.Command(name, args...)
	cmd.Stdout = &buf
	if err := cmd.Run(); err != nil {
		return nil, err
	}
	return buf.Bytes(), nil
}
```

- [ ] **Step 2: Verify it compiles**

```bash
cd ~/Projects/tmux-agent-deck && go build ./internal/tmux/...
```

Expected: no errors

- [ ] **Step 3: Commit**

```bash
git -C ~/Projects/tmux-agent-deck add internal/tmux/client.go
git -C ~/Projects/tmux-agent-deck commit -m "feat: add tmux client (exec-based)"
```

---

### Task 7: State poller

**Files:**
- Create: `internal/state/poller.go`
- Create: `internal/state/poller_test.go`

- [ ] **Step 1: Write failing test**

```go
// internal/state/poller_test.go
package state_test

import (
	"testing"
	"time"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/state"
	"github.com/black-gato/tmux-agent-deck/internal/testutil"
)

type stubTmux struct {
	output string
	exists bool
}

func (s *stubTmux) CapturePaneOutput(name string) (string, error) { return s.output, nil }
func (s *stubTmux) SessionExists(name string) (bool, error)       { return s.exists, nil }

func TestPollerUpdatesStatusToWaiting(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	db.CreateSession(conn, db.Session{
		ID:          "s1",
		Title:       "test",
		GroupPath:   "my-sessions",
		TmuxSession: "tmux-s1",
		ProjectPath: "/p",
		Tool:        "claude",
		Status:      "running",
		CreatedAt:   now,
	})

	stub := &stubTmux{output: "Some output\n> ", exists: true}
	p := state.New(conn, stub)
	p.PollOnce()

	s, _ := db.GetSession(conn, "s1")
	if s.Status != "waiting" {
		t.Errorf("status: got %q want waiting", s.Status)
	}
}

func TestPollerMarksErrorWhenSessionGone(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	db.CreateSession(conn, db.Session{
		ID:          "s2",
		Title:       "gone",
		GroupPath:   "my-sessions",
		TmuxSession: "tmux-s2",
		ProjectPath: "/p",
		Tool:        "claude",
		Status:      "running",
		CreatedAt:   now,
	})

	stub := &stubTmux{exists: false}
	p := state.New(conn, stub)
	p.PollOnce()

	s, _ := db.GetSession(conn, "s2")
	if s.Status != "error" {
		t.Errorf("status: got %q want error", s.Status)
	}
}

func TestPollerSkipsStoppedSessions(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	db.CreateSession(conn, db.Session{
		ID:          "s3",
		Title:       "stopped-one",
		GroupPath:   "my-sessions",
		TmuxSession: "tmux-s3",
		ProjectPath: "/p",
		Tool:        "claude",
		Status:      "stopped",
		CreatedAt:   now,
	})

	callCount := 0
	stub := &countingStub{callCount: &callCount}
	p := state.New(conn, stub)
	p.PollOnce()

	if callCount > 0 {
		t.Errorf("CapturePaneOutput called for stopped session")
	}
}

type countingStub struct{ callCount *int }

func (s *countingStub) CapturePaneOutput(name string) (string, error) {
	*s.callCount++
	return "", nil
}
func (s *countingStub) SessionExists(name string) (bool, error) { return true, nil }
```

- [ ] **Step 2: Run test — expect failure**

```bash
go test ./internal/state/... -v
```

Expected: compile error

- [ ] **Step 3: Write internal/state/poller.go**

```go
package state

import (
	"database/sql"
	"log"
	"time"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/tmux"
)

// TmuxReader is the subset of tmux.Client used by the poller.
// Defined here so tests can stub it without importing the real client.
type TmuxReader interface {
	CapturePaneOutput(name string) (string, error)
	SessionExists(name string) (bool, error)
}

type Poller struct {
	conn       *sql.DB
	tmux       TmuxReader
	lastChange map[string]time.Time
	done       chan struct{}
}

func New(conn *sql.DB, tc TmuxReader) *Poller {
	return &Poller{
		conn:       conn,
		tmux:       tc,
		lastChange: make(map[string]time.Time),
		done:       make(chan struct{}),
	}
}

func (p *Poller) Start() {
	go func() {
		ticker := time.NewTicker(time.Second)
		defer ticker.Stop()
		for {
			select {
			case <-ticker.C:
				p.PollOnce()
			case <-p.done:
				return
			}
		}
	}()
}

func (p *Poller) Stop() {
	close(p.done)
}

func (p *Poller) PollOnce() {
	sessions, err := db.ListSessions(p.conn)
	if err != nil {
		log.Printf("poller: list sessions: %v", err)
		return
	}
	for _, s := range sessions {
		if s.Status == tmux.StatusStopped || s.TmuxSession == "" {
			continue
		}
		exists, err := p.tmux.SessionExists(s.TmuxSession)
		if err != nil {
			continue
		}
		if !exists {
			db.UpdateSessionStatus(p.conn, s.ID, tmux.StatusError)
			delete(p.lastChange, s.ID)
			continue
		}
		out, err := p.tmux.CapturePaneOutput(s.TmuxSession)
		if err != nil {
			continue
		}

		lc, ok := p.lastChange[s.ID]
		if !ok {
			lc = time.Now()
			p.lastChange[s.ID] = lc
		}

		newStatus := tmux.DetectStatus(out, lc)
		if newStatus != s.Status {
			p.lastChange[s.ID] = time.Now()
			db.UpdateSessionStatus(p.conn, s.ID, newStatus)
		}
	}
}
```

- [ ] **Step 4: Run tests — expect pass**

```bash
go test ./internal/state/... -v
```

Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C ~/Projects/tmux-agent-deck add internal/state/poller.go internal/state/poller_test.go
git -C ~/Projects/tmux-agent-deck commit -m "feat: add state poller with tmux status detection"
```

---

### Task 8: TUI — keys, model skeleton, and dialog stub

**Files:**
- Create: `internal/ui/keys.go`
- Create: `internal/ui/app.go`
- Create: `internal/ui/dialog.go` (stub — completed in Task 10)
- Create: `internal/ui/list.go` (stub — completed in Task 9)
- Create: `internal/ui/app_test.go`

- [ ] **Step 1: Write failing model tests**

```go
// internal/ui/app_test.go
package ui_test

import (
	"testing"

	tea "github.com/charmbracelet/bubbletea"
	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/ui"
	"github.com/black-gato/tmux-agent-deck/internal/testutil"
)

func TestModelInitializesWithGroups(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	m := ui.NewModel(conn, nil, nil)
	if err := m.Reload(); err != nil {
		t.Fatalf("reload: %v", err)
	}
	if len(m.Items()) == 0 {
		t.Errorf("expected at least one item (my-sessions group)")
	}
}

func TestModelNavigateDown(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", Expanded: true})
	m := ui.NewModel(conn, nil, nil)
	m.Reload()

	before := m.Cursor()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	after := m.Cursor()
	if before == after && len(m.Items()) > 1 {
		t.Errorf("cursor did not advance on 'j'")
	}
}

func TestModelNavigateUp(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", Expanded: true})
	m := ui.NewModel(conn, nil, nil)
	m.Reload()
	// Move to last item, then back up
	for i := 0; i < len(m.Items()); i++ {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'j'}})
	}
	before := m.Cursor()
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'k'}})
	after := m.Cursor()
	if after >= before {
		t.Errorf("cursor did not decrease on 'k': before=%d after=%d", before, after)
	}
}

func TestModelOpenNewSessionDialog(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	m := ui.NewModel(conn, nil, nil)
	m.Reload()

	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'n'}})
	if m.Mode() != "new-session" {
		t.Errorf("expected mode new-session, got %q", m.Mode())
	}
}

func TestModelEscClosesDialog(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	m := ui.NewModel(conn, nil, nil)
	m.Reload()

	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'n'}})
	m.Update(tea.KeyMsg{Type: tea.KeyEsc})
	if m.Mode() != "" {
		t.Errorf("expected mode empty after Esc, got %q", m.Mode())
	}
}
```

- [ ] **Step 2: Run tests — expect failure**

```bash
go test ./internal/ui/... -run TestModel -v
```

Expected: compile error — ui package not found

- [ ] **Step 3: Write internal/ui/keys.go**

```go
package ui

import tea "github.com/charmbracelet/bubbletea"

var keyTypeMap = map[tea.KeyType]string{
	tea.KeyUp:    "up",
	tea.KeyDown:  "down",
	tea.KeyEnter: "attach",
	tea.KeySpace: "toggle",
	tea.KeyEsc:   "esc",
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

- [ ] **Step 4: Write internal/ui/list.go stub** (full version in Task 9)

```go
package ui

import (
	"github.com/black-gato/tmux-agent-deck/internal/db"
)

type ListItem struct {
	Kind    string // "group" or "session"
	Group   *db.Group
	Session *db.Session
	Depth   int
}

func BuildTree(groups []db.Group, sessions []db.Session) []ListItem {
	return nil // stub — implemented in Task 9
}

func RenderList(items []ListItem, cursor, width, height int) string {
	return "loading..." // stub — implemented in Task 9
}
```

- [ ] **Step 5: Write internal/ui/dialog.go stub** (full version in Task 10)

```go
package ui

import (
	"strings"

	tea "github.com/charmbracelet/bubbletea"
)

type dialogState struct {
	prompt string
	value  string
}

func newDialogState(prompt string) dialogState {
	return dialogState{prompt: prompt}
}

func (m *Model) updateDialog(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	switch msg.Type {
	case tea.KeyEsc:
		m.mode = ""
	case tea.KeyEnter:
		m.commitDialog()
		m.mode = ""
		m.Reload()
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
	return m.dialog.prompt + "\n> " + m.dialog.value
}

func (m *Model) commitDialog() {
	// stub — implemented in Task 10
	_ = strings.TrimSpace(m.dialog.value)
}
```

- [ ] **Step 6: Write internal/ui/app.go**

```go
package ui

import (
	"database/sql"

	tea "github.com/charmbracelet/bubbletea"
	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/state"
	"github.com/black-gato/tmux-agent-deck/internal/tmux"
)

type tickMsg struct{}

type Model struct {
	conn     *sql.DB
	tmuxC    *tmux.Client
	poller   *state.Poller
	groups   []db.Group
	sessions []db.Session
	items    []ListItem
	cursor   int
	width    int
	height   int
	mode     string // "", "new-session", "new-group", "rename", "move"
	dialog   dialogState
}

func NewModel(conn *sql.DB, tc *tmux.Client, poller *state.Poller) *Model {
	return &Model{conn: conn, tmuxC: tc, poller: poller}
}

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
	return nil
}

func (m *Model) Items() []ListItem { return m.items }
func (m *Model) Cursor() int       { return m.cursor }
func (m *Model) Mode() string      { return m.mode }

func (m *Model) Init() tea.Cmd {
	m.Reload()
	return tick()
}

func (m *Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tickMsg:
		m.Reload()
		return m, tick()
	case tea.WindowSizeMsg:
		m.width = msg.Width
		m.height = msg.Height
		return m, nil
	case tea.KeyMsg:
		if m.mode != "" {
			return m.updateDialog(msg)
		}
		return m.updateNavigation(msg)
	}
	return m, nil
}

func (m *Model) updateNavigation(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	action := actionForKey(msg)
	switch action {
	case "down":
		if m.cursor < len(m.items)-1 {
			m.cursor++
		}
	case "up":
		if m.cursor > 0 {
			m.cursor--
		}
	case "toggle":
		if m.cursor < len(m.items) && m.items[m.cursor].Kind == "group" {
			g := m.items[m.cursor].Group
			db.SetGroupExpanded(m.conn, g.Path, !g.Expanded)
			m.Reload()
		}
	case "attach":
		if m.cursor < len(m.items) {
			item := m.items[m.cursor]
			if item.Kind == "session" && m.tmuxC != nil && item.Session.TmuxSession != "" {
				m.tmuxC.AttachSession(item.Session.TmuxSession)
			}
		}
	case "new-session":
		m.mode = "new-session"
		m.dialog = newDialogState("Session title:")
	case "new-group":
		m.mode = "new-group"
		m.dialog = newDialogState("Group path (e.g. work/frontend):")
	case "rename":
		if m.cursor < len(m.items) {
			m.mode = "rename"
			m.dialog = newDialogState("New name:")
		}
	case "move":
		if m.cursor < len(m.items) && m.items[m.cursor].Kind == "session" {
			m.mode = "move"
			m.dialog = newDialogState("Move to group path:")
		}
	case "delete":
		if m.cursor < len(m.items) {
			item := m.items[m.cursor]
			if item.Kind == "session" {
				db.DeleteSession(m.conn, item.Session.ID)
			} else if item.Kind == "group" && item.Group.Path != "my-sessions" {
				db.DeleteGroup(m.conn, item.Group.Path)
			}
			m.Reload()
		}
	case "quit":
		if m.poller != nil {
			m.poller.Stop()
		}
		return m, tea.Quit
	}
	return m, nil
}

func (m *Model) View() string {
	if m.mode != "" {
		return m.renderDialog()
	}
	return RenderList(m.items, m.cursor, m.width, m.height)
}

func tick() tea.Cmd {
	return func() tea.Msg { return tickMsg{} }
}
```

- [ ] **Step 7: Run tests — expect pass**

```bash
go test ./internal/ui/... -run TestModel -v
```

Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git -C ~/Projects/tmux-agent-deck add internal/ui/keys.go internal/ui/app.go internal/ui/app_test.go internal/ui/dialog.go internal/ui/list.go
git -C ~/Projects/tmux-agent-deck commit -m "feat: add TUI model, navigation, and dialog/list stubs"
```

---

### Task 9: TUI — list rendering

**Files:**
- Modify: `internal/ui/list.go`
- Create: `internal/ui/list_test.go`

- [ ] **Step 1: Write failing tests**

```go
// internal/ui/list_test.go
package ui_test

import (
	"strings"
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/ui"
)

func TestBuildTreeFlattensGroups(t *testing.T) {
	groups := []db.Group{
		{Path: "my-sessions", Name: "my-sessions", Expanded: true},
		{Path: "work", Name: "work", Expanded: true},
	}
	sessions := []db.Session{
		{ID: "s1", Title: "app", GroupPath: "my-sessions", Status: "stopped"},
		{ID: "s2", Title: "api", GroupPath: "work", Status: "running"},
	}

	items := ui.BuildTree(groups, sessions)
	// my-sessions header, s1, work header, s2
	if len(items) != 4 {
		t.Errorf("expected 4 items, got %d", len(items))
	}
	if items[0].Kind != "group" || items[0].Group.Path != "my-sessions" {
		t.Errorf("items[0] should be my-sessions group")
	}
	if items[1].Kind != "session" || items[1].Session.Title != "app" {
		t.Errorf("items[1] should be session 'app'")
	}
}

func TestBuildTreeCollapsedGroupHidesChildren(t *testing.T) {
	groups := []db.Group{
		{Path: "work", Name: "work", Expanded: false},
	}
	sessions := []db.Session{
		{ID: "s1", Title: "api", GroupPath: "work", Status: "running"},
	}

	items := ui.BuildTree(groups, sessions)
	if len(items) != 1 {
		t.Errorf("expected 1 item (collapsed group header), got %d", len(items))
	}
}

func TestBuildTreeNestedGroups(t *testing.T) {
	groups := []db.Group{
		{Path: "work", Name: "work", Expanded: true},
		{Path: "work/frontend", Name: "frontend", Expanded: true},
	}
	sessions := []db.Session{
		{ID: "s1", Title: "app", GroupPath: "work/frontend", Status: "stopped"},
	}

	items := ui.BuildTree(groups, sessions)
	// work header, frontend header (depth 1), session (depth 2)
	if len(items) != 3 {
		t.Errorf("expected 3 items, got %d", len(items))
	}
	if items[1].Depth != 1 {
		t.Errorf("nested group depth: got %d want 1", items[1].Depth)
	}
	if items[2].Depth != 2 {
		t.Errorf("session depth: got %d want 2", items[2].Depth)
	}
}

func TestRenderListContainsSessionTitle(t *testing.T) {
	groups := []db.Group{{Path: "my-sessions", Name: "my-sessions", Expanded: true}}
	sessions := []db.Session{{ID: "s1", Title: "my-app", GroupPath: "my-sessions", Status: "running"}}
	items := ui.BuildTree(groups, sessions)

	output := ui.RenderList(items, 1, 80, 24) // cursor on session
	if !strings.Contains(output, "my-app") {
		t.Errorf("render missing session title: %q", output)
	}
}

func TestRenderListContainsStatusSymbol(t *testing.T) {
	groups := []db.Group{{Path: "g", Name: "g", Expanded: true}}
	for status, symbol := range map[string]string{
		"running": "●",
		"waiting": "○",
		"idle":    "◐",
		"error":   "✕",
		"stopped": "—",
	} {
		sessions := []db.Session{{ID: "s1", Title: "x", GroupPath: "g", Status: status}}
		items := ui.BuildTree(groups, sessions)
		output := ui.RenderList(items, 0, 80, 24)
		if !strings.Contains(output, symbol) {
			t.Errorf("status %q: missing symbol %q in output", status, symbol)
		}
	}
}
```

- [ ] **Step 2: Run tests — expect failure**

```bash
go test ./internal/ui/... -run TestBuildTree -v
go test ./internal/ui/... -run TestRenderList -v
```

Expected: FAIL — BuildTree returns nil

- [ ] **Step 3: Replace list.go stub with full implementation**

```go
package ui

import (
	"fmt"
	"strings"

	"github.com/charmbracelet/lipgloss"
	"github.com/black-gato/tmux-agent-deck/internal/db"
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

func RenderList(items []ListItem, cursor, width, height int) string {
	var sb strings.Builder
	sb.WriteString("tmux-agent-deck\n\n")
	for i, item := range items {
		indent := strings.Repeat("  ", item.Depth)
		var line string
		if item.Kind == "group" {
			arrow := "▼"
			if !item.Group.Expanded {
				arrow = "►"
			}
			line = groupStyle.Render(fmt.Sprintf("%s%s %s", indent, arrow, item.Group.Name))
		} else {
			sym := statusSymbol[item.Session.Status]
			if sym == "" {
				sym = "—"
			}
			toolStr := dimStyle.Render(item.Session.Tool)
			line = fmt.Sprintf("%s%s  %-20s %s", indent, sym, item.Session.Title, toolStr)
		}
		if i == cursor {
			line = selectedStyle.Render("> " + strings.TrimLeft(line, " "))
		}
		sb.WriteString(line + "\n")
	}
	sb.WriteString("\n[n]ew  [g]roup  [m]ove  [r]ename  [d]elete  [q]uit")
	return sb.String()
}
```

- [ ] **Step 4: Run tests — expect pass**

```bash
go test ./internal/ui/... -v
```

Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C ~/Projects/tmux-agent-deck add internal/ui/list.go internal/ui/list_test.go
git -C ~/Projects/tmux-agent-deck commit -m "feat: implement TUI list rendering and tree flattening"
```

---

### Task 10: TUI — complete dialog actions

**Files:**
- Modify: `internal/ui/dialog.go`
- Modify: `internal/ui/app_test.go` (add dialog action tests)

- [ ] **Step 1: Write failing test for new-session via dialog**

Add to `internal/ui/app_test.go`:

```go
func TestNewSessionDialogCreatesSession(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	m := ui.NewModel(conn, nil, nil)
	m.Reload()

	// open new-session dialog
	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'n'}})
	// type title
	for _, r := range "my-app" {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
	}
	// confirm
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})

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

func TestNewGroupDialogCreatesGroup(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	m := ui.NewModel(conn, nil, nil)
	m.Reload()

	m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{'g'}})
	for _, r := range "work/frontend" {
		m.Update(tea.KeyMsg{Type: tea.KeyRunes, Runes: []rune{r}})
	}
	m.Update(tea.KeyMsg{Type: tea.KeyEnter})

	groups, err := db.ListGroups(conn)
	if err != nil {
		t.Fatal(err)
	}
	found := false
	for _, g := range groups {
		if g.Path == "work/frontend" {
			found = true
		}
	}
	if !found {
		t.Errorf("group work/frontend not created")
	}
}
```

- [ ] **Step 2: Run tests — expect failure**

```bash
go test ./internal/ui/... -run TestNewSessionDialogCreatesSession -v
go test ./internal/ui/... -run TestNewGroupDialogCreatesGroup -v
```

Expected: FAIL — commitDialog is a stub

- [ ] **Step 3: Replace commitDialog stub in dialog.go with full implementation**

Replace the `commitDialog` method in `internal/ui/dialog.go`:

```go
package ui

import (
	"strings"
	"time"

	tea "github.com/charmbracelet/bubbletea"
	"github.com/google/uuid"
	"github.com/black-gato/tmux-agent-deck/internal/db"
)

type dialogState struct {
	prompt string
	value  string
}

func newDialogState(prompt string) dialogState {
	return dialogState{prompt: prompt}
}

func (m *Model) updateDialog(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	switch msg.Type {
	case tea.KeyEsc:
		m.mode = ""
	case tea.KeyEnter:
		m.commitDialog()
		m.mode = ""
		m.Reload()
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
	return m.dialog.prompt + "\n> " + m.dialog.value
}

func (m *Model) commitDialog() {
	val := strings.TrimSpace(m.dialog.value)
	if val == "" {
		return
	}
	switch m.mode {
	case "new-session":
		groupPath := "my-sessions"
		if m.cursor < len(m.items) && m.items[m.cursor].Kind == "group" {
			groupPath = m.items[m.cursor].Group.Path
		}
		db.CreateSession(m.conn, db.Session{
			ID:          uuid.New().String(),
			Title:       val,
			GroupPath:   groupPath,
			ProjectPath: ".",
			Tool:        "claude",
			Status:      "stopped",
			CreatedAt:   time.Now().Unix(),
		})
	case "new-group":
		parts := strings.Split(val, "/")
		name := parts[len(parts)-1]
		db.CreateGroup(m.conn, db.Group{
			Path:        val,
			Name:        name,
			DefaultTool: "claude",
			Expanded:    true,
		})
	case "rename":
		if m.cursor < len(m.items) {
			item := m.items[m.cursor]
			if item.Kind == "session" {
				db.RenameSession(m.conn, item.Session.ID, val)
			} else if item.Kind == "group" {
				db.RenameGroup(m.conn, item.Group.Path, val)
			}
		}
	case "move":
		if m.cursor < len(m.items) && m.items[m.cursor].Kind == "session" {
			db.MoveSession(m.conn, m.items[m.cursor].Session.ID, val)
		}
	}
}
```

- [ ] **Step 4: Run tests — expect pass**

```bash
go test ./internal/ui/... -v
```

Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C ~/Projects/tmux-agent-deck add internal/ui/dialog.go internal/ui/app_test.go
git -C ~/Projects/tmux-agent-deck commit -m "feat: implement dialog actions (new session/group, rename, move)"
```

---

### Task 11: CLI commands

**Files:**
- Modify: `cmd/root.go`
- Create: `cmd/add.go`
- Create: `cmd/list.go`
- Create: `cmd/remove.go`
- Create: `cmd/session.go`
- Create: `cmd/group.go`
- Create: `cmd/cmd_test.go`

- [ ] **Step 1: Write failing test**

```go
// cmd/cmd_test.go
package cmd_test

import (
	"bytes"
	"testing"

	"github.com/black-gato/tmux-agent-deck/cmd"
)

func TestAddAndList(t *testing.T) {
	t.Setenv("AGENT_DECK_DB", ":memory:")

	var out bytes.Buffer
	if err := cmd.RunWith([]string{"add", "--title", "test-app", "--project", "/tmp"}, &out); err != nil {
		t.Fatalf("add: %v", err)
	}

	out.Reset()
	if err := cmd.RunWith([]string{"list"}, &out); err != nil {
		t.Fatalf("list: %v", err)
	}
	if !bytes.Contains(out.Bytes(), []byte("test-app")) {
		t.Errorf("list output missing 'test-app': %s", out.String())
	}
}

func TestGroupCreateAndMove(t *testing.T) {
	t.Setenv("AGENT_DECK_DB", ":memory:")

	var out bytes.Buffer
	if err := cmd.RunWith([]string{"group", "create", "work"}, &out); err != nil {
		t.Fatalf("group create: %v", err)
	}
	if err := cmd.RunWith([]string{"add", "--title", "my-app", "--project", "/tmp"}, &out); err != nil {
		t.Fatalf("add: %v", err)
	}
	if err := cmd.RunWith([]string{"group", "move", "my-app", "work"}, &out); err != nil {
		t.Fatalf("group move: %v", err)
	}

	out.Reset()
	if err := cmd.RunWith([]string{"list"}, &out); err != nil {
		t.Fatal(err)
	}
	if !bytes.Contains(out.Bytes(), []byte("my-app")) {
		t.Errorf("session missing from list after move: %s", out.String())
	}
}
```

- [ ] **Step 2: Run tests — expect failure**

```bash
go test ./cmd/... -run TestAddAndList -v
```

Expected: compile error — RunWith not defined

- [ ] **Step 3: Rewrite cmd/root.go**

```go
package cmd

import (
	"database/sql"
	"fmt"
	"io"
	"os"
	"path/filepath"

	"github.com/spf13/cobra"
	"github.com/black-gato/tmux-agent-deck/internal/db"
)

var rootDB *sql.DB

var rootCmd = &cobra.Command{
	Use:   "tmux-agent-deck",
	Short: "Manage AI coding agent sessions in tmux",
	RunE: func(cmd *cobra.Command, args []string) error {
		return launchTUI(rootDB)
	},
}

func Execute() error {
	conn, err := openDB()
	if err != nil {
		return err
	}
	defer conn.Close()
	rootDB = conn
	return rootCmd.Execute()
}

// RunWith is used by tests to inject args and capture output without os.Exit.
func RunWith(args []string, out io.Writer) error {
	conn, err := openDB()
	if err != nil {
		return err
	}
	defer conn.Close()
	rootDB = conn
	rootCmd.SetOut(out)
	rootCmd.SetArgs(args)
	return rootCmd.Execute()
}

func openDB() (*sql.DB, error) {
	path := os.Getenv("AGENT_DECK_DB")
	if path == "" {
		home, err := os.UserHomeDir()
		if err != nil {
			return nil, err
		}
		path = filepath.Join(home, ".tmux-agent-deck", "state.db")
		if err := os.MkdirAll(filepath.Dir(path), 0755); err != nil {
			return nil, fmt.Errorf("create db dir: %w", err)
		}
	}
	return db.Open(path)
}
```

- [ ] **Step 4: Write cmd/add.go**

```go
package cmd

import (
	"fmt"
	"time"

	"github.com/google/uuid"
	"github.com/spf13/cobra"
	"github.com/black-gato/tmux-agent-deck/internal/db"
)

var addCmd = &cobra.Command{
	Use:   "add",
	Short: "Add a new session",
	RunE: func(cmd *cobra.Command, args []string) error {
		title, _ := cmd.Flags().GetString("title")
		group, _ := cmd.Flags().GetString("group")
		project, _ := cmd.Flags().GetString("project")
		tool, _ := cmd.Flags().GetString("tool")

		// inherit group's default tool if --tool was not explicitly set
		if !cmd.Flags().Changed("tool") {
			if g, err := db.GetGroup(rootDB, group); err == nil && g.DefaultTool != "" {
				tool = g.DefaultTool
			}
		}

		s := db.Session{
			ID:          uuid.New().String(),
			Title:       title,
			GroupPath:   group,
			ProjectPath: project,
			Tool:        tool,
			Status:      "stopped",
			CreatedAt:   time.Now().Unix(),
		}
		if err := db.CreateSession(rootDB, s); err != nil {
			return fmt.Errorf("create session: %w", err)
		}
		fmt.Fprintf(cmd.OutOrStdout(), "Created session %q in group %q\n", title, group)
		return nil
	},
}

func init() {
	addCmd.Flags().String("title", "", "Session title (required)")
	addCmd.Flags().StringP("group", "g", "my-sessions", "Group path")
	addCmd.Flags().StringP("project", "p", ".", "Project directory")
	addCmd.Flags().StringP("tool", "c", "claude", "AI tool")
	addCmd.MarkFlagRequired("title")
	rootCmd.AddCommand(addCmd)
}
```

- [ ] **Step 5: Write cmd/list.go**

```go
package cmd

import (
	"encoding/json"
	"fmt"

	"github.com/spf13/cobra"
	"github.com/black-gato/tmux-agent-deck/internal/db"
)

var listCmd = &cobra.Command{
	Use:   "list",
	Short: "List all sessions",
	RunE: func(cmd *cobra.Command, args []string) error {
		sessions, err := db.ListSessions(rootDB)
		if err != nil {
			return err
		}
		useJSON, _ := cmd.Flags().GetBool("json")
		if useJSON {
			enc := json.NewEncoder(cmd.OutOrStdout())
			enc.SetIndent("", "  ")
			return enc.Encode(sessions)
		}
		for _, s := range sessions {
			fmt.Fprintf(cmd.OutOrStdout(), "%-36s  %-20s  %-15s  %s\n",
				s.ID, s.Title, s.GroupPath, s.Status)
		}
		return nil
	},
}

func init() {
	listCmd.Flags().Bool("json", false, "Output as JSON")
	rootCmd.AddCommand(listCmd)
}
```

- [ ] **Step 6: Write cmd/remove.go**

```go
package cmd

import (
	"fmt"

	"github.com/spf13/cobra"
	"github.com/black-gato/tmux-agent-deck/internal/db"
)

var removeCmd = &cobra.Command{
	Use:   "remove <id|title>",
	Short: "Remove a session",
	Args:  cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		s, err := resolveSession(args[0])
		if err != nil {
			return err
		}
		if err := db.DeleteSession(rootDB, s.ID); err != nil {
			return err
		}
		fmt.Fprintf(cmd.OutOrStdout(), "Removed session %q\n", s.Title)
		return nil
	},
}

func init() {
	rootCmd.AddCommand(removeCmd)
}
```

- [ ] **Step 7: Write cmd/session.go**

```go
package cmd

import (
	"fmt"

	"github.com/spf13/cobra"
	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/tmux"
)

var sessionCmd = &cobra.Command{
	Use:   "session",
	Short: "Manage session lifecycle",
}

var sessionStartCmd = &cobra.Command{
	Use:   "start <id|title>",
	Short: "Spawn a tmux session for this entry",
	Args:  cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		s, err := resolveSession(args[0])
		if err != nil {
			return err
		}
		tc := tmux.NewClient()
		tmuxName := fmt.Sprintf("ad-%s", s.ID[:8])
		startCmd := fmt.Sprintf("%s --project-dir %s", s.Tool, s.ProjectPath)
		if err := tc.NewSession(tmuxName, s.ProjectPath, startCmd); err != nil {
			return fmt.Errorf("start tmux session: %w", err)
		}
		db.UpdateSessionTmuxName(rootDB, s.ID, tmuxName)
		db.UpdateSessionStatus(rootDB, s.ID, "waiting")
		fmt.Fprintf(cmd.OutOrStdout(), "Started %q as tmux session %q\n", s.Title, tmuxName)
		return nil
	},
}

var sessionStopCmd = &cobra.Command{
	Use:   "stop <id|title>",
	Short: "Kill the tmux session for this entry",
	Args:  cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		s, err := resolveSession(args[0])
		if err != nil {
			return err
		}
		if s.TmuxSession != "" {
			tc := tmux.NewClient()
			tc.KillSession(s.TmuxSession)
		}
		db.UpdateSessionStatus(rootDB, s.ID, "stopped")
		fmt.Fprintf(cmd.OutOrStdout(), "Stopped %q\n", s.Title)
		return nil
	},
}

var sessionAttachCmd = &cobra.Command{
	Use:   "attach <id|title>",
	Short: "Attach to the tmux session",
	Args:  cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		s, err := resolveSession(args[0])
		if err != nil {
			return err
		}
		if s.TmuxSession == "" {
			return fmt.Errorf("session %q has no tmux session — run 'session start' first", s.Title)
		}
		return tmux.NewClient().AttachSession(s.TmuxSession)
	},
}

func resolveSession(ref string) (db.Session, error) {
	s, err := db.GetSession(rootDB, ref)
	if err != nil {
		s, err = db.GetSessionByTitle(rootDB, ref)
	}
	if err != nil {
		return db.Session{}, fmt.Errorf("session not found: %q", ref)
	}
	return s, nil
}

func init() {
	sessionCmd.AddCommand(sessionStartCmd)
	sessionCmd.AddCommand(sessionStopCmd)
	sessionCmd.AddCommand(sessionAttachCmd)
	rootCmd.AddCommand(sessionCmd)
}
```

- [ ] **Step 8: Write cmd/group.go**

```go
package cmd

import (
	"fmt"
	"strings"

	"github.com/spf13/cobra"
	"github.com/black-gato/tmux-agent-deck/internal/db"
)

var groupCmd = &cobra.Command{
	Use:   "group",
	Short: "Manage groups",
}

var groupCreateCmd = &cobra.Command{
	Use:   "create <path>",
	Short: "Create a new group",
	Args:  cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		path := args[0]
		tool, _ := cmd.Flags().GetString("tool")
		parts := strings.Split(path, "/")
		name := parts[len(parts)-1]
		g := db.Group{
			Path:        path,
			Name:        name,
			DefaultTool: tool,
			Expanded:    true,
		}
		if err := db.CreateGroup(rootDB, g); err != nil {
			return fmt.Errorf("create group: %w", err)
		}
		fmt.Fprintf(cmd.OutOrStdout(), "Created group %q\n", path)
		return nil
	},
}

var groupDeleteCmd = &cobra.Command{
	Use:   "delete <path>",
	Short: "Delete a group",
	Args:  cobra.ExactArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		if err := db.DeleteGroup(rootDB, args[0]); err != nil {
			return err
		}
		fmt.Fprintf(cmd.OutOrStdout(), "Deleted group %q\n", args[0])
		return nil
	},
}

var groupMoveCmd = &cobra.Command{
	Use:   "move <session-id-or-title> <group-path>",
	Short: "Move a session to a different group",
	Args:  cobra.ExactArgs(2),
	RunE: func(cmd *cobra.Command, args []string) error {
		s, err := resolveSession(args[0])
		if err != nil {
			return err
		}
		if err := db.MoveSession(rootDB, s.ID, args[1]); err != nil {
			return err
		}
		fmt.Fprintf(cmd.OutOrStdout(), "Moved %q to %q\n", s.Title, args[1])
		return nil
	},
}

func init() {
	groupCreateCmd.Flags().String("tool", "claude", "Default AI tool for sessions in this group")
	groupCmd.AddCommand(groupCreateCmd)
	groupCmd.AddCommand(groupDeleteCmd)
	groupCmd.AddCommand(groupMoveCmd)
	rootCmd.AddCommand(groupCmd)
}
```

- [ ] **Step 9: Run tests — expect pass**

```bash
go test ./cmd/... -v
```

Expected: all PASS

- [ ] **Step 10: Commit**

```bash
git -C ~/Projects/tmux-agent-deck add cmd/root.go cmd/add.go cmd/list.go cmd/remove.go cmd/session.go cmd/group.go cmd/cmd_test.go
git -C ~/Projects/tmux-agent-deck commit -m "feat: add CLI commands (add, list, remove, session, group)"
```

---

### Task 12: Wire main.go and smoke test

**Files:**
- Modify: `cmd/root.go` (add launchTUI import block)
- Modify: `main.go` (verify final state)

- [ ] **Step 1: Add launchTUI to cmd/root.go**

Add these imports to `cmd/root.go`:

```go
import (
	// existing imports...
	"github.com/black-gato/tmux-agent-deck/internal/state"
	"github.com/black-gato/tmux-agent-deck/internal/tmux"
	"github.com/black-gato/tmux-agent-deck/internal/ui"
	tea "github.com/charmbracelet/bubbletea"
)
```

Add this function to `cmd/root.go`:

```go
func launchTUI(conn *sql.DB) error {
	tc := tmux.NewClient()
	poller := state.New(conn, tc)
	poller.Start()

	m := ui.NewModel(conn, tc, poller)
	p := tea.NewProgram(m, tea.WithAltScreen())
	_, err := p.Run()
	return err
}
```

- [ ] **Step 2: Build the final binary**

```bash
cd ~/Projects/tmux-agent-deck
go build -o tmux-agent-deck .
```

Expected: binary `tmux-agent-deck` produced, no errors

- [ ] **Step 3: Run full test suite**

```bash
go test ./... -v
```

Expected: all PASS

- [ ] **Step 4: Smoke test — add and list**

```bash
AGENT_DECK_DB=/tmp/test-agent-deck.db ./tmux-agent-deck add --title smoke-test --project /tmp
AGENT_DECK_DB=/tmp/test-agent-deck.db ./tmux-agent-deck list
```

Expected: list output contains a row with `smoke-test`

- [ ] **Step 5: Smoke test — group commands**

```bash
AGENT_DECK_DB=/tmp/test-agent-deck.db ./tmux-agent-deck group create work
AGENT_DECK_DB=/tmp/test-agent-deck.db ./tmux-agent-deck add --title work-app --project /tmp --group work
AGENT_DECK_DB=/tmp/test-agent-deck.db ./tmux-agent-deck list
```

Expected: list shows both `smoke-test` (in my-sessions) and `work-app` (in work)

- [ ] **Step 6: Clean up smoke test DB**

```bash
rm /tmp/test-agent-deck.db
```

- [ ] **Step 7: Commit**

```bash
git -C ~/Projects/tmux-agent-deck add cmd/root.go main.go
git -C ~/Projects/tmux-agent-deck commit -m "feat: wire TUI launch and complete MVP"
```

---

## Self-Review

**Spec coverage:**
- [x] TUI with nested groups — `BuildTree` + `RenderList` in `list.go`
- [x] Status indicators `●○◐✕—` — `statusSymbol` map in `list.go`
- [x] Keybindings `n g m r d Space Enter q` — `keys.go` + `updateNavigation` in `app.go`
- [x] State detection from `capture-pane` — `status.go` + `poller.go`
- [x] Background polling ~1s — `Poller.Start()` in `poller.go`
- [x] CLI: `add`, `list`, `remove`, `session start/stop/attach`, `group create/delete/move` — `cmd/*.go`
- [x] SQLite at `~/.tmux-agent-deck/state.db` — `openDB()` in `cmd/root.go`
- [x] Schema migrations via `metadata` table — `migrate()` in `db/db.go`
- [x] Default `my-sessions` group — seeded in `migrate()`
- [x] Sessions inherit group's `default_tool` at creation — `add.go` reads group before creating
- [x] Collapse/expand groups — `Space` → `SetGroupExpanded` → `Reload`

**Placeholder scan:** No TBDs, stubs are replaced in their respective tasks. All code steps are complete.

**Type consistency:**
- `db.Group` / `db.Session` defined Task 3/4, used identically in Tasks 5–12
- `ui.ListItem` defined in Task 9 `list.go`, referenced in `app.go` Task 8 — consistent
- `state.TmuxReader` interface (`CapturePaneOutput`, `SessionExists`) matches `tmux.Client` methods exactly
- `tmux.StatusStopped` etc. are `type Status = string` (alias), used as bare strings in DB calls — consistent throughout
- `resolveSession` defined in `cmd/session.go`, called from `cmd/remove.go` and `cmd/group.go` — all in `package cmd`, no import needed
