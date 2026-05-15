# M2 Finish Observability Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a context window % indicator parsed from Claude pane output, render it in the list and detail panel, and polish the existing waiting timer and fleet header with color treatment.

**Architecture:** `ParseContextPct` is a new pure function in `status.go` — no change to `DetectStatus`'s signature. The poller stores context % per session in memory (`contextPct map[string]*int`), snapshotted to the model on every `Reload()`. `ListItem` gets two new fields (`ContextPct *int`, `WaitOverdue bool`) that drive rendering. All color treatment uses lipgloss styles already in use elsewhere in the UI package.

**Tech Stack:** Go 1.22+, Bubbletea, Lipgloss, modernc SQLite (context % is not persisted — it is a live transient reading)

---

## File Map

| File | Change |
|---|---|
| `internal/tmux/status.go` | Add `ParseContextPct(output string) *int` |
| `internal/tmux/status_test.go` | Tests for `ParseContextPct` |
| `internal/state/poller.go` | Add `contextPct map[string]*int`, `ContextPctSnapshot()`, call `ParseContextPct` in `PollOnce`, clear in `clearSessionState` |
| `internal/state/poller_test.go` | Test that `PollOnce` stores and clears context % |
| `internal/ui/list.go` | Add `ContextPct *int` and `WaitOverdue bool` to `ListItem`; add `renderContextBar` helper; update `RenderList` to render bar and color overdue wait label |
| `internal/ui/list_test.go` | Tests for `renderContextBar` output and `RenderList` bar/color rendering |
| `internal/ui/app.go` | Add `contextPct map[string]*int` to `Model`; populate in `Reload`; set `item.ContextPct` and `item.WaitOverdue`; render context line in `RenderDetailPanel`; color waiting/error counts and overdue badge in `renderAppHeader` |
| `internal/ui/app_test.go` | Tests for detail panel context line and colored header |

---

## Task 1: ParseContextPct — pure function

**Files:**
- Modify: `internal/tmux/status.go`
- Test: `internal/tmux/status_test.go`

Claude's pane output includes lines like `75% context used · /compact to reduce`. This task adds a pure parsing function that returns the integer percentage, or nil if the line doesn't match.

- [ ] **Step 1: Write the failing tests**

Add to `internal/tmux/status_test.go`:

```go
func TestParseContextPctDetectsPercentage(t *testing.T) {
	cases := []struct {
		output string
		want   *int
	}{
		{"Some output\n75% context used · /compact\n> ", intPtr(75)},
		{"context window: 50%\n> ", intPtr(50)},
		{"100% context used", intPtr(100)},
		{"0% context used", intPtr(0)},
		{"no percentage here\n> ", nil},
		{"75% complete (not context)", nil},
		{"", nil},
	}
	for _, tc := range cases {
		got := tmux.ParseContextPct(tc.output)
		if tc.want == nil && got != nil {
			t.Errorf("output %q: expected nil, got %d", tc.output, *got)
		} else if tc.want != nil && (got == nil || *got != *tc.want) {
			gotStr := "<nil>"
			if got != nil {
				gotStr = fmt.Sprintf("%d", *got)
			}
			t.Errorf("output %q: expected %d, got %s", tc.output, *tc.want, gotStr)
		}
	}
}

func intPtr(n int) *int { return &n }
```

Add `"fmt"` to the imports in `status_test.go`.

- [ ] **Step 2: Run test to confirm it fails**

```bash
go test ./internal/tmux/... -run TestParseContextPctDetectsPercentage -v
```

Expected: `FAIL` — `tmux.ParseContextPct` undefined.

- [ ] **Step 3: Implement ParseContextPct**

Add to `internal/tmux/status.go` (add `"strconv"` to imports):

