# Hook Handler Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `tmux-agent-deck hook-handler` (receives Claude Code lifecycle events and sends lightweight conductor updates) and `tmux-agent-deck install-hooks` (patches `~/.claude/settings.json` to register the hooks).

**Architecture:** `hook-handler` reads Claude Code event JSON from stdin, resolves the current tmux session name via `tmux display-message`, looks up the session and its conductor in the DB (read-only), then sends a `[deck] <title> | <event>` message via tmux send-keys. `install-hooks` reads `~/.claude/settings.json` into a `map[string]interface{}`, idempotently adds hook entries, and writes back atomically.

**Tech Stack:** Go 1.22+, Cobra, SQLite via modernc.org/sqlite, tmux CLI, encoding/json.

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `internal/db/sessions.go` | Modify | Add `GetSessionByTmuxName` |
| `internal/db/sessions_test.go` | Modify | Tests for `GetSessionByTmuxName` |
| `internal/hook/hook.go` | Create | `HookEvent` type, `ParseEvent(r io.Reader)` |
| `internal/hook/hook_test.go` | Create | Unit tests for `ParseEvent` |
| `cmd/hookhandler.go` | Create | `hook-handler` Cobra subcommand + `runHookHandlerWith` |
| `cmd/hookhandler_test.go` | Create | Integration tests for hook-handler |
| `cmd/installhooks.go` | Create | `install-hooks` Cobra subcommand |
| `cmd/installhooks_test.go` | Create | Tests for install-hooks |

---

## Task 1: `GetSessionByTmuxName` DB function

**Files:**
- Modify: `internal/db/sessions.go`
- Modify: `internal/db/sessions_test.go`

- [ ] **Step 1: Write the failing test**

Add to `internal/db/sessions_test.go`:

```go
func TestGetSessionByTmuxName(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	_ = CreateGroup(conn, Group{Path: "g", Name: "g", DefaultTool: "claude"})
	s := Session{
		ID: "s1", Title: "worker-a", GroupPath: "g",
		TmuxSession: "tmux-abc", Tool: "claude", Status: "running",
		CreatedAt: 1,
	}
	if err := CreateSession(conn, s); err != nil {
		t.Fatal(err)
	}

	got, err := GetSessionByTmuxName(conn, "tmux-abc")
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if got.ID != "s1" {
		t.Errorf("got ID %q, want %q", got.ID, "s1")
	}

	_, err = GetSessionByTmuxName(conn, "not-a-session")
	if err == nil {
		t.Error("expected error for missing session, got nil")
	}
}
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
go test ./internal/db/ -run TestGetSessionByTmuxName -v
```

Expected: `FAIL — GetSessionByTmuxName undefined`

- [ ] **Step 3: Implement `GetSessionByTmuxName`**

Add to `internal/db/sessions.go` (after `GetSessionByTitle`):

```go
func GetSessionByTmuxName(conn *sql.DB, tmuxSession string) (Session, error) {
	var s Session
	var archived int
	err := conn.QueryRow(
		`SELECT id, title, group_path, tmux_session, project_path, tool, status, created_at, last_active, notes, archived, tags, startup_script, tool_flags
		 FROM sessions WHERE tmux_session = ? LIMIT 1`, tmuxSession,
	).Scan(&s.ID, &s.Title, &s.GroupPath, &s.TmuxSession, &s.ProjectPath, &s.Tool, &s.Status, &s.CreatedAt, &s.LastActive, &s.Notes, &archived, &s.Tags, &s.StartupScript, &s.ToolFlags)
	if err != nil {
		return Session{}, fmt.Errorf("get session by tmux name %q: %w", tmuxSession, err)
	}
	s.Archived = archived == 1
	return s, nil
}
```

- [ ] **Step 4: Run the test to verify it passes**

```bash
go test ./internal/db/ -run TestGetSessionByTmuxName -v
```

Expected: `PASS`

- [ ] **Step 5: Run all tests to check for regressions**

```bash
go test ./...
```

Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add internal/db/sessions.go internal/db/sessions_test.go
git commit -m "feat(db): add GetSessionByTmuxName lookup"
```

---

## Task 2: `internal/hook` package

**Files:**
- Create: `internal/hook/hook.go`
- Create: `internal/hook/hook_test.go`

- [ ] **Step 1: Write the failing tests**

Create `internal/hook/hook_test.go`:

```go
package hook_test

