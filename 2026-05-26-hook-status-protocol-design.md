# Hook Status Protocol Design

**Date:** 2026-05-26
**Status:** Proposed
**Supersedes:** the conductor-poke behavior in [2026-05-23-hook-handler-design.md](2026-05-23-hook-handler-design.md)
**Fixes:** BUG-021

## Overview

Replace the current DB-and-tmux-coupled hook handler with a **file-based hook status protocol**: Claude Code (and later Gemini/Codex) lifecycle hooks write an atomic per-session JSON status file, and the poller reads those files on a fast, cheap cadence to drive session status in near-real-time. A fresh hook status overrides pane-scrape detection; otherwise we fall through to `DetectStatus`.

This fixes BUG-021 (hook updates never arrive) by removing the two structural failure modes that cause it — opening a possibly-wrong `state.db` from the hook subprocess, and resolving session identity from a runtime tmux lookup — and gives us sub-second status latency without adding an fsnotify dependency.

The design is informed by a read of `asheshgoplani/agent-deck` (MIT, 2.5k★), whose status system is the same architecture we are adopting here. We take their *protocol* (env-var identity + atomic status files + freshness windows) and their *cold-read backbone*, but deliberately **not** their fsnotify watcher — we fold the read into our existing poll loop instead.

## Motivation

Our current `hook-handler` (`cmd/hookhandler.go`) does a fundamentally different job than a status feed:

1. Resolves identity at hook time via `tmux display-message -p #S` — fragile inside a Claude-spawned hook subprocess.
2. Calls `openDB()` — if the registered binary opens a different `state.db` than the running deck, every lookup misses.
3. `SendKeys` a message straight into the conductor's pane — it **never updates session status**.
4. Has five silent `return nil` dead-ends: any miss is invisible, with no log.

This explains every BUG-021 symptom: "updates never arrive" (silent misses on DB/tmux mismatch) and "duplicate `agent-deck` + `tmux-agent-deck` registrations" (non-idempotent install across binary names). The fix is a redesign of what the hook handler *does*, not a patch.

## Goals

- A hook event updates the owning session's status with ≤~250ms perceived latency.
- The hook handler has **zero** dependency on the DB or on a runtime tmux lookup.
- Hook identity is unambiguous and set at launch, not inferred at hook time.
- Failures are logged, never silent.
- `install-hooks` is idempotent; the duplicate-registration problem is resolved.
- The existing conductor-notification behavior is preserved, decoupled from the hook handler.

## Non-Goals

- fsnotify / inotify / kqueue watching (deferred; additive later if 250ms feels slow).
- Multi-tool hook support beyond Claude (Gemini/Codex follow the same protocol later).
- Flicker telemetry / structured-logging overhaul (low payoff for a single-user TUI).
- Changing the pane-scrape detector (`DetectStatus`) — it remains the fallback.

## Architecture

### 1. Instance identity via tmux session env (`-e`)

When the deck launches an agent into a tmux session, set the session's deck ID as a tmux session environment variable so the launched process and every child (including Claude's hook subprocess) inherit it:

```
tmux new-session -d -s <name> -c <startDir> -e AGENTDECK_INSTANCE_ID=<s.ID> <launchCommand>
```

**Launch-path trace (resolved 2026-05-26):** all launches funnel through the single function `Client.NewSession` (`internal/tmux/client.go:60`), called only from `cmd/session.go:27` and `internal/ui/app.go:993`. The worktree path (`internal/ui/form.go`) only changes `-c <startDir>` (it runs `git worktree add`, then sets `projectPath = worktree`) and reuses the same `NewSession`. The `claude-dangerous` preset is merely a different `<launchCommand>` string (`resolveLaunchCommand`, `client.go:37`). So a single injection point covers every case.

**Why `-e` rather than a `VAR=val` command prefix:** tmux `-e` (tmux 3.0+; target machine runs 3.6b) sets the var in the session environment, inherited without shell-quoting concerns and uniformly across `claude`, `claude-dangerous`, `shell` (`zsh -il`), and flagged tools. The command-prefix approach upstream agent-deck uses (`instance.go:749`) depends on the command going through `sh -c` and is fragile for the interactive-shell and flagged cases.

