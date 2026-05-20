# Conductor Enhancements Implementation Plan

**Status:** Implemented (2026-05-19)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend conductor mode with reply-to-worker routing, a configurable heartbeat digest, and optional CLAUDE.md initialization so a conductor session can supervise workers with less manual relay.

**Architecture:** Feature 1 adds a `@deck-reply` block parser as a pure function module and wires reply scanning into `PollOnce` using per-conductor pane baselines and processed-reply fingerprints. Feature 2 adds a `SetConductorHeartbeat` method to `Poller` and a DB helper for scoped group membership. Feature 3 adds a standalone `conductordocs` package that writes a managed block into `CLAUDE.md` and is triggered when `c` is pressed.

**Tech Stack:** Go 1.22, modernc SQLite, Cobra, Bubble Tea, existing `internal/state`, `internal/db`, `internal/tmux`, `internal/ui` packages.

---

## File Structure

| File | Change |
|---|---|
| `internal/state/reply.go` | New — `replyBlock` type, `parseReplyBlocks`, `newOutputSince` pure functions |
| `internal/state/reply_test.go` | New — parser unit tests |
| `internal/state/escalate.go` | Modify — add worker ID and reply syntax to `escalationMessage` |
| `internal/state/escalate_test.go` | New — escalation message unit tests |
| `internal/state/poller.go` | Modify — add `conductorBaseline`, `processedReplies`, `heartbeatInterval`, `lastHeartbeat` fields; `SetConductorHeartbeat`; `scanConductorReplies`; `runHeartbeats`; `heartbeatMessage` |
| `internal/state/poller_test.go` | Modify — reply routing + heartbeat tests |
| `internal/db/sessions.go` | Modify — add `ListGroupChildSessions` |
| `internal/db/sessions_test.go` | Modify — cover `ListGroupChildSessions` |
| `internal/conductordocs/conductordocs.go` | New — `WriteBlock(projectPath string) error`, managed block writer |
| `internal/conductordocs/conductordocs_test.go` | New — file-rule tests |
| `internal/ui/app.go` | Modify — add `InitConductorDocs bool` field; call `conductordocs.WriteBlock` on `set-conductor` when enabled |
| `cmd/root.go` | Modify — add `--conductor-heartbeat` and `--init-conductor-docs` flags; wire into poller and UI |

---

### Task 1: Reply block parser

**Files:**
- Create: `internal/state/reply.go`
- Create: `internal/state/reply_test.go`

- [ ] **Step 1: Write the failing tests**

Create `internal/state/reply_test.go`:

```go
package state_test

import (
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/state"
)

func TestParseReplyBlocksComplete(t *testing.T) {
	input := "@deck-reply worker=abc\nhello world\n@deck-end"
	blocks := state.ParseReplyBlocks(input)
	if len(blocks) != 1 {
		t.Fatalf("expected 1 block, got %d", len(blocks))
	}
	if blocks[0].WorkerID != "abc" {
		t.Errorf("workerID: got %q want %q", blocks[0].WorkerID, "abc")
	}
	if blocks[0].Body != "hello world" {
		t.Errorf("body: got %q want %q", blocks[0].Body, "hello world")
	}
}

func TestParseReplyBlocksIncompleteIgnored(t *testing.T) {
	input := "@deck-reply worker=abc\nhello world"
	blocks := state.ParseReplyBlocks(input)
	if len(blocks) != 0 {
		t.Fatalf("expected 0 blocks, got %d", len(blocks))
	}
}

func TestParseReplyBlocksMultiple(t *testing.T) {
	input := "@deck-reply worker=a\nfoo\n@deck-end\n@deck-reply worker=b\nbar\n@deck-end"
	blocks := state.ParseReplyBlocks(input)
	if len(blocks) != 2 {
		t.Fatalf("expected 2 blocks, got %d", len(blocks))
	}
	if blocks[0].WorkerID != "a" || blocks[1].WorkerID != "b" {
		t.Errorf("workerIDs: got %q %q", blocks[0].WorkerID, blocks[1].WorkerID)
	}
}

func TestParseReplyBlocksEmptyBodyIgnored(t *testing.T) {
	input := "@deck-reply worker=abc\n\n@deck-end"
	blocks := state.ParseReplyBlocks(input)
	if len(blocks) != 0 {
		t.Fatalf("expected 0 blocks for empty body, got %d", len(blocks))
	}
}

func TestParseReplyBlocksMultilineBodyNormalized(t *testing.T) {
	input := "@deck-reply worker=abc\nline one\nline two\n@deck-end"
	blocks := state.ParseReplyBlocks(input)
	if len(blocks) != 1 {
		t.Fatalf("expected 1 block, got %d", len(blocks))
	}
	if blocks[0].Body != "line one | line two" {
		t.Errorf("body: got %q want %q", blocks[0].Body, "line one | line two")
	}
}

func TestNewOutputSinceBaseline(t *testing.T) {
	baseline := "old content"
	current := "old content\nnew line"
	got := state.NewOutputSince(baseline, current)
	if got != "\nnew line" {
		t.Errorf("got %q want %q", got, "\nnew line")
	}
}

func TestNewOutputSinceBaselineNotFound(t *testing.T) {
	baseline := "old content"
	current := "completely different"
	got := state.NewOutputSince(baseline, current)
	if got != "completely different" {
		t.Errorf("got %q want %q", got, "completely different")
	}
}
```

- [ ] **Step 2: Run to confirm they fail**

```bash
go test ./internal/state/... -run TestParseReply -v 2>&1 | head -20
```

Expected: compile error — `state.ParseReplyBlocks` undefined.

- [ ] **Step 3: Create `internal/state/reply.go`**

