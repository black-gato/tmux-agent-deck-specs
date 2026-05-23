# Session Worktree Options Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the new-session form with BRANCH/BASE/WORKTREE fields so that filling BRANCH triggers `git worktree add` and the session runs inside the new worktree directory.

**Architecture:** Three new `fieldText` fields are inserted at indices 2/3/4 (between PATH and TOOL), shifting TOOL/FLAGS/SCRIPT to 5/6/7. All new logic lives in `internal/ui/form.go`. `commitForm` calls `runGitWorktreeAdd` when BRANCH is non-empty; on failure it sets `formState.formErr` and leaves mode open. No schema change — `project_path` already stores the worktree dir.

**Tech Stack:** Go stdlib `os/exec`, `path/filepath`, `strings`; existing Bubbletea form infrastructure; `lipgloss` for red error style.

---

### Task 1: Insert BRANCH/BASE/WORKTREE fields and extend formState

**Files:**
- Modify: `internal/ui/form.go` — `formState`, `initSessionForm`, `commitForm`, `updateForm`
- Modify: `internal/ui/app.go` — add `FormErr()` accessor
- Test: `internal/ui/form_test.go`

- [ ] **Step 1: Write the failing test**

Add to `internal/ui/form_test.go`:

```go
func TestFormHasEightFields(t *testing.T) {
	m, _ := openModel(t)
	m = sendKey(m, rune_('n'))
	// Down 7 times lands on last field (index 7); once more should clamp, not panic
	for i := 0; i < 8; i++ {
		m = sendKey(m, key(tea.KeyDown))
	}
	if m.Mode() != "new-session" {
		t.Fatalf("expected new-session mode, got %q", m.Mode())
	}
	// Focus clamped at last field (7), not 8
	if m.FormFocusField() != 7 {
		t.Errorf("expected focus at 7, got %d", m.FormFocusField())
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/ui/ -run TestFormHasEightFields -v
```

Expected: FAIL — focus ends at 4 (last of 5 current fields) not 7.

- [ ] **Step 3: Add `formErr` and `worktreeUserEdited` to `formState`**

In `internal/ui/form.go`, change `formState`:

```go
type formState struct {
	fields              []formField
	focusField          int
	candidates          []string
	candIdx             int
	candActive          bool
	candBase            string
	formErr             string
	worktreeUserEdited  bool
}
```

- [ ] **Step 4: Insert BRANCH/BASE/WORKTREE in `initSessionForm`**

Replace the `m.form = formState{...}` block in `initSessionForm`:

```go
m.form = formState{
    fields: []formField{
        {label: "TITLE", kind: fieldText},
        {label: "PATH", kind: fieldText, value: defaultPath, cursor: len([]rune(defaultPath))},
        {label: "BRANCH", kind: fieldText},
        {label: "BASE", kind: fieldText},
        {label: "WORKTREE", kind: fieldText},
        {label: "TOOL", kind: fieldSelect, options: toolOptions, optIdx: toolIdx},
        {label: "FLAGS", kind: fieldText},
        {label: "SCRIPT", kind: fieldText},
    },
}
```

- [ ] **Step 5: Update field indices in `commitForm`**

Replace the field-index reads in `commitForm`:

```go
func (m *Model) commitForm() {
	title := strings.TrimSpace(m.form.fields[0].value)
	if title == "" {
		return
	}
	path := strings.TrimSpace(m.form.fields[1].value)
	if path == "" {
		path = "."
	}
	tool := m.form.fields[5].options[m.form.fields[5].optIdx]
	flags := strings.TrimSpace(m.form.fields[6].value)
	script := strings.TrimSpace(m.form.fields[7].value)

	if err := db.CreateSession(m.conn, db.Session{
		ID:            uuid.New().String(),
		Title:         title,
		GroupPath:     m.currentGroupPath(),
		ProjectPath:   expandPath(path),
		Tool:          tool,
		ToolFlags:     flags,
		Status:        "stopped",
		CreatedAt:     time.Now().Unix(),
		StartupScript: script,
	}); err != nil {
		m.err = err
	}
}
```

