# Escalation Status Accuracy

**Date:** 2026-05-25
**Status:** Implemented 2026-05-25 (commit `200ada3`, PR #4)

## Problem

The auto-escalation message includes a `Status:` field describing the worker's state. It fires on the `running → waiting` transition (`PollOnce` in `internal/state/poller.go`), but the message was built from `s.Status` — the session row read at the **top** of `PollOnce`, before the status was updated. So the field reported the pre-transition value (`running`) even though the escalation fired *because* the worker had just gone `waiting`.

Observed live: every `Escalation from poller | … | Status: running` line in the conductor, despite each one being triggered by a transition into `waiting`. Misleading to the conductor, which uses status to judge whether the worker is blocked.

## Solution

Inside the transition block — after the `s.Status != waiting && newStatus == waiting` guard and after `UpdateSessionStatus` writes the new status to the DB — reconcile the in-memory session before notifying/escalating:

```go
s.Status = newStatus
```

Because this branch only runs on the transition into `waiting`, `newStatus` is always `waiting` here, so the escalation reports `Status: waiting`. The assignment sits *after* the guard, so status-detection and the once-per-transition logic are unaffected. `notifyWaiting(s)` does not read `Status`, so it is unaffected; the reconciliation also keeps the in-memory row consistent with the DB for any future reader.

## Behavioral Summary

| Event | `Status:` field before | after |
|-------|------------------------|-------|
| Worker goes `running → waiting`, escalation fires | `running` (stale) | `waiting` |

## Affected files

- `internal/state/poller.go` — `s.Status = newStatus` in the waiting-transition block
- `internal/state/poller_test.go` — `TestAutoEscalateReportsWaitingStatus`

## Verification

TDD (failing test first, observed `Status: running`, then `Status: waiting`). After the live deck was rebuilt and restarted, new escalations were confirmed to report `Status: waiting`.