```go
package state

import "strings"

type replyBlock struct {
	WorkerID string
	Body     string
}

func ParseReplyBlocks(output string) []replyBlock {
	var results []replyBlock
	lines := strings.Split(output, "\n")
	i := 0
	for i < len(lines) {
		line := strings.TrimSpace(lines[i])
		if !strings.HasPrefix(line, "@deck-reply worker=") {
			i++
			continue
		}
		workerID := strings.TrimSpace(strings.TrimPrefix(line, "@deck-reply worker="))
		if workerID == "" {
			i++
			continue
		}
		i++
		var bodyLines []string
		closed := false
		for i < len(lines) {
			bl := strings.TrimSpace(lines[i])
			if bl == "@deck-end" {
				closed = true
				i++
				break
			}
			bodyLines = append(bodyLines, bl)
			i++
		}
		if !closed {
			break
		}
		var nonEmpty []string
		for _, bl := range bodyLines {
			if bl != "" {
				nonEmpty = append(nonEmpty, bl)
			}
		}
		if len(nonEmpty) == 0 {
			continue
		}
		results = append(results, replyBlock{
			WorkerID: workerID,
			Body:     strings.Join(nonEmpty, " | "),
		})
	}
	return results
}

func NewOutputSince(baseline, current string) string {
	idx := strings.LastIndex(current, baseline)
	if idx < 0 {
		return current
	}
	return current[idx+len(baseline):]
}
```

- [ ] **Step 4: Run tests**

```bash
go test ./internal/state/... -run "TestParseReply|TestNewOutput" -v
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/state/reply.go internal/state/reply_test.go
git commit -m "feat(state): add reply block parser"
```

---

### Task 2: Escalation message — add worker ID and reply syntax

**Files:**
- Modify: `internal/state/escalate.go`
- Create: `internal/state/escalate_test.go`

- [ ] **Step 1: Write failing tests**

Create `internal/state/escalate_test.go`:

```go
package state_test

import (
	"strings"
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/state"
)

func TestEscalationMessageIncludesWorkerID(t *testing.T) {
	s := db.Session{ID: "worker-42", Title: "my-worker", Status: "waiting"}
	msg := state.EscalationMessage(s, "")
	if !strings.Contains(msg, "Worker ID: worker-42") {
		t.Errorf("message missing worker ID: %q", msg)
	}
}

func TestEscalationMessageIncludesReplySyntax(t *testing.T) {
	s := db.Session{ID: "worker-42", Title: "my-worker", Status: "waiting"}
	msg := state.EscalationMessage(s, "")
	if !strings.Contains(msg, "@deck-reply worker=worker-42") {
		t.Errorf("message missing reply syntax: %q", msg)
	}
	if !strings.Contains(msg, "@deck-end") {
		t.Errorf("message missing @deck-end: %q", msg)
	}
}

func TestEscalationMessageIncludesNotes(t *testing.T) {
	s := db.Session{ID: "w1", Title: "worker", Status: "waiting", Notes: "stuck on auth"}
	msg := state.EscalationMessage(s, "")
	if !strings.Contains(msg, "Notes: stuck on auth") {
		t.Errorf("message missing notes: %q", msg)
	}
}

func TestEscalationMessageIncludesContext(t *testing.T) {
	s := db.Session{ID: "w1", Title: "worker", Status: "waiting"}
	msg := state.EscalationMessage(s, "some relevant output line")
	if !strings.Contains(msg, "some relevant output line") {
		t.Errorf("message missing context: %q", msg)
	}
}
```

- [ ] **Step 2: Run to confirm they fail**

```bash
go test ./internal/state/... -run TestEscalation -v 2>&1 | head -20
```

Expected: compile error — `state.EscalationMessage` undefined (currently unexported `escalationMessage`).

- [ ] **Step 3: Update `internal/state/escalate.go`**

Replace the file contents:

```go
package state

import (
	"fmt"
	"strings"

	"github.com/black-gato/tmux-agent-deck/internal/db"
)

func EscalationMessage(session db.Session, lastOutput string) string {
	parts := []string{
		fmt.Sprintf("Escalation from %s", session.Title),
		fmt.Sprintf("Worker ID: %s", session.ID),
		fmt.Sprintf("Status: %s", session.Status),
	}
	if session.Notes != "" {
		parts = append(parts, fmt.Sprintf("Notes: %s", session.Notes))
	}
	parts = append(parts, fmt.Sprintf("Reply with: @deck-reply worker=%s ... @deck-end", session.ID))
	context := strings.Join(contextLines(lastOutput, 5), " | ")
	if strings.TrimSpace(context) != "" {
		parts = append(parts, fmt.Sprintf("Context: %s", context))
	}
	return strings.Join(parts, " | ") + "\n"
}

func contextLines(output string, n int) []string {
	if n <= 0 || output == "" {
		return nil
	}
	all := strings.Split(output, "\n")
	lines := make([]string, 0, n)
	for i := len(all) - 1; i >= 0 && len(lines) < n; i-- {
		line := strings.TrimSpace(all[i])
		if !isContextLine(line) {
			continue
		}
		lines = append(lines, line)
	}
	for i, j := 0, len(lines)-1; i < j; i, j = i+1, j-1 {
		lines[i], lines[j] = lines[j], lines[i]
	}
	return lines
}

func isContextLine(line string) bool {
	if line == "" || line == ">" || line == "❯" {
		return false
	}
	if strings.Contains(line, "-- INSERT --") {
		return false
	}
	if strings.Contains(line, "ctx:") && strings.Contains(line, "@") {
		return false
	}
	return strings.Trim(line, "─━═- ") != ""
}
```

