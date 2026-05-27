# Hook Status Protocol Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix BUG-021 by replacing the DB-coupled hook handler with a file-based hook status protocol (env-var identity + atomic JSON status files), and surface hook-driven status changes in the TUI within ~250ms.

**Architecture:** The deck sets `AGENTDECK_INSTANCE_ID=<session-id>` via tmux `new-session -e` so Claude's hook subprocess inherits it. The `hook-handler` command writes an atomic JSON file `~/.tmux-agent-deck/hooks/{id}.json` (no DB, no tmux lookup). A new 250ms goroutine in the poller cold-reads those files behind a directory-mtime gate, writes status to the DB, and pushes an immediate UI refresh. A fresh hook status overrides pane-scrape `DetectStatus`; otherwise the existing 1s pane loop decides. Conductor notification stays in the 1s loop, edge-tracked separately so the fast loop never double-fires it.

**Tech Stack:** Go 1.22+, Cobra, Bubble Tea, modernc SQLite, tmux 3.x (`-e` requires ≥3.0; target runs 3.6b).

**Spec:** [docs/superpowers/specs/2026-05-26-hook-status-protocol-design.md](../specs/2026-05-26-hook-status-protocol-design.md)

---

## File Structure

| File | Responsibility | Action |
|------|----------------|--------|
| `internal/hook/hook.go` | Parse stdin hook payload (existing) + event→status mapping | Modify |
| `internal/hook/status.go` | `HookStatus` type, hooks dir path, atomic write, read/list, freshness | Create |
| `internal/hook/status_test.go` | Tests for status.go | Create |
| `internal/hook/hook_test.go` | Tests for event→status mapping | Create (if absent) |
| `cmd/hookhandler.go` | Hook entrypoint: env-var identity → write status file; no DB | Rewrite |
| `cmd/hookhandler_test.go` | Tests for the rewritten handler | Create |
| `internal/tmux/client.go` | `NewSession` gains `instanceID`; `-e` injection; `ClientIface` | Modify |
| `internal/tmux/client_test.go` | Test pure `newSessionArgs` builder | Create (if absent) |
| `internal/testutil/tmux.go` | `FakeTmuxClient.NewSession` signature | Modify |
| `cmd/session.go` | Pass `s.ID` to `NewSession` | Modify |
| `internal/ui/app.go` | Pass `s.ID` to `NewSession`; handle `RefreshMsg` | Modify |
| `internal/state/poller.go` | Hook map, `SetRefresh`, fast loop, merge precedence, edge tracker | Modify |
| `internal/state/poller_test.go` | Tests for fast loop + merge + edge tracking | Modify |
| `cmd/root.go` | Wire `poller.SetRefresh` to `program.Send` | Modify |
| `cmd/installhooks.go` | Remove legacy `agent-deck hook-handler` entries | Modify |
| `docs/bugs.md` | Close BUG-021 | Modify |

---

## Task 1: Hook status package (file protocol primitives)

**Files:**
- Modify: `internal/hook/hook.go`
- Create: `internal/hook/status.go`
- Test: `internal/hook/status_test.go`, `internal/hook/hook_test.go`

- [ ] **Step 1: Write the failing test for event→status mapping**

Create `internal/hook/hook_test.go`:

```go
package hook_test

import (
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/hook"
)

func TestEventToStatus(t *testing.T) {
	cases := map[string]string{
		"SessionStart":     "waiting",
		"UserPromptSubmit": "running",
		"Stop":             "waiting",
		"SessionEnd":       "dead",
		"PreCompact":       "",
		"Unknown":          "",
	}
	for event, want := range cases {
		if got := hook.EventToStatus(event); got != want {
			t.Errorf("EventToStatus(%q) = %q, want %q", event, got, want)
		}
	}
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `go test ./internal/hook/ -run TestEventToStatus -v`
Expected: FAIL — `undefined: hook.EventToStatus`.

- [ ] **Step 3: Add `EventToStatus` to `internal/hook/hook.go`**

Append to `internal/hook/hook.go`:

```go
// EventToStatus maps a Claude Code hook event name to a deck status string
// ("running", "waiting", "dead"), or "" for events we do not act on.
func EventToStatus(event string) string {
	switch event {
	case "SessionStart", "Stop":
		return "waiting"
	case "UserPromptSubmit":
		return "running"
	case "SessionEnd":
		return "dead"
	default:
		return ""
	}
}
```

- [ ] **Step 4: Run it to verify it passes**

Run: `go test ./internal/hook/ -run TestEventToStatus -v`
Expected: PASS.

- [ ] **Step 5: Write the failing test for write/read/freshness**

Create `internal/hook/status_test.go`:

```go
package hook_test

import (
	"testing"
	"time"

	"github.com/black-gato/tmux-agent-deck/internal/hook"
)

func TestWriteReadStatus(t *testing.T) {
	dir := t.TempDir()

	if err := hook.WriteStatus(dir, "abc-123", "running", "claude-sess", "UserPromptSubmit"); err != nil {
		t.Fatalf("WriteStatus: %v", err)
	}

	got, ok := hook.ReadStatus(dir, "abc-123")
	if !ok {
		t.Fatal("ReadStatus: not found after write")
	}
	if got.Status != "running" || got.Event != "UserPromptSubmit" || got.SessionID != "claude-sess" {
		t.Errorf("ReadStatus = %+v", got)
	}
	if got.UpdatedAt.IsZero() {
		t.Error("UpdatedAt not set")
	}
}