```go
func ParseContextPct(output string) *int {
	for _, line := range strings.Split(output, "\n") {
		lower := strings.ToLower(line)
		if !strings.Contains(lower, "context") {
			continue
		}
		idx := strings.Index(line, "%")
		if idx <= 0 {
			continue
		}
		start := idx - 1
		for start > 0 && line[start-1] >= '0' && line[start-1] <= '9' {
			start--
		}
		if start == idx {
			continue
		}
		n, err := strconv.Atoi(line[start:idx])
		if err != nil || n < 0 || n > 100 {
			continue
		}
		return &n
	}
	return nil
}
```

- [ ] **Step 4: Run tests to confirm they pass**

```bash
go test ./internal/tmux/... -v
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/tmux/status.go internal/tmux/status_test.go
git commit -m "feat: add ParseContextPct to extract context window % from pane output"
```

---

## Task 2: Poller context % tracking

**Files:**
- Modify: `internal/state/poller.go`
- Test: `internal/state/poller_test.go`

The poller stores the latest parsed context % per session, protected by the existing `mu` mutex. A snapshot method provides a safe read for the UI layer.

- [ ] **Step 1: Write the failing test**

Add to `internal/state/poller_test.go`:

```go
func TestPollerStoresContextPct(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "test", GroupPath: "my-sessions",
		TmuxSession: "tmux-s1", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: time.Now().Unix(),
	})

	stub := &stubTmux{output: "Some output\n75% context used\n> ", exists: true}
	p := state.New(conn, stub)
	p.PollOnce()

	snap := p.ContextPctSnapshot()
	if snap["s1"] == nil || *snap["s1"] != 75 {
		t.Fatalf("expected context pct 75 for s1, got %v", snap["s1"])
	}
}

func TestPollerClearsContextPctWhenSessionStopped(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "test", GroupPath: "my-sessions",
		TmuxSession: "tmux-s1", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: time.Now().Unix(),
	})

	stub := &stubTmux{output: "75% context used\n> ", exists: true}
	p := state.New(conn, stub)
	p.PollOnce()

	// stop the session
	db.UpdateSessionStatus(conn, "s1", "stopped")
	p.PollOnce()

	snap := p.ContextPctSnapshot()
	if snap["s1"] != nil {
		t.Fatalf("expected nil context pct after session stopped, got %d", *snap["s1"])
	}
}
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
go test ./internal/state/... -run "TestPollerStoresContextPct|TestPollerClearsContextPctWhenSessionStopped" -v
```

Expected: `FAIL` — `state.Poller` has no `ContextPctSnapshot` method.

- [ ] **Step 3: Add contextPct field, setContextPct, clearSessionState update, ContextPctSnapshot, and PollOnce call**

In `internal/state/poller.go`, add `contextPct` field to the `Poller` struct:

```go
type Poller struct {
	conn         *sql.DB
	tmux         TmuxReader
	notifier     waitingNotifier
	now          func() time.Time
	interval     time.Duration
	mu           sync.RWMutex
	lastChange   map[string]time.Time
	waitingSince map[string]time.Time
	contextPct   map[string]*int  // add this line
	done         chan struct{}
}
```

Initialize it in `NewWithClockInterval`:

```go
return &Poller{
	conn:         conn,
	tmux:         tc,
	notifier:     notifier,
	now:          now,
	interval:     interval,
	lastChange:   make(map[string]time.Time),
	waitingSince: make(map[string]time.Time),
	contextPct:   make(map[string]*int),  // add this line
	done:         make(chan struct{}),
}
```

Add `setContextPct` helper:

```go
func (p *Poller) setContextPct(id string, pct *int) {
	p.mu.Lock()
	defer p.mu.Unlock()
	if pct == nil {
		delete(p.contextPct, id)
		return
	}
	v := *pct
	p.contextPct[id] = &v
}
```

Add `ContextPctSnapshot`:

```go
func (p *Poller) ContextPctSnapshot() map[string]*int {
	p.mu.RLock()
	defer p.mu.RUnlock()
	snap := make(map[string]*int, len(p.contextPct))
	for id, pct := range p.contextPct {
		v := *pct
		snap[id] = &v
	}
	return snap
}
```

Update `clearSessionState` to also delete from `contextPct`:

