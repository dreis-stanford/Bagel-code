# Bagel — Development Context

## What this is
A family card game called "Bagel" built as a single HTML/JS/CSS file, targeting eventual iPad App Store publication via Xcode/SwiftUI. Development in Claude chat sessions.

## Repository
- GitHub: https://github.com/dreis-stanford/Bagel-code (public)
- Live: https://dreis-stanford.github.io/Bagel-code
- Files: `index.html` (game), `RULES.md` (rules), `CONTEXT.md` (this file)

## Game overview
- 106-card deck (2× standard 52 + 2 jokers, each card has `deck:0` or `deck:1` property)
- 3–5 players; two scoring systems: points (game ends at 1000) and money (winner = most money)
- See RULES.md for full rules

## Terminology
- "Calling cards" = announcing card count before discarding (formerly "declaring")
- "Declaring a joker" = stating what card joker represents (keep this term)
- "Redeemer" = natural card matching joker's declaration
- Special cards: Deuces (wild), Jokers (wild), Queens of Spades (special, not wild)

## Current status — shared for beta testing
URL shared with trusted testers: https://dreis-stanford.github.io/Bagel-code

## Features implemented
- Full game flow: deal, cut, play, score, end game at 1000 pts
- All meld types with wild card rules, Q♠ position check
- Wild position in runs determined by selection order (left=low, right=high)
- Joker declaration (declared as specific card), redemption with normDecl() normalization
- Joker notification modal on meld; redemption notification to all players
- Calling cards: only before discard, cannot go out same turn as calling
- shouldCall(cp) — full rule: ≤3 cards after discard OR canBagel(hand) returns non-null
- missedDeclaration only set on non-going-out turns
- Scoring: points, money, checks (×100 each), bagels (×250, circled ⊙), dream (check ✓)
- Full scorecard with all hands, subtotals, checks/bagels visible
- Game ends automatically at 1000 points
- Conservative CPU: melds greedily, smart pile pickup, discards highest penalty
- Gambler CPU: holds for bagel, bails if someone calls or bagel impossible
- canBagel/canMeldAll: recursive search with 2000-iter limit + canBagelCached() per turn
- CPU names: Conservative = Plain/Pumpernickel/Egg/Cinnamon-Raisin; Gambler = Poppy/Everything/Garlic/Onion
- Auto-names when switching player type in setup
- Three game modes: all-human (pass & play), one human + CPU (continuous), all CPU (watch)
- Show CPU cards checkbox (default OFF — shows melds/discards but hides hand values)
- CPU speed selector: Fast(800ms)/Medium(1600ms)/Slow(2400ms); all-CPU at half speed
- CPU auto-cut; shows cut result briefly when Show CPU cards is ON
- Async CPU turns: one meld at a time with pauses; Continue ▶ button after each CPU discard
- Debug deal screen; Audit Deck button in Scores → Debug (shows popup, no console needed)
- auditDeck() tracks card integrity using deck:0/deck:1 property
- Discard history: last N-1 cards shown beside top card on game screen
- Recent discards shown on handoff screen (last N cards, one full round)
- Calling card loud modal notification to all players
- Joker declaration/redemption modal notification queued per player
- Two-column layout: game main + sidebar (players, hand penalty private)
- Tap to select, hold & drag to reorder hand (8px threshold)
- sortMeld sorts runs by declaredAs rank for wilds
- cancelMeld() clears _pendingDecl to prevent joker pre-mutation on cancel
- Rules reference modal with two tabs: Game Rules and Playing on Screen
- Feedback button: captures game state + user description, copy to clipboard or email

## Architecture
- `G` object = full game state
- Card: `{id, deck, rank, suit, wild, joker, qs, declaredAs}`
- `tryRun()` → `{valid, suit, assignedRanks}` assigns ranks to wilds by selection order
- `valMeld()` only sets declaredAs if `!card.declaredAs` (won't overwrite user's declaration)
- `canBagel(hand)` / `canMeldAll(hand,_iters)` — recursive with 2000-iter limit, works on copies
- `canBagelCached(cp)` — cached by playerId+turnCount+handLength
- `shouldCall(cp)` — ≤3 cards after discard OR canBagelCached returns non-null
- `normDecl(s)` — normalizes suit input (3C→3♣, 10s→10♠)
- Player types: 'human', 'conservative', 'gambler'
- `singleHumanMode` / `allCPUMode` in G control handoff/flow behavior
- `cpuTurnId` guard prevents stale async callbacks after hand ends
- `auditDeck()` — checks all card locations for duplicates/missing; callable in-game

## Key design decisions
- Card order in selection = wild placement intent in runs
- Pile pickup: 2+ naturals from hand OR table melds; if table naturals used, extends existing meld (not duplicate)
- Calling: only before discard, cannot go out same turn as calling, blocked next turn if missed
- Bagel = circled score ⊙ (+250); Dream = check ✓ (+100); both mutually exclusive
- Handoff screen only in all-human mode; single-human+CPU goes direct to game screen
- Hand scores sidebar: table pts public for all, hand penalty shown only for current player
- CPU pile pickup: separates hand cards from table naturals to avoid card duplication

## Known issues / next session priorities
1. **Deck duplication** — believed fixed (cpuDraw table natural handling) but needs auditDeck() confirmation from testers
2. **Gambler hang** — mitigated with 2000-iter limit and caching; monitor tester reports
3. **Self-scoring** — not yet built (each player reviews own score before committing)
4. **Show CPU cards** — works for hand; cut card peek shown when ON
5. **Gambler strategy** — needs real-game testing; may need tuning of bail conditions
6. **Xcode/SwiftUI transition** — planned after prototype is stable

## Session workflow
- Upload index.html + CONTEXT.md + RULES.md at session start
- Edit /mnt/user-data/outputs/bagel_v3.html
- Syntax check: node --check on extracted script
- Download and push to GitHub as index.html
- Test at https://dreis-stanford.github.io/Bagel-code (NOT in Claude artifact viewer)
- Feedback from testers comes via copied game state logs
