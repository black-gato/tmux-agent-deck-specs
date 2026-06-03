# PermissionRequest → waiting Status Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Map the `PermissionRequest` Claude Code hook event to `waiting` status so a session blocked on a tool permission prompt is surfaced immediately via the hook path rather than waiting for pane scraping.

**Architecture:** `EventToStatus` in `internal/hook/hook.go` is the single mapping function. Adding `PermissionRequest` there is the entire change — the hook handler, poller, and DB update path already handle it correctly. The existing `TestEventToStatus` table-driven test needs one new row.

**Tech Stack:** Go 1.22+, `internal/hook` package

---

### Task 1: Map PermissionRequest to waiting

**Files:**
- Modify: `internal/hook/hook.go` (the `EventToStatus` switch)
- Modify: `internal/hook/hook_test.go` (the `TestEventToStatus` cases map)

- [ ] **Step 1: Write the failing test**

Add `"PermissionRequest": "waiting"` to the cases map in `TestEventToStatus` in `internal/hook/hook_test.go`:

```go
func TestEventToStatus(t *testing.T) {
	cases := map[string]string{
		"SessionStart":      "waiting",
		"UserPromptSubmit":  "running",
		"Stop":              "waiting",
		"PermissionRequest": "waiting",
		"SessionEnd":        "dead",
		"PreCompact":        "",
		"Unknown":           "",
	}
	for event, want := range cases {
		if got := hook.EventToStatus(event); got != want {
			t.Errorf("EventToStatus(%q) = %q, want %q", event, got, want)
		}
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
go test ./internal/hook/ -run TestEventToStatus -v
```

Expected: FAIL — `EventToStatus("PermissionRequest") = "", want "waiting"`

- [ ] **Step 3: Add PermissionRequest to the switch**

In `internal/hook/hook.go`, update `EventToStatus`:

```go
func EventToStatus(event string) string {
	switch event {
	case "SessionStart", "Stop", "PermissionRequest":
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

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/hook/ -v
```

Expected: all PASS

- [ ] **Step 5: Run full test suite**

```bash
go test ./...
```

Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add internal/hook/hook.go internal/hook/hook_test.go
git commit -m "feat(hook): map PermissionRequest event to waiting status"
```

- [ ] **Step 7: Sync plan to spec repo**

```bash
cp docs/superpowers/plans/2026-05-27-permission-request-waiting-status.md ../tmux-agent-deck-specs/
```