**Implementation cost:** thread `s.ID` into `NewSession` — signature change touching `ClientIface` (`client.go:17`), `FakeTmuxClient` (`internal/testutil/tmux.go:55`), and the two call sites.

**Edge case — imported sessions:** sessions adopted via the Import feature were started outside the deck, never went through `NewSession`, and have no `AGENTDECK_INSTANCE_ID`. Their hooks cannot be attributed to a deck instance, so they cleanly fall back to pane scraping. Acceptable.

### 2. Hook handler writes an atomic status file

Rewrite `cmd/hookhandler.go`:

- Read `AGENTDECK_INSTANCE_ID` from env. If empty → exit 0 silently (session not deck-managed).
- Validate the ID against `^[a-zA-Z0-9][a-zA-Z0-9_.-]*$` and reject `..` (path-traversal guard).
- Read the JSON payload from stdin (size-limited to 1 MB).
- Map `hook_event_name` → status (see table below).
- Write atomically to `~/.tmux-agent-deck/hooks/{instance_id}.json` via `write tmp → rename`.
- **No DB open. No tmux lookup.** Always exit 0 (never block Claude Code). Log write failures at warn.

Event → status mapping:

| Hook event           | Status    |
|----------------------|-----------|
| `SessionStart`       | `waiting` |
| `UserPromptSubmit`   | `running` |
| `Stop`               | `waiting` |
| `Notification` (permission_prompt / elicitation_dialog) | `waiting` |
| `SessionEnd`         | `dead`    |
| (other)              | (ignored) |

Status file shape:

```json
{ "status": "running", "session_id": "<claude session id>", "event": "UserPromptSubmit", "ts": 1716700000 }
```

The atomic `tmp → rename` discipline is **load-bearing**: it is what makes the directory-mtime gate (below) correct. It must not regress to in-place writes.

### 3. install-hooks: single constant command, idempotent

- One hook command string: `tmux-agent-deck hook-handler` (no per-session args; identity comes from env).
- Read-preserve-modify-write of `~/.claude/settings.json`; an install check that no-ops if our entries are already present for all events.
- Remove/migrate any stale `agent-deck` entry so we don't get double-fires (BUG-021 duplicate registration).

### 4. Fast cold-read loop in the poller

Add a second goroutine alongside the existing 1s `PollOnce` (`internal/state/poller.go`). It reads hook files only — no `CapturePaneOutput` — so it can run far faster than the pane poll.

Loop, every **250ms**:

1. `os.Stat(hooksDir)`. If the directory mtime is unchanged since last tick → **return** (one syscall; the common idle case). Correct because every hook write is a `rename` into `hooksDir`, which bumps the directory mtime.
2. Otherwise `os.ReadDir(hooksDir)`; for each `*.json` whose own `ModTime` advanced past a per-file cursor, parse it.
3. Derive status with per-tool **freshness windows** (Claude: 2m). A stale file falls through (does not override pane scrape).
4. `UpdateSessionStatus` only when the derived status differs from the session's current status.
5. **Force a full rescan every ~5s** regardless of the gate — insurance against coarse-mtime filesystems and same-second writes. (APFS is nanosecond-granular, so this is belt-and-suspenders.)

```
fastHookLoop()  (250ms ticker)
  └─ stat(hooksDir)
       ├─ mtime unchanged && not forced-rescan-tick → return   (1 syscall)
       └─ readdir → for each changed file:
            ├─ parse JSON
            ├─ fresh per freshness window? ── no ──▶ skip (pane scrape decides)
            └─ status != current?          ── yes ─▶ UpdateSessionStatus + signal UI
```

### 5. Status precedence (merge rule)

When both a hook status and a pane-scrape status exist for a session:

- **Fresh hook status wins** (within its per-tool freshness window).
- Otherwise the pane-scrape `DetectStatus` result is used.
- `StatusStopped` is user-intentional and is never overridden by a hook.

The hook loop writes the override into the DB; the existing 1s pane loop continues to write fallback status. Because both gate on "only write when changed," they don't thrash each other for an unchanged session.

### 6. Push UI refresh

The UI currently re-reads the DB on a hardcoded 1s timer (`internal/ui/app.go` `tick()`), which would cap end-to-end latency at ~1s regardless of how fast the poller writes. To realize the 250ms target:

