# Bagel — Development Context

## What this is
A family card game called "Bagel" built as a single HTML/JS/CSS file, targeting eventual iPad App Store publication via Xcode/SwiftUI. Development in Claude chat sessions.

## Repository
- GitHub: https://github.com/dreis-stanford/Bagel-code (public)
- Live: https://dreis-stanford.github.io/Bagel-code
- Files: `index.html` (game), `RULES.md` (rules), `CONTEXT.md` (this file)
- Single branch: `main` — new UI with all bug fixes (old two-branch setup retired)

## Game overview
- 106-card deck (2× standard 52 + 2 jokers, each card has `deck:0` or `deck:1` property)
- 3–5 players; two scoring systems: points (game ends at 1000) and money (winner = most money)
- See RULES.md for full rules

## Terminology
- "Calling cards" = announcing card count before discarding (formerly "declaring")
- "Declaring a joker" = stating what card joker represents (keep this term)
- "Redeemer" = natural card matching joker's declaration
- Joker redemption: players may only redeem a Joker from their **own** melds (never another player's), and there is **no requirement to immediately meld** it after redeeming — it simply returns to their hand to be melded whenever they choose
- Special cards: Deuces (wild), Jokers (wild), Queens of Spades (special, not wild)

## Current status — in beta testing
URL: https://dreis-stanford.github.io/Bagel-code

## Features implemented
- Full game flow: deal, cut, play, score, end game at 1000 pts
- All meld types with wild card rules, Q♠ position check
- Wild position in runs determined by selection order (left=low, right=high)
- Joker declaration, redemption with normDecl() normalization (3C→3♣)
- Joker redemption restricted to a player's own melds only; redeemed Joker returns to hand with no immediate-meld requirement (openRedeem only lists current player's own melds; confirmRedeem no longer sets G.redeemMustMeld/G.mustMeldTopId)
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
- New UI: all players' zones visible at once on left panel; draw/discard piles and scores in right sidebar
- Active player zone highlighted/enlarged with action buttons inline
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
- `sortMeld(meld)` — sorts run cards by rank (uses declaredAs for wilds)
- `rcSmall(c)` — renders a small card element (used in meld displays)
- `isHumanTurn()` — returns true only when current player is human; gates all human action functions
- `renderAllPlayers()` — renders all player zones; replaces old renderOthers/renderHand/renderAllMelds; calls updateBtns()+updateDeclUI() at end to restore button state after every re-render
- `G.awaitingCPUAck` flag controls Continue ▶ button rendering inside CPU zone
- `tSel()` updates card CSS classes in-place (no full re-render) to prevent button/touch interference
- `htouchStart` ignores touches on button elements
- Action buttons only render for active human player zone (not CPU zones)
- Piles in sidebar panel
- Player types: 'human', 'conservative', 'gambler'
- `singleHumanMode` / `allCPUMode` in G control handoff/flow
- `cpuTurnId` guard prevents stale async callbacks
- `auditDeck()` — checks all card locations for duplicates/missing; in-game via Scores→Debug
- `openRedeem()` — lists only the current player's own melds' jokers (not other players')
- `confirmRedeem()` — swaps natural into the meld in place and returns the Joker to hand; does NOT set any "must meld" flag (rule change: own-Joker redemption has no immediate-meld requirement)
- `G.redeemMustMeld` / `G.mustMeldTopId` fields still exist in state (used by pile-pickup enforcement) but are no longer set by joker redemption
- `commitNewMeld(cp, cards)` / `commitAddToMeld(meld, addCards)` — the ONLY functions allowed to push a new meld onto `cp.melds` or mutate `meld.cards`; both internally re-run `valMeld` regardless of what the caller believes, so a mixed run+set, sub-3-card, or otherwise illegal meld can never be committed. `pushMeld()` is now a thin wrapper around `commitNewMeld()` kept for call-site compatibility.
- Fixed CPU pile-pickup bug (`cpuDraw`): when extending an existing meld with additional hand cards failed validation and there weren't enough hand cards (≥3) for a standalone meld, the old code force-merged the cards onto the existing meld anyway with NO validation — this was the source of the reported "CPU creates illegal meld (run+set mashup)" bug. Now it simply leaves those cards in the player's hand rather than ever performing an unvalidated mutation.
- `declRank(c)` — extracts the rank a wild's `declaredAs` represents (e.g. "A♥"→14), used by `tryRun` to honor an explicit player declaration
- `tryRun()` — when assigning a wild to a rank slot, if that wild already has `declaredAs` set, the attempt is rejected unless the assigned rank matches the declaration. Fixes the bug where e.g. Q♥,K♥,+wild was silently interpreted using selection-click-order (deuce clicked first → low extension → Jack) even when the player explicitly wanted it declared as an Ace.
- `openMeld()`/`confirmMeld()` — the "ambiguous wild position, please declare" UI (previously only triggered for Jokers via `sel.filter(c=>c.joker)`) now triggers for ANY wild card via `sel.filter(c=>c.wild)`, since Deuces have the identical ambiguous-run-position problem as Jokers (e.g. Q,K,+wild could be J-Q-K or Q-K-A). Skipped entirely when the selection is unambiguously a set (`trySet(sel)` true).

## Key design decisions
- Card order in selection = wild placement intent in runs
- Pile pickup: 2+ naturals from hand OR table melds; table naturals extend existing meld (not duplicate)
- CPU pile pickup: separates hand vs table cards to avoid object duplication
- Calling: only before discard, cannot go out same turn as calling
- Bagel = circled ⊙ (+250); Dream = check ✓ (+100); mutually exclusive
- Handoff screen only in all-human mode; single-human+CPU goes direct to game screen
- Hidden CPU mode: game screen shown BEFORE runCPUInstant is called
- first-discard phase applies only to handFirstPlayer (who holds 14 cards); all other players start in draw phase
- Joker redemption: own-melds-only, no immediate-meld requirement — since a player can only redeem their own Joker, they've necessarily already melded this hand, so the once-per-turn-meld condition is already satisfied; holding a redeemed Joker is a known strategic blunder (risk of being caught with it, no Bagel benefit) but is now legal

## Bug fixes in main (stable)
- Pile pickup meld extension now validates before merging (was illegally adding mismatched cards to existing melds)
- cpuDiscard calls endHand instead of silently returning when hand is empty
- doUndeclare resets calledThisTurn (fixes edge case in undeclare/redeclare flow)
- autoDecl updates CPU call count downward when CPU melds further after calling
- CPU calling cards now shows loud modal notification (same as human confirmDecl)
- first-discard phase preserved correctly across all players in opening round (endTurnHO fix)
- doDiscard no longer sets phase='draw' prematurely in first-discard path
- New UI: updateBtns()/updateDeclUI() null-guard when no active human zone in DOM (CPU turns)
- New UI: rcSmall() and sortMeld() were missing from new UI file — added
- New UI: startGame() was missing jokerNotifications, awaitingCPUAck, keptByCutter, cutterBot, topPart, botPart fields
- New UI: first-discard phase now only applies to handFirstPlayer; endTurnHO always sets phase='draw' for all subsequent players
- New UI: renderAllPlayers() calls updateBtns()+updateDeclUI() after every re-render to prevent buttons resetting to disabled when G.sel is non-empty
- New UI: cpuAfterDiscard() skips awaitingCPUAck during first-discard phase
- New UI: draw pile button visually disabled during first-discard phase
- New UI: isHumanTurn() guard on all human action entry points (tSel, drawCard, startPickup, openMeld, openAdd, openRedeem, doDiscard) prevents human from interacting during CPU turns
- CPU single-card meld bug: added pushMeld() helper that guards all CPU meld creation against sub-3-card melds; fixed cpuDraw pile pickup path where meldFromHand could be <3 cards and get pushed as an invalid meld
- UI: moved Hand # and current player display from top-left header to bottom of sidebar, freeing space in main player area
- Rule change: Joker redemption restricted to own melds only, and immediate-meld requirement removed (openRedeem/confirmRedeem updated; rules modal and RULES.md updated)
- CPU illegal meld bug: centralized ALL meld creation/extension through `commitNewMeld()`/`commitAddToMeld()`, which always re-validate via `valMeld()` no matter the call site. Fixed the specific root cause in `cpuDraw()`'s pile-pickup path, which was force-merging hand cards onto an existing meld without validation when the "nice" merge failed and there weren't enough cards for a standalone meld.
- Ambiguous wild-in-run declaration bug: a Deuce (or Joker) that could validly extend a run from either end (e.g. Q,K,+wild = J-Q-K or Q-K-A) was silently resolved via selection click-order with no way for the player to confirm/override, unlike Jokers which already had a "declare" prompt for ambiguous cases — except that prompt was gated to Jokers only (`c.joker`) and ignored Deuces (`c.wild`). Generalized the declare/ambiguous-detection UI in `openMeld()`/`confirmMeld()` to all wild cards, and made `tryRun()` honor an explicit `declaredAs` so the chosen run interpretation matches the player's stated intent rather than click order.

## Known issues / next session
1. **Self-scoring** — not yet built
2. **Gambler strategy** — needs real-game testing; bail conditions may need tuning
3. **Xcode/SwiftUI transition** — planned after prototype is stable
4. **Joker redemption meld not recognized** — likely resolved/moot now that immediate melding is no longer required; revisit if a related bug resurfaces when the player does choose to meld a held redeemed Joker
5. **Joker redemption — add to existing meld** — should allow adding redeemed joker to existing meld (still applies whenever the player chooses to meld it)
6. **Call/Discard combined button** — consider single "Discard (calling 2)" button to reduce taps
7. **CPU hand count visible when Show CPU cards OFF** — fix in player zone meta label
8. **CPU does not currently use joker redemption at all** — no CPU logic calls confirmRedeem; out of scope for this session but worth tracking since CPU could now reasonably redeem-and-hold a Joker
9. **Wild-declare prompt may over-trigger for sets with coincidentally same-suit naturals** — e.g. melding two same-rank-and-suit cards from different physical decks plus a wild: `trySet` correctly identifies it as a set (so the new generalized prompt is skipped), but this edge case wasn't exhaustively tested — watch for any regression where a wild-in-a-set is asked to "declare" unnecessarily.

## Session workflow
- Upload index.html + CONTEXT.md at session start
- Syntax check: node -e with new Function()
- When porting functions between files: run missing-function audit immediately — node -e comparing defined vs called functions between old and new file
- Download and push to GitHub as index.html to main branch
- Test at https://dreis-stanford.github.io/Bagel-code
- Feedback via in-game Feedback button → copy to clipboard

## Branch strategy
- Single branch: `main`
- GitHub web interface used for all pushes (iPad workflow)
- No local Git — revert via GitHub commit history if needed
