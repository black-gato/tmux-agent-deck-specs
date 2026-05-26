# Escalation Context Fidelity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Status: Complete** — all tasks implemented and verified (commit `200ada3`, PR #4).

**Goal:** Make the escalation `Context:` field carry the worker's entire last Claude message instead of a truncated, chrome-polluted 5-line tail.

**Architecture:** Add `lastClaudeBlock` to `internal/state/escalate.go` to extract the final `⏺` (U+23FA) assistant block plus its indented continuations, stopping at chrome. `EscalationMessage` uses it when present and falls back to the existing `contextLines` scan for non-Claude workers, whose `isContextLine` filter is corrected for the NBSP `❯` prompt and the `●` (U+25CF) survey bullet.

**Tech Stack:** Go 1.22, existing `internal/state` package.

---

## File Map

| File | Change |
|------|--------|
| `internal/state/escalate.go` | New `lastClaudeBlock`; `EscalationMessage` prefers it over `contextLines`; `isContextLine` `❯`/`●` fixes |
| `internal/state/escalate_test.go` | `TestEscalationMessageCapturesFullLastBlock` |

---

### Task 1: Failing test for full-block capture

- [x] Add `TestEscalationMessageCapturesFullLastBlock` with a realistic multi-line `⏺` answer followed by the `✻` timing line, the `●` survey, separators, and a `❯`+U+00A0 unsent draft.
- [x] Assert the message includes the block's first and last lines and excludes the survey, rating menu, unsent prompt, timing, and status lines.
- [x] Verify it fails against the old 5-line `contextLines` implementation.

### Task 2: Implement `lastClaudeBlock`

- [x] Scan bottom-up for the last line prefixed with `⏺` (U+23FA).
- [x] Collect that line plus following blank/indented lines; break at the first non-blank, unindented line.
- [x] Trim, strip the leading `⏺`, and drop blank and separator-only (`─━═- `) lines.
- [x] Wire `EscalationMessage` to use the block when non-empty, else fall back to `contextLines`.

### Task 3: Fix the fallback filter

- [x] Change the `isContextLine` prompt check from `"❯ "` to a `❯` prefix so the NBSP-rendered prompt is dropped.
- [x] Drop lines beginning with the survey bullet `●` (U+25CF).

### Task 4: Verify

- [x] `go test ./internal/state/` green; `go vet ./...` clean.
- [x] Confirmed against a live Claude pane and end-to-end through the running deck (full block captured; leftover `❯` draft excluded).