func TestReadStatusMissing(t *testing.T) {
	if _, ok := hook.ReadStatus(t.TempDir(), "nope"); ok {
		t.Error("expected not-found for missing file")
	}
}

func TestReadStatusRejectsTraversal(t *testing.T) {
	if _, ok := hook.ReadStatus(t.TempDir(), "../escape"); ok {
		t.Error("expected traversal id to be rejected")
	}
}

func TestFresh(t *testing.T) {
	now := time.Unix(1_000_000, 0)
	fresh := hook.HookStatus{UpdatedAt: now.Add(-time.Minute)}
	stale := hook.HookStatus{UpdatedAt: now.Add(-3 * time.Minute)}
	if !hook.Fresh(fresh, now) {
		t.Error("1m-old status should be fresh")
	}
	if hook.Fresh(stale, now) {
		t.Error("3m-old status should be stale")
	}
}

func TestDeckStatus(t *testing.T) {
	cases := map[string]string{"running": "running", "waiting": "waiting", "dead": "error"}
	for raw, want := range cases {
		hs := hook.HookStatus{Status: raw}
		if got := hs.DeckStatus(); got != want {
			t.Errorf("DeckStatus(%q) = %q, want %q", raw, got, want)
		}
	}
}

func TestListStatuses(t *testing.T) {
	dir := t.TempDir()
	_ = hook.WriteStatus(dir, "one", "running", "", "UserPromptSubmit")
	_ = hook.WriteStatus(dir, "two", "waiting", "", "Stop")
	all := hook.ListStatuses(dir)
	if len(all) != 2 || all["one"].Status != "running" || all["two"].Status != "waiting" {
		t.Errorf("ListStatuses = %+v", all)
	}
}
```

- [ ] **Step 6: Run it to verify it fails**

Run: `go test ./internal/hook/ -run 'TestWriteReadStatus|TestReadStatus|TestFresh|TestDeckStatus|TestListStatuses' -v`
Expected: FAIL — undefined symbols.

- [ ] **Step 7: Create `internal/hook/status.go`**

```go
package hook

import (
	"encoding/json"
	"os"
	"path/filepath"
	"regexp"
	"strings"
	"time"
)

// FreshWindow is how long a hook status is trusted before the pane-scrape
// fallback takes over. Matches upstream agent-deck's HookFastPathWindow.
const FreshWindow = 2 * time.Minute

// validInstanceID guards against path traversal via crafted IDs.
var validInstanceID = regexp.MustCompile(`^[a-zA-Z0-9][a-zA-Z0-9_.-]*$`)

// HookStatus is the decoded contents of a hook status file.
type HookStatus struct {
	Status    string    // "running", "waiting", "dead"
	SessionID string    // Claude session id (informational)
	Event     string    // originating hook event name
	UpdatedAt time.Time // decoded from the file's unix ts
}

// statusFile is the on-disk JSON shape.
type statusFile struct {
	Status    string `json:"status"`
	SessionID string `json:"session_id,omitempty"`
	Event     string `json:"event"`
	Timestamp int64  `json:"ts"`
}

// DeckStatus maps the wire status onto a deck status constant. "dead" becomes
// "error"; "running"/"waiting" pass through unchanged.
func (h HookStatus) DeckStatus() string {
	if h.Status == "dead" {
		return "error"
	}
	return h.Status
}

// Fresh reports whether the status is within FreshWindow of now.
func Fresh(h HookStatus, now time.Time) bool {
	return !h.UpdatedAt.IsZero() && now.Sub(h.UpdatedAt) <= FreshWindow
}

// HooksDir is the directory that holds per-instance status files. Home-anchored
// so the hook subprocess and the deck resolve the same path; falls back to
// $TMPDIR when $HOME is unavailable.
func HooksDir() string {
	home, err := os.UserHomeDir()
	if err != nil || home == "" {
		return filepath.Join(os.TempDir(), ".tmux-agent-deck", "hooks")
	}
	return filepath.Join(home, ".tmux-agent-deck", "hooks")
}

func validID(instanceID string) bool {
	return validInstanceID.MatchString(instanceID) && !strings.Contains(instanceID, "..")
}

// WriteStatus atomically writes a status file for one instance via tmp+rename.
// The rename is load-bearing: it bumps the directory mtime, which the poller's
// fast loop uses as a change gate.
func WriteStatus(dir, instanceID, status, sessionID, event string) error {
	if !validID(instanceID) {
		return nil
	}
	if err := os.MkdirAll(dir, 0700); err != nil {
		return err
	}
	data, err := json.Marshal(statusFile{
		Status:    status,
		SessionID: sessionID,
		Event:     event,
		Timestamp: time.Now().Unix(),
	})
	if err != nil {
		return err
	}
	final := filepath.Join(dir, instanceID+".json")
	tmp := final + ".tmp"
	if err := os.WriteFile(tmp, data, 0600); err != nil {
		return err
	}
	return os.Rename(tmp, final)
}

// ReadStatus reads one instance's status file. Returns (zero, false) if the
// file is missing, the id is invalid, or the JSON is corrupt.
func ReadStatus(dir, instanceID string) (HookStatus, bool) {
	if !validID(instanceID) {
		return HookStatus{}, false
	}
	data, err := os.ReadFile(filepath.Join(dir, instanceID+".json"))
	if err != nil {
		return HookStatus{}, false
	}
	var f statusFile
	if err := json.Unmarshal(data, &f); err != nil || f.Status == "" {
		return HookStatus{}, false
	}
	return HookStatus{
		Status:    f.Status,
		SessionID: f.SessionID,
		Event:     f.Event,
		UpdatedAt: time.Unix(f.Timestamp, 0),
	}, true
}