(BRANCH logic is added in Task 5; this step only corrects the indices.)

- [ ] **Step 6: Add `FormErr()` accessor to `app.go`**

Add alongside the existing `Mode()` and `FormFocusField()` methods:

```go
func (m *Model) FormErr() string { return m.form.formErr }
```

- [ ] **Step 7: Run the failing test — it should pass now**

```bash
go test ./internal/ui/ -run TestFormHasEightFields -v
```

Expected: PASS.

- [ ] **Step 8: Run full suite to catch regressions**

```bash
go test ./internal/ui/ -v 2>&1 | tail -30
```

Three tests are expected to fail at this point — `TestToolSelectorCycles`, `TestNewSessionFlowCreatesSessionWithTool`, `TestNewSessionDialogPersistsToolFlags` — because they navigate PATH→TOOL in one Tab, which now lands on BRANCH. All other tests should pass.

- [ ] **Step 9: Commit**

```bash
git add internal/ui/form.go internal/ui/app.go internal/ui/form_test.go
git commit -m "feat(ui): insert BRANCH/BASE/WORKTREE fields in session form (8 fields)"
```

---

### Task 2: Add git helpers

**Files:**
- Modify: `internal/ui/form.go` — add `slugBranch`, `deriveWorktreePath`, `resolveDefaultBranch`, `runGitWorktreeAdd`; add imports `fmt`, `os/exec`, `path/filepath`
- Test: `internal/ui/form_test.go`

- [ ] **Step 1: Write the failing tests**

Add to `internal/ui/form_test.go`:

```go
func TestDeriveWorktreePath(t *testing.T) {
	cases := []struct {
		repo, branch, want string
	}{
		{"/home/user/myrepo", "feature/my-branch", "/home/user/myrepo-feature-my-branch"},
		{"/home/user/myrepo", "main", "/home/user/myrepo-main"},
		{"/home/user/myrepo", "FEATure_X", "/home/user/myrepo-feature-x"},
	}
	for _, c := range cases {
		got := ui.DeriveWorktreePath(c.repo, c.branch)
		if got != c.want {
			t.Errorf("DeriveWorktreePath(%q, %q) = %q, want %q", c.repo, c.branch, got, c.want)
		}
	}
}

func TestResolveDefaultBranch(t *testing.T) {
	if _, err := exec.LookPath("git"); err != nil {
		t.Skip("git not available")
	}
	repo := t.TempDir()
	run := func(args ...string) {
		cmd := exec.Command("git", args...)
		cmd.Dir = repo
		if out, err := cmd.CombinedOutput(); err != nil {
			t.Fatalf("git %v: %s", args, out)
		}
	}
	run("init")
	run("config", "user.email", "test@test.com")
	run("config", "user.name", "Test")
	run("commit", "--allow-empty", "-m", "init")

	branch := ui.ResolveDefaultBranch(repo)
	if branch == "" {
		t.Fatal("expected non-empty default branch")
	}
}
```

Note: `DeriveWorktreePath` and `ResolveDefaultBranch` must be exported for black-box tests. The unexported helpers `slugBranch`, `runGitWorktreeAdd` are tested indirectly via `TestCommitWithWorktree` in Task 5.

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/ui/ -run "TestDeriveWorktreePath|TestResolveDefaultBranch" -v
```

Expected: FAIL — functions not defined.

- [ ] **Step 3: Add imports to `form.go`**

Add to the import block in `internal/ui/form.go`:

```go
import (
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"time"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	tea "github.com/charmbracelet/bubbletea"
	"github.com/charmbracelet/lipgloss"
	"github.com/google/uuid"
)
```

- [ ] **Step 4: Implement the helpers in `form.go`**

Add after `resetPathCompletion`:

```go
func slugBranch(branch string) string {
	var b strings.Builder
	for _, r := range strings.ToLower(branch) {
		if (r >= 'a' && r <= 'z') || (r >= '0' && r <= '9') || r == '-' || r == '_' {
			b.WriteRune(r)
		} else {
			b.WriteRune('-')
		}
	}
	return b.String()
}

