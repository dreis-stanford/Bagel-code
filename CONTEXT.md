# Bagel — Development Context

## What this is
A family card game called "Bagel" being built as a web prototype (single HTML file) with the goal of eventually publishing as an iPad app via Xcode/SwiftUI. Development is happening in Claude chat sessions using vanilla HTML/CSS/JavaScript.

## Repository
- GitHub: https://github.com/dreis-stanford/Bagel-code (public)
- Live: https://dreis-stanford.github.io/Bagel-code
- Files: `index.html` (game), `RULES.md` (rules)

## Game overview
- 106-card deck (2× standard 52 + 2 jokers)
- 3–5 players best; 2 possible
- Two scoring systems: **points** (cumulative, game ends at 1000) and **money** (collected per hand and at game end)
- Winner = most money, not most points
- See RULES.md for full rules

## Current prototype status (bagel_v3.html)
Full playable prototype with:
- Deal, cut (slider UI), first-discard phase
- Full turn loop: draw → redeem joker → meld → discard
- Pile pickup with 2-natural requirement (hand or table melds)
- Meld validation: sets (3–4), runs (3+), wild rules, Q♠ position check
- Card order = wild placement intent in runs
- Joker declaration (declared as specific card), redemption, label-in-hand
- Joker notification modal shown to all players when joker is melded
- Calling cards (formerly "declaring"): only before discard, cannot go out same turn
- Scoring: points, money, checks (×100), bagels (×250), dream
- End of hand scorecard with circled bagel scores, ✓ check ticks
- Full scorecard in Scores modal (all hands, subtotals)
- Game ends automatically at 1000 points
- **Conservative CPU player** — melds greedily, picks up pile if valuable, discards highest penalty
- Three game modes: all-human (pass & play), one human + CPU (continuous), all CPU (watch)
- Show CPU actions checkbox + speed selector (Fast/Medium/Slow)
- CPU auto-cut when cutter is CPU
- Debug deal screen — specify any player's hand and discard top card
- Undo last hand / restart current hand (in Scores modal)
- Two-column layout (game main + sidebar with live scores)
- Drag-to-reorder hand (hold & drag), tap to select
- Compact inline meld display with sorted runs

## Terminology
- "Calling cards" = announcing how many cards you hold (formerly "declaring")
- "Declaring a joker" = stating what card a joker represents (keep this term)
- Special cards: Deuces (wild), Jokers (wild), Queens of Spades (special, not wild)

## Known bugs / issues to fix next session
1. **Show CPU cards** — currently "Show CPU actions" hides hand. Change to "Show CPU cards": when ON, show CPU hand and cut cards (everything CPU sees). When OFF, hide those.
2. **Pile pickup table naturals** — the `_fromMeldIdx` approach for using table melds as naturals may have edge cases; needs more testing.
3. **Card selection on iPad** — using `onmousedown` + `sourceCapabilities` to avoid double-fire on touch; may still have issues on some devices.
4. **CPU missed call** — CPU doesn't handle the calling requirement properly (it auto-calls via code but the timing rule isn't enforced for CPU).

## Next features to build
1. **Gambler CPU player** — holds cards hoping for a bagel; only melds if:
   - Someone else calls (threat that they might go out)
   - A bagel is no longer possible (hand too large/diverse)
   - NOTE: picking up pile only helps bagel if ALL pile+hand cards can be melded out leaving 1 discard
2. **Self-scoring** — at end of hand, each player reviews/confirms their own score before it's committed; friendly correction if disputed
3. **Show CPU cards option** — see CPU hand for debugging

## Architecture notes (for Xcode transition)
- All game logic is in vanilla JS — translates directly to Swift structs/functions
- Card order = wild placement intent (important for run validation)
- `tryRun()` returns `{valid, suit, assignedRanks}` — assigns ranks to wilds based on position
- Q♠ position check in spades runs: only blocks if NO valid placement avoids rank 12
- `G` object = full game state; snapshot/restore used for undo
- Player types: `'human'`, `'conservative'` (more to come: `'gambler'`)
- `singleHumanMode` / `allCPUMode` flags control handoff screen behavior
- Drag-to-reorder uses HTML5 drag API + touch events with 8px movement threshold

## Key design decisions made
- Card order in selection = wild placement intent (player reorders hand to specify)
- Pile pickup: top card must be melded with 2+ naturals (from hand OR table melds); if table naturals used, top card extends existing meld rather than creating duplicate
- Calling: only before discard, cannot go out same turn as calling
- Bagel = circled score on scorecard; checks = ✓ ticks next to score
- Dream = check (100pts), not bagel
- Handoff screen only in all-human mode; single-human + CPU goes direct to game
- No "watch" player type — all-CPU auto-detected when no humans selected
- CPU speed: Fast=800ms, Medium=1600ms, Slow=2400ms; all-CPU runs at half speed

## File locations
- Game: `/mnt/user-data/outputs/bagel_v3.html` (or `index.html` on GitHub)
- Rules: `/mnt/user-data/outputs/RULES.md`
- This file: `CONTEXT.md`
