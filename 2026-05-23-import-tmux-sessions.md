# Import Tmux Sessions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let the deck adopt running tmux sessions that have no row in its SQLite DB, via both a CLI `import` subcommand and a TUI picker bound to `I`.

**Architecture:** A new `tmux.Client.SessionInfo` reads `#{pane_current_path}` from a live tmux session. A new `db.ListUntrackedTmuxSessions` diffs `tmux list-sessions` against rows with non-empty `tmux_session`. The CLI command (`cmd/import.go`) and a new TUI picker+form dialog (`internal/ui/import.go`) call a shared insert helper that builds a `db.Session` with `Status="unknown"` and defers status detection to the next poller tick.

**Tech Stack:** Go stdlib `os/exec`; existing Cobra command pattern (mirroring `cmd/add.go`); existing Bubbletea dialog scaffolding (`internal/ui/dialog.go`, `internal/ui/form.go`); `testutil.FakeTmuxClient`.

**Spec:** [docs/superpowers/specs/2026-05-23-import-tmux-sessions-design.md](../specs/2026-05-23-import-tmux-sessions-design.md)

---

### Task 1: Add `tmux.Client.SessionInfo` to read `pane_current_path`

**Files:**
- Modify: `internal/tmux/client.go` — new struct + method, add to `ClientIface`
- Modify: `internal/testutil/tmux.go` — fake implementation
- Test: `internal/tmux/client_test.go` (create if missing — black-box `package tmux_test`)

- [ ] **Step 1: Write the failing test**

Create or extend `internal/tmux/client_test.go`:

```go
package tmux_test

import (
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/tmux"
)

func TestParseSessionInfoOutput(t *testing.T) {
	got, err := tmux.ParseSessionInfoOutput("my-sess", "/tmp/work|1716480000")
	if err != nil {
		t.Fatalf("parse: %v", err)
	}
	if got.Name != "my-sess" {
		t.Errorf("name: got %q want %q", got.Name, "my-sess")
	}
	if got.CurrentPath != "/tmp/work" {
		t.Errorf("path: got %q want %q", got.CurrentPath, "/tmp/work")
	}
	if got.Activity.Unix() != 1716480000 {
		t.Errorf("activity: got %d want %d", got.Activity.Unix(), 1716480000)
	}
}

func TestParseSessionInfoOutputBlankActivity(t *testing.T) {
	got, err := tmux.ParseSessionInfoOutput("s", "/tmp|0")
	if err != nil {
		t.Fatalf("parse: %v", err)
	}
	if !got.Activity.IsZero() {
		t.Errorf("expected zero activity, got %v", got.Activity)
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/tmux/ -run TestParseSessionInfoOutput -v
```

Expected: FAIL — `ParseSessionInfoOutput` undefined.

- [ ] **Step 3: Add `SessionInfo` struct, parser, and `SessionInfo` method**

Append to `internal/tmux/client.go` (alongside `SessionActivity`):

```go
type SessionInfo struct {
	Name        string
	CurrentPath string
	Activity    time.Time
}

func ParseSessionInfoOutput(name, raw string) (SessionInfo, error) {
	raw = strings.TrimSpace(raw)
	parts := strings.SplitN(raw, "|", 2)
	if len(parts) != 2 {
		return SessionInfo{}, fmt.Errorf("session info %q: unexpected output %q", name, raw)
	}
	info := SessionInfo{Name: name, CurrentPath: parts[0]}
	sec, err := strconv.ParseInt(strings.TrimSpace(parts[1]), 10, 64)
	if err != nil {
		return SessionInfo{}, fmt.Errorf("session info %q: parse activity: %w", name, err)
	}
	if sec > 0 {
		info.Activity = time.Unix(sec, 0)
	}
	return info, nil
}

func (c *Client) SessionInfo(name string) (SessionInfo, error) {
	out, err := cmdOutput("tmux", "display-message", "-p", "-t", name, "-F", "#{pane_current_path}|#{session_activity}")
	if err != nil {
		return SessionInfo{}, fmt.Errorf("session info %q: %w", name, err)
	}
	return ParseSessionInfoOutput(name, string(out))
}
```

- [ ] **Step 4: Add `SessionInfo` to `ClientIface`**

In `internal/tmux/client.go`, add to the interface block:

```go
SessionInfo(name string) (SessionInfo, error)
```

- [ ] **Step 5: Extend `FakeTmuxClient` with a `SessionInfos` map**

In `internal/testutil/tmux.go`:

```go
// (add field)
SessionInfos map[string]tmux.SessionInfo
```

In `NewFakeTmuxClient()` add:

```go
SessionInfos: make(map[string]tmux.SessionInfo),
```

Add the method:

```go
func (f *FakeTmuxClient) SessionInfo(name string) (tmux.SessionInfo, error) {
	if info, ok := f.SessionInfos[name]; ok {
		return info, nil
	}
	if _, exists := f.Sessions[name]; !exists {
		return tmux.SessionInfo{}, fmt.Errorf("no session %q", name)
	}
	return tmux.SessionInfo{Name: name}, nil
}
```

- [ ] **Step 6: Run tests to verify they pass**

```bash
go test ./internal/tmux/ ./internal/testutil/ -v
```

Expected: PASS (parser tests pass; existing tests still pass).

- [ ] **Step 7: Confirm full build still compiles**

```bash
go build ./...
```

Expected: no errors. (Adding to `ClientIface` will surface any missing implementations elsewhere.)

- [ ] **Step 8: Commit**

```bash
git add internal/tmux/client.go internal/tmux/client_test.go internal/testutil/tmux.go
git commit -m "feat(tmux): add SessionInfo for pane_current_path + activity"
```

---

### Task 2: Add `db.ListUntrackedTmuxSessions`

**Files:**
- Modify: `internal/db/sessions.go`
- Test: `internal/db/sessions_test.go`

- [ ] **Step 1: Write the failing test**

Append to `internal/db/sessions_test.go`:

```go
func TestListUntrackedTmuxSessions(t *testing.T) {
	conn := testutil.OpenTestDB(t)

	tracked := db.Session{
		ID:          "id-tracked",
		Title:       "tracked",
		GroupPath:   "my-sessions",
		TmuxSession: "ad-tracked",
		Tool:        "claude",
		Status:      "running",
		CreatedAt:   1,
	}
	if err := db.CreateSession(conn, tracked); err != nil {
		t.Fatalf("create: %v", err)
	}

	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["ad-tracked"] = ""
	fake.Sessions["scratch-foo"] = ""
	fake.Sessions["scratch-bar"] = ""

	got, err := db.ListUntrackedTmuxSessions(conn, fake)
	if err != nil {
		t.Fatalf("list: %v", err)
	}
	sort.Strings(got)
	want := []string{"scratch-bar", "scratch-foo"}
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}

func TestListUntrackedTmuxSessionsIgnoresEmptyTmuxName(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	if err := db.CreateSession(conn, db.Session{
		ID: "id-1", Title: "row", GroupPath: "my-sessions",
		TmuxSession: "", Tool: "claude", Status: "stopped", CreatedAt: 1,
	}); err != nil {
		t.Fatalf("create: %v", err)
	}
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["live-session"] = ""

	got, err := db.ListUntrackedTmuxSessions(conn, fake)
	if err != nil {
		t.Fatalf("list: %v", err)
	}
	if len(got) != 1 || got[0] != "live-session" {
		t.Errorf("got %v want [live-session]", got)
	}
}
```

Also ensure imports include `sort`, `reflect`, and `internal/testutil`.

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/db/ -run TestListUntrackedTmuxSessions -v
```

Expected: FAIL — `db.ListUntrackedTmuxSessions` undefined.

- [ ] **Step 3: Add the helper**

In `internal/db/sessions.go`, append:

```go
// TmuxLister is the subset of tmux.ClientIface needed to enumerate live sessions.
// Defined here to avoid an import cycle between db and tmux.
type TmuxLister interface {
	ListSessions() ([]string, error)
}