- Hand the `*tea.Program` (or a `func()` send-closure) to the poller from `launchTUI()` in `cmd/root.go`.
- When the fast hook loop writes a status change, call `program.Send(tickMsg{})` to trigger an immediate `Reload()` instead of waiting for the next 1s tick. `Program.Send` is goroutine-safe.
- The 1s `tick()` stays as the baseline refresh for pane-scrape changes.

### 7. Conductor notification (preserve, decoupled)

The current hook handler's only real function — poking the group conductor — moves into the poller's existing `→ waiting` transition point (where `notifyWaiting` / `autoEscalate` already fire). This keeps the behavior while removing it from the hook handler, so the hook path stays a pure status feed.

## Data Flow

```
Claude Code hook fires
  └─ tmux-agent-deck hook-handler          (inherits AGENTDECK_INSTANCE_ID)
       └─ write ~/.tmux-agent-deck/hooks/{id}.json   (atomic, no DB)

fastHookLoop (250ms)                         pane loop (1s, unchanged)
  └─ dir-mtime gate → parse changed files      └─ CapturePaneOutput → DetectStatus
       └─ fresh? override status in DB               └─ fallback status in DB
            └─ program.Send(tickMsg{})  ──────────────▶ UI Reload() (immediate)
```

## File / Path Conventions

- Hooks dir: `~/.tmux-agent-deck/hooks/` (home-anchored, **not** working-dir-relative — this is the file-path analog of the DB-path bug we are fixing; it must be stable across the deck process and the hook subprocess).
- Fallback when `$HOME` is unset: `os.TempDir()/.tmux-agent-deck/hooks/`.
- File perms: dir `0700`, files `0600`.

## Failure Modes & Mitigations

| Failure | Mitigation |
|---|---|
| `AGENTDECK_INSTANCE_ID` not propagated (worktree / DSP launch) | Verify both paths in implementation; integration test asserts the env var reaches a hook subprocess. |
| Hook subprocess and deck resolve different `$HOME` | Home-anchored fixed path; log the resolved path on write failure. |
| Stale file after a crash | Freshness window: stale status is ignored, pane scrape takes over. |
| Coarse-granularity / same-second mtime | 5s forced full rescan. |
| Duplicate `agent-deck` registration still present | `install-hooks` migration removes it. |
| Partial/corrupt file mid-write | Atomic rename means readers never see a partial file; a parse error on one file skips it and retries next tick. |

## Testing

- `internal/hook` (or successor): event→status mapping table; malformed/oversized payload handling; instance-ID validation rejects traversal.
- Hook handler: writes the expected file atomically; no-ops with empty env; exits 0 on all paths.
- Poller fast loop: dir-mtime gate skips when unchanged; parses only advanced files; freshness window causes stale files to fall through; status written only on change; 5s forced rescan fires.
- Precedence: fresh hook overrides pane scrape; stale hook does not; `stopped` never overridden.
- `install-hooks`: idempotent; removes legacy `agent-deck` entry.
- Integration (`testutil.FakeTmuxClient` + temp `$HOME`): hook file write → status reflected within the loop interval; `program.Send` triggers a Reload.
- All black-box `package *_test`, `testutil.OpenTestDB(t)`.

## Open Questions

1. ~~Does `AGENTDECK_INSTANCE_ID` survive our worktree and `claude-dangerous` launch paths?~~ **Resolved 2026-05-26** — single launch path through `NewSession`; use tmux `-e` (see Architecture §1).
2. Final per-tool freshness window for Claude — start at 2m (matches upstream `HookFastPathWindow`); tune if Stop→idle feels slow.
3. Should the conductor poke remain edge-triggered in the poller long-term, or become its own transition-watcher once multi-tool hooks land?

## Rollout

1. Thread `s.ID` through `NewSession`; set `AGENTDECK_INSTANCE_ID` via tmux `-e` (launch path confirmed single — see §1).
2. Rewrite hook handler to write status files; keep old conductor poke temporarily behind the same command.
3. Add the fast cold-read loop + precedence merge.
4. Wire `program.Send` push refresh.
5. Make `install-hooks` idempotent + migrate duplicate registration.
6. Move conductor notification into the poller transition point; delete the old handler behavior.
7. Close BUG-021.
