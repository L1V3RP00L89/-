# Repertoire Import & Drill Mode Layout Spec

**From:** Dara, UX/Visual Designer (content direction from Renee, Chess Improvement Consultant)
**For:** Priya
**Scope:** New feature, `web-chess` — M11 (see `docs/PROJECT_BRIEF.md` M11 Detail)
**Grounded in:** Kris's actual purchased files, `Team Inbox/ChessGoalsCaroPGN.pgn` (Black) and `Team Inbox/London-System-Variations.pgn` (White)

## The files are not the same shape — this drives everything below

I opened both before speccing anything, because "PGN repertoire" undersells how different these two are.

**`London-System-Variations.pgn` (White):** 292 separate PGN games, zero comments, zero nested variations. Each game is one flat move sequence — e.g. `1. d4 Nf6 2. Bf4 d5 3. e3 c5 4. c3 *`. PlyCount ranges 4–39, averaging ~19. This is effectively **already decomposed into individual lines** — every branch point in the real repertoire has been unrolled into its own separate game. Covers seven Black setups by ECO (A45 Torre/London ×154, D00 Queen's Pawn ×67, A40 ×32, A80 Dutch ×24, A43/A44/A41 the rest).

**`ChessGoalsCaroPGN.pgn` (Black):** only 10 games — one "Course Outline" overview plus 9 chapters (Advanced 4.c3, Panov Attack, Exchange Variation, Tartakower, Two Knights, Fantasy Variation, KIA, etc.) — but those 10 games contain 592 comment blocks and 96 nested variation branches. This is a real annotated course: prose explaining *why* a move is played, sub-variations for sidelines, the works. A chapter game looks like `4. c3 Nc6 {Putting pressure on White's center. We wait for the white knight to come to f3...} 5. Nf3 (5. Bb5 {This is not a popular move...} Qa5 6. Bxc6+ ...)`.

**What this means practically:** the import step cannot treat "a PGN game" as "a drillable line" uniformly. For London, that mapping already works — 292 games in, roughly 292 lines out. For Caro, each of the 10 games is a *tree*, and Priya's import needs to walk it and enumerate root-to-leaf paths as separate drillable lines — 96 branch points across 10 games means this could expand to well over 100 individual lines, not 10. I haven't hand-counted the exact number (that's a parsing job, not something to eyeball), but the UI needs to plan for "a handful of files in, a few hundred lines out," not a 1:1 count.

## 1. Import flow

Two separate imports, matching Kris's own framing — "my White repertoire" and "my Black repertoire," not a generic multi-file library manager (per the brief's scoped-down v1: two repertoires, one per side).

- Add a **Repertoires** entry point — I'd put it inside the M8 Training tab rather than as a new top-level nav item, since it's practice/habit content, not analysis. A card similar to the existing "Reviewed Games" / "Historical Library" cards: **"My Repertoires"**, two slots — **White** and **Black** — each showing either "Not imported" + an upload control, or once imported: the repertoire's source filename, a line count, and a "Re-import" / "Remove" action.
- Reuse the app's existing PGN import dialog/dropzone pattern (`PgnDialog`) rather than inventing new file-picking UI — same drag-and-drop-or-browse affordance users already know from importing a game.
- **Import must lift the existing multi-game restriction.** Both these files trip `hasMultiplePgnGames`/`PGN_MULTIPLE_GAMES_ERROR` in `src/engine/pgn.ts` today — that error path is correct for "importing one game to review," wrong for "importing a repertoire," so this needs a second import mode, not a removed restriction (single-game import elsewhere in the app should keep rejecting multi-game files; only the new repertoire-import path accepts them).
- **After import, show a parse summary before committing:** "Imported 292 lines from London-System-Variations.pgn" or "Imported ~140 lines across 9 chapters from ChessGoalsCaroPGN.pgn." This is the moment Kris finds out the Caro import expanded past 10 games into many more lines — better to surface that count plainly than let it be a surprise mid-drill.
- The "Course Outline" game in the Caro file (headers `[White "Course Outline"]`, no `[Black]` chapter label) is overview prose describing what each chapter covers, not a memorizable line — Priya, exclude games whose White-tag reads as a section header rather than a real chapter (this one's identifiable by having no distinct `[Black]` variation label — flag to Renee if that heuristic turns out too fragile once more files get imported later).

## 2. What happens to the Caro file's prose comments

This is the part I want to protect, not throw away. Comments like *"Putting pressure on White's center. We wait for the white knight to come to f3 before developing our bishop to g4"* are exactly the "curated human-designed" content M4's audit said the app doesn't have anywhere. Losing them on import would be a real loss, not just a formatting detail.

- Store the comment attached to its move/node when parsing (`parsePgnMoveTree` already carries comments through the tree — confirm this in implementation, don't silently drop them).
- **During an active drill, don't show the comment before the user has attempted the move** — showing "here's why you should play X" before they've tried defeats the point of drilling from memory, and conflicts with the app's own Coach-mode principle (engines/content should guide, not spoon-feed, per M4).
- **After the user plays their move (right or wrong), reveal the line's comment for that node if one exists** — this is where it earns its keep, as the "why" behind the answer they just committed to, not a hint beforehand.
- If a line has no comment (true for 100% of the London file, and true for individual moves inside Caro chapters that aren't annotated), just skip that step silently — no "no note for this move" filler text.

## 3. Drill-session visuals

- **Starting a drill:** pick a side (White or Black) — if only one repertoire is imported, skip the picker and go straight in.
- **Board + a single line status strip above it**, plain text, no card chrome — matches the restraint I asked for in the M6 spec: `Line 14 of ~140 · London System vs 1...d5` or similar, pulling the chapter/ECO label from the source PGN where available (Caro's chapter names are real: "Panov Attack", "Tartakower Variation"; London's are ECO codes only — fall back to `"A45 · Torre/London setups"` style text if no better label exists).
- The engine plays the "book" side's moves automatically (the opponent's half of the repertoire); the user plays their own side. This mirrors the existing Play-vs-computer flow visually — reuse that board/move interaction, don't build a second board component.
- **On a correct move:** brief, quiet positive feedback (a green flash on the piece/square, consistent with the existing "in-book" green dot styling I've seen elsewhere in the review UI) and immediate continuation — no modal, no interrupting confirmation.
- **On an incorrect move:** the attempted move is rejected (piece returns / doesn't move — same as an illegal move today), a small inline correction shows the actual repertoire move, then the comment for that node (if any) reveals per section 2. This position gets requeued into the SRS schedule per the brief's decision — no additional UI needed here beyond what M8's Woodpecker queue UI already establishes, once that exists.
- **End of line:** short completion state — "Line complete" — then either auto-advance to the next due line or return to a summary, mirroring whatever pattern Priya lands on for M8's drill queue overall (this should be one shared drill-session shell between M8's tactics queue and M11's repertoire lines, not two divergent UIs — flagging that for Priya's implementation call, not deciding the internals here).

## Open items back to the team

- **Priya:** confirm `parsePgnMoveTree` preserves comments per-node today, or whether that needs extending — I'm assuming yes but haven't read the parser closely enough to be sure.
- **Priya:** ran a rough branch-count pass on the Caro file before finalizing this spec (counting `(` variation-open tokens per chapter, not a full tree parse) — 99 branch points across the 9 real chapters (Course Outline excluded), 10–11 per chapter fairly evenly. That puts the real leaf-path count somewhere in the **~100–150 line range** once fully decomposed, not the "well over 100" I'd guessed before checking — close enough to my estimate that the "Line N of ~M" framing above holds up fine as one flat count, no per-chapter breakdown needed. Treat this as a sanity check, not a substitute for your actual tree-walk during implementation — a real parse may turn up a different exact number once nested (branch-within-a-branch) paths are counted properly.
- **Renee:** the "exclude the Course Outline game" heuristic above is a guess on my part, not a pedagogy call — flag if there's a cleaner signal to use once more repertoire files get imported by other users down the line (not just Kris's two).
