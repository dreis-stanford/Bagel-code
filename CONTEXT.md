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

## Current status — in beta testing
URL: https://dreis-stanford.github.io/Bagel-code

## Features implemented
- Full game flow: deal, cut, play, score, end game at 1000 pts
- All meld types with wild card rules, Q♠ position check
- Wild position in runs determined by selection order (left=low, right=high)
- Joker declaration, redemption with normDecl() normalization (3C→3♣)
- _pendingDecl prevents joker pre-mutation on meld cancel
- Joker notification modal on meld; redemption notification queued per player
- Calling cards: only before discard, cannot go out same turn as calling
- shouldCall(cp): ≤3 cards after discard OR canBagelCached returns non-null
- missedDeclaration only set on non-going-out turns
- Scoring: points, money, checks (×100), bagels (×250 circled ⊙), dream (check ✓)
- Full scorecard with all hands, subtotals, checks/bagels visible
- Game ends automatically at 1000 points
- Conservative CPU: melds greedily, smart pile pickup, discards highest penalty
- Gambler CPU: holds for bagel, bails if someone calls or >4 turns without bagel possible
- canBagel/canMeldAll: recursive with 2000-iter limit, works on copies to avoid mutation
- canBagelCached(cp): cached by playerId+turnCount+handLength
- CPU names auto-assigned: Conservative=Plain/Pumpernickel/Egg/Cinnamon-Raisin; Gambler=Poppy/Everything/Garlic/Onion
- Three modes: all-human (pass & play), one human+CPU (continuous), all CPU (watch)
- Show CPU cards checkbox (default OFF — shows melds/discards, hides hand values)
- CPU speed: Fast(800ms)/Medium(1600ms)/Slow(2400ms); all-CPU at half speed
- CPU auto-cut; cut result shown briefly when Show CPU cards ON
- Async CPU turns: one meld at a time with pauses; Continue ▶ after each CPU discard (single-human mode)
- Deck integrity: deck:0/deck:1 property on every card; auditDeck() accessible via Scores→Debug
- Discard history: last N-1 cards beside top card; recent discards on handoff screen
- Calling card loud modal notification to all players
- Two-column layout: game main + sidebar (table pts public, hand penalty private)
- Tap to select, hold & drag to reorder hand (8px threshold)
- sortMeld sorts runs by declaredAs rank for wilds
- Rules modal: two tabs — Game Rules and Playing on Screen
- Feedback button: type selector + description + auto-captured game state; copy to clipboard or email
- Audit Deck button in Scores→Debug: shows popup with card counts and duplicates (no console needed)

## Critical bugs fixed this session
- **optShowCPU always true**: `checked!==false` evaluates incorrectly when element missing; fixed to `checked||false`
- **Hang on startup with CPU first player in hidden mode**: `showHandoff` called `runCPUInstant` before switching to game screen; DOM elements didn't exist yet. Fixed by calling `showSc('game');updateGame()` before `runCPUInstant` in hidden mode path.

## Architecture
- `G` object = full game state
- Card: `{id, deck, rank, suit, wild, joker, qs, declaredAs}`
- `tryRun()` → `{valid, suit, assignedRanks}` assigns ranks to wilds by selection order
- `valMeld()` only sets declaredAs if `!card.declaredAs`
- `canBagel(hand)` / `canMeldAll(hand,_iters)` — recursive, 2000-iter limit, works on copies
- `canBagelCached(cp)` — cached by id+turnCount+handLength
- `shouldCall(cp)` — ≤3 cards after discard OR canBagelCached non-null
- `normDecl(s)` — normalizes suit input (3C→3♣, 10s→10♠)
- Player types: 'human', 'conservative', 'gambler'
- `singleHumanMode` / `allCPUMode` in G control handoff/flow
- `cpuTurnId` guard prevents stale async callbacks
- `auditDeck()` — checks all card locations for duplicates/missing; in-game via Scores→Debug

## Key design decisions
- Card order in selection = wild placement intent in runs
- Pile pickup: 2+ naturals from hand OR table melds; table naturals extend existing meld (not duplicate)
- CPU pile pickup: separates hand vs table cards to avoid object duplication
- Calling: only before discard, cannot go out same turn as calling
- Bagel = circled ⊙ (+250); Dream = check ✓ (+100); mutually exclusive
- Handoff screen only in all-human mode; single-human+CPU goes direct to game screen
- Hidden CPU mode: game screen shown BEFORE runCPUInstant is called

## Known issues / next session
1. **Deck duplication** — believed fixed (cpuDraw table natural handling); verify with auditDeck() during testing
2. **Gambler hang** — mitigated with iter limit + caching; monitor tester reports
3. **Self-scoring** — not yet built
4. **Gambler strategy** — needs real-game testing; bail conditions may need tuning
5. **Xcode/SwiftUI transition** — planned after prototype is stable
6. **Call/Discard combined button** — currently Call and Discard are separate steps. Consider a single "Call & Discard" button that handles both actions at once. The call count could be shown inline (e.g. "Discard (calling 2)") to reduce the number of taps. Current two-step flow works but is awkward.
7. **CPU hand count visible when Show CPU cards is OFF** — the sidebar currently shows card counts for CPU players even when their cards should be hidden. In hidden mode, CPU card counts should not be displayed (only show that it's their turn and what they discarded/melded). Fix: in renderOthers, when !G.optShowCPU, hide the 🃏 count for CPU players.

## Session workflow
- Upload index.html + CONTEXT.md + RULES.md at session start
- Edit /mnt/user-data/outputs/bagel_v3.html
- Syntax check: node --check on extracted script
- Download and push to GitHub as index.html
- Test at https://dreis-stanford.github.io/Bagel-code (NOT in Claude artifact viewer)
- Feedback via in-game Feedback button → copy to clipboard