func ListUntrackedTmuxSessions(conn *sql.DB, tc TmuxLister) ([]string, error) {
	names, err := tc.ListSessions()
	if err != nil {
		return nil, fmt.Errorf("list tmux sessions: %w", err)
	}
	out := make([]string, 0, len(names))
	for _, name := range names {
		if name == "" {
			continue
		}
		_, err := GetSessionByTmuxName(conn, name)
		if err == nil {
			continue
		}
		if errors.Is(err, sql.ErrNoRows) {
			out = append(out, name)
			continue
		}
		// GetSessionByTmuxName wraps sql.ErrNoRows in fmt.Errorf with %w; unwrap once.
		if errors.Is(errors.Unwrap(err), sql.ErrNoRows) {
			out = append(out, name)
			continue
		}
		return nil, err
	}
	return out, nil
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
go test ./internal/db/ -run TestListUntrackedTmuxSessions -v
```

Expected: PASS (both tests).

- [ ] **Step 5: Commit**

```bash
git add internal/db/sessions.go internal/db/sessions_test.go
git commit -m "feat(db): ListUntrackedTmuxSessions diffs live tmux against DB"
```

---

### Task 3: Shared `ImportSession` helper for CLI + TUI

**Files:**
- Create: `internal/db/import.go` (lives in `db` package so both CLI and UI can reuse)
- Test: `internal/db/import_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/db/import_test.go`:

```go
package db_test

import (
	"strings"
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/testutil"
)

func TestImportSessionDefaults(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["scratch-foo"] = ""
	fake.SessionInfos["scratch-foo"] = tmuxInfo("scratch-foo", "/tmp/work")

	got, err := db.ImportSession(conn, fake, db.ImportRequest{TmuxName: "scratch-foo"})
	if err != nil {
		t.Fatalf("import: %v", err)
	}
	if got.Title != "scratch-foo" {
		t.Errorf("title: %q", got.Title)
	}
	if got.GroupPath != "my-sessions" {
		t.Errorf("group: %q", got.GroupPath)
	}
	if got.ProjectPath != "/tmp/work" {
		t.Errorf("project: %q", got.ProjectPath)
	}
	if got.Tool != "claude" {
		t.Errorf("tool: %q", got.Tool)
	}
	if got.TmuxSession != "scratch-foo" {
		t.Errorf("tmux: %q", got.TmuxSession)
	}
	if got.Status != "unknown" {
		t.Errorf("status: %q", got.Status)
	}
	if got.ID == "" {
		t.Errorf("expected generated id")
	}

	roundtrip, err := db.GetSession(conn, got.ID)
	if err != nil {
		t.Fatalf("get: %v", err)
	}
	if roundtrip.TmuxSession != "scratch-foo" {
		t.Errorf("roundtrip tmux: %q", roundtrip.TmuxSession)
	}
}

func TestImportSessionRejectsDuplicate(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["dup"] = ""
	if _, err := db.ImportSession(conn, fake, db.ImportRequest{TmuxName: "dup"}); err != nil {
		t.Fatalf("first: %v", err)
	}
	_, err := db.ImportSession(conn, fake, db.ImportRequest{TmuxName: "dup"})
	if err == nil || !strings.Contains(err.Error(), "already imported") {
		t.Errorf("expected already-imported error, got %v", err)
	}
}

func TestImportSessionRejectsMissing(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	_, err := db.ImportSession(conn, fake, db.ImportRequest{TmuxName: "ghost"})
	if err == nil || !strings.Contains(err.Error(), "not found") {
		t.Errorf("expected not-found error, got %v", err)
	}
}

func TestImportSessionInheritsGroupTool(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	if err := db.CreateGroup(conn, db.Group{Path: "work", Name: "work", DefaultTool: "gemini"}); err != nil {
		t.Fatalf("group: %v", err)
	}
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["s"] = ""
	got, err := db.ImportSession(conn, fake, db.ImportRequest{TmuxName: "s", GroupPath: "work"})
	if err != nil {
		t.Fatalf("import: %v", err)
	}
	if got.Tool != "gemini" {
		t.Errorf("tool: got %q want gemini", got.Tool)
	}
}

func tmuxInfo(name, path string) tmux.SessionInfo {
	return tmux.SessionInfo{Name: name, CurrentPath: path}
}
```

Add to file's imports: `"github.com/black-gato/tmux-agent-deck/internal/tmux"`.

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/db/ -run TestImportSession -v
```

Expected: FAIL — `db.ImportSession`, `db.ImportRequest` undefined.

- [ ] **Step 3: Implement `ImportSession`**

Create `internal/db/import.go`:

```go
package db

import (
	"database/sql"
	"errors"
	"fmt"
	"time"

	"github.com/black-gato/tmux-agent-deck/internal/tmux"
	"github.com/google/uuid"
)

// ImportInspector is the subset of tmux.ClientIface needed to import sessions.
type ImportInspector interface {
	ListSessions() ([]string, error)
	SessionInfo(name string) (tmux.SessionInfo, error)
}

type ImportRequest struct {
	TmuxName  string
	Title     string // empty → defaults to TmuxName
	GroupPath string // empty → "my-sessions"
}

func ImportSession(conn *sql.DB, tc ImportInspector, req ImportRequest) (Session, error) {
	if req.TmuxName == "" {
		return Session{}, errors.New("tmux name required")
	}

	names, err := tc.ListSessions()
	if err != nil {
		return Session{}, fmt.Errorf("list tmux sessions: %w", err)
	}
	found := false
	for _, n := range names {
		if n == req.TmuxName {
			found = true
			break
		}
	}
	if !found {
		return Session{}, fmt.Errorf("tmux session %q not found", req.TmuxName)
	}

	if existing, err := GetSessionByTmuxName(conn, req.TmuxName); err == nil {
		return Session{}, fmt.Errorf("tmux session %q already imported (deck id %s)", req.TmuxName, existing.ID)
	}

	title := req.Title
	if title == "" {
		title = req.TmuxName
	}
	groupPath := req.GroupPath
	if groupPath == "" {
		groupPath = "my-sessions"
	}

	tool := "claude"
	if g, err := GetGroup(conn, groupPath); err == nil && g.DefaultTool != "" {
		tool = g.DefaultTool
	}

	info, err := tc.SessionInfo(req.TmuxName)
	if err != nil {
		return Session{}, fmt.Errorf("session info %q: %w", req.TmuxName, err)
	}

	s := Session{
		ID:          uuid.New().String(),
		Title:       title,
		GroupPath:   groupPath,
		TmuxSession: req.TmuxName,
		ProjectPath: info.CurrentPath,
		Tool:        tool,
		Status:      "unknown",
		CreatedAt:   time.Now().Unix(),
		LastActive:  info.Activity.Unix(),
	}
	if info.Activity.IsZero() {
		s.LastActive = 0
	}
	if err := CreateSession(conn, s); err != nil {
		return Session{}, fmt.Errorf("create imported session: %w", err)
	}
	return s, nil
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/db/ -v
```

Expected: PASS (all four `TestImportSession*` plus existing tests).

- [ ] **Step 5: Commit**

```bash
git add internal/db/import.go internal/db/import_test.go
git commit -m "feat(db): ImportSession inserts row for live tmux session"
```

---

### Task 4: `tmux-agent-deck import` CLI command

**Files:**
- Create: `cmd/import.go`
- Modify: `cmd/cmd_test.go`

- [ ] **Step 1: Write the failing test**

Append to `cmd/cmd_test.go`:

```go
func TestImportCommandList(t *testing.T) {
	withTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["alpha"] = ""
	fake.Sessions["beta"] = ""
	// Pre-track one
	mustCreate(t, db.Session{
		ID: "id-a", Title: "alpha-deck", GroupPath: "my-sessions",
		TmuxSession: "alpha", Tool: "claude", Status: "running", CreatedAt: 1,
	})

	var out bytes.Buffer
	if err := cmd.RunWithContextAndClient(context.Background(),
		[]string{"import", "--list"}, &out, fake); err != nil {
		t.Fatalf("import --list: %v", err)
	}
	if got := out.String(); !strings.Contains(got, "beta") || strings.Contains(got, "alpha") {
		t.Errorf("unexpected output: %q", got)
	}
}

func TestImportCommandSingle(t *testing.T) {
	withTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["gamma"] = ""
	fake.SessionInfos["gamma"] = tmux.SessionInfo{Name: "gamma", CurrentPath: "/srv/code"}

	var out bytes.Buffer
	if err := cmd.RunWithContextAndClient(context.Background(),
		[]string{"import", "gamma", "--title", "Gamma"}, &out, fake); err != nil {
		t.Fatalf("import: %v", err)
	}
	s, err := db.GetSessionByTmuxName(testDBConn(t), "gamma")
	if err != nil {
		t.Fatalf("lookup: %v", err)
	}
	if s.Title != "Gamma" || s.ProjectPath != "/srv/code" || s.Status != "unknown" {
		t.Errorf("session row wrong: %+v", s)
	}
}

func TestImportCommandAll(t *testing.T) {
	withTestDB(t)
	fake := testutil.NewFakeTmuxClient()
	fake.Sessions["one"] = ""
	fake.Sessions["two"] = ""
	var out bytes.Buffer
	if err := cmd.RunWithContextAndClient(context.Background(),
		[]string{"import", "--all"}, &out, fake); err != nil {
		t.Fatalf("import --all: %v", err)
	}
	for _, name := range []string{"one", "two"} {
		if _, err := db.GetSessionByTmuxName(testDBConn(t), name); err != nil {
			t.Errorf("missing %s: %v", name, err)
		}
	}
}
```

If `withTestDB`, `mustCreate`, `testDBConn` don't already exist in `cmd/cmd_test.go`, follow the same DB injection pattern other tests use (set `AGENT_DECK_DB` to a temp path) — adapt the existing helpers rather than inventing new ones.

Add imports: `"github.com/black-gato/tmux-agent-deck/internal/tmux"`, `"github.com/black-gato/tmux-agent-deck/internal/testutil"`, `"github.com/black-gato/tmux-agent-deck/internal/db"`, `"context"`, `"strings"`, `"bytes"`.

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./cmd/ -run TestImportCommand -v
```

Expected: FAIL — unknown command `import`.

- [ ] **Step 3: Implement `cmd/import.go`**

```go
package cmd

import (
	"fmt"
	"sort"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/tmux"
	"github.com/spf13/cobra"
)

var importCmd = &cobra.Command{
	Use:   "import [tmux-name]",
	Short: "Import an existing tmux session into the deck",
	Long: `Import a running tmux session that isn't tracked by the deck.

Use --list to print untracked tmux session names (one per line).
Use --all to import every untracked session with defaults.`,
	Args: cobra.MaximumNArgs(1),
	RunE: func(cmd *cobra.Command, args []string) error {
		listOnly, _ := cmd.Flags().GetBool("list")
		all, _ := cmd.Flags().GetBool("all")
		title, _ := cmd.Flags().GetString("title")
		group, _ := cmd.Flags().GetString("group")

		tc := rootTmuxClient
		if tc == nil {
			tc = tmux.NewClient()
		}

		if listOnly {
			names, err := db.ListUntrackedTmuxSessions(rootDB, tc)
			if err != nil {
				return err
			}
			sort.Strings(names)
			for _, n := range names {
				fmt.Fprintln(cmd.OutOrStdout(), n)
			}
			return nil
		}

		if all {
			names, err := db.ListUntrackedTmuxSessions(rootDB, tc)
			if err != nil {
				return err
			}
			sort.Strings(names)
			var firstErr error
			for _, n := range names {
				if _, err := db.ImportSession(rootDB, tc, db.ImportRequest{TmuxName: n, GroupPath: group}); err != nil {
					fmt.Fprintf(cmd.OutOrStdout(), "  FAIL %s: %v\n", n, err)
					if firstErr == nil {
						firstErr = err
					}
					continue
				}
				fmt.Fprintf(cmd.OutOrStdout(), "  imported %s\n", n)
			}
			return firstErr
		}

		if len(args) != 1 {
			return fmt.Errorf("tmux-name required (or use --list / --all)")
		}
		s, err := db.ImportSession(rootDB, tc, db.ImportRequest{
			TmuxName:  args[0],
			Title:     title,
			GroupPath: group,
		})
		if err != nil {
			return err
		}
		fmt.Fprintf(cmd.OutOrStdout(), "Imported %q as %q in group %q\n", s.TmuxSession, s.Title, s.GroupPath)
		return nil
	},
}

func init() {
	importCmd.Flags().Bool("list", false, "List untracked tmux sessions and exit")
	importCmd.Flags().Bool("all", false, "Import every untracked tmux session with defaults")
	importCmd.Flags().String("title", "", "Title for the imported session (defaults to tmux name)")
	importCmd.Flags().StringP("group", "g", "my-sessions", "Group path")
	rootCmd.AddCommand(importCmd)
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./cmd/ -v
```

Expected: PASS (the three new tests plus all existing).

- [ ] **Step 5: Commit**

```bash
git add cmd/import.go cmd/cmd_test.go
git commit -m "feat(cmd): tmux-agent-deck import [--list|--all] CLI"
```

---

### Task 5: TUI key binding `I` → import picker dialog

**Files:**
- Modify: `internal/ui/keys.go` — add binding
- Modify: `internal/ui/app.go` — route action, render dialog
- Create: `internal/ui/import.go` — picker + form state + commit
- Test: `internal/ui/import_test.go`

- [ ] **Step 1: Write the failing test**

Create `internal/ui/import_test.go`:

```go
package ui_test

import (
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/testutil"
	"github.com/black-gato/tmux-agent-deck/internal/tmux"
	tea "github.com/charmbracelet/bubbletea"
)

func TestImportPickerOpensOnCapitalI(t *testing.T) {
	m, fake := openModel(t)
	fake.Sessions["live-foo"] = ""
	fake.Sessions["live-bar"] = ""
	fake.SessionInfos["live-foo"] = tmux.SessionInfo{Name: "live-foo", CurrentPath: "/tmp/foo"}
	fake.SessionInfos["live-bar"] = tmux.SessionInfo{Name: "live-bar", CurrentPath: "/tmp/bar"}

	m = sendKey(m, rune_('I'))

	if m.Mode() != "import-picker" {
		t.Fatalf("expected import-picker mode, got %q", m.Mode())
	}
	if got := m.ImportCandidates(); len(got) != 2 {
		t.Errorf("expected 2 candidates, got %v", got)
	}
}

func TestImportPickerNoUntrackedShowsMessage(t *testing.T) {
	m, _ := openModel(t)
	m = sendKey(m, rune_('I'))
	if m.Mode() == "import-picker" {
		t.Errorf("should not enter picker with no untracked sessions")
	}
	if !strings.Contains(m.StatusMessage(), "no untracked") {
		t.Errorf("expected status message, got %q", m.StatusMessage())
	}
}

func TestImportPickerEnterOpensForm(t *testing.T) {
	m, fake := openModel(t)
	fake.Sessions["pickme"] = ""
	fake.SessionInfos["pickme"] = tmux.SessionInfo{Name: "pickme", CurrentPath: "/srv"}

	m = sendKey(m, rune_('I'))
	m = sendKey(m, key(tea.KeyEnter))

	if m.Mode() != "import-form" {
		t.Fatalf("expected import-form mode, got %q", m.Mode())
	}
	if m.ImportFormTitle() != "pickme" {
		t.Errorf("expected title prefilled, got %q", m.ImportFormTitle())
	}
}

func TestImportFormCommitInsertsRow(t *testing.T) {
	m, fake := openModel(t)
	fake.Sessions["adopt-me"] = ""
	fake.SessionInfos["adopt-me"] = tmux.SessionInfo{Name: "adopt-me", CurrentPath: "/work"}

	m = sendKey(m, rune_('I'))
	m = sendKey(m, key(tea.KeyEnter)) // pick first
	m = sendKey(m, key(tea.KeyEnter)) // commit form with defaults

	conn := testDBFromModel(m)
	got, err := db.GetSessionByTmuxName(conn, "adopt-me")
	if err != nil {
		t.Fatalf("lookup: %v", err)
	}
	if got.Title != "adopt-me" || got.ProjectPath != "/work" {
		t.Errorf("session wrong: %+v", got)
	}
	if m.Mode() != "" {
		t.Errorf("expected mode cleared after commit, got %q", m.Mode())
	}
}
```

If `openModel`, `sendKey`, `key`, `rune_`, `testDBFromModel` don't already exist with these signatures, mirror the helpers used in `internal/ui/form_test.go` / `dialog_test.go`. Add accessors `Mode()`, `StatusMessage()`, `ImportCandidates() []string`, `ImportFormTitle() string` on `Model` (these are test-only, but the codebase already exposes similar helpers — match style).

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/ui/ -run TestImport -v
```

Expected: FAIL — `import-picker` mode does not exist.

- [ ] **Step 3: Add `I` binding in `internal/ui/keys.go`**

Insert in `KeyBindings` (under "Editing"):

```go
{Section: "Editing", Rune: 'I', Key: "I", Description: "Import tmux session", Action: "import"},
```

- [ ] **Step 4: Add import state to `Model` and wire the action**

In `internal/ui/import.go` (new file):

```go
package ui

import (
	"sort"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/tmux"
)

type importState struct {
	candidates []importCandidate
	cursor     int
	selected   string
	title      string
	group      string
	formErr    string
}

type importCandidate struct {
	Name string
	Path string
}

func (m *Model) openImportPicker() {
	names, err := db.ListUntrackedTmuxSessions(m.db, m.tmux)
	if err != nil {
		m.status = "import: " + err.Error()
		return
	}
	if len(names) == 0 {
		m.status = "no untracked tmux sessions"
		return
	}
	sort.Strings(names)
	cands := make([]importCandidate, 0, len(names))
	for _, n := range names {
		path := ""
		if info, err := m.tmux.SessionInfo(n); err == nil {
			path = info.CurrentPath
		}
		cands = append(cands, importCandidate{Name: n, Path: path})
	}
	m.imp = importState{candidates: cands}
	m.mode = "import-picker"
}

func (m *Model) commitImport() error {
	req := db.ImportRequest{
		TmuxName:  m.imp.selected,
		Title:     m.imp.title,
		GroupPath: m.imp.group,
	}
	if _, err := db.ImportSession(m.db, m.tmux, req); err != nil {
		m.imp.formErr = err.Error()
		return err
	}
	m.mode = ""
	m.imp = importState{}
	m.Reload()
	return nil
}

// helper for tests
func (m *Model) ImportCandidates() []string {
	out := make([]string, 0, len(m.imp.candidates))
	for _, c := range m.imp.candidates {
		out = append(out, c.Name)
	}
	return out
}

func (m *Model) ImportFormTitle() string { return m.imp.title }
```

Add an `imp importState` field to `Model` in `internal/ui/app.go`, and an `m.tmux` field if not present (`tmux.ClientIface`). The model already holds `m.db`; add `m.tmux` if missing — wire it in the constructor where the model is built. Confirm by searching `app.go` for the constructor.

Route the action in `Model.Update`'s key handler:

```go
case "import":
    m.openImportPicker()
    return m, nil
```

- [ ] **Step 5: Handle picker keys**

Extend the `Update` switch (where existing modes like `"new-session"` are handled). For `m.mode == "import-picker"`:

```go
case "import-picker":
    switch msg.Type {
    case tea.KeyUp:
        if m.imp.cursor > 0 {
            m.imp.cursor--
        }
    case tea.KeyDown:
        if m.imp.cursor < len(m.imp.candidates)-1 {
            m.imp.cursor++
        }
    case tea.KeyEsc:
        m.mode = ""
        m.imp = importState{}
    case tea.KeyEnter:
        c := m.imp.candidates[m.imp.cursor]
        m.imp.selected = c.Name
        m.imp.title = c.Name
        m.imp.group = m.currentGroupPath() // existing helper; if absent, fall back to "my-sessions"
        if m.imp.group == "" {
            m.imp.group = "my-sessions"
        }
        m.mode = "import-form"
    }
    return m, nil
```

For `m.mode == "import-form"`:

```go
case "import-form":
    switch msg.Type {
    case tea.KeyEsc:
        m.mode = ""
        m.imp = importState{}
    case tea.KeyTab:
        // toggle focus between title and group (track in imp.focus int)
        m.imp.focus = 1 - m.imp.focus
    case tea.KeyBackspace:
        if m.imp.focus == 0 && len(m.imp.title) > 0 {
            m.imp.title = m.imp.title[:len(m.imp.title)-1]
        } else if m.imp.focus == 1 && len(m.imp.group) > 0 {
            m.imp.group = m.imp.group[:len(m.imp.group)-1]
        }
    case tea.KeyEnter:
        m.commitImport()
    default:
        if len(msg.Runes) == 1 {
            r := msg.Runes[0]
            if m.imp.focus == 0 {
                m.imp.title += string(r)
            } else {
                m.imp.group += string(r)
            }
        }
    }
    return m, nil
```

Add `focus int` to `importState`. (The two-field form is intentionally simpler than `form.go`'s 8-field machinery — no candidate completion, no dimming.)

- [ ] **Step 6: Render the picker and form in `View`**

Add View branches for the two modes, matching the visual style of other dialogs. Minimum:

```
IMPORT TMUX SESSION
  scratch-foo     /tmp
> work-bar        /Users/.../some-repo

enter: select   esc: cancel
```

```
IMPORT: scratch-foo
> TITLE  scratch-foo
  GROUP  my-sessions

tab: next   enter: import   esc: cancel
formErr: <red if set>
```

Reuse `formHeaderStyle`, `formLabelActive`, `formErrStyle` from `form.go` — they're already package-level vars.

- [ ] **Step 7: Run tests to verify they pass**

```bash
go test ./internal/ui/ -v
```

Expected: PASS (all `TestImport*` plus existing).

- [ ] **Step 8: Run full test suite**

```bash
go test ./...
go vet ./...
```

Expected: clean.

- [ ] **Step 9: Commit**

```bash
git add internal/ui/keys.go internal/ui/app.go internal/ui/import.go internal/ui/import_test.go
git commit -m "feat(ui): I opens import picker + 2-field form for untracked tmux sessions"
```

---

### Task 6: Help table + roadmap update

**Files:**
- Modify: `CLAUDE.md` — append row to roadmap table
- Modify: `internal/ui/app.go` (or wherever the `?` help overlay reads `KeyBindings`) — verify the new `I` row renders without changes; if not, fix it

- [ ] **Step 1: Verify `?` overlay shows `I`**

```bash
go build -o /tmp/tad . && AGENT_DECK_DB=/tmp/tad.db /tmp/tad &
# In another terminal, press `?` inside the TUI.
```

Confirm "I — Import tmux session" appears under Editing.

- [ ] **Step 2: Add roadmap entry to CLAUDE.md**

Append to the roadmap table in `CLAUDE.md` (between "Vim Mode Auto-Detection" and "Session Worktree Options" lines, alphabetic by feature within the trailing block is fine — just keep the table well-formed):

```
| Import Tmux Sessions | Adopt running tmux sessions not tracked by the deck (`I` in TUI; `import` CLI) | complete | [spec](docs/superpowers/specs/2026-05-23-import-tmux-sessions-design.md) | [plan](docs/superpowers/plans/2026-05-23-import-tmux-sessions.md) |
```

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add Import Tmux Sessions row to roadmap"
```

---

### Verification (manual, end-to-end)

After all tasks complete:

```bash
go build -o tmux-agent-deck .
tmux new -d -s scratch-foo -c /tmp
tmux new -d -s scratch-bar -c "$HOME"

./tmux-agent-deck import --list
# expect:
# scratch-bar
# scratch-foo

./tmux-agent-deck import scratch-foo --title "Foo"
# expect: Imported "scratch-foo" as "Foo" in group "my-sessions"

./tmux-agent-deck list
# expect Foo present with TmuxSession=scratch-foo

./tmux-agent-deck
# TUI: press `I`, see picker with scratch-bar; Enter; commit with default title
# After one poll interval, scratch-bar shows real status (running/idle/waiting).
# Press Enter on the row to attach — should land in the live tmux session with its original scrollback.
```

Cleanup:

```bash
tmux kill-session -t scratch-foo
tmux kill-session -t scratch-bar
```

Confirm: `go test ./...` and `go vet ./...` clean.

---

## Self-review notes

- Spec coverage: tmux read (Task 1), DB diff (Task 2), insert helper (Task 3), CLI (Task 4), TUI picker+form (Task 5), discoverability/docs (Task 6) — all spec sections covered.
- The `ImportInspector` interface (Task 3) is a superset of `TmuxLister` (Task 2). Keep both: the diff helper genuinely only needs `ListSessions`, and the smaller interface keeps tests honest.
- `Status="unknown"` relies on the poller overwriting it on next tick. If the existing poller doesn't handle `"unknown"` as a valid starting state, treat that as a separate bug — but a scan of the poller code in Phase 1 didn't reveal a status whitelist; it computes status from pane output unconditionally.
- The `m.tmux` field on `Model`: confirm during Task 5 whether the TUI model already holds a `tmux.ClientIface`. If it does (it must, to attach), reuse it; if it holds the concrete `*tmux.Client`, no change needed — `ImportInspector` is satisfied either way once `SessionInfo` is on `ClientIface`.
