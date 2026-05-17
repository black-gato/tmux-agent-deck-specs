# Auto-Escalation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Status: Complete** — all tasks implemented and verified.

**Goal:** When a worker session transitions to `waiting` and its group has a conductor, automatically send the escalation message to the conductor's tmux pane — opt-in via `--auto-escalate` flag.

**Architecture:** Add a `TmuxSender` interface and optional `sender` field to `Poller`, with a `SetSender` method for injection. A new `autoEscalate` method fires once per `→ waiting` transition, looks up the group conductor, and calls `SendKeys` with the same message format as the manual `C` key. A `--auto-escalate` CLI flag wires the real tmux client as the sender.

**Tech Stack:** Go 1.22, modernc SQLite, Cobra, existing `internal/state`, `internal/db`, `internal/tmux` packages.

---

## File Map

| File | Change |
|------|--------|
| `internal/state/poller.go` | Add `TmuxSender` interface, `sender` field, `SetSender`, `autoEscalate`, call from `PollOnce` |
| `internal/state/escalate.go` | New — `escalationMessage` pure function, `tailLines` helper |
| `internal/state/poller_test.go` | New test cases: 6 auto-escalation scenarios + message format |
| `cmd/root.go` | Add `--auto-escalate` flag, call `poller.SetSender(tc)` when enabled |

---

### Task 1: Add `TmuxSender` interface, `sender` field, and `SetSender` to Poller

**Files:**
- Modify: `internal/state/poller.go`

- [ ] **Step 1: Add `TmuxSender` interface and `sender` field**

Open `internal/state/poller.go`. After the `waitingNotifier` interface (around line 37), add:

```go
type TmuxSender interface {
	SendKeys(session string, pane int, keys string) error
}
```

Add `sender TmuxSender` as a field to the `Poller` struct (after `notifier waitingNotifier`):

```go
type Poller struct {
	conn         *sql.DB
	tmux         TmuxReader
	notifier     waitingNotifier
	sender       TmuxSender
	now          func() time.Time
	interval     time.Duration
	mu           sync.RWMutex
	lastChange   map[string]time.Time
	lastOutput   map[string]string
	waitingSince map[string]time.Time
	contextPct   map[string]*int
	done         chan struct{}
}
```

- [ ] **Step 2: Add `SetSender` method**

After `NewWithClockInterval`, add:

```go
func (p *Poller) SetSender(s TmuxSender) {
	p.sender = s
}
```

- [ ] **Step 3: Verify it compiles**

```bash
go build ./internal/state/...
```

Expected: no output, exit 0.

- [ ] **Step 4: Commit**

```bash
git add internal/state/poller.go
git commit -m "feat(state): add TmuxSender interface and SetSender to Poller"
```

---

### Task 2: Add `escalationMessage` pure function

**Files:**
- Create: `internal/state/escalate.go`

- [ ] **Step 1: Create `internal/state/escalate.go`**

```go
package state

import (
	"fmt"
	"strings"

	"github.com/black-gato/tmux-agent-deck/internal/db"
)

func escalationMessage(session db.Session, lastOutput string) string {
	lines := []string{
		fmt.Sprintf("Escalation from %s", session.Title),
		fmt.Sprintf("Status: %s", session.Status),
	}
	if session.Notes != "" {
		lines = append(lines, fmt.Sprintf("Notes: %s", session.Notes))
	}
	context := strings.Join(tailLines(lastOutput, 3), "\n")
	if strings.TrimSpace(context) != "" {
		lines = append(lines, "Current issue context:")
		lines = append(lines, context)
	}
	return strings.Join(lines, "\n")
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

- [ ] **Step 2: Verify it compiles**

```bash
go build ./internal/state/...
```

Expected: no output, exit 0.

- [ ] **Step 3: Run existing tests to confirm no regression**

```bash
go test ./internal/state/...
```

Expected: all PASS.

- [ ] **Step 4: Commit**

```bash
git add internal/state/escalate.go
git commit -m "feat(state): add escalationMessage pure function"
```

---

### Task 3: Add `autoEscalate` method and wire into `PollOnce`

**Files:**
- Modify: `internal/state/poller.go`

- [ ] **Step 1: Add `autoEscalate` method**

Add this method to `internal/state/poller.go` after `notifyDigest`:

```go
func (p *Poller) autoEscalate(session db.Session, output string) {
	if p.sender == nil {
		return
	}
	conductor, err := db.GetGroupConductorSession(p.conn, session.GroupPath)
	if err != nil || conductor.Title == "" {
		return
	}
	if conductor.ID == session.ID {
		return
	}
	if conductor.Status != tmux.StatusRunning || conductor.TmuxSession == "" {
		log.Printf("poller: auto-escalate %q: conductor %q not running", session.ID, conductor.Title)
		return
	}
	if err := p.sender.SendKeys(conductor.TmuxSession, 0, escalationMessage(session, output)); err != nil {
		log.Printf("poller: auto-escalate send keys %q: %v", session.ID, err)
	}
}
```

- [ ] **Step 2: Call `autoEscalate` from `PollOnce` at the `→ waiting` transition point**

In `PollOnce`, locate this block (around line 156):

```go
			if s.Status != tmux.StatusWaiting && newStatus == tmux.StatusWaiting {
				if p.notifier != nil && p.notifier.Style() == notify.StyleDigest {
					digestGroups[s.GroupPath] = true
				} else {
					p.notifyWaiting(s)
				}
			}