import (
	"strings"
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/hook"
)

func TestParseEvent_Stop(t *testing.T) {
	r := strings.NewReader(`{"hook_event_name":"Stop","session_id":"abc"}`)
	ev, err := hook.ParseEvent(r)
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if ev.EventName != "Stop" {
		t.Errorf("got EventName %q, want %q", ev.EventName, "Stop")
	}
	if ev.ToolName != "" {
		t.Errorf("got ToolName %q, want empty", ev.ToolName)
	}
	if ev.Message != "" {
		t.Errorf("got Message %q, want empty", ev.Message)
	}
}

func TestParseEvent_PermissionRequest(t *testing.T) {
	r := strings.NewReader(`{"hook_event_name":"PermissionRequest","tool_name":"Bash"}`)
	ev, err := hook.ParseEvent(r)
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if ev.EventName != "PermissionRequest" {
		t.Errorf("got EventName %q, want %q", ev.EventName, "PermissionRequest")
	}
	if ev.ToolName != "Bash" {
		t.Errorf("got ToolName %q, want %q", ev.ToolName, "Bash")
	}
}

func TestParseEvent_Notification(t *testing.T) {
	r := strings.NewReader(`{"hook_event_name":"Notification","message":"needs input"}`)
	ev, err := hook.ParseEvent(r)
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if ev.Message != "needs input" {
		t.Errorf("got Message %q, want %q", ev.Message, "needs input")
	}
}

func TestParseEvent_UnknownEvent(t *testing.T) {
	r := strings.NewReader(`{"hook_event_name":"SomeFutureEvent"}`)
	ev, err := hook.ParseEvent(r)
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if ev.EventName != "SomeFutureEvent" {
		t.Errorf("got EventName %q, want %q", ev.EventName, "SomeFutureEvent")
	}
}

func TestParseEvent_InvalidJSON(t *testing.T) {
	r := strings.NewReader(`not json`)
	_, err := hook.ParseEvent(r)
	if err == nil {
		t.Error("expected error for invalid JSON, got nil")
	}
}
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
go test ./internal/hook/ -v
```

Expected: `FAIL — cannot find package`

- [ ] **Step 3: Implement `internal/hook/hook.go`**

Create `internal/hook/hook.go`:

```go
package hook

import (
	"encoding/json"
	"io"
)

type HookEvent struct {
	EventName string
	ToolName  string
	Message   string
}

type hookPayload struct {
	EventName string `json:"hook_event_name"`
	ToolName  string `json:"tool_name"`
	Message   string `json:"message"`
}

func ParseEvent(r io.Reader) (HookEvent, error) {
	var p hookPayload
	if err := json.NewDecoder(r).Decode(&p); err != nil {
		return HookEvent{}, err
	}
	return HookEvent{
		EventName: p.EventName,
		ToolName:  p.ToolName,
		Message:   p.Message,
	}, nil
}
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
go test ./internal/hook/ -v
```

Expected: all `PASS`

- [ ] **Step 5: Run all tests**

```bash
go test ./...
```

Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add internal/hook/
git commit -m "feat(hook): add ParseEvent for Claude Code hook JSON"
```

---

## Task 3: `hook-handler` subcommand

**Files:**
- Create: `cmd/hookhandler.go`
- Create: `cmd/hookhandler_test.go`

- [ ] **Step 1: Write the failing tests**

Create `cmd/hookhandler_test.go`:

```go
package cmd_test

import (
	"strings"
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/testutil"
)

func setupHookHandlerDB(t *testing.T) (*db.Group, *db.Session, *db.Session) {
	t.Helper()
	conn := testutil.OpenTestDB(t)
	g := db.Group{Path: "grp", Name: "grp", DefaultTool: "claude"}
	if err := db.CreateGroup(conn, g); err != nil {
		t.Fatal(err)
	}
	conductor := db.Session{
		ID: "cond-1", Title: "conductor", GroupPath: "grp",
		TmuxSession: "tmux-cond", Tool: "claude", Status: "running", CreatedAt: 1,
	}
	worker := db.Session{
		ID: "work-1", Title: "worker-a", GroupPath: "grp",
		TmuxSession: "tmux-work", Tool: "claude", Status: "running", CreatedAt: 2,
	}
	if err := db.CreateSession(conn, conductor); err != nil {
		t.Fatal(err)
	}
	if err := db.CreateSession(conn, worker); err != nil {
		t.Fatal(err)
	}
	if err := db.SetGroupConductor(conn, "grp", "cond-1"); err != nil {
		t.Fatal(err)
	}
	return &g, &conductor, &worker
}
```

