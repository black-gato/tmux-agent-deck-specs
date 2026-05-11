# Return to TUI Keybinding Design

**Date:** 2026-05-09  
**Status:** Approved

## Overview

When a user attaches to an agent session from the TUI, they are dropped into a tmux session with no obvious way back. This feature adds a `ctrl + q` keybinding that detaches the client and returns to the plain terminal where the TUI is running.

---

## Behavior

1. User presses `Enter` on a session in the TUI → `Attach()` is called
2. Before attaching, the existing `ctrl + q` binding in the `root` table (if any) is saved
3. `ctrl + q` (`C-q` in the `root` table) is set to `detach-client`
4. `tmux attach-session -t <session>` blocks until the user detaches
5. On return (any exit), the original `ctrl + q` binding is restored

The TUI process is still running while the user is in the tmux session; detaching returns them to it automatically.

---

## Implementation

**File:** `internal/tmux/client.go`

All changes are confined to the `Attach()` method. No DB, TUI, CLI, or schema changes.

### Flow

```
saveBinding("C-q")                   → tmux list-keys -T root C-q
setBinding("C-q", "detach-client")   → tmux bind-key -T root C-q detach-client
defer restoreBinding("C-q", saved)   → runs on any return from Attach()
tmux attach-session -t <session>     → blocks
```

### Parsing `list-keys` output

Output format: `bind-key -T root C-q <command>`

`ParseBindingCommand(output, key)` searches for `" <key> "` in the line and returns everything after it. If `list-keys` returns nothing or exits non-zero, there was no existing binding — restore is `tmux unbind-key -T root C-q`.

### Error handling

If `bind-key` or `unbind-key` calls fail, log the error but do not block the attach. The core operation (attaching to the session) always proceeds.

### Key

Hardcoded to `C-q` (ctrl+q) in the `root` table — no prefix required. Can be made configurable later via a field on the `Client` struct.

---

## Out of Scope

- Configurable key (post-MVP)
- Support for launching the TUI inside tmux (separate concern)
