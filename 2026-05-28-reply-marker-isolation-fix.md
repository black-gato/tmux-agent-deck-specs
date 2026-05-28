# Reply Marker Isolation Fix

**Date:** 2026-05-28
**Status:** Implemented
**Related:** [Conductor Enhancements Plan](2026-05-18-conductor-enhancements-plan.md), BUG-022

## Problem

The conductor reply protocol (`@deck-reply worker=<id> … @deck-end`) was vulnerable to a feedback bug: the auto-escalation message itself contained the literal markers as part of its inline "Reply with:" instructions. When that message was sent into the conductor's pane, it sat in the scrollback as parseable text. On the next poll cycle, `ParseReplyBlocks` matched the echoed markers as a *real* reply block with body `...` and routed `...` back to the worker as if it were the conductor's answer.

Real conductor pane snippet that triggered the bug:

```
⏺ @deck-reply worker=6d2199…
  You're doing great…
  @deck-end

❯ Escalation from timer-debug | Worker ID: 6d2199… | Status: waiting |
  Reply with: @deck-reply worker=6d2199… ... @deck-end | Context: …
```

Both lines parsed. The first delivered the genuine reply. The second delivered the literal three dots.

### Why the existing safeguards did not catch it

- **Feedback-loop guard** rejected bodies containing `@deck-reply` — body here was `...`, not a marker.
- **First-scan seeding** only protects against historical blocks present at first sight of a conductor. The escalation arrives *after* seeding, so its echo looks new.
- **Fingerprint dedup** (`workerID:body`) does not help when the phantom block has a distinct body (`...`) from the real reply.
- **`LastIndex(@deck-end)`** — intentional to let bodies legitimately mention the marker mid-sentence — happily matched the only `@deck-end` in the echoed line.

## Fix

Two defensive changes:

### 1. Remove literal markers from the escalation message

`internal/state/escalate.go:19` previously emitted:

```text
Reply with: @deck-reply worker=<id> ... @deck-end
```

It now emits:

```text
Reply to worker <id> using your conductor role reply protocol.
```

The protocol itself is taught via the managed `## Conductor Role` block written by `--init-conductor-docs` (see `2026-05-18-conductor-enhancements-plan.md`, Feature 3). Conductors learn the syntax from their `CLAUDE.md`, not from a literal instance embedded in every escalation.

### 2. Require markers at the logical start of a line

`internal/state/reply.go` `ParseReplyBlocks` previously used `strings.Index` to find `@deck-reply worker=` *anywhere* on a line, which was tolerant of prompt prefixes like `❯ @deck-reply …` but also matched mid-sentence text like `Reply with: @deck-reply …`.

The new `trimPromptPrefix` helper strips leading whitespace and a fixed set of prompt-prefix glyphs (`❯`, `>`, `⏺`) and then requires the remainder to start with `@deck-reply worker=`. Mid-line occurrences no longer start a block.

```go
func trimPromptPrefix(line string) string {
    for {
        trimmed := strings.TrimLeftFunc(line, unicode.IsSpace)
        switch {
        case strings.HasPrefix(trimmed, "❯"):
            line = strings.TrimPrefix(trimmed, "❯")
        case strings.HasPrefix(trimmed, ">"):
            line = strings.TrimPrefix(trimmed, ">")
        case strings.HasPrefix(trimmed, "⏺"):
            line = strings.TrimPrefix(trimmed, "⏺")
        default:
            return trimmed
        }
    }
}
```

### Defense in depth, not either-or

Either change alone leaves a gap:

- **Tightening alone** does not stop the bug if tmux line-wrapping pushes the marker to column 0 of a continuation line. The marker would then sit at logical start and parse as before.
- **Template change alone** depends on the marker never landing back in the pane through some other path (e.g. a conductor pasting a reply that quotes the escalation). Search-anywhere matching would still bite us.

Both changes together eliminate the class of bug: the message stops shipping the markers in the first place, and the parser stops matching incidental occurrences if they ever appear.

## Tests

Added in `internal/state/reply_test.go`:

- `TestParseReplyBlocksIgnoresMidLineMarker` — `"Reply with: @deck-reply worker=abc ... @deck-end"` → zero blocks.
- `TestParseReplyBlocksIgnoresEchoedEscalationLine` — full single-line escalation echo from the real pane snippet → zero blocks.
- `TestParseReplyBlocksClaudeBulletPrefixStillParses` — `⏺ @deck-reply worker=abc\n  hello world\n  @deck-end` still parses to a single block (preserves the existing path through prompt-prefix stripping).

Added in `internal/state/escalate_test.go`:

- `TestEscalationMessageOmitsLiteralReplyMarkers` — the message contains neither `@deck-reply` nor `@deck-end` as literal substrings.

Replaced: the old `TestEscalationMessageIncludesReplySyntax` (which asserted the bug-causing behavior).

All pre-existing reply, escalation, routing, and heartbeat tests continue to pass.

## Tradeoff

Conductors whose project has no managed `## Conductor Role` block in `CLAUDE.md` (i.e. `--init-conductor-docs` was never used) will receive escalations that reference "your conductor role reply protocol" without explaining what that protocol is. This is acceptable for v1: the recommended setup runs `--init-conductor-docs` at conductor assignment time, and the canonical reply syntax lives in one place rather than being duplicated into every escalation message.

## Files

- `internal/state/escalate.go` — drop literal reply markers from the message.
- `internal/state/reply.go` — `trimPromptPrefix` + start-of-line requirement.
- `internal/state/reply_test.go` — three new tests.
- `internal/state/escalate_test.go` — new omits-markers test; replaces includes-syntax test.
- `docs/bugs.md` — BUG-022 entry.