Note: `setupHookHandlerDB` returns the shared test DB connection. You need to use `testutil.OpenTestDB(t)` inside `runHookHandlerWith` in tests by passing `conn` directly.

Add these test functions to `cmd/hookhandler_test.go`:

```go
func TestHookHandler_StopSendsMessage(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	g := db.Group{Path: "grp", Name: "grp", DefaultTool: "claude"}
	_ = db.CreateGroup(conn, g)
	conductor := db.Session{ID: "c1", Title: "conductor", GroupPath: "grp", TmuxSession: "tmux-cond", Tool: "claude", Status: "running", CreatedAt: 1}
	worker := db.Session{ID: "w1", Title: "worker-a", GroupPath: "grp", TmuxSession: "tmux-work", Tool: "claude", Status: "running", CreatedAt: 2}
	_ = db.CreateSession(conn, conductor)
	_ = db.CreateSession(conn, worker)
	_ = db.SetGroupConductor(conn, "grp", "c1")

	fake := testutil.NewFakeTmuxClient()
	r := strings.NewReader(`{"hook_event_name":"Stop"}`)
	err := runHookHandlerWith(r, conn, hookHandlerDeps{
		resolveSession: func() (string, error) { return "tmux-work", nil },
		sender:         fake,
	})
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if len(fake.SentKeys) != 1 {
		t.Fatalf("expected 1 SendKeys call, got %d", len(fake.SentKeys))
	}
	if fake.SentKeys[0].Keys != "[deck] worker-a | Stop | task complete" {
		t.Errorf("unexpected message: %q", fake.SentKeys[0].Keys)
	}
	if fake.SentKeys[0].Session != "tmux-cond" {
		t.Errorf("sent to wrong session: %q", fake.SentKeys[0].Session)
	}
	if len(fake.SentRawKeys) != 1 || fake.SentRawKeys[0].Keys != "Enter" {
		t.Errorf("expected Enter raw key submit")
	}
}

func TestHookHandler_PermissionRequestIncludesTool(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	g := db.Group{Path: "grp", Name: "grp", DefaultTool: "claude"}
	_ = db.CreateGroup(conn, g)
	conductor := db.Session{ID: "c1", Title: "conductor", GroupPath: "grp", TmuxSession: "tmux-cond", Tool: "claude", Status: "running", CreatedAt: 1}
	worker := db.Session{ID: "w1", Title: "worker-a", GroupPath: "grp", TmuxSession: "tmux-work", Tool: "claude", Status: "running", CreatedAt: 2}
	_ = db.CreateSession(conn, conductor)
	_ = db.CreateSession(conn, worker)
	_ = db.SetGroupConductor(conn, "grp", "c1")

	fake := testutil.NewFakeTmuxClient()
	r := strings.NewReader(`{"hook_event_name":"PermissionRequest","tool_name":"Bash"}`)
	_ = runHookHandlerWith(r, conn, hookHandlerDeps{
		resolveSession: func() (string, error) { return "tmux-work", nil },
		sender:         fake,
	})
	if len(fake.SentKeys) != 1 {
		t.Fatalf("expected 1 SendKeys call, got %d", len(fake.SentKeys))
	}
	if fake.SentKeys[0].Keys != "[deck] worker-a | PermissionRequest | tool: Bash" {
		t.Errorf("unexpected message: %q", fake.SentKeys[0].Keys)
	}
}

func TestHookHandler_NoConductorSilentExit(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	g := db.Group{Path: "grp", Name: "grp", DefaultTool: "claude"}
	_ = db.CreateGroup(conn, g)
	worker := db.Session{ID: "w1", Title: "worker-a", GroupPath: "grp", TmuxSession: "tmux-work", Tool: "claude", Status: "running", CreatedAt: 1}
	_ = db.CreateSession(conn, worker)
	// no conductor set

	fake := testutil.NewFakeTmuxClient()
	r := strings.NewReader(`{"hook_event_name":"Stop"}`)
	err := runHookHandlerWith(r, conn, hookHandlerDeps{
		resolveSession: func() (string, error) { return "tmux-work", nil },
		sender:         fake,
	})
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if len(fake.SentKeys) != 0 {
		t.Errorf("expected no SendKeys, got %d", len(fake.SentKeys))
	}
}

func TestHookHandler_UnknownTmuxSessionSilentExit(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	r := strings.NewReader(`{"hook_event_name":"Stop"}`)
	err := runHookHandlerWith(r, conn, hookHandlerDeps{
		resolveSession: func() (string, error) { return "not-tracked", nil },
		sender:         fake,
	})
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if len(fake.SentKeys) != 0 {
		t.Errorf("expected no SendKeys, got %d", len(fake.SentKeys))
	}
}

func TestHookHandler_NotificationTruncatesLongMessage(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	g := db.Group{Path: "grp", Name: "grp", DefaultTool: "claude"}
	_ = db.CreateGroup(conn, g)
	conductor := db.Session{ID: "c1", Title: "conductor", GroupPath: "grp", TmuxSession: "tmux-cond", Tool: "claude", Status: "running", CreatedAt: 1}
	worker := db.Session{ID: "w1", Title: "worker-a", GroupPath: "grp", TmuxSession: "tmux-work", Tool: "claude", Status: "running", CreatedAt: 2}
	_ = db.CreateSession(conn, conductor)
	_ = db.CreateSession(conn, worker)
	_ = db.SetGroupConductor(conn, "grp", "c1")

	longMsg := strings.Repeat("x", 80)
	fake := testutil.NewFakeTmuxClient()
	r := strings.NewReader(`{"hook_event_name":"Notification","message":"` + longMsg + `"}`)
	_ = runHookHandlerWith(r, conn, hookHandlerDeps{
		resolveSession: func() (string, error) { return "tmux-work", nil },
		sender:         fake,
	})
	if len(fake.SentKeys) != 1 {
		t.Fatalf("expected 1 SendKeys, got %d", len(fake.SentKeys))
	}
	// message context must be truncated to 60 chars
	expected := "[deck] worker-a | Notification | " + strings.Repeat("x", 60)
	if fake.SentKeys[0].Keys != expected {
		t.Errorf("unexpected message: %q", fake.SentKeys[0].Keys)
	}
}
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
go test ./cmd/ -run TestHookHandler -v
```