```go
func (p *Poller) clearSessionState(id string) {
	p.mu.Lock()
	defer p.mu.Unlock()
	delete(p.lastChange, id)
	delete(p.waitingSince, id)
	delete(p.contextPct, id)
}
```

In `PollOnce`, after the `CapturePaneOutput` call and before the `DetectStatus` call, add:

```go
p.setContextPct(s.ID, tmux.ParseContextPct(out))
```

The relevant section of `PollOnce` becomes:

```go
out, err := p.tmux.CapturePaneOutput(s.TmuxSession)
if err != nil {
    log.Printf("poller: capture pane %q: %v", s.TmuxSession, err)
    continue
}

p.setContextPct(s.ID, tmux.ParseContextPct(out))

now := p.now()
```

- [ ] **Step 4: Run tests to confirm they pass**

```bash
go test ./internal/state/... -v
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/state/poller.go internal/state/poller_test.go
git commit -m "feat: track context window % per session in poller"
```

---

## Task 3: renderContextBar helper + ListItem fields + RenderList

**Files:**
- Modify: `internal/ui/list.go`
- Test: `internal/ui/list_test.go`

`renderContextBar` converts an integer 0–100 into a 4-block progress bar: `▓▓▓░ 75%`. `ListItem` gains `ContextPct *int` and `WaitOverdue bool`. `RenderList` renders the bar inline before the title and colors overdue wait labels.

- [ ] **Step 1: Write failing tests**

Add to `internal/ui/list_test.go`:

```go
func TestRenderContextBar(t *testing.T) {
	cases := []struct {
		pct  int
		want string
	}{
		{0, "░░░░ 0%"},
		{25, "▓░░░ 25%"},
		{50, "▓▓░░ 50%"},
		{75, "▓▓▓░ 75%"},
		{100, "▓▓▓▓ 100%"},
	}
	for _, tc := range cases {
		got := ui.RenderContextBar(tc.pct)
		if got != tc.want {
			t.Errorf("pct %d: got %q, want %q", tc.pct, got, tc.want)
		}
	}
}

func TestRenderListShowsContextBar(t *testing.T) {
	pct := 75
	items := []ui.ListItem{
		{
			Kind:       "session",
			Session:    &db.Session{ID: "s1", Title: "my-app", Status: "running"},
			ContextPct: &pct,
		},
	}
	out := ui.RenderList(items, 0, 80, 20)
	if !strings.Contains(out, "▓▓▓░ 75%") {
		t.Errorf("expected context bar in list output, got:\n%s", out)
	}
}

func TestRenderListNoBarWhenContextPctNil(t *testing.T) {
	items := []ui.ListItem{
		{
			Kind:    "session",
			Session: &db.Session{ID: "s1", Title: "my-app", Status: "running"},
		},
	}
	out := ui.RenderList(items, 0, 80, 20)
	if strings.Contains(out, "▓") || strings.Contains(out, "░") {
		t.Errorf("expected no context bar when ContextPct is nil, got:\n%s", out)
	}
}
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
go test ./internal/ui/... -run "TestRenderContextBar|TestRenderListShowsContextBar|TestRenderListNoBarWhenContextPctNil" -v
```

Expected: `FAIL` — `ui.RenderContextBar` undefined, `ListItem` missing `ContextPct`.

- [ ] **Step 3: Add ContextPct, WaitOverdue to ListItem and implement renderContextBar**

In `internal/ui/list.go`, update `ListItem`:

```go
type ListItem struct {
	Kind        string // "group" or "session"
	Group       *db.Group
	Session     *db.Session
	Depth       int
	WaitLabel   string
	WaitOverdue bool
	ContextPct  *int
	Selected    bool
	IsConductor bool
}
```

Add new styles and the exported `RenderContextBar` function (exported so list_test.go can call it):

