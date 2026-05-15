# Return to TUI Keybinding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When a user attaches to an agent session, automatically bind `ctrl + q` (`C-q` in the tmux `root` table) to `detach-client` so they can return to the TUI with a single keystroke, then restore the original binding on return.

**Architecture:** All changes are in `AttachSession()` in `internal/tmux/client.go`. Before attaching, save the current `prefix + q` binding, set it to `detach-client`, block on attach, then restore in a deferred call. The only testable pure logic is the `list-keys` output parser, which gets its own test.

**Tech Stack:** Go stdlib (`os/exec`), tmux CLI (`list-keys`, `bind-key`, `unbind-key`)

---

### Task 1: Add parseBindingCommand + test

**Files:**
- Modify: `internal/tmux/client.go`
- Test: `internal/tmux/status_test.go` (same package, append)

- [ ] **Step 1: Write the failing tests**

Append to `internal/tmux/status_test.go`:

```go
func TestParseBindingCommand(t *testing.T) {
	tests := []struct {
		name   string
		input  string
		want   string
	}{
		{
			name:  "simple command",
			input: "bind-key -T prefix q display-panes",
			want:  "display-panes",
		},
		{
			name:  "multi-word command",
			input: "bind-key -T prefix q run-shell 'echo hi'",
			want:  "run-shell 'echo hi'",
		},
		{
			name:  "repeatable flag",
			input: "bind-key -rT prefix q resize-pane -D 5",
			want:  "resize-pane -D 5",
		},
		{
			name:  "empty output means no binding",
			input: "",
			want:  "",
		},
		{
			name:  "whitespace only",
			input: "   ",
			want:  "",
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := tmux.ParseBindingCommand(tt.input)
			if got != tt.want {
				t.Errorf("ParseBindingCommand(%q) = %q, want %q", tt.input, got, tt.want)
			}
		})
	}
}
```

- [ ] **Step 2: Run the tests to confirm they fail**

```bash
go test ./internal/tmux/... -run TestParseBindingCommand -v
```

Expected: `FAIL — tmux.ParseBindingCommand undefined`

- [ ] **Step 3: Add ParseBindingCommand to client.go**

Add this function to `internal/tmux/client.go` (before `runCmd`):

```go
// ParseBindingCommand extracts the command from a tmux list-keys output line.
// Input format: "bind-key [-r] -T prefix q <command>"
// Returns "" if the input is empty or unparseable.
func ParseBindingCommand(listKeysOutput string) string {
	line := strings.TrimSpace(listKeysOutput)
	if line == "" {
		return ""
	}
	// Find " q " and take everything after it — the key is always "q" here.
	idx := strings.Index(line, " q ")
	if idx == -1 {
		return ""
	}
	return strings.TrimSpace(line[idx+3:])
}
```

- [ ] **Step 4: Run the tests to confirm they pass**

```bash
go test ./internal/tmux/... -run TestParseBindingCommand -v
```

Expected: all 5 sub-tests PASS

- [ ] **Step 5: Commit**

```bash
git add internal/tmux/client.go internal/tmux/status_test.go
git commit -m "feat: add ParseBindingCommand for tmux list-keys output"
```

---

### Task 2: Binding helpers + AttachSession update

**Files:**
- Modify: `internal/tmux/client.go`

- [ ] **Step 1: Add the three binding helper functions**

Add these three functions to `internal/tmux/client.go` (after `ParseBindingCommand`, before `runCmd`):

```go
// saveBinding returns the current command bound to prefix+key, or "" if none.
func saveBinding(key string) string {
	out, err := cmdOutput("tmux", "list-keys", "-T", "prefix", key)
	if err != nil {
		return ""
	}
	return ParseBindingCommand(string(out))
}

// setBinding sets prefix+key to the given command.
func setBinding(key, command string) {
	if err := runCmd("tmux", "bind-key", "-T", "prefix", key, command); err != nil {
		fmt.Fprintf(os.Stderr, "tmux-agent-deck: bind-key %s: %v\n", key, err)
	}
}

// restoreBinding restores prefix+key to savedCmd, or unbinds it if savedCmd is "".
func restoreBinding(key, savedCmd string) {
	var err error
	if savedCmd == "" {
		err = runCmd("tmux", "unbind-key", "-T", "prefix", key)
	} else {
		err = runCmd("tmux", "bind-key", "-T", "prefix", key, savedCmd)
	}
	if err != nil {
		fmt.Fprintf(os.Stderr, "tmux-agent-deck: restore binding %s: %v\n", key, err)
	}
}
```

- [ ] **Step 2: Update AttachSession to save/restore the binding**

Replace the current `AttachSession` method:

```go
func (c *Client) AttachSession(name string) error {
	if os.Getenv("TMUX") != "" {
		return runInteractive("tmux", "switch-client", "-t", name)
	}
	saved := saveBinding("q")
	setBinding("q", "detach-client")
	defer restoreBinding("q", saved)
	return runInteractive("tmux", "attach-session", "-t", name)
}
```

- [ ] **Step 3: Verify the package still compiles**

```bash
go build ./internal/tmux/...
```

Expected: no output, exit 0

- [ ] **Step 4: Run the full test suite**

```bash
go test ./...
```

Expected: all tests pass (the new helpers aren't unit-tested since they wrap tmux exec calls — their correctness is covered by the ParseBindingCommand tests and manual smoke test)

- [ ] **Step 5: Manual smoke test**

Run outside tmux:

```bash
go run . 
```

- Select a session and press `Enter` to attach
- Verify `prefix + q` detaches you back to the TUI
- Verify the previous `q` binding is restored (run `tmux list-keys -T prefix q` in a separate pane before attaching to record it, then check after returning)

- [ ] **Step 6: Commit**

```bash
git add internal/tmux/client.go
git commit -m "feat: bind prefix+q to detach-client on session attach"
```