Expected: `FAIL — runHookHandlerWith undefined`

- [ ] **Step 3: Implement `cmd/hookhandler.go`**

Create `cmd/hookhandler.go`:

```go
package cmd

import (
	"database/sql"
	"io"
	"os"
	"os/exec"
	"strings"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/hook"
	"github.com/black-gato/tmux-agent-deck/internal/tmux"
	"github.com/spf13/cobra"
)

type hookSender interface {
	SendKeys(session string, pane int, keys string) error
	SendRawKeys(session string, pane int, keys string) error
}

type hookHandlerDeps struct {
	resolveSession func() (string, error)
	sender         hookSender
}

var hookHandlerCmd = &cobra.Command{
	Use:   "hook-handler",
	Short: "Handle Claude Code lifecycle hook events (called by Claude Code hooks)",
	RunE: func(cmd *cobra.Command, args []string) error {
		conn, err := openDB()
		if err != nil {
			return err
		}
		defer conn.Close()
		return runHookHandlerWith(os.Stdin, conn, hookHandlerDeps{
			resolveSession: resolveCurrentTmuxSession,
			sender:         tmux.NewClient(),
		})
	},
}

func resolveCurrentTmuxSession() (string, error) {
	out, err := exec.Command("tmux", "display-message", "-p", "#S").Output()
	if err != nil {
		return "", err
	}
	return strings.TrimSpace(string(out)), nil
}

func runHookHandlerWith(r io.Reader, conn *sql.DB, deps hookHandlerDeps) error {
	event, err := hook.ParseEvent(r)
	if err != nil || event.EventName == "" {
		return nil
	}

	tmuxName, err := deps.resolveSession()
	if err != nil || tmuxName == "" {
		return nil
	}

	session, err := db.GetSessionByTmuxName(conn, tmuxName)
	if err != nil {
		return nil
	}

	conductor, err := db.GetGroupConductorSession(conn, session.GroupPath)
	if err != nil || conductor.Title == "" || conductor.TmuxSession == "" {
		return nil
	}
	if conductor.Status == tmux.StatusStopped || conductor.Status == tmux.StatusError {
		return nil
	}

	msg := hookMessage(session.Title, event)
	if err := deps.sender.SendKeys(conductor.TmuxSession, 0, msg); err != nil {
		return err
	}
	return deps.sender.SendRawKeys(conductor.TmuxSession, 0, "Enter")
}

func hookMessage(title string, event hook.HookEvent) string {
	base := "[deck] " + title + " | " + event.EventName
	switch event.EventName {
	case "Stop":
		return base + " | task complete"
	case "PermissionRequest":
		if event.ToolName != "" {
			return base + " | tool: " + event.ToolName
		}
	case "Notification":
		if event.Message != "" {
			msg := event.Message
			if len(msg) > 60 {
				msg = msg[:60]
			}
			return base + " | " + msg
		}
	}
	return base
}

func init() {
	rootCmd.AddCommand(hookHandlerCmd)
}
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
go test ./cmd/ -run TestHookHandler -v
```