```

Replace it with:

```go
			if s.Status != tmux.StatusWaiting && newStatus == tmux.StatusWaiting {
				if p.notifier != nil && p.notifier.Style() == notify.StyleDigest {
					digestGroups[s.GroupPath] = true
				} else {
					p.notifyWaiting(s)
				}
				p.autoEscalate(s, out)
			}
```

- [ ] **Step 3: Verify it compiles and existing tests pass**

```bash
go test ./internal/state/...
```

Expected: all PASS.

- [ ] **Step 4: Commit**

```bash
git add internal/state/poller.go
git commit -m "feat(state): add autoEscalate, fires on waiting transition"
```

---

### Task 4: Test `autoEscalate` in `poller_test.go`

**Files:**
- Modify: `internal/state/poller_test.go`

- [ ] **Step 1: Add `stubSender` type** at the bottom of `internal/state/poller_test.go`:

```go
type stubSender struct {
	calls []sentKeysCall
}

type sentKeysCall struct {
	session string
	pane    int
	keys    string
}

func (s *stubSender) SendKeys(session string, pane int, keys string) error {
	s.calls = append(s.calls, sentKeysCall{session, pane, keys})
	return nil
}
```

- [ ] **Step 2: Write the failing tests** — add these six test functions to `poller_test.go`:

```go
func TestAutoEscalateSendsToCondutorOnWaitingTransition(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "conductor"})
	db.CreateSession(conn, db.Session{
		ID: "conductor", Title: "conductor", GroupPath: "work", TmuxSession: "tmux-conductor",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})
	db.CreateSession(conn, db.Session{
		ID: "worker", Title: "worker", GroupPath: "work", TmuxSession: "tmux-worker",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now, Notes: "stuck on auth",
	})

	stub := &stubTmux{output: "line1\nline2\nline3\n> ", exists: true}
	sender := &stubSender{}
	p := state.New(conn, stub)
	p.SetSender(sender)
	p.PollOnce()

	if len(sender.calls) != 1 {
		t.Fatalf("expected 1 SendKeys call, got %d", len(sender.calls))
	}
	got := sender.calls[0]
	if got.session != "tmux-conductor" {
		t.Errorf("session: got %q want %q", got.session, "tmux-conductor")
	}
	if got.pane != 0 {
		t.Errorf("pane: got %d want 0", got.pane)
	}
	if !strings.Contains(got.keys, "Escalation from worker") {
		t.Errorf("keys missing escalation header: %q", got.keys)
	}
	if !strings.Contains(got.keys, "Notes: stuck on auth") {
		t.Errorf("keys missing notes: %q", got.keys)
	}
	if !strings.Contains(got.keys, "Current issue context:") {
		t.Errorf("keys missing context section: %q", got.keys)
	}
}

func TestAutoEscalateFiresOncePerTransition(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "conductor"})
	db.CreateSession(conn, db.Session{
		ID: "conductor", Title: "conductor", GroupPath: "work", TmuxSession: "tmux-conductor",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})
	db.CreateSession(conn, db.Session{
		ID: "worker", Title: "worker", GroupPath: "work", TmuxSession: "tmux-worker",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})

	stub := &stubTmux{output: "Some output\n> ", exists: true}
	sender := &stubSender{}
	p := state.New(conn, stub)
	p.SetSender(sender)
	p.PollOnce() // running → waiting, should fire
	p.PollOnce() // still waiting, should NOT fire again

	if len(sender.calls) != 1 {
		t.Fatalf("expected 1 SendKeys call across 2 polls, got %d", len(sender.calls))
	}
}

func TestAutoEscalateSkipsWhenSenderNil(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "conductor"})
	db.CreateSession(conn, db.Session{
		ID: "conductor", Title: "conductor", GroupPath: "work", TmuxSession: "tmux-conductor",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})
	db.CreateSession(conn, db.Session{
		ID: "worker", Title: "worker", GroupPath: "work", TmuxSession: "tmux-worker",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})

	stub := &stubTmux{output: "Some output\n> ", exists: true}
	p := state.New(conn, stub) // no SetSender call
	p.PollOnce()               // should not panic or send anything
}

