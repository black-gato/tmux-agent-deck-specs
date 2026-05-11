# Split Panel TUI — Manual QA Checklist

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

- [x] `v` expands detail panel to full width; list disappears
- [x] `v` again returns to split layout

**Feedback:**

---

## Edit Notes (`e`)

- [x] `e` on a session → notes editor opens in right panel with existing notes pre-filled
- [ ] Typing and pressing Enter → notes saved, normal view restores
- [x] `Esc` → discards changes, notes unchanged
- [x] `e` on a group → no effect (no crash)

**Feedback:** when in split layout you can't actually create a note. The tui seems to be frozen but when I hit escape I can interact with the tui again

---

## Live Output

- [x] Output tail scrolls to latest lines when a session is running
- [x] Output area respects available height (doesn't overflow into footer)

**Feedback:**

---

## App Header

- [ ] Status counts (running / waiting / idle) update as session states change

**Feedback:**
Icons need to be fixed I can't tell the difference between running and waiting interms of usage.
I want running to mean that something is processing in that session (either claude is running or something is process like test). Running currently seems to mean that the session is not empty. 

waiting should be waiting for input from the user.

idel should be nothing is either waiting for a response or actively processing data

dead should be session that aren't responding

---

## Footer

- [x] `v Expand output` key hint is visible and accurate

**Feedback:** 

---

## General Notes

