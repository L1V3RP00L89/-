# Repertoire Import & Drill Mode Layout Spec

**From:** Dara, UX/Visual Designer (content direction from Renee, Chess Improvement Consultant)
**For:** Priya
**Scope:** New feature, `web-chess` — M11 (see `docs/PROJECT_BRIEF.md` M11 Detail)
**Grounded in:** Kris's actual purchased files, `Team Inbox/ChessGoalsCaroPGN.pgn` (Black) and `Team Inbox/London-System-Variations.pgn` (White)

## The files are not the same shape — this drives everything below

I opened both before speccing anything, because "PGN repertoire" undersells how different these two are.

**`London-System-Variations.pgn` (White):** 292 separate PGN games, zero comments, zero nested variations. Each game is one flat move sequence — e.g. `1. d4 Nf6 2. Bf4 d5 3. e3 c5 4. c3 *`. PlyCount ranges 4–39, averaging ~19. This is effectively **already decomposed into individual lines** — every branch point in the real repertoire has been unrolled into its own separate game. Covers seven Black setups by ECO (A45 Torre/London ×154, D00 Queen's Pawn ×67, A40 ×32, A80 Dutch ×24, A43/A44/A41 the rest).

**`ChessGoalsCaroPGN.pgn` (Black):** only 10 games — one "Course Outline" overview plus 9 chapters (Advanced 4.c3, Panov Attack, Exchange Variation, Tartakower, Two Knights, Fantasy Variation, KIA, etc.) — but those 10 games contain 592 comment blocks and 96 nested variation branches. This is a real annotated course: prose explaining *why* a move is played, sub-variations for sidelines, the works. A chapter game looks like `4. c3 Nc6 {Putting pressure on White's center. We wait for the white knight to come to f3...} 5. Nf3 (5. Bb5 {This is not a popular move...} Qa5 6. Bxc6+ ...)`.

**What this means practically:** the import step cannot treat "a PGN game" as "a drillable line" uniformly — and, per Renee's pedagogy sign-off below, it shouldn't treat "a PGN game" as "a drillable unit" at all, even for London where that mapping looks 1:1 on the surface.

## Revision (2026-08-19) — drill at branch points in a merged tree, not raw lines

Kris asked whether the White/Black imbalance meant content needed trimming to a "more realistic" load. Checked the real numbers before answering: London's 292 games looked like 292 independent things to learn, but merging them into one tree (deduping shared prefixes) shows most of that is the same trunk replayed from move 1 with a different tail — 5,489 raw plies collapse to 3,208 unique positions and only **379 real branch points** (positions with more than one repertoire-legal reply). Caro's 99 branch points were never inflated this way — it's a genuine tree already, so its real count is close to what it looked like: roughly **~100–150 decision points**.

**Renee's sign-off:** drill at branch points in a merged tree, not full lines replayed from move 1. Spaced repetition is supposed to spend review time on what's actually uncertain — re-typing `d4 Nf6 Bf4 d5 e3` forty times to reach one different move at the end is padding, not learning, and it's the same "over-analysis is the enemy" principle M4 already established for this app. This also makes the White-vs-Black workload comparison honest for the first time: ~379 vs ~150 real decisions, not 292 vs ~150 raw lines — still an uneven ask, but an honest one.

**What changes below as a result:**
- Import still parses every source game/chapter in full (all 292 London games, all 10 Caro chapters) — nothing gets dropped or trimmed.
- But the import step merges same-side games into **one tree per side**, not 292/10 independent lines — identical prefixes collapse into shared nodes.
- The drill queue operates on that tree's branch points (~379 White, ~150 Black), not on raw source-file line count.
- Each drilled attempt starts a few plies before its branch point, not from move 1 of the whole repertoire — enough runway to establish context, not a full replay of the shared trunk.

## 1. Import flow

Two separate imports, matching Kris's own framing — "my White repertoire" and "my Black repertoire," not a generic multi-file library manager (per the brief's scoped-down v1: two repertoires, one per side).