func TestAutoEscalateSkipsWhenNoConductor(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work"}) // no ConductorSessionID
	db.CreateSession(conn, db.Session{
		ID: "worker", Title: "worker", GroupPath: "work", TmuxSession: "tmux-worker",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})

	stub := &stubTmux{output: "Some output\n> ", exists: true}
	sender := &stubSender{}
	p := state.New(conn, stub)
	p.SetSender(sender)
	p.PollOnce()

	if len(sender.calls) != 0 {
		t.Fatalf("expected 0 SendKeys calls when no conductor, got %d", len(sender.calls))
	}
}

func TestAutoEscalateSkipsWhenSessionIsConductor(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "conductor"})
	db.CreateSession(conn, db.Session{
		ID: "conductor", Title: "conductor", GroupPath: "work", TmuxSession: "tmux-conductor",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})

	stub := &stubTmux{output: "Some output\n> ", exists: true}
	sender := &stubSender{}
	p := state.New(conn, stub)
	p.SetSender(sender)
	p.PollOnce() // conductor itself transitions to waiting — should not self-escalate

	if len(sender.calls) != 0 {
		t.Fatalf("expected 0 SendKeys calls when conductor is the waiting session, got %d", len(sender.calls))
	}
}

func TestAutoEscalateSkipsWhenConductorNotRunning(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "conductor"})
	db.CreateSession(conn, db.Session{
		ID: "conductor", Title: "conductor", GroupPath: "work", TmuxSession: "tmux-conductor",
		ProjectPath: "/p", Tool: "claude", Status: "idle", CreatedAt: now,
	})
	db.CreateSession(conn, db.Session{
		ID: "worker", Title: "worker", GroupPath: "work", TmuxSession: "tmux-worker",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})

	stub := &stubTmux{output: "Some output\n> ", exists: true}
	sender := &stubSender{}
	p := state.New(conn, stub)
	p.SetSender(sender)
	p.PollOnce()

	if len(sender.calls) != 0 {
		t.Fatalf("expected 0 SendKeys calls when conductor not running, got %d", len(sender.calls))
	}
}
```

- [ ] **Step 3: Run the tests to verify they fail (before implementation is complete)**

```bash
go test ./internal/state/... -run TestAutoEscalate -v
```

Expected: `FAIL` — `SetSender` not yet exposed (Task 1 added it, so these should compile but the logic in Task 3 drives the behavior). If Task 1–3 are already done, run to confirm PASS.

- [ ] **Step 4: Run all state tests**

```bash
go test ./internal/state/...
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/state/poller_test.go
git commit -m "test(state): add auto-escalation test coverage"
```

---

### Task 5: Wire `--auto-escalate` flag in `cmd/root.go`

**Files:**
- Modify: `cmd/root.go`

- [ ] **Step 1: Add package-level variable**

After `var headlessMode bool` (around line 27), add:

```go
var autoEscalate bool
```

- [ ] **Step 2: Register the flag in `init()`**

In the `init()` function, after the `--headless` flag registration, add:

```go
rootCmd.PersistentFlags().BoolVar(&autoEscalate, "auto-escalate", false, "Automatically send escalation message to group conductor when a worker session goes waiting")
```

- [ ] **Step 3: Reset the flag in `resetRootOptions()`**

In `resetRootOptions()`, after `_ = rootCmd.PersistentFlags().Set("headless", "false")`, add:

```go
autoEscalate = false
_ = rootCmd.PersistentFlags().Set("auto-escalate", "false")
```

- [ ] **Step 4: Pass sender in `launchTUI`**

Replace the existing `launchTUI` function with:

```go
func launchTUI(conn *sql.DB, tc tmux.ClientIface) error {
	for {
		poller := state.NewWithNotifierInterval(conn, tc, notify.New(notify.Config{
			Enabled: notifyEnabled,
			Style:   notify.Style(notifyStyle),
			Quiet:   notifyQuiet,
		}), pollInterval)
		if autoEscalate {
			poller.SetSender(tc)
		}
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

- [ ] **Step 5: Pass sender in `launchHeadless`**

Replace the existing `launchHeadless` function with:

```go
func launchHeadless(ctx context.Context, conn *sql.DB, tc tmux.ClientIface) error {
	poller := state.NewWithNotifierInterval(conn, tc, notify.New(notify.Config{
		Enabled: notifyEnabled,
		Style:   notify.Style(notifyStyle),
		Quiet:   notifyQuiet,
	}), pollInterval)
	if autoEscalate {
		poller.SetSender(tc)
	}
	poller.Start()
	defer poller.Stop()
	<-ctx.Done()
	return nil
}
```

- [ ] **Step 6: Run all tests**

```bash
go test ./...
```

Expected: all PASS.

- [ ] **Step 7: Build and verify flag appears in help**

```bash
go build -o tmux-agent-deck . && ./tmux-agent-deck --help
```

Expected: `--auto-escalate` appears in the flag list.

- [ ] **Step 8: Commit**

```bash
git add cmd/root.go
git commit -m "feat(cmd): add --auto-escalate flag"
```