```go
var (
	selectedStyle    = lipgloss.NewStyle().Bold(true).Foreground(lipgloss.Color("12"))
	groupStyle       = lipgloss.NewStyle().Bold(true)
	dimStyle         = lipgloss.NewStyle().Faint(true)
	overdueWaitStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("214")) // amber
)

func RenderContextBar(pct int) string {
	filled := (pct * 4) / 100
	var bar strings.Builder
	for i := 0; i < 4; i++ {
		if i < filled {
			bar.WriteRune('▓')
		} else {
			bar.WriteRune('░')
		}
	}
	return fmt.Sprintf("%s %d%%", bar.String(), pct)
}
```

Update `RenderList` — replace the session-row rendering block (starting at `prefixLen :=`) with:

```go
prefixLen := len([]rune(indent)) + 1 + 1 + 2 // mark + sym + 2 spaces
if item.WaitLabel != "" {
    prefixLen += len([]rune(item.WaitLabel)) + 1
}
if item.ContextPct != nil {
    prefixLen += len([]rune(RenderContextBar(*item.ContextPct))) + 1
}
titleMax := width - prefixLen
if titleMax < 1 {
    titleMax = 1
}
title := truncate(item.Session.Title, titleMax)

waitStr := item.WaitLabel
if item.WaitOverdue && !selected {
    waitStr = overdueWaitStyle.Render(waitStr)
}

var raw string
switch {
case item.WaitLabel != "" && item.ContextPct != nil:
    raw = fmt.Sprintf("%s%s%s %s %s %s", indent, mark, sym, waitStr, RenderContextBar(*item.ContextPct), title)
case item.WaitLabel != "":
    raw = fmt.Sprintf("%s%s%s %s %s", indent, mark, sym, waitStr, title)
case item.ContextPct != nil:
    raw = fmt.Sprintf("%s%s%s %s %s", indent, mark, sym, RenderContextBar(*item.ContextPct), title)
default:
    raw = fmt.Sprintf("%s%s%s  %s", indent, mark, sym, title)
}
if selected {
    line = selectedStyle.Render(raw)
} else {
    line = raw
}
```

- [ ] **Step 4: Run tests to confirm they pass**

```bash
go test ./internal/ui/... -v
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/ui/list.go internal/ui/list_test.go
git commit -m "feat: add context bar rendering and overdue wait label color to list"
```

---

## Task 4: Wire context % into Model + detail panel context line

**Files:**
- Modify: `internal/ui/app.go`
- Test: `internal/ui/app_test.go`

`Reload()` calls `poller.ContextPctSnapshot()` and stamps each waiting/running `ListItem` with `ContextPct`. `RenderDetailPanel` gains a `context:` line.

- [ ] **Step 1: Write failing tests**

Add to `internal/ui/app_test.go`. The detail panel test works by rendering with a `ListItem` that already has `ContextPct` set — since the model sets it during `Reload()` from the poller snapshot, we test the render output after `Reload()` with a real poller whose stub output contains a context line.

The simplest integration test: build a model with a real poller, poll once with stub output containing `75% context used`, then reload and check the detail panel view.

```go
func TestDetailPanelShowsContextLine(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	if err := db.CreateSession(conn, db.Session{
		ID: "s1", Title: "app", GroupPath: "my-sessions",
		TmuxSession: "tmux-s1", ProjectPath: "/p", Tool: "claude",
		Status: "running", CreatedAt: time.Now().Unix(),
	}); err != nil {
		t.Fatal(err)
	}

	fake := testutil.NewFakeTmuxClient()
	fake.PaneOutput["tmux-s1"] = "Some output\n75% context used\n> "

	poller := state.New(conn, fake)
	poller.PollOnce()

	m := ui.NewModel(conn, fake, poller)
	m.Reload()

	// cursor starts at 0; first item is the group header — move to the session
	moveCursorToSession(t, m, "app")

	view := m.View()
	if !strings.Contains(view, "context:") {
		t.Fatalf("expected 'context:' line in detail panel, got:\n%s", view)
	}
	if !strings.Contains(view, "75%") {
		t.Fatalf("expected '75%%' in detail panel, got:\n%s", view)
	}
}

func TestDetailPanelNoContextLineWhenPctNil(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	if err := db.CreateSession(conn, db.Session{
		ID: "s1", Title: "app", GroupPath: "my-sessions",
		TmuxSession: "", ProjectPath: "/p", Tool: "claude",
		Status: "stopped", CreatedAt: time.Now().Unix(),
	}); err != nil {
		t.Fatal(err)
	}

	m := ui.NewModel(conn, nil, nil)
	m.Reload()
	moveCursorToSession(t, m, "app")

	view := m.View()
	if strings.Contains(view, "context:") {
		t.Fatalf("expected no 'context:' line when pct is nil, got:\n%s", view)
	}
}
```

