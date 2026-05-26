# Escalation Status Accuracy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Status: Complete** — all tasks implemented and verified (commit `200ada3`, PR #4).

**Goal:** Make the auto-escalation `Status:` field report the post-transition status (`waiting`) instead of the stale pre-transition value (`running`).

**Architecture:** In `PollOnce`, after the `running → waiting` guard and the `UpdateSessionStatus` call, set `s.Status = newStatus` before notifying/escalating so `EscalationMessage` reads the current status. The assignment is scoped inside the transition branch, leaving detection and once-per-transition behavior untouched.

**Tech Stack:** Go 1.22, existing `internal/state` package.

---

## File Map

| File | Change |
|------|--------|
| `internal/state/poller.go` | `s.Status = newStatus` inside the waiting-transition block |
| `internal/state/poller_test.go` | `TestAutoEscalateReportsWaitingStatus` |

---

### Task 1: Failing test

- [x] Add `TestAutoEscalateReportsWaitingStatus`: worker `running` in DB, pane at a prompt, single `PollOnce`.
- [x] Assert the sent escalation contains `Status: waiting` and not `Status: running`.
- [x] Verify it fails against the current code (observed `Status: running`).

### Task 2: Reconcile status before escalating

- [x] Inside the `s.Status != waiting && newStatus == waiting` block, after `UpdateSessionStatus`, add `s.Status = newStatus`.
- [x] Confirm placement is after the guard so detection/once-per-transition logic is unchanged.

### Task 3: Verify

- [x] `go test ./internal/state/` green (`TestAutoEscalate*`, `TestEscalationMessage*`, `TestPoller*`); `go vet ./...` clean.
- [x] After live deck rebuild + restart, confirmed new escalations report `Status: waiting`.
