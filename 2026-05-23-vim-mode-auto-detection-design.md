# Vim Mode Auto-Detection for send-pane / broadcast

**Date:** 2026-05-23
**Status:** Implemented

## Overview

When sending text to a Claude Code session via send-pane (`x`) or broadcast (`b`), the session may be in readline vi INSERT mode or COMMAND (NORMAL) mode. Sending literal text to a session in COMMAND mode causes the characters to be interpreted as vi motions rather than typed input, so the message is never received.

The original `Ctrl+V` toggle (`dialog.vimMode`) was a single global switch that applied to every target session. In broadcast, each session can independently be in INSERT or COMMAND mode, making a global toggle unreliable.

This spec covers the auto-detection approach that replaced the toggle as the primary mechanism for Claude-family sessions.

## Detection Signal

Claude Code's readline vi mode emits `-- INSERT --` in the visible pane area when the session is in INSERT mode. COMMAND mode shows nothing. This is the only reliable signal available from pane content alone — pane content cannot distinguish "vim COMMAND mode" from "no vim at all."

Two signals are combined to make a routing decision:

1. **Session tool** — `session.Tool == "claude"` or `"claude-dangerous"` means vim mode is in use.
2. **`-- INSERT --` in current pane view** — presence means INSERT mode; absence means COMMAND mode (or session is processing output, not at the readline prompt).

## Decision Table

| Tool is claude? | `-- INSERT --` visible? | Action |
|---|---|---|
| Yes | Yes | Send text directly — session already in INSERT mode |
| Yes | No | Send `i` prefix to enter INSERT from COMMAND mode, then text |
| Yes | Capture failed | Fall back to `dialog.vimMode` (false by default — assume INSERT) |
| No | — | Fall back to `dialog.vimMode` (manual `Ctrl+V` toggle) |

The manual `Ctrl+V` toggle is retained as the mechanism for non-claude vim sessions (e.g. nvim, aider with vim bindings).

## Why `i` Only — Not `Escape+i`

The initial design proposed `Escape+i` as the prefix: Escape normalises any mode to COMMAND, and `i` re-enters INSERT. This was subsequently found to be unreliable.

Each key is sent as a separate `exec.Command` call:

```
SendRawKeys(session, pane, "Escape")  // tmux send-keys -t session:0.0 Escape
SendKeys(session, pane, "i")          // tmux send-keys -l -t session:0.0 i
```

These two processes launch and complete sequentially. The time between the Escape byte (0x1B) arriving at the pty and the `i` byte (0x69) arriving is a few milliseconds — well within readline's ~100ms escape sequence timeout. Readline vi mode interprets 0x1B followed by 0x69 within the timeout as `Meta-i`, not as standalone `Escape` then `i`. `Meta-i` is either unbound or has an unexpected binding in COMMAND mode, leaving the session in COMMAND mode. The subsequent text then executes as vi motion keys.

The fix: send `i` alone (no Escape). In COMMAND mode, `i` unambiguously enters INSERT mode. There is no multi-byte sequence to misparse. The only edge case is if the session is actually in INSERT mode but was misdetected as COMMAND mode (e.g. `-- INSERT --` scrolled off the visible area) — in that case `i` types a stray `i` character at the head of the input. This is a minor cosmetic issue compared to total message loss.

## CapturePaneView

The detection reads the **current visible pane content only**, not scrollback history:

```
tmux capture-pane -t <session>:0.<pane> -p
```

`-p` without `-S -` captures only what is currently visible on screen. This is fast (no scrollback traversal) and sufficient — the `-- INSERT --` indicator appears in the visible area when at the readline prompt in INSERT mode.

The method is added to `ClientIface` as `CapturePaneView(session string, paneIndex int) (string, error)`, keeping it distinct from the existing `CapturePaneOutput` (which uses `-S -` for full scrollback, used by the status poller).

## Shared Logic Between UI and Poller

Both the UI dialogs (`internal/ui/dialog.go`) and the background poller (`internal/state/poller.go`) send text to sessions. Both use the same detection approach:

- **UI `sendToPane`**: called by send-pane (`x`) and broadcast (`b`). Sends text from user's dialog input, followed by Enter to submit.
- **Poller `sendToSession`**: called by `autoEscalate`, `scanConductorReplies`, and `runHeartbeats`. Sends programmatic messages to conductor or worker sessions.

Both check `isInVimInsertMode(paneContent)` and use `i`-only prefix when entering INSERT from COMMAND mode. Both send Enter as the final raw key.

The poller uses the `TmuxReader` interface (a subset of `ClientIface`) which includes `CapturePaneView`.

## Enter Submission

Before this work, `sendToPane` sent text and ctrl keys but no Enter. Text was typed into the readline buffer but the message was never submitted — it piled up waiting for Enter. Enter is now sent as the final `SendRawKeys` call on every `sendToPane` invocation. The dialog's own Enter key (which triggered `commitDialog`) is not forwarded to the target session; this final `SendRawKeys(Enter)` is separate.

## File Layout

```
internal/tmux/client.go         CapturePaneView on ClientIface and *Client
internal/testutil/tmux.go       PaneViews map + CapturePaneView on FakeTmuxClient
internal/ui/dialog.go           isClaudeTool(), isInVimInsertMode(), sendToPane()
internal/state/poller.go        isClaudeSession(), isInVimInsertMode(), sendToSession()
                                TmuxReader interface includes CapturePaneView
internal/ui/app_test.go         regression tests (see below)
internal/state/poller_test.go   regression tests (see below)
```

## Test Coverage

**UI (`internal/ui/app_test.go`)**
- `TestSendPaneClaudeInInsertModeSkipsVimPrefix` — INSERT detected: no `i` prefix, only Enter raw key
- `TestSendPaneClaudeInNormalModeAddsVimPrefix` — no INSERT: `i` via SendKeys, text, Enter raw key; no Escape raw key
- `TestSendPaneVimModeSendsEscapeIPrefixBeforeText` — vimMode fallback (capture fails, manual toggle): `i` via SendKeys then text; no Escape raw key
- `TestSendPaneWithoutVimModeDoesNotSendEscapePrefix` — no INSERT, vimMode=false: no `i`, no Escape; only Enter raw key
- `TestBroadcastAutoDetectsPerSession` — two sessions: INSERT session gets text+Enter; COMMAND session gets `i`+text+Enter; no Escape raw keys anywhere
- `TestSendPaneLiteralTextSentViaSendKeys` — literal text with Enter submit
- `TestSendPaneCtrlJSendsEnter` — ctrl key queued, then Enter from ctrl key + Enter from submit

**Poller (`internal/state/poller_test.go`)**
- `TestAutoEscalateSendsToConductorOnWaitingTransition` — conductor gets `i`+text+Enter when no INSERT visible
- `TestReplyRoutingSendsToWorker` — worker reply delivery with Enter
- `TestReplyRoutingDuplicateNotResent` — dedup with Enter accounting

## Invariants

- `CapturePaneView` failure is always safe: fall back to conservative assumption (INSERT, no prefix)
- Escape is never sent as a raw key in the INSERT-mode detection path (avoids Meta-i)
- Enter is always the last action in `sendToPane` and `sendToSession`
- The `Ctrl+V` toggle and `[vim]` label remain for non-claude sessions