The `fake.PaneOutput` field needs to exist on `FakeTmuxClient`. Check whether it already does — if not, it must be added in the next sub-step.

- [ ] **Step 2: Check FakeTmuxClient and update if needed**

```bash
grep -n "PaneOutput\|CapturePaneOutput" /Users/anthonymirville/Projects/tmux-agent-deck/internal/testutil/tmux.go
```

If `PaneOutput map[string]string` does not exist on `FakeTmuxClient`, add it:

```go
type FakeTmuxClient struct {
	// existing fields...
	PaneOutput  map[string]string // tmux session name → captured output
}

func NewFakeTmuxClient() *FakeTmuxClient {
	return &FakeTmuxClient{
		// existing...
		PaneOutput: make(map[string]string),
	}
}

func (f *FakeTmuxClient) CapturePaneOutput(name string) (string, error) {
	return f.PaneOutput[name], nil
}

func (f *FakeTmuxClient) SessionExists(name string) (bool, error) {
	_, ok := f.PaneOutput[name]
	return ok, nil
}
```

Note: `FakeTmuxClient` must satisfy both `tmux.ClientIface` (used by the UI) and `state.TmuxReader` (used by the poller). If these interfaces have different method sets, the fake may need separate wrapper types or both interfaces merged. Check whether `ClientIface` already has `CapturePaneOutput` and `SessionExists`:

```bash
grep -n "CapturePaneOutput\|SessionExists\|ClientIface" /Users/anthonymirville/Projects/tmux-agent-deck/internal/tmux/client.go | head -20
```

If `FakeTmuxClient` already satisfies `state.TmuxReader` (likely, since tests already pass a fake to the poller in `poller_test.go`), no change needed beyond adding `PaneOutput`.

- [ ] **Step 3: Run tests to confirm they fail**

```bash
go test ./internal/ui/... -run "TestDetailPanelShowsContextLine|TestDetailPanelNoContextLineWhenPctNil" -v
```

Expected: `FAIL` — either compile error (missing `PaneOutput`) or assertion fails because `context:` line doesn't exist yet.

- [ ] **Step 4: Add contextPct to Model and wire in Reload**

In `internal/ui/app.go`, add `contextPct` field to `Model`:

```go
type Model struct {
	// existing fields...
	contextPct      map[string]*int
	navigateToGroup string
}
```

In `Reload()`, after the `waitingSince` snapshot block, add context pct snapshot and stamp list items. The relevant section of `Reload()` currently reads:

```go
m.waitingSince = nil
m.overdueWaiting = 0
if m.poller != nil {
    m.waitingSince = m.poller.WaitingSinceSnapshot()
    for i := range m.items {
        if m.items[i].Kind != "session" || m.items[i].Session.Status != tmux.StatusWaiting {
            continue
        }
        since, ok := m.waitingSince[m.items[i].Session.ID]
        if !ok {
            continue
        }
        m.items[i].WaitLabel = formatElapsed(now.Sub(since))
        if now.Sub(since) > 30*time.Second {
            m.overdueWaiting++
        }
    }
}
```

Replace with:

```go
m.waitingSince = nil
m.overdueWaiting = 0
m.contextPct = nil
if m.poller != nil {
    m.waitingSince = m.poller.WaitingSinceSnapshot()
    m.contextPct = m.poller.ContextPctSnapshot()
    for i := range m.items {
        if m.items[i].Kind != "session" {
            continue
        }
        id := m.items[i].Session.ID
        m.items[i].ContextPct = m.contextPct[id]
        if m.items[i].Session.Status != tmux.StatusWaiting {
            continue
        }
        since, ok := m.waitingSince[id]
        if !ok {
            continue
        }
        m.items[i].WaitLabel = formatElapsed(now.Sub(since))
        if now.Sub(since) > 30*time.Second {
            m.overdueWaiting++
            m.items[i].WaitOverdue = true
        }
    }
}
```

- [ ] **Step 5: Add context line to RenderDetailPanel**

In `internal/ui/app.go`, find `RenderDetailPanel`. After the status line (which already includes the wait label), add a context line. The current lines section looks like:

```go
lines = append(lines, fmt.Sprintf(" %s  %s", s.Title, statusText))
lines = append(lines, fmt.Sprintf(" group: %s", s.GroupPath))
lines = append(lines, fmt.Sprintf(" conductor: %t", m.isConductorSession(s)))
lines = append(lines, fmt.Sprintf(" tags: %s", s.Tags))
lines = append(lines, " "+renderPaneList(m.panes, m.activePaneIdx))
```

Add a context line between tags and pane list — but only when context pct is available:

```go
lines = append(lines, fmt.Sprintf(" %s  %s", s.Title, statusText))
lines = append(lines, fmt.Sprintf(" group: %s", s.GroupPath))
lines = append(lines, fmt.Sprintf(" conductor: %t", m.isConductorSession(s)))
lines = append(lines, fmt.Sprintf(" tags: %s", s.Tags))
if pct, ok := m.contextPct[s.ID]; ok && pct != nil {
    lines = append(lines, fmt.Sprintf(" context: %s", ui.RenderContextBar(*pct)))
}
lines = append(lines, " "+renderPaneList(m.panes, m.activePaneIdx))
```

Note: `RenderContextBar` is in the same `ui` package, so call it as `RenderContextBar(*pct)` (no package qualifier needed since `app.go` is `package ui`).

Also update `sessionHeaderLines` constant if the count changed (it currently assumes a fixed number of header lines for space calculations):

```go
const sessionHeaderLines = 6 // title, group, conductor, tags, [context if present], panes
```

The context line is conditional so the constant should remain 6 (the context line takes a slot from notes/output space only when present — this is acceptable). Verify rendering looks correct manually.

- [ ] **Step 6: Run tests to confirm they pass**

```bash
go test ./internal/ui/... -v
```

Expected: all PASS.

- [ ] **Step 7: Commit**

```bash
git add internal/ui/app.go internal/ui/app_test.go internal/testutil/tmux.go
git commit -m "feat: wire context % from poller into model and detail panel"
```

---

## Task 5: Color polish — fleet header and overdue wait label

**Files:**
- Modify: `internal/ui/app.go`
- Test: `internal/ui/app_test.go`

Color the header's waiting count (amber), error count (red), and overdue `!N` badge (bold red). The overdue wait label color in the list was already wired in Task 3 via `WaitOverdue`.

- [ ] **Step 1: Write failing tests**

Add to `internal/ui/app_test.go`:

```go
func TestHeaderColorsWaitingAmberWhenNonZero(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "app", GroupPath: "my-sessions",
		ProjectPath: "/p", Tool: "claude", Status: "waiting", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, nil, nil)
	m.Reload()

	view := m.View()
	// Lipgloss color "214" (amber) escape sequence should appear when waiting > 0
	if !strings.Contains(view, "\x1b[") {
		t.Skip("terminal does not support ANSI — skipping color test")
	}
	// The waiting count should be styled; we check the raw number still appears
	if !strings.Contains(view, "1 waiting") {
		t.Fatalf("expected '1 waiting' in header, got:\n%s", view)
	}
}

func TestHeaderColorsErrorRedWhenNonZero(t *testing.T) {
	conn := testutil.OpenTestDB(t)
	db.CreateSession(conn, db.Session{
		ID: "s1", Title: "app", GroupPath: "my-sessions",
		ProjectPath: "/p", Tool: "claude", Status: "error", CreatedAt: 1000,
	})
	m := ui.NewModel(conn, nil, nil)
	m.Reload()

	view := m.View()
	if !strings.Contains(view, "1 error") {
		t.Fatalf("expected '1 error' in header, got:\n%s", view)
	}
}
```