- [ ] **Step 4: Update call site in `internal/state/poller.go`**

In `poller.go`, find the `autoEscalate` function call to `escalationMessage` and update it to `EscalationMessage`:

```go
if err := p.sender.SendKeys(conductor.TmuxSession, 0, EscalationMessage(session, output)); err != nil {
```

- [ ] **Step 5: Update call site in `internal/ui/app.go`**

Search for `escalationMessage` in `app.go` and update:

```bash
grep -n "escalationMessage" internal/ui/app.go
```

Open `internal/ui/app.go` and find the `escalateSelectedSession` method. It calls `m.escalationMessage(session)`. Find that private method and replace it with:

```go
func (m *Model) escalationMessage(session db.Session) string {
	out := ""
	if m.output != "" {
		out = m.output
	}
	return state.EscalationMessage(session, out)
}
```

Add `"github.com/black-gato/tmux-agent-deck/internal/state"` to the imports in `app.go` if not already present.

- [ ] **Step 6: Run all tests**

```bash
go test ./...
```

Expected: all PASS.

- [ ] **Step 7: Commit**

```bash
git add internal/state/escalate.go internal/state/escalate_test.go internal/state/poller.go internal/ui/app.go
git commit -m "feat(state): add worker ID and reply syntax to escalation messages"
```

---

### Task 3: Conductor baseline and reply scanning in the poller

**Files:**
- Modify: `internal/state/poller.go`
- Modify: `internal/state/poller_test.go`

- [ ] **Step 1: Write failing tests**

Append to `internal/state/poller_test.go`:

```go
func TestReplyRoutingSendsToWorker(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "conductor"})
	db.CreateSession(conn, db.Session{
		ID: "conductor", Title: "conductor", GroupPath: "work", TmuxSession: "tmux-conductor",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})
	db.CreateSession(conn, db.Session{
		ID: "worker-1", Title: "worker", GroupPath: "work", TmuxSession: "tmux-worker",
		ProjectPath: "/p", Tool: "claude", Status: "waiting", CreatedAt: now,
	})

	conductorOutput := "some prior output\n@deck-reply worker=worker-1\nfix the failing test\n@deck-end"
	stub := &stubTmux{
		outputs: map[string]string{
			"tmux-conductor": conductorOutput,
			"tmux-worker":    "Some output\n> ",
		},
		exists: true,
	}
	sender := &stubSender{}
	p := state.New(conn, stub)
	p.SetSender(sender)
	p.PollOnce() // first pass: establishes baseline for conductor
	p.PollOnce() // second pass: no new output → no reply

	if len(sender.calls) != 0 {
		t.Fatalf("expected 0 calls on first two polls (baseline pass then no new output), got %d", len(sender.calls))
	}

	// Simulate new reply appearing after baseline
	stub.outputs["tmux-conductor"] = conductorOutput + "\n@deck-reply worker=worker-1\nunblock: run go test\n@deck-end"
	p.PollOnce()

	var replyCalls []sentKeysCall
	for _, c := range sender.calls {
		if c.session == "tmux-worker" {
			replyCalls = append(replyCalls, c)
		}
	}
	if len(replyCalls) == 0 {
		t.Fatal("expected a SendKeys call to worker session")
	}
	if !strings.Contains(replyCalls[0].keys, "Conductor reply:") {
		t.Errorf("reply missing prefix: %q", replyCalls[0].keys)
	}
	if !strings.Contains(replyCalls[0].keys, "unblock: run go test") {
		t.Errorf("reply missing body: %q", replyCalls[0].keys)
	}
}

func TestReplyRoutingDuplicateNotResent(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "conductor"})
	db.CreateSession(conn, db.Session{
		ID: "conductor", Title: "conductor", GroupPath: "work", TmuxSession: "tmux-conductor",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})
	db.CreateSession(conn, db.Session{
		ID: "worker-1", Title: "worker", GroupPath: "work", TmuxSession: "tmux-worker",
		ProjectPath: "/p", Tool: "claude", Status: "waiting", CreatedAt: now,
	})

	baseOutput := "baseline"
	replyOutput := baseOutput + "\n@deck-reply worker=worker-1\ndo the thing\n@deck-end"
	stub := &stubTmux{
		outputs: map[string]string{
			"tmux-conductor": baseOutput,
			"tmux-worker":    "> ",
		},
		exists: true,
	}
	sender := &stubSender{}
	p := state.New(conn, stub)
	p.SetSender(sender)
	p.PollOnce() // establishes baseline

	stub.outputs["tmux-conductor"] = replyOutput
	p.PollOnce() // sends reply
	p.PollOnce() // reply still visible — must NOT resend

	var replyCalls int
	for _, c := range sender.calls {
		if c.session == "tmux-worker" {
			replyCalls++
		}
	}
	if replyCalls != 1 {
		t.Errorf("expected 1 reply delivery, got %d", replyCalls)
	}
}

func TestReplyRoutingSkipsUnknownWorker(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "conductor"})
	db.CreateSession(conn, db.Session{
		ID: "conductor", Title: "conductor", GroupPath: "work", TmuxSession: "tmux-conductor",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})

	baseOutput := "baseline"
	stub := &stubTmux{
		outputs: map[string]string{"tmux-conductor": baseOutput},
		exists:  true,
	}
	sender := &stubSender{}
	p := state.New(conn, stub)
	p.SetSender(sender)
	p.PollOnce()

	stub.outputs["tmux-conductor"] = baseOutput + "\n@deck-reply worker=nonexistent\nfoo\n@deck-end"
	p.PollOnce()

	if len(sender.calls) != 0 {
		t.Fatalf("expected 0 calls for unknown worker, got %d", len(sender.calls))
	}
}

func TestReplyRoutingSkipsStoppedWorker(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	now := time.Now().Unix()
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "conductor"})
	db.CreateSession(conn, db.Session{
		ID: "conductor", Title: "conductor", GroupPath: "work", TmuxSession: "tmux-conductor",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})
	db.CreateSession(conn, db.Session{
		ID: "worker-1", Title: "worker", GroupPath: "work", TmuxSession: "tmux-worker",
		ProjectPath: "/p", Tool: "claude", Status: "stopped", CreatedAt: now,
	})

	baseOutput := "baseline"
	stub := &stubTmux{
		outputs: map[string]string{"tmux-conductor": baseOutput},
		exists:  true,
	}
	sender := &stubSender{}
	p := state.New(conn, stub)
	p.SetSender(sender)
	p.PollOnce()

	stub.outputs["tmux-conductor"] = baseOutput + "\n@deck-reply worker=worker-1\nfoo\n@deck-end"
	p.PollOnce()

	var replyCalls int
	for _, c := range sender.calls {
		if c.session == "tmux-worker" {
			replyCalls++
		}
	}
	if replyCalls != 0 {
		t.Errorf("expected 0 reply calls for stopped worker, got %d", replyCalls)
	}
}
```