Expected: all `PASS`

- [ ] **Step 5: Run all tests**

```bash
go test ./...
```

Expected: all pass.

- [ ] **Step 6: Commit**

```bash
git add cmd/hookhandler.go cmd/hookhandler_test.go
git commit -m "feat(cmd): add hook-handler subcommand for Claude Code lifecycle events"
```

---

## Task 4: `install-hooks` subcommand

**Files:**
- Create: `cmd/installhooks.go`
- Create: `cmd/installhooks_test.go`

- [ ] **Step 1: Write the failing tests**

Create `cmd/installhooks_test.go`:

```go
package cmd_test

import (
	"encoding/json"
	"io"
	"os"
	"path/filepath"
	"testing"
)

func TestInstallHooks_FreshFile(t *testing.T) {
	dir := t.TempDir()
	path := filepath.Join(dir, "settings.json")

	if err := runInstallHooks(path, false, io.Discard); err != nil {
		t.Fatalf("unexpected error: %v", err)
	}

	data, err := os.ReadFile(path)
	if err != nil {
		t.Fatal(err)
	}
	var settings map[string]interface{}
	if err := json.Unmarshal(data, &settings); err != nil {
		t.Fatalf("invalid JSON written: %v", err)
	}
	hooks, ok := settings["hooks"].(map[string]interface{})
	if !ok {
		t.Fatal("expected hooks object")
	}
	for _, event := range []string{"Stop", "SessionStart", "SessionEnd", "UserPromptSubmit", "PermissionRequest", "PreCompact", "Notification"} {
		arr, ok := hooks[event].([]interface{})
		if !ok || len(arr) == 0 {
			t.Errorf("event %q: expected at least one hook entry", event)
			continue
		}
		found := false
		for _, entry := range arr {
			m, ok := entry.(map[string]interface{})
			if !ok {
				continue
			}
			if m["command"] == "tmux-agent-deck hook-handler" {
				found = true
				break
			}
		}
		if !found {
			t.Errorf("event %q: tmux-agent-deck hook-handler not found in hooks", event)
		}
	}
}

func TestInstallHooks_Idempotent(t *testing.T) {
	dir := t.TempDir()
	path := filepath.Join(dir, "settings.json")

	if err := runInstallHooks(path, false, io.Discard); err != nil {
		t.Fatal(err)
	}
	if err := runInstallHooks(path, false, io.Discard); err != nil {
		t.Fatal(err)
	}

	data, _ := os.ReadFile(path)
	var settings map[string]interface{}
	_ = json.Unmarshal(data, &settings)
	hooks := settings["hooks"].(map[string]interface{})
	stop := hooks["Stop"].([]interface{})
	count := 0
	for _, entry := range stop {
		m, ok := entry.(map[string]interface{})
		if ok && m["command"] == "tmux-agent-deck hook-handler" {
			count++
		}
	}
	if count != 1 {
		t.Errorf("expected exactly 1 hook-handler entry for Stop after two installs, got %d", count)
	}
}

func TestInstallHooks_PreservesExistingEntries(t *testing.T) {
	dir := t.TempDir()
	path := filepath.Join(dir, "settings.json")

	existing := map[string]interface{}{
		"env": map[string]interface{}{"EDITOR": "nvim"},
		"hooks": map[string]interface{}{
			"Stop": []interface{}{
				map[string]interface{}{"type": "command", "command": "other-tool", "async": true},
			},
		},
	}
	data, _ := json.Marshal(existing)
	_ = os.WriteFile(path, data, 0644)

	if err := runInstallHooks(path, false, io.Discard); err != nil {
		t.Fatal(err)
	}

	data, _ = os.ReadFile(path)
	var settings map[string]interface{}
	_ = json.Unmarshal(data, &settings)

	// env preserved
	env, ok := settings["env"].(map[string]interface{})
	if !ok || env["EDITOR"] != "nvim" {
		t.Error("expected env.EDITOR to be preserved")
	}

	// other-tool preserved in Stop
	hooks := settings["hooks"].(map[string]interface{})
	stop := hooks["Stop"].([]interface{})
	foundOther := false
	for _, entry := range stop {
		m, ok := entry.(map[string]interface{})
		if ok && m["command"] == "other-tool" {
			foundOther = true
		}
	}
	if !foundOther {
		t.Error("expected other-tool entry to be preserved in Stop hooks")
	}
}

func TestInstallHooks_Uninstall(t *testing.T) {
	dir := t.TempDir()
	path := filepath.Join(dir, "settings.json")

	// install first
	_ = runInstallHooks(path, false, io.Discard)
	// then uninstall
	if err := runInstallHooks(path, true, io.Discard); err != nil {
		t.Fatalf("uninstall error: %v", err)
	}

	data, _ := os.ReadFile(path)
	var settings map[string]interface{}
	_ = json.Unmarshal(data, &settings)
	hooks, _ := settings["hooks"].(map[string]interface{})
	for _, event := range []string{"Stop", "SessionStart", "SessionEnd", "UserPromptSubmit", "PermissionRequest", "PreCompact", "Notification"} {
		arr, _ := hooks[event].([]interface{})
		for _, entry := range arr {
			m, ok := entry.(map[string]interface{})
			if ok && m["command"] == "tmux-agent-deck hook-handler" {
				t.Errorf("event %q: hook-handler entry still present after uninstall", event)
			}
		}
	}
}

func TestInstallHooks_PermissionRequestIsSync(t *testing.T) {
	dir := t.TempDir()
	path := filepath.Join(dir, "settings.json")
	_ = runInstallHooks(path, false, io.Discard)

	data, _ := os.ReadFile(path)
	var settings map[string]interface{}
	_ = json.Unmarshal(data, &settings)
	hooks := settings["hooks"].(map[string]interface{})
	arr := hooks["PermissionRequest"].([]interface{})
	for _, entry := range arr {
		m, ok := entry.(map[string]interface{})
		if !ok || m["command"] != "tmux-agent-deck hook-handler" {
			continue
		}
		// async must be absent (false/omitted) for PermissionRequest
		if v, exists := m["async"]; exists && v == true {
			t.Error("PermissionRequest hook-handler must not be async")
		}
	}
}
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
go test ./cmd/ -run TestInstallHooks -v
```