- Add a **Repertoires** entry point — I'd put it inside the M8 Training tab rather than as a new top-level nav item, since it's practice/habit content, not analysis. A card similar to the existing "Reviewed Games" / "Historical Library" cards: **"My Repertoires"**, two slots — **White** and **Black** — each showing either "Not imported" + an upload control, or once imported: the repertoire's source filename, a line count, and a "Re-import" / "Remove" action.
- Reuse the app's existing PGN import dialog/dropzone pattern (`PgnDialog`) rather than inventing new file-picking UI — same drag-and-drop-or-browse affordance users already know from importing a game.
- **Import must lift the existing multi-game restriction.** Both these files trip `hasMultiplePgnGames`/`PGN_MULTIPLE_GAMES_ERROR` in `src/engine/pgn.ts` today — that error path is correct for "importing one game to review," wrong for "importing a repertoire," so this needs a second import mode, not a removed restriction (single-game import elsewhere in the app should keep rejecting multi-game files; only the new repertoire-import path accepts them).
- **After import, show a parse summary before committing** — now framed as decisions, not lines, per the branch-point revision above: "Imported London-System-Variations.pgn — 379 decision points across 292 source lines" or "Imported ChessGoalsCaroPGN.pgn — ~150 decision points across 9 chapters." Showing both numbers (source lines/games *and* real decision points) is worth keeping — it's the honest version of the "292 lines" figure Kris originally asked about, and makes clear the drill queue isn't 292 (or even ~150) separate slogs.
- The "Course Outline" game in the Caro file (headers `[White "Course Outline"]`, no `[Black]` chapter label) is overview prose describing what each chapter covers, not a memorizable line — Priya, exclude games whose White-tag reads as a section header rather than a real chapter (this one's identifiable by having no distinct `[Black]` variation label — flag to Renee if that heuristic turns out too fragile once more files get imported later).

## 2. What happens to the Caro file's prose comments

This is the part I want to protect, not throw away. Comments like *"Putting pressure on White's center. We wait for the white knight to come to f3 before developing our bishop to g4"* are exactly the "curated human-designed" content M4's audit said the app doesn't have anywhere. Losing them on import would be a real loss, not just a formatting detail.

- Store the comment attached to its move/node when parsing (`parsePgnMoveTree` already carries comments through the tree — confirm this in implementation, don't silently drop them).
- **During an active drill, don't show the comment before the user has attempted the move** — showing "here's why you should play X" before they've tried defeats the point of drilling from memory, and conflicts with the app's own Coach-mode principle (engines/content should guide, not spoon-feed, per M4).
- **After the user plays their move (right or wrong), reveal the line's comment for that node if one exists** — this is where it earns its keep, as the "why" behind the answer they just committed to, not a hint beforehand.
- If a line has no comment (true for 100% of the London file, and true for individual moves inside Caro chapters that aren't annotated), just skip that step silently — no "no note for this move" filler text.

## 3. Drill-session visuals

- **Starting a drill:** pick a side (White or Black) — if only one repertoire is imported, skip the picker and go straight in.
- **A drilled attempt starts a few plies before its branch point** (per the revision above), not from move 1 of the whole repertoire — engine auto-plays the run-up moves quickly (visually distinct from the "live" portion, e.g. dimmed/fast-forwarded) to establish context, then hands control to the user right as the real decision arrives.
- **Board + a single line status strip above it**, plain text, no card chrome — matches the restraint I asked for in the M6 spec: `Decision 14 of ~379 · London System vs 1...d5` or similar, pulling the chapter/ECO label from the source PGN where available (Caro's chapter names are real: "Panov Attack", "Tartakower Variation"; London's are ECO codes only — fall back to `"A45 · Torre/London setups"` style text if no better label exists).
- The engine plays the "book" side's moves automatically (the opponent's half of the repertoire); the user plays their own side. This mirrors the existing Play-vs-computer flow visually — reuse that board/move interaction, don't build a second board component.
- **On a correct move:** brief, quiet positive feedback (a green flash on the piece/square, consistent with the existing "in-book" green dot styling I've seen elsewhere in the review UI) and immediate continuation — no modal, no interrupting confirmation.
- **On an incorrect move:** the attempted move is rejected (piece returns / doesn't move — same as an illegal move today), a small inline correction shows the actual repertoire move, then the comment for that node (if any) reveals per section 2. This position gets requeued into the SRS schedule per the brief's decision — no additional UI needed here beyond what M8's Woodpecker queue UI already establishes, once that exists.
- **End of line:** short completion state — "Line complete" — then either auto-advance to the next due line or return to a summary, mirroring whatever pattern Priya lands on for M8's drill queue overall (this should be one shared drill-session shell between M8's tactics queue and M11's repertoire lines, not two divergent UIs — flagging that for Priya's implementation call, not deciding the internals here).

## Open items back to the team

- **Priya:** confirm `parsePgnMoveTree` preserves comments per-node today, or whether that needs extending — I'm assuming yes but haven't read the parser closely enough to be sure.
- **Priya:** ran a rough branch-count pass on the Caro file before finalizing this spec (counting `(` variation-open tokens per chapter, not a full tree parse) — 99 branch points across the 9 real chapters (Course Outline excluded), 10–11 per chapter fairly evenly. That puts the real leaf-path count somewhere in the **~100–150 line range** once fully decomposed, not the "well over 100" I'd guessed before checking — close enough to my estimate that the "Line N of ~M" framing above holds up fine as one flat count, no per-chapter breakdown needed. Treat this as a sanity check, not a substitute for your actual tree-walk during implementation — a real parse may turn up a different exact number once nested (branch-within-a-branch) paths are counted properly.
- **Renee:** the "exclude the Course Outline game" heuristic above is a guess on my part, not a pedagogy call — flag if there's a cleaner signal to use once more repertoire files get imported by other users down the line (not just Kris's two).
- **Priya:** merging London's 292 flat games into one tree is straightforward (walk each game's moves, coalesce shared prefixes into one node chain, same shape as the sanity-check script used to find the 379 figure). Merging Caro's 9 already-tree-shaped chapters is a different job — each chapter is its own tree already (via nested PGN variations), so those 9 trees likely stay as 9 separate chapter-rooted trees rather than merging into one Black-wide tree, unless two chapters happen to share an opening trunk (unlikely here, they're responses to different White tries). Don't force a single merged tree where the source data doesn't actually share structure.