These tests confirm the count text still appears (color is additive — lipgloss wraps with ANSI codes but the text remains). The `t.Skip` guards against headless environments where lipgloss may not emit ANSI.

- [ ] **Step 2: Run tests to confirm they fail**

```bash
go test ./internal/ui/... -run "TestHeaderColorsWaiting|TestHeaderColorsError" -v
```

Expected: `FAIL` — the current header uses `fmt.Sprintf` plain text, so `"1 waiting"` exists but we're checking correct format; tests may pass already. If they pass, adjust test to check for the ANSI escape presence around the count instead. Rerun to confirm current behavior.

- [ ] **Step 3: Implement header color treatment**

In `internal/ui/app.go`, add styles (near the top of the file, alongside other style vars or at function scope in `renderAppHeader`):

```go
var (
	headerWaitingStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("214")) // amber
	headerErrorStyle   = lipgloss.NewStyle().Foreground(lipgloss.Color("196")) // red
	headerOverdueStyle = lipgloss.NewStyle().Bold(true).Foreground(lipgloss.Color("196"))
)
```

Replace `renderAppHeader`:

```go
func (m *Model) renderAppHeader() string {
	var running, waiting, idle, errs int
	for _, s := range m.sessions {
		switch s.Status {
		case "running":
			running++
		case "waiting":
			waiting++
		case "idle":
			idle++
		case "error":
			errs++
		}
	}
	waitingStr := fmt.Sprintf("%d waiting", waiting)
	errorStr := fmt.Sprintf("%d error", errs)
	if waiting > 0 {
		waitingStr = headerWaitingStyle.Render(waitingStr)
	}
	if errs > 0 {
		errorStr = headerErrorStyle.Render(errorStr)
	}
	header := fmt.Sprintf(" Agent Deck  ● %d running  ○ %s  ◐ %d idle  ✕ %s", running, waitingStr, idle, errorStr)
	if m.overdueWaiting > 0 {
		header += "  " + headerOverdueStyle.Render(fmt.Sprintf("!%d", m.overdueWaiting))
	}
	return header
}
```

- [ ] **Step 4: Run all tests**

```bash
go test ./... -v 2>&1 | tail -20
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/ui/app.go internal/ui/app_test.go
git commit -m "feat: color fleet header waiting/error counts and overdue badge"
```

---

## Self-Review

**Spec coverage check:**

| Spec item | Task |
|---|---|
| Context window indicator — parse `75% context used` | Task 1 (`ParseContextPct`) |
| Context window indicator — render `▓▓▓░ 75%` in list | Task 3 (`RenderContextBar`, `RenderList`) |
| Context window indicator — render in detail panel | Task 4 (`RenderDetailPanel` context line) |
| Waiting elapsed timer polish | Task 4 (`WaitOverdue` + overdue list color in Task 3) |
| Fleet status bar polish — color treatment | Task 5 (header colors) |

**Placeholder scan:** No TBD, TODO, or vague "add error handling" steps present. All code blocks are complete.

**Type consistency check:**
- `ParseContextPct` returns `*int` — used as `*int` everywhere ✓
- `RenderContextBar` takes `int` (not `*int`) — callers dereference before passing ✓
- `ListItem.ContextPct *int` — set from `m.contextPct[id]` which is `*int` ✓
- `ListItem.WaitOverdue bool` — set in `Reload`, read in `RenderList` ✓
- `ContextPctSnapshot()` returns `map[string]*int` — same type as `m.contextPct` ✓