Expected: `FAIL — runInstallHooks undefined`

- [ ] **Step 3: Implement `cmd/installhooks.go`**

Create `cmd/installhooks.go`:

```go
package cmd

import (
	"encoding/json"
	"fmt"
	"io"
	"os"
	"path/filepath"

	"github.com/spf13/cobra"
)

const deckHookCommand = "tmux-agent-deck hook-handler"

var managedHookEvents = []struct {
	name  string
	async bool
}{
	{"Stop", true},
	{"SessionStart", true},
	{"SessionEnd", true},
	{"UserPromptSubmit", true},
	{"PermissionRequest", false},
	{"PreCompact", false},
	{"Notification", true},
}

var installHooksUninstall bool

var installHooksCmd = &cobra.Command{
	Use:   "install-hooks",
	Short: "Register tmux-agent-deck hook-handler in ~/.claude/settings.json",
	RunE: func(cmd *cobra.Command, args []string) error {
		home, err := os.UserHomeDir()
		if err != nil {
			return err
		}
		path := filepath.Join(home, ".claude", "settings.json")
		return runInstallHooks(path, installHooksUninstall, cmd.OutOrStdout())
	},
}

func runInstallHooks(settingsPath string, uninstall bool, out io.Writer) error {
	settings, err := readSettings(settingsPath)
	if err != nil {
		return err
	}

	hooks := settingsHooks(settings)

	for _, ev := range managedHookEvents {
		arr := hooksForEvent(hooks, ev.name)
		if uninstall {
			before := len(arr)
			arr = removeOurEntry(arr)
			hooks[ev.name] = arr
			if len(arr) < before {
				fmt.Fprintf(out, "%-20s removed\n", ev.name)
			} else {
				fmt.Fprintf(out, "%-20s not registered\n", ev.name)
			}
		} else {
			if hasOurEntry(arr) {
				fmt.Fprintf(out, "%-20s already registered\n", ev.name)
			} else {
				hooks[ev.name] = append(arr, buildEntry(ev.async))
				fmt.Fprintf(out, "%-20s added\n", ev.name)
			}
		}
	}

	settings["hooks"] = hooks
	return writeSettings(settingsPath, settings)
}

func readSettings(path string) (map[string]interface{}, error) {
	data, err := os.ReadFile(path)
	if os.IsNotExist(err) {
		return map[string]interface{}{}, nil
	}
	if err != nil {
		return nil, err
	}
	var m map[string]interface{}
	if err := json.Unmarshal(data, &m); err != nil {
		return nil, fmt.Errorf("parse %s: %w", path, err)
	}
	return m, nil
}

func writeSettings(path string, settings map[string]interface{}) error {
	data, err := json.MarshalIndent(settings, "", "    ")
	if err != nil {
		return err
	}
	if err := os.MkdirAll(filepath.Dir(path), 0755); err != nil {
		return err
	}
	tmp := path + ".tmp"
	if err := os.WriteFile(tmp, data, 0644); err != nil {
		return err
	}
	return os.Rename(tmp, path)
}

func settingsHooks(settings map[string]interface{}) map[string]interface{} {
	if h, ok := settings["hooks"].(map[string]interface{}); ok {
		return h
	}
	h := map[string]interface{}{}
	settings["hooks"] = h
	return h
}

func hooksForEvent(hooks map[string]interface{}, event string) []interface{} {
	arr, _ := hooks[event].([]interface{})
	return arr
}

func hasOurEntry(arr []interface{}) bool {
	for _, entry := range arr {
		m, ok := entry.(map[string]interface{})
		if !ok {
			continue
		}
		if m["command"] == deckHookCommand {
			return true
		}
	}
	return false
}

func removeOurEntry(arr []interface{}) []interface{} {
	var out []interface{}
	for _, entry := range arr {
		m, ok := entry.(map[string]interface{})
		if ok && m["command"] == deckHookCommand {
			continue
		}
		out = append(out, entry)
	}
	return out
}

func buildEntry(async bool) map[string]interface{} {
	entry := map[string]interface{}{
		"type":    "command",
		"command": deckHookCommand,
	}
	if async {
		entry["async"] = true
	}
	return entry
}

func init() {
	installHooksCmd.Flags().BoolVar(&installHooksUninstall, "uninstall", false, "Remove tmux-agent-deck hook-handler entries")
	rootCmd.AddCommand(installHooksCmd)
}
```

- [ ] **Step 4: Run the tests to verify they pass**

```bash
go test ./cmd/ -run TestInstallHooks -v
```

Expected: all `PASS`

- [ ] **Step 5: Run all tests and vet**

```bash
go test ./...
go vet ./...
```

Expected: all pass, no vet issues.

- [ ] **Step 6: Commit**

```bash
git add cmd/installhooks.go cmd/installhooks_test.go
git commit -m "feat(cmd): add install-hooks subcommand to register Claude Code hooks"
```

---

## Task 5: Update CLAUDE.md roadmap

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Add hook-handler to the roadmap table**

In `CLAUDE.md`, add a row to the Roadmap table after the last entry:

```markdown
| Hook Handler | Claude Code lifecycle hooks → real-time conductor updates; `install-hooks` to register | complete | [spec](docs/superpowers/specs/2026-05-23-hook-handler-design.md) | [plan](docs/superpowers/plans/2026-05-23-hook-handler.md) |
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add hook-handler to roadmap"
```