// ListStatuses reads every *.json status file in dir, keyed by instance id.
// Corrupt or mid-write files are skipped.
func ListStatuses(dir string) map[string]HookStatus {
	out := make(map[string]HookStatus)
	entries, err := os.ReadDir(dir)
	if err != nil {
		return out
	}
	for _, e := range entries {
		if e.IsDir() || filepath.Ext(e.Name()) != ".json" {
			continue
		}
		id := strings.TrimSuffix(e.Name(), ".json")
		if hs, ok := ReadStatus(dir, id); ok {
			out[id] = hs
		}
	}
	return out
}
```

- [ ] **Step 8: Run it to verify it passes**

Run: `go test ./internal/hook/ -v`
Expected: PASS (all tests).

- [ ] **Step 9: Commit**

```bash
git add internal/hook/
git commit -m "feat(hook): file-based status protocol primitives"
```

---

## Task 2: Rewrite the hook handler to write status files

**Files:**
- Rewrite: `cmd/hookhandler.go`
- Test: `cmd/hookhandler_test.go`
- Reference: `internal/hook/status.go` (Task 1)

The current handler opens the DB and pokes the conductor's pane. The rewrite drops both: identity comes from `AGENTDECK_INSTANCE_ID`, output is a status file. Conductor notification moves to the poller (Task 5).

- [ ] **Step 1: Write the failing test**

Create `cmd/hookhandler_test.go`:

```go
package cmd

import (
	"strings"
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/hook"
)

func TestRunHookHandlerWritesStatus(t *testing.T) {
	dir := t.TempDir()
	payload := `{"hook_event_name":"UserPromptSubmit","session_id":"claude-1"}`

	runHookHandler(strings.NewReader(payload), "inst-42", dir)

	got, ok := hook.ReadStatus(dir, "inst-42")
	if !ok {
		t.Fatal("no status file written")
	}
	if got.Status != "running" {
		t.Errorf("status = %q, want running", got.Status)
	}
}

func TestRunHookHandlerNoInstanceID(t *testing.T) {
	dir := t.TempDir()
	runHookHandler(strings.NewReader(`{"hook_event_name":"Stop"}`), "", dir)
	if len(hook.ListStatuses(dir)) != 0 {
		t.Error("expected no write when instance id is empty")
	}
}

