# Escalation Context Fidelity

**Date:** 2026-05-25
**Status:** Implemented 2026-05-25 (commit `200ada3`, PR #4)

## Problem

When a worker session transitions to `waiting`, the poller escalates to the group conductor with a `Context:` field built from the worker's captured pane (`EscalationMessage` → `contextLines` in `internal/state/escalate.go`). The original implementation collected the **last 5 non-chrome lines** bottom-up. On real Claude Code panes this failed in three ways, so the conductor received a garbled tail instead of the worker's actual last message:

1. **Truncation.** Claude answers routinely span 10–40 lines. A 5-line window dropped most of the message and, because it had no notion of where the last message began, bled backward across earlier UI.

2. **NBSP prompt leak.** The chrome filter dropped lines starting with `"❯ "` (`❯` + ASCII space, bytes `e2 9d af 20`). Claude Code renders the input prompt as `❯` + **U+00A0 non-breaking space** (`e2 9d af c2 a0`), so the filter never matched — and the user's typed-but-unsent next prompt leaked into the context as if it were Claude's reply.

3. **Survey leak.** Claude Code's feedback survey (`● How is Claude doing this session?` and `1: Bad 2: Fine 3: Good 0: Dismiss`) passed the filter and polluted the context.

Verified live: an escalation's `Context:` came through as the last two answer lines plus the survey plus the unsent prompt, with the bulk of the answer missing.

## Solution

### Last assistant block extraction

Replace the line-window model with `lastClaudeBlock(output string) []string`. Claude marks every assistant block — prose responses and tool calls alike — with the bullet `⏺` (U+23FA). The function:

1. Scans bottom-up for the last line beginning with `⏺`.
2. Collects that line plus following lines that are blank or indented (continuation), stopping at the first non-blank, unindented line (chrome: the `✻` timing line, a separator, the input box, the status line, or the next bullet).
3. Trims each line, drops blank and separator-only (`─━═- `) lines, and strips the leading `⏺`.

This captures the **entire** final answer regardless of length and structurally excludes everything below it.

### Glyph distinction

The response/tool bullet `⏺` (U+23FA) is a different code point from the feedback-survey bullet `●` (U+25CF). Because block collection stops before reaching the survey, the survey is excluded by construction — no string-matching heuristic needed.

### Fallback filter corrections

For non-Claude workers (shell, aider) there is no `⏺` block, so `EscalationMessage` falls back to the line scan. Two `isContextLine` fixes were applied there too:

- Match `❯` as a prefix regardless of the following byte, so the NBSP-rendered prompt is dropped.
- Drop lines beginning with the survey bullet `●` (U+25CF).

## Behavioral Summary

| Worker pane | Old context | New context |
|-------------|-------------|-------------|
| Multi-line Claude answer | last ~2 answer lines + survey + unsent `❯` draft | the full last `⏺` block, nothing else |
| Tool-call as last block | partial tail | the `⏺` tool line + indented `⎿` result |
| Shell/aider worker | line scan (NBSP/survey could leak) | line scan with `❯`/`●` correctly filtered |

## Affected files

- `internal/state/escalate.go` — `lastClaudeBlock`, `EscalationMessage` wiring, `isContextLine` fixes
- `internal/state/escalate_test.go` — `TestEscalationMessageCapturesFullLastBlock`

## Verification

TDD (failing test first); confirmed against a live Claude pane and end-to-end through the running deck — the conductor received the full last block, and a leftover `❯` draft in the worker's input box was excluded.