- [ ] **Step 2: Run to verify they fail**

```bash
go test ./internal/state/... -run TestReplyRouting -v 2>&1 | head -20
```

Expected: FAIL — `scanConductorReplies` not yet implemented.

- [ ] **Step 3: Add fields to Poller struct in `internal/state/poller.go`**

In the `Poller` struct, add after `contextPct map[string]*int`:

```go
conductorBaseline map[string]string
processedReplies  map[string]bool
```

In `NewWithClockInterval`, initialize them:

```go
conductorBaseline: make(map[string]string),
processedReplies:  make(map[string]bool),
```

- [ ] **Step 4: Add `scanConductorReplies` method to `internal/state/poller.go`**

Add after `autoEscalate`:

```go
func (p *Poller) scanConductorReplies(sessions []db.Session) {
	if p.sender == nil {
		return
	}
	groups, err := db.ListGroups(p.conn)
	if err != nil {
		log.Printf("poller: scan replies list groups: %v", err)
		return
	}
	conductorIDs := make(map[string]bool, len(groups))
	for _, g := range groups {
		if g.ConductorSessionID != "" {
			conductorIDs[g.ConductorSessionID] = true
		}
	}
	sessionMap := make(map[string]db.Session, len(sessions))
	for _, s := range sessions {
		sessionMap[s.ID] = s
	}
	for _, s := range sessions {
		if !conductorIDs[s.ID] || s.TmuxSession == "" {
			continue
		}
		out, err := p.tmux.CapturePaneOutput(s.TmuxSession)
		if err != nil {
			continue
		}
		p.mu.Lock()
		baseline, seen := p.conductorBaseline[s.ID]
		if !seen {
			p.conductorBaseline[s.ID] = out
			p.mu.Unlock()
			continue
		}
		newOut := NewOutputSince(baseline, out)
		p.mu.Unlock()

		for _, block := range ParseReplyBlocks(newOut) {
			fingerprint := block.WorkerID + ":" + block.Body
			p.mu.Lock()
			if p.processedReplies[fingerprint] {
				p.mu.Unlock()
				continue
			}
			p.processedReplies[fingerprint] = true
			p.mu.Unlock()

			worker, err := db.GetSession(p.conn, block.WorkerID)
			if err != nil {
				log.Printf("poller: reply routing: unknown worker %q", block.WorkerID)
				continue
			}
			if worker.Status == tmux.StatusStopped || worker.Status == tmux.StatusError || worker.TmuxSession == "" {
				log.Printf("poller: reply routing: worker %q not active (status=%s)", block.WorkerID, worker.Status)
				continue
			}
			msg := "Conductor reply: " + block.Body
			if err := p.sender.SendKeys(worker.TmuxSession, 0, msg); err != nil {
				log.Printf("poller: reply routing send %q: %v", block.WorkerID, err)
				continue
			}
			if err := p.sender.SendRawKeys(worker.TmuxSession, 0, "Enter"); err != nil {
				log.Printf("poller: reply routing submit %q: %v", block.WorkerID, err)
			}
		}
	}
}
```

- [ ] **Step 5: Call `scanConductorReplies` from `PollOnce`**

At the end of `PollOnce`, after the `for groupPath := range digestGroups` loop, add:

```go
p.scanConductorReplies(sessions)
```

- [ ] **Step 6: Run tests**

```bash
go test ./internal/state/... -run TestReplyRouting -v
```

Expected: all PASS.

- [ ] **Step 7: Run full test suite**

```bash
go test ./...
```

Expected: all PASS.

- [ ] **Step 8: Commit**

```bash
git add internal/state/poller.go internal/state/poller_test.go
git commit -m "feat(state): add conductor reply scanning and routing to workers"
```

---

### Task 4: DB helper for group child sessions

**Files:**
- Modify: `internal/db/sessions.go`
- Modify: `internal/db/sessions_test.go`

- [ ] **Step 1: Write failing tests**

Append to `internal/db/sessions_test.go`:

```go
func TestListGroupChildSessionsIncludesOwnGroup(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "c1"})
	now := time.Now().Unix()
	db.CreateSession(conn, db.Session{
		ID: "c1", Title: "conductor", GroupPath: "work",
		TmuxSession: "t1", ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})
	db.CreateSession(conn, db.Session{
		ID: "w1", Title: "worker", GroupPath: "work",
		TmuxSession: "t2", ProjectPath: "/p", Tool: "claude", Status: "waiting", CreatedAt: now,
	})

	sessions, err := db.ListGroupChildSessions(conn, "work", "c1")
	if err != nil {
		t.Fatal(err)
	}
	ids := make(map[string]bool)
	for _, s := range sessions {
		ids[s.ID] = true
	}
	if !ids["c1"] || !ids["w1"] {
		t.Errorf("expected both sessions, got IDs: %v", ids)
	}
}

func TestListGroupChildSessionsIncludesInheritedChildren(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "c1"})
	db.CreateGroup(conn, db.Group{Path: "work/sub", Name: "sub"}) // no own conductor
	now := time.Now().Unix()
	db.CreateSession(conn, db.Session{
		ID: "c1", Title: "conductor", GroupPath: "work",
		TmuxSession: "t1", ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})
	db.CreateSession(conn, db.Session{
		ID: "w2", Title: "sub-worker", GroupPath: "work/sub",
		TmuxSession: "t3", ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})

	sessions, err := db.ListGroupChildSessions(conn, "work", "c1")
	if err != nil {
		t.Fatal(err)
	}
	ids := make(map[string]bool)
	for _, s := range sessions {
		ids[s.ID] = true
	}
	if !ids["w2"] {
		t.Errorf("expected sub-group worker in results, got: %v", ids)
	}
}

func TestListGroupChildSessionsExcludesOwnConductorSubgroups(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "c1"})
	db.CreateGroup(conn, db.Group{Path: "work/sub", Name: "sub", ConductorSessionID: "c2"}) // own conductor
	now := time.Now().Unix()
	db.CreateSession(conn, db.Session{
		ID: "c1", Title: "conductor", GroupPath: "work",
		TmuxSession: "t1", ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})
	db.CreateSession(conn, db.Session{
		ID: "c2", Title: "sub-conductor", GroupPath: "work/sub",
		TmuxSession: "t2", ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})
	db.CreateSession(conn, db.Session{
		ID: "w3", Title: "sub-worker", GroupPath: "work/sub",
		TmuxSession: "t3", ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})

	sessions, err := db.ListGroupChildSessions(conn, "work", "c1")
	if err != nil {
		t.Fatal(err)
	}
	ids := make(map[string]bool)
	for _, s := range sessions {
		ids[s.ID] = true
	}
	if ids["c2"] || ids["w3"] {
		t.Errorf("expected sub-group with own conductor excluded, got: %v", ids)
	}
}
```

- [ ] **Step 2: Run to verify they fail**

```bash
go test ./internal/db/... -run TestListGroupChild -v 2>&1 | head -10
```

Expected: compile error — `db.ListGroupChildSessions` undefined.

- [ ] **Step 3: Add `ListGroupChildSessions` to `internal/db/sessions.go`**

Add before `scanSessions`:

```go
func ListGroupChildSessions(conn *sql.DB, groupPath, conductorID string) ([]Session, error) {
	escaped := strings.NewReplacer("%", `\%`, "_", `\_`).Replace(groupPath)
	prefix := escaped + "/%"
	rows, err := conn.Query(
		`SELECT s.id, s.title, s.group_path, s.tmux_session, s.project_path, s.tool, s.status,
		        s.created_at, s.last_active, s.notes, s.archived, s.tags, s.startup_script, s.tool_flags
		 FROM sessions s
		 LEFT JOIN groups g ON s.group_path = g.path
		 WHERE (s.group_path = ? OR s.group_path LIKE ? ESCAPE '\')
		   AND (g.conductor_session_id IS NULL OR g.conductor_session_id = '' OR g.conductor_session_id = ?)
		   AND s.archived = 0`,
		groupPath, prefix, conductorID,
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	return scanSessions(rows)
}
```

- [ ] **Step 4: Run tests**

```bash
go test ./internal/db/... -run TestListGroupChild -v
```

Expected: all PASS.

- [ ] **Step 5: Run full suite**

```bash
go test ./...
```

Expected: all PASS.

- [ ] **Step 6: Commit**

```bash
git add internal/db/sessions.go internal/db/sessions_test.go
git commit -m "feat(db): add ListGroupChildSessions for heartbeat scoping"
```

---

### Task 5: Conductor heartbeat

**Files:**
- Modify: `internal/state/poller.go`
- Modify: `internal/state/poller_test.go`
- Modify: `cmd/root.go`

- [ ] **Step 1: Write failing tests**

Append to `internal/state/poller_test.go`:

```go
func TestHeartbeatSendsDigestAfterInterval(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	start := time.Now()
	now := start.Unix()
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "conductor"})
	db.CreateSession(conn, db.Session{
		ID: "conductor", Title: "conductor", GroupPath: "work", TmuxSession: "tmux-conductor",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})
	db.CreateSession(conn, db.Session{
		ID: "worker-1", Title: "worker", GroupPath: "work", TmuxSession: "tmux-worker",
		ProjectPath: "/p", Tool: "claude", Status: "waiting", CreatedAt: now,
	})

	tick := start
	stub := &stubTmux{output: "> ", exists: true}
	sender := &stubSender{}
	p := state.NewWithClock(conn, stub, nil, func() time.Time { return tick })
	p.SetSender(sender)
	p.SetConductorHeartbeat(5 * time.Minute)

	p.PollOnce() // t=0, no heartbeat yet (first call)

	tick = start.Add(6 * time.Minute)
	p.PollOnce() // t=6m, heartbeat should fire

	var heartbeatCalls int
	for _, c := range sender.calls {
		if c.session == "tmux-conductor" && strings.Contains(c.keys, "Heartbeat") {
			heartbeatCalls++
		}
	}
	if heartbeatCalls == 0 {
		t.Fatal("expected at least one heartbeat call to conductor")
	}
}

func TestHeartbeatNotFiredBeforeInterval(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	start := time.Now()
	now := start.Unix()
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "conductor"})
	db.CreateSession(conn, db.Session{
		ID: "conductor", Title: "conductor", GroupPath: "work", TmuxSession: "tmux-conductor",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})

	tick := start
	stub := &stubTmux{output: "> ", exists: true}
	sender := &stubSender{}
	p := state.NewWithClock(conn, stub, nil, func() time.Time { return tick })
	p.SetSender(sender)
	p.SetConductorHeartbeat(10 * time.Minute)

	p.PollOnce()
	tick = start.Add(3 * time.Minute)
	p.PollOnce()

	for _, c := range sender.calls {
		if strings.Contains(c.keys, "Heartbeat") {
			t.Errorf("unexpected heartbeat call before interval: %+v", c)
		}
	}
}

func TestHeartbeatAllClearWhenNoWaiting(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	start := time.Now()
	now := start.Unix()
	db.CreateGroup(conn, db.Group{Path: "work", Name: "work", ConductorSessionID: "conductor"})
	db.CreateSession(conn, db.Session{
		ID: "conductor", Title: "conductor", GroupPath: "work", TmuxSession: "tmux-conductor",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})
	db.CreateSession(conn, db.Session{
		ID: "worker-1", Title: "worker", GroupPath: "work", TmuxSession: "tmux-worker",
		ProjectPath: "/p", Tool: "claude", Status: "running", CreatedAt: now,
	})

	tick := start
	stub := &stubTmux{output: "running output", exists: true}
	sender := &stubSender{}
	p := state.NewWithClock(conn, stub, nil, func() time.Time { return tick })
	p.SetSender(sender)
	p.SetConductorHeartbeat(5 * time.Minute)

	p.PollOnce()
	tick = start.Add(6 * time.Minute)
	p.PollOnce()

	var allClear bool
	for _, c := range sender.calls {
		if c.session == "tmux-conductor" && strings.Contains(c.keys, "All clear") {
			allClear = true
		}
	}
	if !allClear {
		t.Error("expected All clear heartbeat when no workers waiting")
	}
}
```

- [ ] **Step 2: Run to verify they fail**

```bash
go test ./internal/state/... -run TestHeartbeat -v 2>&1 | head -20
```

Expected: compile error — `state.SetConductorHeartbeat` undefined.

- [ ] **Step 3: Add fields to Poller struct in `poller.go`**

In the `Poller` struct, add after `processedReplies`:

```go
heartbeatInterval time.Duration
lastHeartbeat     map[string]time.Time
```

In `NewWithClockInterval`, initialize:

```go
lastHeartbeat:    make(map[string]time.Time),
```

- [ ] **Step 4: Add `SetConductorHeartbeat` method**

After `SetSender`:

```go
func (p *Poller) SetConductorHeartbeat(interval time.Duration) {
	p.heartbeatInterval = interval
}
```

- [ ] **Step 5: Add `runHeartbeats` and `heartbeatMessage` methods**

Add after `scanConductorReplies`:

```go
func (p *Poller) runHeartbeats(sessions []db.Session) {
	if p.heartbeatInterval <= 0 || p.sender == nil {
		return
	}
	now := p.now()
	groups, err := db.ListGroups(p.conn)
	if err != nil {
		log.Printf("poller: heartbeat list groups: %v", err)
		return
	}
	sessionMap := make(map[string]db.Session, len(sessions))
	for _, s := range sessions {
		sessionMap[s.ID] = s
	}
	for _, g := range groups {
		if g.ConductorSessionID == "" {
			continue
		}
		conductor, ok := sessionMap[g.ConductorSessionID]
		if !ok || conductor.TmuxSession == "" {
			continue
		}
		p.mu.RLock()
		last := p.lastHeartbeat[g.Path]
		p.mu.RUnlock()
		if !last.IsZero() && now.Sub(last) < p.heartbeatInterval {
			continue
		}
		groupSessions, err := db.ListGroupChildSessions(p.conn, g.Path, g.ConductorSessionID)
		if err != nil {
			log.Printf("poller: heartbeat list sessions %q: %v", g.Path, err)
			continue
		}
		msg := p.heartbeatMessage(g.Path, groupSessions)
		if err := p.sender.SendKeys(conductor.TmuxSession, 0, msg); err != nil {
			log.Printf("poller: heartbeat send %q: %v", g.Path, err)
			continue
		}
		_ = p.sender.SendRawKeys(conductor.TmuxSession, 0, "Enter")
		p.mu.Lock()
		p.lastHeartbeat[g.Path] = now
		p.mu.Unlock()
	}
}

func (p *Poller) heartbeatMessage(groupPath string, sessions []db.Session) string {
	var waiting []db.Session
	for _, s := range sessions {
		if s.Status == tmux.StatusWaiting {
			waiting = append(waiting, s)
		}
	}
	if len(waiting) == 0 {
		return fmt.Sprintf("Heartbeat for %s | All clear | %d sessions active", groupPath, len(sessions))
	}
	parts := []string{fmt.Sprintf("Heartbeat for %s | %d waiting", groupPath, len(waiting))}
	p.mu.RLock()
	now := p.now()
	for _, s := range waiting {
		part := fmt.Sprintf("Worker ID: %s | Title: %s", s.ID, s.Title)
		if since, ok := p.waitingSince[s.ID]; ok {
			part += fmt.Sprintf(" | Waiting: %s", now.Sub(since).Round(time.Second))
		}
		if s.Notes != "" {
			part += fmt.Sprintf(" | Notes: %s", s.Notes)
		}
		parts = append(parts, part)
	}
	p.mu.RUnlock()
	return strings.Join(parts, " | ")
}
```

Add `"fmt"` to the imports in `poller.go` if not already present.

- [ ] **Step 6: Call `runHeartbeats` from `PollOnce`**

After the `p.scanConductorReplies(sessions)` call, add:

```go
p.runHeartbeats(sessions)
```

- [ ] **Step 7: Run tests**

```bash
go test ./internal/state/... -run TestHeartbeat -v
```

Expected: all PASS.

- [ ] **Step 8: Add `--conductor-heartbeat` flag in `cmd/root.go`**

After `var autoEscalate bool`, add:

```go
var conductorHeartbeat time.Duration
```

In `resetRootOptions()`, add:

```go
conductorHeartbeat = 0
_ = rootCmd.PersistentFlags().Set("conductor-heartbeat", "0s")
```

In `init()`, after the `--auto-escalate` flag, add:

```go
rootCmd.PersistentFlags().DurationVar(&conductorHeartbeat, "conductor-heartbeat", 0, "Send a waiting-worker digest to each group conductor on this interval (0 disables)")
```

In `launchTUI`, after `poller.SetSender(tc)`, add:

```go
if conductorHeartbeat > 0 {
    poller.SetConductorHeartbeat(conductorHeartbeat)
}
```

In `launchHeadless`, after `poller.SetSender(tc)`, add the same block.

- [ ] **Step 9: Run full test suite**

```bash
go test ./...
```

Expected: all PASS.

- [ ] **Step 10: Build and verify flag**

```bash
go build -o tmux-agent-deck . && ./tmux-agent-deck --help | grep conductor
```

Expected: `--conductor-heartbeat` appears.

- [ ] **Step 11: Commit**

```bash
git add internal/state/poller.go internal/state/poller_test.go cmd/root.go
git commit -m "feat(state,cmd): add conductor heartbeat with --conductor-heartbeat flag"
```

---

### Task 6: CLAUDE.md managed block writer

**Files:**
- Create: `internal/conductordocs/conductordocs.go`
- Create: `internal/conductordocs/conductordocs_test.go`

- [ ] **Step 1: Write failing tests**

Create `internal/conductordocs/conductordocs_test.go`:

```go
package conductordocs_test

import (
	"os"
	"path/filepath"
	"strings"
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/conductordocs"
)

func TestWriteBlockCreatesFile(t *testing.T) {
	dir := t.TempDir()
	if err := conductordocs.WriteBlock(dir); err != nil {
		t.Fatal(err)
	}
	data, err := os.ReadFile(filepath.Join(dir, "CLAUDE.md"))
	if err != nil {
		t.Fatal(err)
	}
	if !strings.Contains(string(data), "<!-- tmux-agent-deck:conductor-role:start -->") {
		t.Error("missing block start marker")
	}
	if !strings.Contains(string(data), "<!-- tmux-agent-deck:conductor-role:end -->") {
		t.Error("missing block end marker")
	}
	if !strings.Contains(string(data), "@deck-reply worker=<session-id>") {
		t.Error("missing reply syntax in block")
	}
}

func TestWriteBlockReplacesExistingBlock(t *testing.T) {
	dir := t.TempDir()
	path := filepath.Join(dir, "CLAUDE.md")
	initial := "# My Project\n\n<!-- tmux-agent-deck:conductor-role:start -->\nold content\n<!-- tmux-agent-deck:conductor-role:end -->\n\nmore user content\n"
	os.WriteFile(path, []byte(initial), 0644)

	if err := conductordocs.WriteBlock(dir); err != nil {
		t.Fatal(err)
	}
	data, err := os.ReadFile(path)
	if err != nil {
		t.Fatal(err)
	}
	content := string(data)
	if strings.Contains(content, "old content") {
		t.Error("old block content should be replaced")
	}
	if !strings.Contains(content, "more user content") {
		t.Error("user content after block should be preserved")
	}
	if !strings.Contains(content, "# My Project") {
		t.Error("user content before block should be preserved")
	}
}

func TestWriteBlockAppendsWhenNoBlock(t *testing.T) {
	dir := t.TempDir()
	path := filepath.Join(dir, "CLAUDE.md")
	os.WriteFile(path, []byte("# My Project\n\nExisting content.\n"), 0644)

	if err := conductordocs.WriteBlock(dir); err != nil {
		t.Fatal(err)
	}
	data, err := os.ReadFile(path)
	if err != nil {
		t.Fatal(err)
	}
	content := string(data)
	if !strings.Contains(content, "# My Project") {
		t.Error("existing content should be preserved")
	}
	if !strings.Contains(content, "<!-- tmux-agent-deck:conductor-role:start -->") {
		t.Error("block should be appended")
	}
}

func TestWriteBlockIdempotent(t *testing.T) {
	dir := t.TempDir()
	if err := conductordocs.WriteBlock(dir); err != nil {
		t.Fatal(err)
	}
	if err := conductordocs.WriteBlock(dir); err != nil {
		t.Fatal(err)
	}
	data, _ := os.ReadFile(filepath.Join(dir, "CLAUDE.md"))
	content := string(data)
	startCount := strings.Count(content, "<!-- tmux-agent-deck:conductor-role:start -->")
	if startCount != 1 {
		t.Errorf("expected exactly 1 start marker after two calls, got %d", startCount)
	}
}
```

- [ ] **Step 2: Run to verify they fail**

```bash
go test ./internal/conductordocs/... 2>&1 | head -10
```

Expected: compile error — package does not exist.

- [ ] **Step 3: Create `internal/conductordocs/conductordocs.go`**

```go
package conductordocs

import (
	"os"
	"path/filepath"
	"strings"
)

const blockStart = "<!-- tmux-agent-deck:conductor-role:start -->"
const blockEnd = "<!-- tmux-agent-deck:conductor-role:end -->"

const blockBody = `## Conductor Role

You are the conductor for tmux-agent-deck worker sessions.

When you receive a message beginning with "Escalation from ...":

- Identify what the worker is blocked on.
- Use the included status, notes, and context to decide the next action.
- If more repo context is needed, inspect the local project files before answering.
- Reply with a concise unblock instruction the worker can follow.
- Prefer specific commands, file paths, tests, or implementation steps.
- Do not make broad unrelated changes.
- If the escalation lacks enough context, ask one targeted follow-up question.

When sending a reply back to a worker, use:

@deck-reply worker=<session-id>
<reply body>
@deck-end`

func WriteBlock(projectPath string) error {
	claudePath := filepath.Join(projectPath, "CLAUDE.md")
	managed := blockStart + "\n" + blockBody + "\n" + blockEnd

	data, err := os.ReadFile(claudePath)
	if err != nil && !os.IsNotExist(err) {
		return err
	}

	content := string(data)
	startIdx := strings.Index(content, blockStart)
	endIdx := strings.Index(content, blockEnd)

	if startIdx >= 0 && endIdx > startIdx {
		newContent := content[:startIdx] + managed + content[endIdx+len(blockEnd):]
		return os.WriteFile(claudePath, []byte(newContent), 0644)
	}

	if content != "" && !strings.HasSuffix(content, "\n") {
		content += "\n"
	}
	content += "\n" + managed + "\n"
	return os.WriteFile(claudePath, []byte(content), 0644)
}
```

- [ ] **Step 4: Run tests**

```bash
go test ./internal/conductordocs/... -v
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/conductordocs/conductordocs.go internal/conductordocs/conductordocs_test.go
git commit -m "feat(conductordocs): add CLAUDE.md managed block writer"
```

---

### Task 7: Wire `--init-conductor-docs` flag into UI and CLI

**Files:**
- Modify: `internal/ui/app.go`
- Modify: `cmd/root.go`

- [ ] **Step 1: Add `InitConductorDocs` field to Model in `internal/ui/app.go`**

In the `Model` struct, after `contextPct map[string]*int`, add:

```go
InitConductorDocs bool
```

Add the import for `conductordocs` at the top of `app.go`:

```go
"github.com/black-gato/tmux-agent-deck/internal/conductordocs"
```

- [ ] **Step 2: Call `conductordocs.WriteBlock` in the `set-conductor` case**

In `app.go`, find the `set-conductor` case (around line 315). After the `m.Reload()` call, add:

```go
if m.InitConductorDocs && session.ProjectPath != "" {
    if err := conductordocs.WriteBlock(session.ProjectPath); err != nil {
        m.err = fmt.Errorf("init conductor docs: %w", err)
    }
}
```

The full updated `set-conductor` case:

```go
case "set-conductor":
    if m.cursor < len(m.items) && m.items[m.cursor].Kind == "session" {
        session := m.items[m.cursor].Session
        if err := db.SetGroupConductor(m.conn, session.GroupPath, session.ID); err != nil {
            m.err = err
            break
        }
        if err := m.Reload(); err != nil {
            m.err = err
            break
        }
        if m.InitConductorDocs && session.ProjectPath != "" {
            if err := conductordocs.WriteBlock(session.ProjectPath); err != nil {
                m.err = fmt.Errorf("init conductor docs: %w", err)
            }
        }
    }
```

- [ ] **Step 3: Add `--init-conductor-docs` flag in `cmd/root.go`**

After `var conductorHeartbeat time.Duration`, add:

```go
var initConductorDocs bool
```

In `resetRootOptions()`, add:

```go
initConductorDocs = false
_ = rootCmd.PersistentFlags().Set("init-conductor-docs", "false")
```

In `init()`, after `--conductor-heartbeat`, add:

```go
rootCmd.PersistentFlags().BoolVar(&initConductorDocs, "init-conductor-docs", false, "Write conductor role instructions into CLAUDE.md when setting a conductor with c")
```

In `launchTUI`, after the `poller.SetConductorHeartbeat` block, set the field on the model. The model is created inside the loop as `m := ui.NewModel(conn, tc, poller)`. Update that line to:

```go
m := ui.NewModel(conn, tc, poller)
m.InitConductorDocs = initConductorDocs
```

- [ ] **Step 4: Build and verify**

```bash
go build -o tmux-agent-deck . && ./tmux-agent-deck --help | grep init-conductor
```

Expected: `--init-conductor-docs` appears.

- [ ] **Step 5: Run full test suite**

```bash
go test ./...
```

Expected: all PASS.

- [ ] **Step 6: Commit**

```bash
git add internal/ui/app.go cmd/root.go
git commit -m "feat(ui,cmd): add --init-conductor-docs flag; write CLAUDE.md block on conductor assignment"
```

---

### Task 8: Final verification

**Files:**
- Modify as needed based on results

- [ ] **Step 1: Run all tests**

```bash
go test ./...
```

Expected: all PASS.

- [ ] **Step 2: Build**

```bash
go build -o tmux-agent-deck .
```

Expected: no errors.

- [ ] **Step 3: Verify all three flags appear**

```bash
./tmux-agent-deck --help | grep -E "auto-escalate|conductor-heartbeat|init-conductor-docs"
```

Expected: all three flags listed.

- [ ] **Step 4: Run vet**

```bash
go vet ./...
```

Expected: no output.