func DeriveWorktreePath(repo, branch string) string {
	return filepath.Join(filepath.Dir(repo), filepath.Base(repo)+"-"+slugBranch(branch))
}

func ResolveDefaultBranch(repo string) string {
	out, err := exec.Command("git", "-C", repo, "symbolic-ref", "--short", "refs/remotes/origin/HEAD").Output()
	if err == nil {
		s := strings.TrimSpace(string(out))
		if idx := strings.LastIndex(s, "/"); idx >= 0 {
			return s[idx+1:]
		}
		return s
	}
	out, err = exec.Command("git", "-C", repo, "symbolic-ref", "--short", "HEAD").Output()
	if err != nil {
		return ""
	}
	return strings.TrimSpace(string(out))
}

func runGitWorktreeAdd(repo, branch, dir, base string) error {
	cmd := exec.Command("git", "-C", repo, "worktree", "add", "-b", branch, dir, base)
	out, err := cmd.CombinedOutput()
	if err != nil {
		return fmt.Errorf("%s", strings.TrimSpace(string(out)))
	}
	return nil
}
```

- [ ] **Step 5: Run tests — they should pass**

```bash
go test ./internal/ui/ -run "TestDeriveWorktreePath|TestResolveDefaultBranch" -v
```

Expected: PASS.

- [ ] **Step 6: Run full suite**

```bash
go test ./internal/ui/ -v 2>&1 | grep -E "^(PASS|FAIL|---)"
```

Same three tests still failing (tab navigation), no new failures.

- [ ] **Step 7: Commit**

```bash
git add internal/ui/form.go internal/ui/form_test.go
git commit -m "feat(ui): add git worktree helpers (DeriveWorktreePath, ResolveDefaultBranch, runGitWorktreeAdd)"
```

---

### Task 3: WORKTREE auto-fill and formErr clearing in `updateForm`

**Files:**
- Modify: `internal/ui/form.go` — `updateForm`, add `updateWorktreeDefault`
- Test: `internal/ui/form_test.go`

- [ ] **Step 1: Write the failing tests**

Add to `internal/ui/form_test.go`:

```go
func TestWorktreeAutoFillsFromBranch(t *testing.T) {
	m, _ := openModel(t)
	m = sendKey(m, rune_('n'))
	// Down twice: TITLE → PATH → BRANCH
	m = sendKey(m, key(tea.KeyDown))
	m = sendKey(m, key(tea.KeyDown))
	// Type branch name
	for _, r := range "feat-x" {
		m = sendKey(m, rune_(r))
	}
	// Down twice: BRANCH → BASE → WORKTREE
	m = sendKey(m, key(tea.KeyDown))
	m = sendKey(m, key(tea.KeyDown))
	// WORKTREE field should be auto-filled (non-empty and contains "feat-x")
	view := m.View()
	if !strings.Contains(view, "feat-x") {
		t.Errorf("expected WORKTREE to auto-fill with branch slug, view:\n%s", view)
	}
}

