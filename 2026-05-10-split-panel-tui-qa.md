# Split Panel TUI — Manual QA Checklist

## Current Coverage

Automated coverage today is split across unit tests in `internal/ui` and the build-tagged E2E suite in `test/e2e`.

Covered automatically:

- split renders at narrow and wide widths
- terminal resize redraws both panels
- long session titles truncate at the divider
- session detail panel shows title, status, group, panes, notes, and captured output
- group selection leaves the detail panel empty without crashing
- missing tmux session output/pane data is handled safely
- notes edit open/save/discard behavior and group-row no-op
- output tail stays within the available panel height
- dialog escape and post-resize rendering recovery in the real TUI

Manual-only checks still needed:

- `v` full-screen toggle behavior in a real terminal session
- live output tail behavior under rapid streaming output
- header status counts changing under real multi-session churn
- icon clarity and wording for `running`, `waiting`, `idle`, and `error`
- note editing ergonomics in a real interactive terminal, beyond unit coverage

## Layout

- [x] Split renders correctly at various terminal widths (narrow ~80, wide ~200)
- [x] Resizing the terminal reflows both panels without artifacts
- [x] Left panel truncates long session names at the divider

**Feedback:** The text seems to shift when selecting text, making the right panel shift not be aligned with it self

---

## Detail Panel Content

- [x] Cursor on a session → detail panel shows title, status symbol, group path, pane list
- [x] Cursor on a group → detail panel is empty (no crash)
- [x] Session with no tmux session attached → pane list and output handle gracefully

**Feedback:**

---

## Full-Screen Toggle (`v`)

- [ ] `v` expands detail panel to full width; list disappears
- [ ] `v` again returns to split layout

**Feedback:**

---

## Edit Notes (`e`)

- [x] `e` on a session → notes editor opens in right panel with existing notes pre-filled
- [ ] Typing and pressing Enter → notes saved, normal view restores
- [x] `Esc` → discards changes, notes unchanged
- [x] `e` on a group → no effect (no crash)

**Feedback:** Unit coverage passes for save/discard, but this should still be spot-checked in a real terminal session.

---

## Live Output

- [x] Output tail scrolls to latest lines when a session is running
- [x] Output area respects available height (doesn't overflow into footer)

**Feedback:**

---

## App Header

- [ ] Status counts (running / waiting / idle) update as session states change

**Feedback:**
Icons still need a product pass. `running` should mean active processing, `waiting` should mean blocked on user input, `idle` should mean stable but not waiting or processing, and `error` should cover dead or missing sessions.

---

## Footer

- [x] `v Output` key hint is visible and accurate

**Feedback:** 

---

## General Notes
