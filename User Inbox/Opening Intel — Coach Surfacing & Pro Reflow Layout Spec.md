# Opening Intel — Coach Surfacing & Pro Reflow Layout Spec

**From:** Dara, UX/Visual Designer (content direction from Renee, Chess Improvement Consultant)
**For:** Priya
**Scope:** `web-chess` Analyze tab, `src/App.tsx` (opening-intel block ~4318–4445) + `src/App.css` (`.opening-intel-card` ~1870–1960)
**Milestone:** M6 — see `docs/PROJECT_BRIEF.md` M6 Detail for Renee's full pedagogy pass this spec implements.

## What I found reviewing the current implementation

The whole "Opening Intel" card (`App.tsx:4318`, `{isProMode && (...)}`) is invisible in Coach mode — confirmed, not assumed, by reading the gate. Renee's finding that Coach-mode users get zero opening guidance is real, not a hypothesis.

I also traced why the token-friction problem (Renee's item 3) is worse than "annoying UX": `useOpeningExplorer.ts:84` has `if (!hasAuth) return` — **no request fires at all, for either Masters or Lichess source, without a token.** That's a client-side restriction, not a Lichess API requirement (their public Explorer API allows unauthenticated Masters/Lichess reads at a lower rate limit). This isn't just friction to soften — it's actively wrong for the Masters case, which needs no auth on Lichess's own API. Flagging this as a fix Priya should make regardless of the rest of this spec, since it blocks item 1 below outright: a Coach-mode line that only works after a user pastes an API token isn't "at a glance," it's just a smaller version of the same wall.

## 1. New: minimal Coach-mode opening line

**Placement:** new block in `App.tsx`, gated on `isCoachMode` (not `isProMode`), inserted where the `isProMode && <div className="opening-intel-card">` block currently sits (~4318), so the two are siblings — one or the other renders, never both, no visual gap either way.

**Content — one line, no card chrome:**

```
Ruy Lopez, Berlin Defence · usually Nf3 (68%)
```

- No border/card background — this is inline text in the existing Analyze panel flow, not a new bordered box. Adding a card here re-introduces the "theory dump" weight Renee is explicitly trying to avoid for Coach mode.
- Reuse the `.panel-copy small` class already used elsewhere in this panel for consistent type (0.68rem-equivalent, `--text-muted`).
- Opening name comes from the existing `useOpening` FEN-lookup hook (works offline, no token — confirmed in `src/hooks/useOpening.ts`). The "usually {SAN} ({win%})" clause needs exactly one move from the explorer response — Priya, once the auth fix above lands, this can be a Masters-source, no-token request scoped to the current FEN only (not the full move list, not the speed/rating pickers — those stay Pro-only).
- No opening detected at the position: render nothing (not an empty-state message) — Coach mode shouldn't show "no data" chrome for something this minor.
- No click target, no button — Coach mode gets information, not another control surface to parse.

**Mobile (<768px):** single line wraps naturally in the existing panel column; no special-casing needed, verified against how adjacent `.panel-copy small` lines already wrap in that column.

## 2. Pro mode: lead with the top move before the table

**Current problem:** `openingTopMoves.length > 0` renders the full `.opening-move-list` (`App.tsx:4414`, up to 5 rows) immediately after the aggregate stats line, with equal visual weight per row — no hierarchy, matches Renee's "table, not a takeaway" critique.

**Change:** split `openingTopMoves` into "lead move" (`[0]`) and "rest" (`[1..]`).

- Render the lead move as its own row, directly under the aggregate `Games / White / Draw / Black` line, visually distinct — larger text (bump to `--text-secondary` at current `.opening-move-row strong` size, ~0.75rem vs the list's 0.68rem) and a short label prefix: `Top book move: Nf3 · 68% (14,200 games)`. This is a text row, not a button — keep the click-to-search behavior on rows inside the list below, not duplicated here.
- The remaining moves (`openingTopMoves.slice(1)`) render in the existing `.opening-move-list` grid, unchanged structurally, but **behind a disclosure** closed by default: `Other book moves (4) ▾` — same `<details>` pattern Priya already shipped for the Review tab's full move list (`bd92c40`), so this is a known, low-risk implementation path, not a new interaction to design from scratch.
- This mirrors the "cap what's shown, disclose the rest" principle already applied to Critical Moments — consistent with the app's existing progressive-disclosure language, not a new pattern.

**CSS:** add `.opening-lead-move` (new rule, place after `.opening-token-meta` in `App.css`) — flex row, `justify-content: space-between`, `font-size: 0.75rem`, `padding: 0.3rem 0` — no border, sits inside the existing card padding. The `<details>` summary for "Other book moves" reuses the disclosure styling from the Review tab work rather than introducing a third pattern; check `App.css` for whatever class that shipped under (search for the Review tab's `<details>` block) and extend it rather than writing new CSS from scratch.

## 3. Token gating — narrowed, not removed

Per the auth-fix finding above: Masters-source requests (both the new Coach-mode line and Pro mode's default view) should work with no token. Keep the token requirement only for:
- Lichess-source stats (rating-bucket/time-control filtering needs the authenticated endpoint's rate limit)
- The full move list beyond the lead move, if Priya determines the anonymous rate limit can't sustain it — Priya's call, flag it to me if the anonymous tier turns out too restrictive for the list view and I'll revisit whether the disclosure should itself be token-gated.

This isn't a full auth redesign — just: don't block the single highest-value, lowest-cost display (one name, one move) behind a wall that Lichess's own API doesn't require.

## Open item back to Priya

Confirm the anonymous Masters request rate limit is sufficient for a per-move-hover or per-position Coach-mode line before wiring it live — if it's too aggressive, the Coach line may need its own lighter debounce/cache tier separate from the existing 250ms/5-min-TTL Pro-mode cache in `openingExplorer.ts`.