func TestRunHookHandlerIgnoredEvent(t *testing.T) {
	dir := t.TempDir()
	runHookHandler(strings.NewReader(`{"hook_event_name":"PreCompact"}`), "inst-1", dir)
	if len(hook.ListStatuses(dir)) != 0 {
		t.Error("expected no write for unmapped event")
	}
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `go test ./cmd/ -run TestRunHookHandler -v`
Expected: FAIL — `undefined: runHookHandler`.

- [ ] **Step 3: Rewrite `cmd/hookhandler.go`**

Replace the entire file with:

```go
package cmd

import (
	"io"
	"os"

	"github.com/black-gato/tmux-agent-deck/internal/hook"
	"github.com/spf13/cobra"
)

const maxHookPayloadSize = 1 << 20 // 1 MB

var hookHandlerCmd = &cobra.Command{
	Use:   "hook-handler",
	Short: "Handle Claude Code lifecycle hook events (called by Claude Code hooks)",
	RunE: func(cmd *cobra.Command, args []string) error {
		runHookHandler(os.Stdin, os.Getenv("AGENTDECK_INSTANCE_ID"), hook.HooksDir())
		return nil
	},
}

// runHookHandler reads a hook payload, maps it to a status, and writes a status
// file for instanceID into dir. It never touches the DB or tmux, and never
// errors out (hooks must not block Claude Code). A blank instanceID means the
// session is not deck-managed, so we no-op.
func runHookHandler(r io.Reader, instanceID, dir string) {
	if instanceID == "" {
		return
	}
	data, err := io.ReadAll(io.LimitReader(r, maxHookPayloadSize))
	if err != nil || len(data) == 0 {
		return
	}
	event, err := hook.ParseEvent(stringsReader(data))
	if err != nil || event.EventName == "" {
		return
	}
	status := hook.EventToStatus(event.EventName)
	if status == "" {
		return
	}
	_ = hook.WriteStatus(dir, instanceID, status, event.SessionID, event.EventName)
}

func init() {
	rootCmd.AddCommand(hookHandlerCmd)
}
```

- [ ] **Step 4: Add `SessionID` to the parsed event and a bytes reader helper**

In `internal/hook/hook.go`, extend `HookEvent` and `hookPayload` with the session id:

```go
type HookEvent struct {
	EventName string
	ToolName  string
	Message   string
	SessionID string
}

type hookPayload struct {
	EventName string `json:"hook_event_name"`
	ToolName  string `json:"tool_name"`
	Message   string `json:"message"`
	SessionID string `json:"session_id"`
}
```

And set it in `ParseEvent`'s return:

```go
	return HookEvent{
		EventName: p.EventName,
		ToolName:  p.ToolName,
		Message:   p.Message,
		SessionID: p.SessionID,
	}, nil
```

In `cmd/hookhandler.go` add the small helper (avoids importing bytes in two spots):

```go
import "bytes"

func stringsReader(b []byte) io.Reader { return bytes.NewReader(b) }
```

(Adjust the import block to include `bytes`.)

- [ ] **Step 5: Run it to verify it passes**

Run: `go test ./cmd/ -run TestRunHookHandler -v && go test ./internal/hook/ -v`
Expected: PASS.

- [ ] **Step 6: Verify nothing else referenced the deleted symbols**

Run: `go build ./...`
Expected: builds clean. If `internal/hook/hook.go` no longer compiles because old tests reference removed fields, update them. (The old `cmd/hookhandler.go` symbols `hookSender`, `hookMessage`, `resolveCurrentTmuxSession`, `runHookHandlerWith` are gone — grep to confirm no references remain: `grep -rn "runHookHandlerWith\|hookMessage\|resolveCurrentTmuxSession" --include=*.go .`)

- [ ] **Step 7: Commit**

```bash
git add cmd/hookhandler.go cmd/hookhandler_test.go internal/hook/hook.go
git commit -m "feat(hook): rewrite hook-handler to write status files (no DB/tmux)"
```

---

## Task 3: Inject `AGENTDECK_INSTANCE_ID` via tmux `-e`

**Files:**
- Modify: `internal/tmux/client.go` (`ClientIface`, `NewSession`, add `newSessionArgs`)
- Create: `internal/tmux/client_test.go`
- Modify: `internal/testutil/tmux.go`
- Modify: `cmd/session.go:27`, `internal/ui/app.go:993`

- [ ] **Step 1: Write the failing test for the args builder**

Create `internal/tmux/client_test.go`:

```go
package tmux

import (
	"strings"
	"testing"
)

func TestNewSessionArgsIncludesInstanceEnv(t *testing.T) {
	args := newSessionArgs("sess", "/work", "claude-dangerous", "", "inst-7")
	joined := strings.Join(args, " ")
	if !strings.Contains(joined, "-e AGENTDECK_INSTANCE_ID=inst-7") {
		t.Errorf("missing -e env flag: %v", args)
	}
	if !strings.Contains(joined, "claude --dangerously-skip-permissions") {
		t.Errorf("missing resolved launch command: %v", args)
	}
	// -e must appear before the launch command (it is a new-session flag).
	if strings.Index(joined, "-e") > strings.Index(joined, "claude") {
		t.Errorf("-e must precede the command: %v", args)
	}
}

func TestNewSessionArgsNoInstanceID(t *testing.T) {
	args := newSessionArgs("sess", "/work", "shell", "", "")
	if strings.Contains(strings.Join(args, " "), "AGENTDECK_INSTANCE_ID") {
		t.Errorf("should not set env when instance id is empty: %v", args)
	}
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `go test ./internal/tmux/ -run TestNewSessionArgs -v`
Expected: FAIL — `undefined: newSessionArgs`.

- [ ] **Step 3: Refactor `NewSession` to use a pure args builder**

In `internal/tmux/client.go`, replace the existing `NewSession` (lines ~60-66) with:

```go
func newSessionArgs(name, startDir, tool, toolFlags, instanceID string) []string {
	args := []string{"new-session", "-d", "-s", name, "-c", startDir}
	if instanceID != "" {
		args = append(args, "-e", "AGENTDECK_INSTANCE_ID="+instanceID)
	}
	if launch := buildLaunchCommand(tool, toolFlags); launch != "" {
		args = append(args, launch)
	}
	return args
}

func (c *Client) NewSession(name, startDir, tool, toolFlags, instanceID string) error {
	return runCmd("tmux", newSessionArgs(name, startDir, tool, toolFlags, instanceID)...)
}
```

And update the interface (line ~17):

```go
	NewSession(name, startDir, tool, toolFlags, instanceID string) error
```

- [ ] **Step 4: Update the fake and both call sites**

In `internal/testutil/tmux.go`:

```go
type NewSessionCall struct {
	Name       string
	Dir        string
	Tool       string
	ToolFlags  string
	InstanceID string
}

func (f *FakeTmuxClient) NewSession(name, startDir, tool, toolFlags, instanceID string) error {
	f.NewSessionCalls = append(f.NewSessionCalls, NewSessionCall{name, startDir, tool, toolFlags, instanceID})
	if f.NewSessionErr != nil {
		return f.NewSessionErr
	}
	f.Sessions[name] = ""
	return nil
}
```

In `cmd/session.go` line 27, change:

```go
		if err := tc.NewSession(tmuxName, s.ProjectPath, s.Tool, s.ToolFlags, s.ID); err != nil {
```

In `internal/ui/app.go` line ~993, change:

```go
	if err := m.tmuxC.NewSession(tmuxName, s.ProjectPath, s.Tool, s.ToolFlags, s.ID); err != nil {
```

- [ ] **Step 5: Run tests + build**

Run: `go build ./... && go test ./internal/tmux/ -run TestNewSessionArgs -v`
Expected: PASS, clean build. Fix any other `NewSession(` call the build flags (grep: `grep -rn "\.NewSession(" --include=*.go .`).

- [ ] **Step 6: Run the full suite to catch fake-signature fallout**

Run: `go test ./...`
Expected: PASS. Any test constructing `NewSessionCall` positionally needs the new 5th field.

- [ ] **Step 7: Commit**

```bash
git add internal/tmux/client.go internal/tmux/client_test.go internal/testutil/tmux.go cmd/session.go internal/ui/app.go
git commit -m "feat(tmux): set AGENTDECK_INSTANCE_ID on session launch via -e"
```

---

## Task 4: Poller fast hook loop (250ms, dir-mtime gated)

**Files:**
- Modify: `internal/state/poller.go`
- Test: `internal/state/poller_test.go`

Adds the fast loop that reads hook files and updates DB status + UI refresh. Merge precedence and notification edge-tracking come in Task 5.

- [ ] **Step 1: Write the failing test**

Add to `internal/state/poller_test.go`:

```go
func TestPollHooksOnceUpdatesStatus(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	must(t, db.CreateSession(conn, db.Session{ID: "inst-1", Title: "w", TmuxSession: "tmux-1", Tool: "claude", Status: "running"}))

	hooksDir := t.TempDir()
	must(t, hook.WriteStatus(hooksDir, "inst-1", "waiting", "", "Stop"))

	tc := testutil.NewFakeTmuxClient()
	p := state.NewWithClockInterval(conn, tc, nil, time.Now, time.Second)
	p.SetHooksDir(hooksDir)

	changed := p.PollHooksOnce()
	if !changed {
		t.Fatal("expected PollHooksOnce to report a change")
	}
	s, _ := db.GetSession(conn, "inst-1")
	if s.Status != "waiting" {
		t.Errorf("status = %q, want waiting", s.Status)
	}
}

func TestPollHooksOnceStaleIgnored(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	must(t, db.CreateSession(conn, db.Session{ID: "inst-1", TmuxSession: "tmux-1", Tool: "claude", Status: "running"}))

	hooksDir := t.TempDir()
	must(t, hook.WriteStatus(hooksDir, "inst-1", "waiting", "", "Stop"))

	tc := testutil.NewFakeTmuxClient()
	// Clock 3 minutes ahead → the just-written file is stale (>FreshWindow).
	p := state.NewWithClockInterval(conn, tc, nil, func() time.Time { return time.Now().Add(3 * time.Minute) }, time.Second)
	p.SetHooksDir(hooksDir)

	p.PollHooksOnce()
	s, _ := db.GetSession(conn, "inst-1")
	if s.Status != "running" {
		t.Errorf("stale hook should not change status; got %q", s.Status)
	}
}
```

Ensure the test file imports `"github.com/black-gato/tmux-agent-deck/internal/hook"` and has a `must(t, err)` helper (add if absent):

```go
func must(t *testing.T, err error) {
	t.Helper()
	if err != nil {
		t.Fatal(err)
	}
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `go test ./internal/state/ -run TestPollHooksOnce -v`
Expected: FAIL — `undefined: SetHooksDir` / `PollHooksOnce`.

- [ ] **Step 3: Add hook fields + methods to the poller**

In `internal/state/poller.go`, add to the imports: `"github.com/black-gato/tmux-agent-deck/internal/hook"`.

Add fields to the `Poller` struct:

```go
	hooksDir      string
	hookStatus    map[string]hook.HookStatus // instance id -> latest fresh hook status
	lastHooksMod  time.Time
	refresh       func()
```

In `NewWithClockInterval`, initialise them:

```go
		hooksDir:   hook.HooksDir(),
		hookStatus: make(map[string]hook.HookStatus),
```

Add accessors and the loop body:

```go
// SetHooksDir overrides the hooks directory (tests).
func (p *Poller) SetHooksDir(dir string) {
	p.mu.Lock()
	p.hooksDir = dir
	p.mu.Unlock()
}

// SetRefresh registers a callback fired when the fast hook loop changes a
// session's status, so the TUI can re-render immediately.
func (p *Poller) SetRefresh(fn func()) {
	p.mu.Lock()
	p.refresh = fn
	p.mu.Unlock()
}

const hookInterval = 250 * time.Millisecond
const hookForcedRescan = 5 * time.Second

// PollHooksOnce reads changed hook files (behind a directory-mtime gate),
// records fresh statuses, applies them to DB status, and returns whether any
// session's status changed.
func (p *Poller) PollHooksOnce() bool {
	p.mu.RLock()
	dir := p.hooksDir
	lastMod := p.lastHooksMod
	p.mu.RUnlock()

	info, err := os.Stat(dir)
	if err != nil {
		return false
	}
	forced := p.now().Sub(lastMod) > hookForcedRescan
	if !forced && info.ModTime().Equal(lastMod) {
		return false
	}

	statuses := hook.ListStatuses(dir)
	now := p.now()

	p.mu.Lock()
	p.lastHooksMod = info.ModTime()
	p.hookStatus = make(map[string]hook.HookStatus, len(statuses))
	for id, hs := range statuses {
		if hook.Fresh(hs, now) {
			p.hookStatus[id] = hs
		}
	}
	fresh := p.hookStatus
	p.mu.Unlock()

	changed := false
	sessions, err := db.ListSessions(p.conn)
	if err != nil {
		return false
	}
	for _, s := range sessions {
		hs, ok := fresh[s.ID]
		if !ok || s.Status == tmux.StatusStopped {
			continue
		}
		want := hs.DeckStatus()
		if want != "" && want != s.Status {
			if err := db.UpdateSessionStatus(p.conn, s.ID, want); err != nil {
				log.Printf("poller: hook status %q: %v", s.ID, err)
				continue
			}
			changed = true
		}
	}
	return changed
}
```

Add `"os"` to the imports if not present.

- [ ] **Step 4: Start the fast loop alongside the main ticker**

In `Start()`, add a second goroutine:

```go
func (p *Poller) Start() {
	go func() {
		ticker := time.NewTicker(p.interval)
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
	go func() {
		ticker := time.NewTicker(hookInterval)
		defer ticker.Stop()
		for {
			select {
			case <-ticker.C:
				if p.PollHooksOnce() {
					p.mu.RLock()
					fn := p.refresh
					p.mu.RUnlock()
					if fn != nil {
						fn()
					}
				}
			case <-p.done:
				return
			}
		}
	}()
}
```

- [ ] **Step 5: Run it to verify it passes**

Run: `go test ./internal/state/ -run TestPollHooksOnce -v`
Expected: PASS (both cases).

- [ ] **Step 6: Commit**

```bash
git add internal/state/poller.go internal/state/poller_test.go
git commit -m "feat(poller): 250ms hook-file fast loop with dir-mtime gate"
```

---

## Task 5: Merge precedence + notification edge-tracking in PollOnce

**Files:**
- Modify: `internal/state/poller.go` (`PollOnce`, add `lastNotified` map)
- Test: `internal/state/poller_test.go`

The fast loop already writes hook status to the DB. Now make the 1s pane loop respect hook precedence and fire `notifyWaiting`/`autoEscalate` exactly once per real `→ waiting` edge, tracked independently of the DB status the fast loop also writes.

- [ ] **Step 1: Write the failing test**

Add to `internal/state/poller_test.go`:

```go
func TestPollOnceFreshHookOverridesPaneScrape(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	must(t, db.CreateSession(conn, db.Session{ID: "inst-1", TmuxSession: "tmux-1", Tool: "claude", Status: "running"}))

	hooksDir := t.TempDir()
	must(t, hook.WriteStatus(hooksDir, "inst-1", "waiting", "", "Stop"))

	tc := testutil.NewFakeTmuxClient()
	tc.Sessions["tmux-1"] = "✢ working… (3s · ↓ 120 tokens)" // pane scrape would say running
	tc.Activities["tmux-1"] = time.Now()

	p := state.NewWithClockInterval(conn, tc, nil, time.Now, time.Second)
	p.SetHooksDir(hooksDir)

	p.PollHooksOnce() // populate fresh hook map
	p.PollOnce()

	s, _ := db.GetSession(conn, "inst-1")
	if s.Status != "waiting" {
		t.Errorf("fresh hook should win over pane scrape; got %q", s.Status)
	}
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `go test ./internal/state/ -run TestPollOnceFreshHookOverrides -v`
Expected: FAIL — pane scrape sets `running`, overwriting the hook's `waiting`.

- [ ] **Step 3: Add the `lastNotified` map and a merge helper**

In `internal/state/poller.go`, add struct field:

```go
	lastNotified  map[string]string // id -> last status we fired notify/escalate on
```

Initialise in `NewWithClockInterval`: `lastNotified: make(map[string]string),`

Add a helper that returns the effective status given the pane-derived status:

```go
// effectiveStatus returns the fresh hook status if one exists for id, else the
// pane-derived status. Mirrors sessionstatus.Derive precedence.
func (p *Poller) effectiveStatus(id, paneStatus string) string {
	p.mu.RLock()
	hs, ok := p.hookStatus[id]
	p.mu.RUnlock()
	if ok {
		if want := hs.DeckStatus(); want != "" {
			return want
		}
	}
	return paneStatus
}
```

- [ ] **Step 4: Rewire the decision block in `PollOnce`**

In `PollOnce`, replace the block from `newStatus := tmux.DetectStatus(...)` through the `if newStatus != s.Status { ... }` body (currently lines ~174-189) with:

```go
			paneStatus := tmux.DetectStatus(out, lc, now, s.Tool)
			newStatus := p.effectiveStatus(s.ID, paneStatus)
			p.updateWaitingState(s.ID, s.Status, newStatus, now)
			if newStatus != s.Status {
				if err := db.UpdateSessionStatus(p.conn, s.ID, newStatus); err != nil {
					log.Printf("poller: update status %q: %v", s.ID, err)
				}
			}
			// Notification edge: fire once per real transition into waiting,
			// tracked independently of DB status (the fast hook loop also
			// writes DB status, so we cannot key transitions off s.Status).
			prevNotified := p.notifiedStatus(s.ID)
			if prevNotified != newStatus {
				p.setNotifiedStatus(s.ID, newStatus)
				if prevNotified != tmux.StatusWaiting && newStatus == tmux.StatusWaiting {
					s.Status = newStatus
					if p.notifier != nil && p.notifier.Style() == notify.StyleDigest {
						digestGroups[s.GroupPath] = true
					} else {
						p.notifyWaiting(s)
					}
					p.autoEscalate(s, out)
				}
			}
```

Add the accessors:

```go
func (p *Poller) notifiedStatus(id string) string {
	p.mu.RLock()
	defer p.mu.RUnlock()
	return p.lastNotified[id]
}

func (p *Poller) setNotifiedStatus(id, status string) {
	p.mu.Lock()
	p.lastNotified[id] = status
	p.mu.Unlock()
}
```

- [ ] **Step 5: Clear the edge tracker when a session is cleared**

`clearSessionState(id)` already holds `p.mu.Lock()` and deletes from `lastChange`, `lastOutput`, `waitingSince`, `contextPct`. Add two more deletes in that same block:

```go
	delete(p.lastNotified, id)
	delete(p.hookStatus, id)
```

- [ ] **Step 6: Run it to verify it passes**

Run: `go test ./internal/state/ -v`
Expected: PASS, including the existing waiting-notification tests. If a pre-existing test asserted notify fired off `s.Status` transitions, it still holds because `lastNotified` starts empty and tracks the first observed status.

- [ ] **Step 7: Commit**

```bash
git add internal/state/poller.go internal/state/poller_test.go
git commit -m "feat(poller): hook-status precedence + edge-tracked waiting notifications"
```

---

## Task 6: Push UI refresh on hook-driven changes

**Files:**
- Modify: `internal/ui/app.go` (add `RefreshMsg`, handle in `Update`)
- Modify: `cmd/root.go` (`launchTUI`: wire `poller.SetRefresh`)
- Test: `internal/ui/app_test.go`

- [ ] **Step 1: Write the failing test**

Add to `internal/ui/app_test.go`:

```go
func TestUpdateHandlesRefreshMsg(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	tc := testutil.NewFakeTmuxClient()
	m := ui.NewModel(conn, tc, nil)
	if _, err := m.Update(ui.RefreshMsg{}); err != nil {
		// Update returns (tea.Model, tea.Cmd); adapt to the real signature.
		_ = err
	}
}
```

Adapt the assertion to the real `Update` signature in this codebase (it returns `(tea.Model, tea.Cmd)`). The meaningful check: `Update(ui.RefreshMsg{})` does not panic and returns the model. If `Update` currently has no error return, write:

```go
func TestUpdateHandlesRefreshMsg(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	m := ui.NewModel(conn, testutil.NewFakeTmuxClient(), nil)
	got, _ := m.Update(ui.RefreshMsg{})
	if got == nil {
		t.Fatal("Update(RefreshMsg) returned nil model")
	}
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `go test ./internal/ui/ -run TestUpdateHandlesRefreshMsg -v`
Expected: FAIL — `undefined: ui.RefreshMsg`.

- [ ] **Step 3: Add `RefreshMsg` and handle it**

In `internal/ui/app.go`, next to `tickMsg`:

```go
// RefreshMsg triggers an immediate Reload without rescheduling the periodic
// tick. Sent by the poller's hook loop via Program.Send for sub-second updates.
type RefreshMsg struct{}
```

In `Update`, add a case alongside `case tickMsg:` (line 198). The existing `tickMsg` case is `if err := m.Reload(); err != nil { m.err = err }; return m, tick()`. `RefreshMsg` uses the same error handling but returns a nil cmd (must NOT call `tick()`, which would spawn a second timer chain):

```go
	case RefreshMsg:
		if err := m.Reload(); err != nil {
			m.err = err
		}
		return m, nil
```

(`m.err` is the existing `error` field at `app.go:40`.)

- [ ] **Step 4: Wire the poller refresh to the program in `launchTUI`**

In `cmd/root.go`, inside the `for` loop in `launchTUI`, after `p := tea.NewProgram(...)` (line ~159) and before `p.Run()`:

```go
		p := tea.NewProgram(m, tea.WithAltScreen())
		poller.SetRefresh(func() { p.Send(ui.RefreshMsg{}) })
		finalModel, err := p.Run()
		poller.SetRefresh(nil) // program is done; stop sending to it
```

- [ ] **Step 5: Run tests + build**

Run: `go build ./... && go test ./internal/ui/ -run TestUpdateHandlesRefreshMsg -v`
Expected: PASS, clean build.

- [ ] **Step 6: Commit**

```bash
git add internal/ui/app.go internal/ui/app_test.go cmd/root.go
git commit -m "feat(ui): push immediate refresh on hook-driven status changes"
```

---

## Task 7: De-duplicate hook registration in install-hooks

**Files:**
- Modify: `cmd/installhooks.go`
- Test: `cmd/installhooks_test.go` (create if absent)

BUG-021 left duplicate `agent-deck` + `tmux-agent-deck` entries. On install, also strip any legacy `agent-deck hook-handler` entry so only our command remains.

- [ ] **Step 1: Write the failing test**

Add to `cmd/installhooks_test.go`:

```go
package cmd

import "testing"

func TestRemoveLegacyEntryStripsAgentDeck(t *testing.T) {
	arr := []any{
		map[string]any{"hooks": []any{
			map[string]any{"type": "command", "command": "agent-deck hook-handler"},
		}},
	}
	out := removeLegacyEntry(arr)
	if len(out) != 0 {
		t.Errorf("expected legacy agent-deck entry removed, got %v", out)
	}
}

func TestRemoveLegacyEntryKeepsOthers(t *testing.T) {
	arr := []any{
		map[string]any{"hooks": []any{
			map[string]any{"type": "command", "command": "some-other-tool"},
		}},
	}
	out := removeLegacyEntry(arr)
	if len(out) != 1 {
		t.Errorf("expected unrelated entry kept, got %v", out)
	}
}
```

- [ ] **Step 2: Run it to verify it fails**

Run: `go test ./cmd/ -run TestRemoveLegacyEntry -v`
Expected: FAIL — `undefined: removeLegacyEntry`.

- [ ] **Step 3: Add `removeLegacyEntry` and call it during install**

In `cmd/installhooks.go`, add the legacy command constant and a remover that mirrors `removeOurEntry` but targets `agent-deck hook-handler`:

```go
const legacyHookCommand = "agent-deck hook-handler"

// removeLegacyEntry strips entries whose command is the upstream agent-deck
// binary (the duplicate-registration half of BUG-021).
func removeLegacyEntry(arr []any) []any {
	var out []any
	for _, entry := range arr {
		m, ok := entry.(map[string]any)
		if !ok {
			out = append(out, entry)
			continue
		}
		if m["command"] == legacyHookCommand {
			continue
		}
		if hooks, ok := m["hooks"].([]any); ok {
			var remaining []any
			for _, h := range hooks {
				if hm, ok := h.(map[string]any); ok && hm["command"] == legacyHookCommand {
					continue
				}
				remaining = append(remaining, h)
			}
			if len(remaining) == 0 {
				continue
			}
			updated := make(map[string]any, len(m))
			for k, v := range m {
				updated[k] = v
			}
			updated["hooks"] = remaining
			out = append(out, updated)
			continue
		}
		out = append(out, m)
	}
	return out
}
```

In `runInstallHooks`, in the non-uninstall branch, strip legacy entries before checking/appending ours:

```go
		} else {
			arr = removeLegacyEntry(arr)
			if hasOurEntry(arr) {
				hooks[ev.name] = arr
				fmt.Fprintf(out, "%-20s already registered\n", ev.name)
			} else {
				hooks[ev.name] = append(arr, buildEntry(ev.async))
				fmt.Fprintf(out, "%-20s added\n", ev.name)
			}
		}
```

(Note the added `hooks[ev.name] = arr` in the already-registered path so a stripped legacy entry is persisted even when ours is already present.)

- [ ] **Step 4: Run it to verify it passes**

Run: `go test ./cmd/ -run 'TestRemoveLegacyEntry|TestRunInstallHooks' -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add cmd/installhooks.go cmd/installhooks_test.go
git commit -m "fix(hooks): strip legacy agent-deck registration on install (BUG-021)"
```

---

## Task 8: Manual verification + close BUG-021

**Files:**
- Modify: `docs/bugs.md`
- Modify: `CLAUDE.md` (Known Bugs section)

- [ ] **Step 1: Full suite + vet**

Run: `go test ./... && go vet ./...`
Expected: all PASS, no vet issues.

- [ ] **Step 2: Build and register hooks**

```bash
go build -o tmux-agent-deck .
./tmux-agent-deck install-hooks
```
Expected: each event prints `added` (or `already registered`); no `agent-deck` entries remain in `~/.claude/settings.json` (verify: `grep -c "agent-deck hook-handler" ~/.claude/settings.json` shows only `tmux-agent-deck` ones).

- [ ] **Step 3: Live smoke test**

Start a Claude session through the deck, submit a prompt, and confirm:
- A file appears at `~/.tmux-agent-deck/hooks/<session-id>.json` (verify: `ls ~/.tmux-agent-deck/hooks/`).
- Its `status` flips to `running` on submit and `waiting` on Stop (verify: watch the file, or the TUI status column).
- The TUI status updates within ~250ms of the transition, not on the 1s tick.

Confirm the env var reached the agent (the gating risk from the spec):
```bash
tmux show-environment -t <tmux-session-name> AGENTDECK_INSTANCE_ID
```
Expected: prints `AGENTDECK_INSTANCE_ID=<session-id>`. Repeat for a `claude-dangerous` session and a worktree session.

- [ ] **Step 4: Close BUG-021 in docs**

In `docs/bugs.md`, move BUG-021 to fixed with a one-line root cause: "Hook handler opened a DB and resolved identity via runtime tmux lookup with silent failures; replaced by env-var identity + atomic status files read by the poller. Install now strips the duplicate `agent-deck` registration." In `CLAUDE.md`, remove BUG-021 from the **Open** list.

- [ ] **Step 5: Commit**

```bash
git add docs/bugs.md CLAUDE.md
git commit -m "docs: close BUG-021 (hook status protocol)"
```

---

## Self-Review Notes

- **Spec coverage:** §1 env-var → Task 3; §2 handler file write → Task 2 (+ primitives Task 1); §3 idempotent install + dedupe → Task 7; §4 fast cold-read + dir-mtime gate + 5s rescan → Task 4; §5 precedence → Task 5; §6 push refresh → Task 6; §7 conductor poke moved off the handler → Tasks 2 (removed from handler) + 5 (fires in poller). Failure-mode table → Tasks 1 (traversal guard, home-anchored path, freshness), 4 (forced rescan), 7 (dedupe). Testing section → per-task tests + Task 8 manual.
- **Deferred (documented, not built):** fsnotify watcher (spec non-goal); multi-tool Gemini/Codex hooks (spec non-goal) — `EventToStatus` is Claude-only by design.
- **Type consistency:** `HookStatus`, `DeckStatus()`, `Fresh()`, `HooksDir()`, `WriteStatus()`, `ReadStatus()`, `ListStatuses()`, `EventToStatus()`, `newSessionArgs()`, `PollHooksOnce()`, `SetHooksDir()`, `SetRefresh()`, `effectiveStatus()`, `RefreshMsg` used consistently across tasks.
- **Verified against source (2026-05-26):** `clearSessionState` locking + delete pattern (Task 5 step 5), the `tickMsg` case shape and `m.err` field (Task 6), `NewSession` call sites (Task 3), and `runInstallHooks` structure (Task 7) all match the current code as written in this plan.
