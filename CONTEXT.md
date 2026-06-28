# Bagel — Development Context

## What this is
A family card game called "Bagel" built as a single HTML/JS/CSS file, targeting eventual iPad App Store publication via Xcode/SwiftUI. Development in Claude chat sessions.

## Repository
- GitHub: https://github.com/dreis-stanford/Bagel-code (public)
- Live: https://dreis-stanford.github.io/Bagel-code
- Files: `index.html` (game), `RULES.md` (rules), `CONTEXT.md` (this file)
- Branch `main`: stable old UI with all bug fixes
- Branch `new-ui`: new all-players layout (in progress, has bugs)

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
- shouldCall(cp): ≤3 cards after discard OR canBagelCached non-null
- normDecl(s): normalizes suit input (3C→3♣, 10s→10♠)
- CPU names auto-assigned: Conservative=Plain/Pumpernickel/Egg/Cinnamon-Raisin; Gambler=Poppy/Everything/Garlic/Onion
- Three modes: all-human (pass & play), one human+CPU (continuous), all CPU (watch)
- Show CPU cards checkbox (default OFF — shows melds/discards, hides hand values)
- CPU speed: Fast(800ms)/Medium(1600ms)/Slow(2400ms); all-CPU at half speed
- CPU auto-cut; cut result shown briefly when Show CPU cards ON
- Async CPU turns: one meld at a time with pauses; Continue ▶ after each CPU discard (single-human mode)
- Deck integrity: deck:0/deck:1 property on every card; auditDeck() accessible via Scores→Debug
- Discard history: last N-1 cards beside top card; recent discards on handoff screen
- Calling card loud modal notification to all players (human and CPU)
- Two-column layout: game main + sidebar (table pts public, hand penalty private)
- Tap to select, hold & drag to reorder hand (8px threshold)
- sortMeld sorts runs by declaredAs rank for wilds
- Rules modal: two tabs — Game Rules and Playing on Screen
- Feedback button: type selector + description + auto-captured game state; copy to clipboard or email
- Audit Deck button in Scores→Debug: shows popup with card counts and duplicates

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

## Bug fixes in main (stable)
- Pile pickup meld extension now validates before merging (was illegally adding mismatched cards to existing melds)
- cpuDiscard calls endHand instead of silently returning when hand is empty
- doUndeclare resets calledThisTurn (fixes edge case in undeclare/redeclare flow)
- autoDecl updates CPU call count downward when CPU melds further after calling
- CPU calling cards now shows loud modal notification (same as human confirmDecl)
- first-discard phase preserved correctly across all players in opening round (endTurnHO fix)
- doDiscard no longer sets phase='draw' prematurely in first-discard path

## New UI branch (new-ui) — in progress
Design goal: all players' hands visible at once on the left panel, face-up or face-down per context, grouped with each player's melds. Draw/discard piles and scores moved to right sidebar. Active player zone highlighted/enlarged with action buttons inline.

### New UI key changes
- `renderAllPlayers()` replaces `renderOthers()`, `renderAllMelds()`, `renderHand()` — renders all player zones
- `G.awaitingCPUAck` flag controls Continue ▶ button rendering inside CPU zone
- `tSel()` updates card CSS classes in-place (no full re-render) to prevent button/touch interference
- `htouchStart` ignores touches on button elements
- Action buttons only render for active human player zone (not CPU zones)
- Piles moved to sidebar panel

### New UI known bugs (as of end of session)
1. **All-human mode**: second player cannot draw or see discard pile after first player discards
2. **Human+CPU**: flow still getting stuck in first-discard round in some cases
3. **Discard pile**: intermittently not showing (renderPiles was accidentally deleted once — now restored)
4. **Browser caching**: GitHub Pages propagation delay can cause old version to appear after push — wait ~2 min and hard refresh

## Known issues / next session
1. **New UI bugs** — see above; fix all-human and human+CPU first-discard flow
2. **Self-scoring** — not yet built
3. **Gambler strategy** — needs real-game testing; bail conditions may need tuning
4. **Xcode/SwiftUI transition** — planned after prototype is stable
5. **Joker redemption meld not recognized** — redeemMustMeld flag may conflict with normal meld validation
6. **Joker redemption — add to existing meld** — should allow adding redeemed joker to existing meld
7. **Call/Discard combined button** — consider single "Discard (calling 2)" button to reduce taps
8. **CPU hand count visible when Show CPU cards OFF** — fix in renderOthers

## Session workflow
- Upload index.html + CONTEXT.md at session start
- For new-ui work: also upload the new-ui branch index.html
- Syntax check: node -e with new Function()
- Download and push to GitHub as index.html (to appropriate branch)
- Test at https://dreis-stanford.github.io/Bagel-code (NOT in Claude artifact viewer)
- Feedback via in-game Feedback button → copy to clipboard

## Branch strategy
- `main`: stable old UI, safe for beta testers
- `new-ui`: new all-players layout, work in progress
- GitHub web interface used for all pushes (iPad workflow)
- No local Git — revert via GitHub commit history if needed