func TestWorktreeUserEditPreventsAutoFill(t *testing.T) {
	m, _ := openModel(t)
	m = sendKey(m, rune_('n'))
	// Navigate to WORKTREE (index 4): Down 4 times
	for i := 0; i < 4; i++ {
		m = sendKey(m, key(tea.KeyDown))
	}
	// User manually types something in WORKTREE
	for _, r := range "my-custom-dir" {
		m = sendKey(m, rune_(r))
	}
	// Go back to BRANCH (index 2): Up 2 times
	m = sendKey(m, key(tea.KeyUp))
	m = sendKey(m, key(tea.KeyUp))
	// Type a branch name
	for _, r := range "new-branch" {
		m = sendKey(m, rune_(r))
	}
	// Go to WORKTREE again
	m = sendKey(m, key(tea.KeyDown))
	m = sendKey(m, key(tea.KeyDown))
	// WORKTREE should still be "my-custom-dir", not overwritten
	view := m.View()
	if !strings.Contains(view, "my-custom-dir") {
		t.Errorf("expected WORKTREE to preserve user edit, view:\n%s", view)
	}
	if strings.Contains(view, "new-branch") {
		t.Errorf("expected WORKTREE NOT to auto-fill after user edit, view:\n%s", view)
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/ui/ -run "TestWorktreeAutoFills|TestWorktreeUserEdit" -v
```

Expected: FAIL — WORKTREE never changes.

- [ ] **Step 3: Add `updateWorktreeDefault` to `form.go`**

Add after `resetPathCompletion`:

```go
func (m *Model) updateWorktreeDefault() {
	if m.form.worktreeUserEdited {
		return
	}
	branch := m.form.fields[2].value
	if branch == "" {
		m.form.fields[4].value = ""
		m.form.fields[4].cursor = 0
		return
	}
	path := m.form.fields[1].value
	derived := DeriveWorktreePath(path, branch)
	m.form.fields[4].value = derived
	m.form.fields[4].cursor = len([]rune(derived))
}
```

- [ ] **Step 4: Update `updateForm` to clear `formErr`, track `worktreeUserEdited`, and call `updateWorktreeDefault`**

At the very top of `updateForm`, before the switch, add:

```go
func (m *Model) updateForm(msg tea.KeyMsg) (tea.Model, tea.Cmd) {
	m.form.formErr = ""
	switch msg.Type {
```

In the `tea.KeyBackspace` case, add WORKTREE tracking and BRANCH auto-fill trigger:

```go
	case tea.KeyBackspace:
		f := &m.form.fields[m.form.focusField]
		if f.kind == fieldText {
			deleteRune(f)
			if m.form.focusField == 1 {
				m.resetPathCompletion()
			}
			if m.form.focusField == 4 {
				m.form.worktreeUserEdited = true
			}
			if m.form.focusField == 2 {
				m.updateWorktreeDefault()
			}
		}
		return m, nil
```

In the `default` case (rune insertion), add WORKTREE tracking and BRANCH auto-fill trigger:

```go
	default:
		if len(msg.Runes) == 1 {
			f := &m.form.fields[m.form.focusField]
			if f.kind == fieldText {
				insertRune(f, msg.Runes[0])
				if m.form.focusField == 1 {
					m.resetPathCompletion()
				}
				if m.form.focusField == 4 {
					m.form.worktreeUserEdited = true
				}
				if m.form.focusField == 2 {
					m.updateWorktreeDefault()
				}
			}
		}
		return m, nil
```

- [ ] **Step 5: Update the `tea.KeyEnter` handler to keep form open on `formErr`**

```go
	case tea.KeyEnter:
		m.commitForm()
		if m.form.formErr != "" {
			return m, nil
		}
		m.mode = ""
		if err := m.Reload(); err != nil {
			m.err = err
		}
		return m, nil
```

- [ ] **Step 6: Run tests — they should pass**

```bash
go test ./internal/ui/ -run "TestWorktreeAutoFills|TestWorktreeUserEdit" -v
```

Expected: PASS.

- [ ] **Step 7: Run full suite**

```bash
go test ./internal/ui/ -v 2>&1 | grep -E "^(PASS|FAIL|---)"
```

Same three tab-navigation tests still failing, no new failures.

- [ ] **Step 8: Commit**

```bash
git add internal/ui/form.go internal/ui/form_test.go
git commit -m "feat(ui): WORKTREE auto-fills from BRANCH; formErr cleared on each keystroke"
```

---

### Task 4: Update `renderForm` — dim BASE/WORKTREE when BRANCH empty, show `formErr`

**Files:**
- Modify: `internal/ui/form.go` — `renderForm`, add `formErrStyle`
- Test: `internal/ui/form_test.go`

- [ ] **Step 1: Write the failing test**

Add to `internal/ui/form_test.go`:

```go
func TestFormErrAppearsInView(t *testing.T) {
	m, _ := openModel(t)
	m = sendKey(m, rune_('n'))
	// Navigate to BRANCH, type a value so commitForm attempts worktree
	m = sendKey(m, key(tea.KeyDown)) // → PATH
	m = sendKey(m, key(tea.KeyDown)) // → BRANCH
	for _, r := range "any-branch" {
		m = sendKey(m, rune_(r))
	}
	// Submit — no valid git repo, commitForm will set formErr
	// (current dir may or may not be a repo; we need a known-bad PATH)
	// Navigate to PATH and set it to a nonexistent dir
	m = sendKey(m, key(tea.KeyUp))    // BRANCH → PATH
	for i := 0; i < 512; i++ {
		m = sendKey(m, key(tea.KeyBackspace))
	}
	for _, r := range "/nonexistent-repo-xyz" {
		m = sendKey(m, rune_(r))
	}
	m = sendKey(m, key(tea.KeyUp)) // PATH → TITLE
	for _, r := range "wttest" {
		m = sendKey(m, rune_(r))
	}
	m = sendKey(m, key(tea.KeyEnter)) // submit — should fail, form stays open
	if m.Mode() != "new-session" {
		t.Fatalf("expected form to stay open, got mode %q", m.Mode())
	}
	view := m.View()
	if m.FormErr() == "" {
		t.Fatal("expected FormErr to be set")
	}
	if !strings.Contains(view, m.FormErr()) {
		t.Errorf("expected formErr %q in view:\n%s", m.FormErr(), view)
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/ui/ -run TestFormErrAppearsInView -v
```

Expected: FAIL — form closes after Enter (formErr not checked yet in commitForm) or view doesn't contain error text.

- [ ] **Step 3: Add `formErrStyle` and update `renderForm`**

Add to the style vars block in `form.go`:

```go
formErrStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("#f38ba8"))
```

In `renderForm`, change the label rendering so BASE (index 3) and WORKTREE (index 4) use `formLabelDim` even when active if BRANCH is empty:

```go
	for i, f := range m.form.fields {
		isActive := i == m.form.focusField
		branchEmpty := m.form.fields[2].value == ""
		inertField := (i == 3 || i == 4) && branchEmpty

		var labelStr string
		if isActive && !inertField {
			labelStr = formBarStyle.Render("▌ ") + formLabelActive.Render(f.label)
		} else if isActive && inertField {
			labelStr = formBarStyle.Render("▌ ") + formLabelDim.Render(f.label)
		} else {
			labelStr = "  " + formLabelDim.Render(f.label)
		}
		label := labelStr + "  "
		// ... rest unchanged
```

Add `formErr` line above the hint:

```go
	if m.form.formErr != "" {
		sb.WriteString(formErrStyle.Render("  "+m.form.formErr) + "\n")
	}
	sb.WriteString(formHintStyle.Render("  Tab · Space · Enter to create · Esc cancel"))
```

- [ ] **Step 4: Run test — it should pass**

```bash
go test ./internal/ui/ -run TestFormErrAppearsInView -v
```

Expected: PASS.

- [ ] **Step 5: Run full suite**

```bash
go test ./internal/ui/ -v 2>&1 | grep -E "^(PASS|FAIL|---)"
```

Same three tab-navigation tests still failing, no new failures.

- [ ] **Step 6: Commit**

```bash
git add internal/ui/form.go internal/ui/form_test.go
git commit -m "feat(ui): dim BASE/WORKTREE when BRANCH empty; show formErr in red"
```

---

### Task 5: `commitForm` worktree logic

**Files:**
- Modify: `internal/ui/form.go` — `commitForm`
- Test: `internal/ui/form_test.go`

- [ ] **Step 1: Write the failing tests**

Add to `internal/ui/form_test.go` (add `"os/exec"` import if not already present):

```go
func TestCommitWithoutWorktree(t *testing.T) {
	m, conn := openModel(t)
	m = sendKey(m, rune_('n'))
	for _, r := range "plain-session" {
		m = sendKey(m, rune_(r))
	}
	// Leave BRANCH blank — submit from TITLE field
	m = sendKey(m, key(tea.KeyEnter))
	if m.Mode() != "" {
		t.Fatalf("expected mode cleared, got %q", m.Mode())
	}
	sessions, err := db.ListSessions(conn)
	if err != nil {
		t.Fatal(err)
	}
	if len(sessions) != 1 {
		t.Fatalf("expected 1 session, got %d", len(sessions))
	}
	// project_path should not be empty and no worktree was created
	if sessions[0].ProjectPath == "" {
		t.Error("expected non-empty project_path")
	}
}

func TestCommitWithWorktree(t *testing.T) {
	if _, err := exec.LookPath("git"); err != nil {
		t.Skip("git not available")
	}
	repo := t.TempDir()
	run := func(args ...string) {
		cmd := exec.Command("git", args...)
		cmd.Dir = repo
		if out, err := cmd.CombinedOutput(); err != nil {
			t.Fatalf("git %v: %s", args, out)
		}
	}
	run("init")
	run("config", "user.email", "test@test.com")
	run("config", "user.name", "Test")
	run("commit", "--allow-empty", "-m", "init")

	m, conn := openModel(t)
	m = sendKey(m, rune_('n'))
	for _, r := range "wt-session" {
		m = sendKey(m, rune_(r))
	}
	// Navigate to PATH and set it to repo
	m = sendKey(m, key(tea.KeyDown))
	for i := 0; i < 512; i++ {
		m = sendKey(m, key(tea.KeyBackspace))
	}
	for _, r := range repo {
		m = sendKey(m, rune_(r))
	}
	// Navigate to BRANCH
	m = sendKey(m, key(tea.KeyDown))
	for _, r := range "feature/x" {
		m = sendKey(m, rune_(r))
	}
	// Submit
	m = sendKey(m, key(tea.KeyEnter))
	if m.Mode() != "" {
		t.Fatalf("expected mode cleared after worktree commit, got %q (formErr: %q)", m.Mode(), m.FormErr())
	}
	sessions, err := db.ListSessions(conn)
	if err != nil {
		t.Fatal(err)
	}
	if len(sessions) != 1 {
		t.Fatalf("expected 1 session, got %d", len(sessions))
	}
	worktreeDir := sessions[0].ProjectPath
	if _, statErr := os.Stat(worktreeDir); statErr != nil {
		t.Fatalf("worktree dir %q should exist: %v", worktreeDir, statErr)
	}
	out, err := exec.Command("git", "-C", worktreeDir, "rev-parse", "--abbrev-ref", "HEAD").Output()
	if err != nil {
		t.Fatalf("git rev-parse: %v", err)
	}
	if branch := strings.TrimSpace(string(out)); branch != "feature/x" {
		t.Errorf("expected branch feature/x, got %q", branch)
	}
}

func TestCommitWorktreeErrorKeepsFormOpen(t *testing.T) {
	if _, err := exec.LookPath("git"); err != nil {
		t.Skip("git not available")
	}
	repo := t.TempDir()
	run := func(args ...string) {
		cmd := exec.Command("git", args...)
		cmd.Dir = repo
		if out, err := cmd.CombinedOutput(); err != nil {
			t.Fatalf("git %v: %s", args, out)
		}
	}
	run("init")
	run("config", "user.email", "test@test.com")
	run("config", "user.name", "Test")
	run("commit", "--allow-empty", "-m", "init")
	// Create the branch beforehand so `git worktree add -b <branch>` fails
	run("branch", "existing-branch")

	m, conn := openModel(t)
	m = sendKey(m, rune_('n'))
	for _, r := range "fail-session" {
		m = sendKey(m, rune_(r))
	}
	m = sendKey(m, key(tea.KeyDown)) // → PATH
	for i := 0; i < 512; i++ {
		m = sendKey(m, key(tea.KeyBackspace))
	}
	for _, r := range repo {
		m = sendKey(m, rune_(r))
	}
	m = sendKey(m, key(tea.KeyDown)) // → BRANCH
	for _, r := range "existing-branch" {
		m = sendKey(m, rune_(r))
	}
	m = sendKey(m, key(tea.KeyEnter))

	if m.Mode() != "new-session" {
		t.Fatalf("expected form to stay open, got mode %q", m.Mode())
	}
	if m.FormErr() == "" {
		t.Error("expected FormErr to be set after git failure")
	}
	sessions, err := db.ListSessions(conn)
	if err != nil {
		t.Fatal(err)
	}
	if len(sessions) != 0 {
		t.Errorf("expected 0 sessions, got %d", len(sessions))
	}
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/ui/ -run "TestCommitWithoutWorktree|TestCommitWithWorktree|TestCommitWorktreeError" -v
```

Expected: `TestCommitWithWorktree` and `TestCommitWorktreeErrorKeepsFormOpen` fail — worktree branch not evaluated.

- [ ] **Step 3: Implement worktree logic in `commitForm`**

Replace `commitForm` entirely:

```go
func (m *Model) commitForm() {
	m.form.formErr = ""
	title := strings.TrimSpace(m.form.fields[0].value)
	if title == "" {
		return
	}
	path := strings.TrimSpace(m.form.fields[1].value)
	if path == "" {
		path = "."
	}
	branch := strings.TrimSpace(m.form.fields[2].value)
	tool := m.form.fields[5].options[m.form.fields[5].optIdx]
	flags := strings.TrimSpace(m.form.fields[6].value)
	script := strings.TrimSpace(m.form.fields[7].value)

	var projectPath string
	if branch == "" {
		projectPath = expandPath(path)
	} else {
		base := strings.TrimSpace(m.form.fields[3].value)
		if base == "" {
			base = ResolveDefaultBranch(path)
		}
		if base == "" {
			m.form.formErr = "could not resolve base branch; set BASE manually"
			return
		}
		worktree := strings.TrimSpace(m.form.fields[4].value)
		if worktree == "" {
			worktree = DeriveWorktreePath(path, branch)
		}
		worktree = expandPath(worktree)
		if err := runGitWorktreeAdd(path, branch, worktree, base); err != nil {
			m.form.formErr = err.Error()
			return
		}
		projectPath = worktree
	}

	if err := db.CreateSession(m.conn, db.Session{
		ID:            uuid.New().String(),
		Title:         title,
		GroupPath:     m.currentGroupPath(),
		ProjectPath:   projectPath,
		Tool:          tool,
		ToolFlags:     flags,
		Status:        "stopped",
		CreatedAt:     time.Now().Unix(),
		StartupScript: script,
	}); err != nil {
		m.err = err
	}
}
```

- [ ] **Step 4: Add `"os"` and `"strings"` imports to form_test.go if not already present**

Ensure `internal/ui/form_test.go` imports:

```go
import (
	"database/sql"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"testing"

	"github.com/black-gato/tmux-agent-deck/internal/db"
	"github.com/black-gato/tmux-agent-deck/internal/testutil"
	"github.com/black-gato/tmux-agent-deck/internal/ui"
	tea "github.com/charmbracelet/bubbletea"
)
```

- [ ] **Step 5: Run tests — they should pass**

```bash
go test ./internal/ui/ -run "TestCommitWithoutWorktree|TestCommitWithWorktree|TestCommitWorktreeError" -v
```

Expected: PASS (or SKIP if git not available for the two git tests).

- [ ] **Step 6: Run full suite**

```bash
go test ./internal/ui/ -v 2>&1 | grep -E "^(PASS|FAIL|---)"
```

Same three tab-navigation tests still failing, no new failures.

- [ ] **Step 7: Commit**

```bash
git add internal/ui/form.go internal/ui/form_test.go
git commit -m "feat(ui): commitForm runs git worktree add when BRANCH set; formErr on failure"
```

---

### Task 6: Fix three broken existing tests

**Files:**
- Modify: `internal/ui/form_test.go` — `TestToolSelectorCycles`
- Modify: `internal/ui/app_test.go` — `TestNewSessionFlowCreatesSessionWithTool`, `TestNewSessionDialogPersistsToolFlags`

Each test navigates from PATH directly to TOOL using one Tab (old behavior). With the new 8-field form, Tab from PATH advances to BRANCH (index 2), so three additional Tabs are needed to reach TOOL (index 5): BRANCH→BASE→WORKTREE→TOOL.

- [ ] **Step 1: Run the three failing tests to confirm the failure reason**

```bash
go test ./internal/ui/ -run "TestToolSelectorCycles|TestNewSessionFlowCreatesSessionWithTool|TestNewSessionDialogPersistsToolFlags" -v
```

Expected: FAIL with wrong tool selected (tool defaulted to "claude" because Right arrow was applied to BRANCH field, not TOOL).

- [ ] **Step 2: Fix `TestToolSelectorCycles` in `form_test.go`**

Find the Tab-from-PATH section and add 3 more Tabs:

```go
	m = sendKey(m, key(tea.KeyTab)) // PATH (no candidates) → BRANCH

	// Navigate past BRANCH, BASE, WORKTREE to reach TOOL
	m = sendKey(m, key(tea.KeyTab)) // BRANCH → BASE
	m = sendKey(m, key(tea.KeyTab)) // BASE → WORKTREE
	m = sendKey(m, key(tea.KeyTab)) // WORKTREE → TOOL

	// Right arrow on TOOL cycles forward: claude → claude-dangerous
	m = sendKey(m, key(tea.KeyRight))
	m = sendKey(m, key(tea.KeyEnter))
```

- [ ] **Step 3: Fix `TestNewSessionFlowCreatesSessionWithTool` in `app_test.go`**

Find line 273 and replace the single `Tab` comment:

```go
	m.Update(tea.KeyMsg{Type: tea.KeyTab}) // PATH (no candidates) → BRANCH
	m.Update(tea.KeyMsg{Type: tea.KeyTab}) // BRANCH → BASE
	m.Update(tea.KeyMsg{Type: tea.KeyTab}) // BASE → WORKTREE
	m.Update(tea.KeyMsg{Type: tea.KeyTab}) // WORKTREE → TOOL
```

- [ ] **Step 4: Fix `TestNewSessionDialogPersistsToolFlags` in `app_test.go`**

Find line 361 and replace the two `Tab` lines:

```go
	m.Update(tea.KeyMsg{Type: tea.KeyTab}) // PATH (no candidates) → BRANCH
	m.Update(tea.KeyMsg{Type: tea.KeyTab}) // BRANCH → BASE
	m.Update(tea.KeyMsg{Type: tea.KeyTab}) // BASE → WORKTREE
	m.Update(tea.KeyMsg{Type: tea.KeyTab}) // WORKTREE → TOOL
	m.Update(tea.KeyMsg{Type: tea.KeyTab}) // TOOL → FLAGS
```

- [ ] **Step 5: Run the three fixed tests**

```bash
go test ./internal/ui/ -run "TestToolSelectorCycles|TestNewSessionFlowCreatesSessionWithTool|TestNewSessionDialogPersistsToolFlags" -v
```

Expected: PASS.

- [ ] **Step 6: Run full suite — all tests must pass**

```bash
go test ./... 2>&1 | tail -20
go vet ./...
```

Expected: all PASS, no vet warnings.

- [ ] **Step 7: Commit**

```bash
git add internal/ui/form_test.go internal/ui/app_test.go
git commit -m "fix(ui): update tab-navigation tests for 8-field session form"
```
