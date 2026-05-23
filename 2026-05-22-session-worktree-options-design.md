# Session Worktree Options — Design

**Status:** Draft
**Date:** 2026-05-22
**Related:** [2026-05-17-session-form.md](2026-05-17-session-form.md), [2026-05-17-session-form-design.md](2026-05-17-session-form-design.md)

## Goal

Let the new-session form spawn a fresh `git worktree` per session. When the user fills in a branch, the session runs inside a new worktree of the repo at PATH instead of inside PATH directly. Leaving the branch field blank preserves today's behavior exactly.

## Decisions

- **Mode:** spawn a new worktree per session (no picking existing worktrees, no central worktree root).
- **Activation:** worktree fields are always visible; an empty BRANCH means "no worktree."
- **Persistence:** no schema change. `sessions.project_path` already stores the directory the session runs in; for worktree sessions, it stores the worktree path. Branch/base/repo are not persisted — git is the source of truth post-creation.
- **Errors:** if `git worktree add` fails, the error is shown inline in the form footer and the form stays open. No session row is created. No automatic cleanup of stale directories — the user reads the git error and edits the form (or cleans up in another terminal).

## Form layout

Field order, top to bottom:

```
TITLE     <text>
PATH      <text, defaults to cwd or group default>        ← the repo
BRANCH    <text, blank>                                    ← worktree trigger
BASE      <text, defaults to repo's default branch>        ← only used when BRANCH set
WORKTREE  <text, defaults to "<PATH>/../<repo>-<branch>">  ← only used when BRANCH set
TOOL      <select>
FLAGS     <text>
SCRIPT    <text>
```

BRANCH/BASE/WORKTREE sit immediately after PATH so the form reads top-to-bottom as: *where the repo is → what worktree to make → how to run it*. This is a deliberate change from the prior tab order; the alternative (appending the worktree group after SCRIPT) preserved muscle memory but buried the new fields.

### Field behavior

- **BRANCH blank** → BASE, WORKTREE ignored on submit. Session uses `expandPath(PATH)` for `project_path`. No git is invoked.
- **BRANCH non-blank** → session uses the worktree dir for `project_path`. PATH is treated as the source repo.
- **BASE default** is resolved lazily — on first focus of BASE, or on submit if still blank. Implementation: `git -C <PATH> symbolic-ref --short refs/remotes/origin/HEAD` with a fallback to `git -C <PATH> symbolic-ref --short HEAD`. If PATH is not a git repo, BASE stays blank and submit fails with the git error.
- **WORKTREE default** is recomputed whenever BRANCH changes — *unless* the user has manually edited WORKTREE. A `worktreeUserEdited bool` in `formState` tracks this and is set the first time `updateForm` writes a rune into the WORKTREE field. Once set, BRANCH edits no longer touch WORKTREE.
- BASE and WORKTREE render visually dimmed when BRANCH is empty, signalling they're inert. They remain focusable and editable (no special Tab-skip logic).

## Commit flow

In `commitForm`, after the existing title check:

```
title  := trimmed; if "" → return
path   := trimmed PATH (or ".")
branch := trimmed BRANCH

if branch == "":
    projectPath = expandPath(path)
else:
    base     := trimmed BASE; if "" → resolveDefaultBranch(path); if still "" → set formErr, return
    worktree := trimmed WORKTREE; if "" → deriveWorktreePath(path, branch)
    worktree = expandPath(worktree)

    if err := runGitWorktreeAdd(path, branch, worktree, base); err != nil:
        m.formErr = err.Error()
        return
    projectPath = worktree

db.CreateSession(... ProjectPath: projectPath ...)
```

Then existing post-commit handling runs unchanged (mode cleared, `Reload`, etc.) — but only if `formErr` was not set.

## Error display

`formState` gets a `formErr string` field. `renderForm` shows it on its own line above the hint row, in a red lipgloss style, when non-empty. Any keystroke in `updateForm` clears `formErr` before processing (so a user immediately sees their next attempt is "fresh"). The existing `m.err` channel is **not** used — keeping form-local errors local prevents the global error banner from competing with the form for the user's attention.

## Helpers

All new code lives in `internal/ui/form.go`. No new packages.

- `runGitWorktreeAdd(repo, branch, dir, base string) error` — `exec.Command("git", "-C", repo, "worktree", "add", "-b", branch, dir, base)`; on failure returns an error whose message is the combined stdout+stderr.
- `resolveDefaultBranch(repo string) string` — best-effort; returns `""` if `repo` is not a git repo or the lookups fail. Used by both the lazy BASE default and the submit-time fallback.
- `deriveWorktreePath(repo, branch string) string` — `filepath.Join(filepath.Dir(repo), filepath.Base(repo)+"-"+slug(branch))`. `slug` lowercases and replaces any non `[a-z0-9-_]` rune with `-`.

## Schema

No migration. `sessions.project_path` already exists and is the single column that varies between worktree and non-worktree sessions. The rest of the system (start/stop, list, observability) is unchanged — a worktree-backed session is indistinguishable from a regular one.

## Testing

Three tests in `internal/ui/form_test.go`, black-box through the `Model` API:

1. **`TestCommitWithoutWorktree`** — fill TITLE only, leave BRANCH blank, submit. Assert one session created, `project_path == expandPath(default PATH)`. Confirms the no-worktree path is untouched.
2. **`TestCommitWithWorktree`** — `t.TempDir()` + `git init` + `git commit --allow-empty -m init`. Fill TITLE, set PATH to the repo, BRANCH=`feature/x`, leave BASE/WORKTREE to defaults. Submit. Assert: session created; the directory at `project_path` exists; `git -C <project_path> rev-parse --abbrev-ref HEAD` returns `feature/x`.
3. **`TestCommitWorktreeErrorKeepsFormOpen`** — same setup as #2, but set BRANCH to an already-existing branch. Submit. Assert: `m.Mode() == "new-session"`, `formErr` non-empty, zero sessions in the DB.

Tests #2 and #3 skip via `t.Skip` when `exec.LookPath("git")` fails (CI without git).

Manual smoke checklist:
- Open form, type a branch name, watch WORKTREE auto-fill.
- Edit WORKTREE manually, then change BRANCH — WORKTREE should no longer auto-update.
- Submit with a bad BASE — form stays open, red error visible, no session in list.
- Submit a good worktree — session appears in list; `git worktree list` in the source repo shows the new entry.

## Out of scope

- Listing or attaching to existing worktrees.
- Worktree cleanup when a session is deleted.
- A global "worktree root" setting.
- Storing branch/base/repo on the session row.
- Pre-flight checks for stale WORKTREE directories (we just surface git's error).
