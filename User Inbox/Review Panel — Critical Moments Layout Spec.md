# Review Panel — Critical Moments Layout Spec

**From:** Dara, UX/Visual Designer (content direction from Renee, Chess Improvement Consultant)
**For:** Priya
**Scope:** `web-chess` Analysis sidebar → Review tab (`App.tsx` review-scaffold + critical-moments-card, `App.css` ~2260-2710)

## Problem

The Review tab currently renders, top to bottom: side filter → accuracy summary → classification pills → the **full per-move list** (`ReviewMoveList`, one row per move) → **Critical Moments** card.

Two issues:

1. **Content/hierarchy (Renee):** the full move list duplicates the classification pills (22 Best / 14 Good / 3 Inaccuracy / 2 Mistake) without adding anything actionable. The Critical Moments card — the 5 moves actually worth an adult improver's study time, each with SAN, impact, and a "Try best" action — is the useful artifact, but it's stuck below the full list and pushed off-screen.
2. **Bug (Dara):** `.moves-list` is specced with `max-height: 220px; overflow: auto` (`App.css:2568-2577`), but in the live app the list is rendering well past that — a long column of near-illegible colored hairlines instead of a scrollable 220px box. Root cause not yet diagnosed; flagging for investigation before the layout change below, since the reorder alone won't fix illegible rows if this persists.

## Changes

### 1. Reorder: Critical Moments above the full move list

New order: side filter → accuracy summary → pills → **Critical Moments** → full move list (collapsed, see #2).

In `App.tsx`, move the `critical-moments-card` block (currently ~line 4485) to render immediately after the `review-chips` block (~line 4467), before `ReviewMoveList` is rendered.

### 2. Collapse the full move list behind a disclosure

Wrap the existing `ReviewMoveList` render in a `<details>`/disclosure, closed by default:

```
All 41 moves ▾   (closed by default)
```

- Label should reflect the actual count: `All {visibleReviewRows.length} moves`.
- When open, give the list a real fixed height — **180px**, not the current 220px — with its own `overflow-y: auto`. This is intentionally shown only on demand, so it doesn't need to reserve as much vertical budget as before.
- Verify `max-height`/`overflow` actually clip the box once open (see bug above) — if the current CSS isn't containing it, the disclosure will have the same problem, just deferred.

### 3. Trim the move row columns

Current grid (`App.css:2598`):
```css
grid-template-columns: auto auto minmax(0, 1fr) minmax(4.6rem, auto) auto auto auto;
```
7 columns (index, SAN, uci, best, impact, confidence, quality) is too many for the ~380px sidebar and is what's collapsing rows into unreadable hairlines.

Drop `move-uci` and `move-confidence` from the default row — engine-internal detail, not something a sub-2000 player reads. New column set: index / SAN / best-move hint / impact / quality (5 columns). Keep uci/confidence available via row expansion or tooltip if that data's needed elsewhere (e.g. Engine Lab), just not in this default row.

Re-check the `max-width: 700px` grid-template-areas block (`App.css:2636-2676`) against the trimmed column set — it currently lays out uci/confidence explicitly and will need the same columns removed.

### 4. Critical Moments card copy (Renee's note, low effort, bundle if convenient)

`critical-moment-impact` currently reads raw eval loss ("Lost 2.13"). Reframe in plain language — e.g. "cost about a piece's worth of advantage" — rather than centipawn units. Not a layout change; flagging here so it's not lost since Priya will already be in this card for the reorder.

## Not in scope here

- Visual styling of the disclosure control (chevron, hover state) — standard app pattern, no new component needed.
- Mobile breakpoint (<768px) pass — reorder + disclosure should hold up as-is, but Dara will check on a real device once this lands before sign-off.
