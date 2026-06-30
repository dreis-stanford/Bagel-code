# Bagel — Deferred Bug Tracker

Bugs that are known/reported but not yet fixed, with enough detail to pick the
investigation back up without re-deriving context. Resolved bugs are moved to
the "Resolved this session" log at the bottom for reference, then can be
pruned once stable.

---

## OPEN — CPU can merge two unrelated melds into one illegal combined meld

**⚠ Before re-investigating: confirm this was reproduced on the latest
pushed `index.html`, not a stale browser tab or an un-pushed GitHub Pages
deploy.** This bug was reported in the same session as (and immediately
after) the fixes for the "stale wild declaration" and "force-merge without
validation" bugs above — it's possible the browser/page wasn't refreshed and
the test was actually run against the pre-fix code. First step next session:
hard-refresh (or open a private/incognito tab) on
https://dreis-stanford.github.io/Bagel-code, confirm the deployed file
matches the `index.html` with this session's fixes (e.g. check that
`commitAddToMeld` exists and clears `declaredAs` before validating), and only
then try to reproduce this specific meld again. If it no longer reproduces,
this entry can likely be closed/merged into "Resolved this session" below.

**Reported example (Hand 3, Turn 2, Garlic/conservative):**
Meld shown as one 7-card group: `9♥ 2♣ J♥ J♣ J♣ Q♥ 2♦`
(2♣ and 2♦ are wild deuces.)

**Why this is illegal:** Decomposed, this is actually two structurally valid
melds that got fused into a single meld object:
- A 4-card heart run: `9♥, (wild=10♥), J♥, Q♥`
- A 3-card set: `J♣, J♣, (wild)` (two Jacks of clubs from the two physical
  decks, plus a wild)

As a single 7-card group it is neither a valid set (naturals aren't all the
same rank — 9, J, J, Q) nor a valid run (naturals aren't all the same suit —
hearts and clubs mixed). `valMeld()` would reject this combo immediately if
ever checked as one unit — `trySet`'s rank check and `tryRun`'s suit check
both correctly fail on it. This means the 7-card group was almost certainly
never validated as a whole; it was assembled incrementally through multiple
"add to meld" operations, and **some single step incorrectly approved an
addition that should have been rejected, or two previously-separate melds got
collapsed into one meld object reference.**

**What's already been ruled out / already fixed this session (so the bug is
something else, or a remaining gap in the same area):**
- `commitNewMeld()` / `commitAddToMeld()` are now the only functions allowed
  to push a new meld or mutate `meld.cards`, and both always re-run
  `valMeld()` on the complete resulting card set before committing — this
  was confirmed to fix the earlier "CPU force-merges without validation"
  bug (`cpuDraw`'s pile-pickup) and the "stale wild declaration after
  extending a run" bug (`commitAddToMeld` now clears wild `declaredAs`
  before re-validating an extension). Both of those are fixed in the current
  `index.html`.
- Despite that, this NEW report still shows a clearly-illegal combined meld,
  so there is at least one more code path that either (a) isn't routed
  through `commitNewMeld`/`commitAddToMeld`, or (b) is routed through them
  but is feeding in a card set that incorrectly spans two existing melds.

**Investigation leads for next session:**
1. Audit `cpuDraw()`'s pile-pickup meld-detection (`testCards3`/`testCards4`
   construction, `meldFromTable`/`meldFromHand` split) — `tableNatIds` is
   built by flattening cards across *all* of a player's melds
   (`cp.melds.flatMap(m=>m.cards.map(c=>c.id))`), so if the best-pickup combo
   incorrectly draws "table naturals" from two *different* existing melds at
   once, `meldIdx` (which does `cp.melds.findIndex(...)`) would only resolve
   to the *first* matching meld, silently ignoring that the second natural
   actually belongs to a different meld. Confirm whether this can let a combo
   validate using cards from meld A + meld B together even though only meld A
   ever gets mutated.
2. Re-check whether any meld-creation path can run `combinations(hand, size)`
   (in `cpuFindBestMeld` / `cpuFindOneMeld`) against a pool that accidentally
   includes already-melded table cards (not just hand cards), which could let
   a "new meld" combo silently include cards that are already part of another
   meld.
3. Confirm there isn't a path where two independent `commitAddToMeld` calls
   in the same turn both validate successfully in isolation against the
   *same* target meld object before either one's mutation is "seen" by the
   other (a sequencing issue), resulting in a final state that was never
   validated as a whole. (Current code is synchronous/single-threaded per
   call, so this is unlikely, but worth ruling out for the async CPU turn
   functions specifically.)
4. Add a defensive **post-turn / post-meld consistency audit** (similar in
   spirit to `auditDeck()`) that runs `valMeld(meld.cards)` against every
   meld currently on the table after each turn and surfaces a warning/log if
   any meld fails — this would catch the *moment* an illegal meld is created
   rather than only being noticed when a player reads the table later, and
   would help pinpoint which code path is responsible.
5. Given the user's own diagnosis ("could be two different melds... but
   depending on circumstances, might not have been possible") — there may
   also be a *design* question to resolve alongside the bug: should "add to
   meld" ever be allowed to implicitly split/merge melds, or should the UI
   (and CPU logic) only ever support add-to-ONE-existing-meld or
   create-ONE-new-meld, never a combination that spans two pre-existing
   melds? Worth deciding explicitly so the fix has a clear target behavior.

**Suggested next step:** Add the consistency-audit hook (#4) first — it's
low-risk and will make root-causing #1–#3 much faster by catching the exact
turn/function where the corruption happens, rather than reasoning about it
after the fact from a static snapshot.

---

## Resolved this session (for reference — already fixed in current index.html)

- **CPU pile-pickup force-merge without validation** (`cpuDraw`) — when
  extending an existing meld failed validation and there weren't enough hand
  cards for a standalone meld, the old code merged anyway with zero
  validation. Fixed via `commitAddToMeld`/`commitNewMeld` centralization.
- **`cpuMeld()`'s single-card add-to-meld loop never called `sortMeld`** —
  fixed by routing through `commitAddToMeld`.
- **Stale wild `declaredAs` after extending an existing run** — extending a
  run with new cards can change which rank a wild needs to represent (e.g. a
  wild declared "6♥" in a 3-card run needs to become "8♥" once the run grows
  to 6 cards), but the old code only ever *set* `declaredAs` if it was unset,
  never *corrected* it. This produced exactly the "wild card visually stuck
  between two naturals it shouldn't be near" artifact (reported as Garlic's
  `4,5,2,6,7,2` meld). Fixed: `commitAddToMeld` now clears `declaredAs` on
  every wild card in the combined set before re-validating, so `valMeld`
  recomputes and re-labels them correctly for the new run shape.
- **Ambiguous wild run position silently resolved by click order** (e.g.
  Q♥,K♥,+wild defaulting to Jack instead of the intended Ace) — the
  "ambiguous position, please declare" UI only ever triggered for Jokers
  (`c.joker`), never Deuces (`c.wild`). Generalized to all wild cards in
  `openMeld()`/`confirmMeld()`, and `tryRun()` now honors an explicit
  `declaredAs` set during the *current* meld action as a hard constraint.
- **CPU call counts never auto-corrected downward** — `shouldCall()`
  short-circuits to `false` once `cp.declared` is already `true`, so a CPU
  that melded further after calling (reducing its hand below the originally
  declared count) kept showing the old, now-too-high, call forever.
  `autoDecl()` already contained the correct downward-correction logic but
  was only ever invoked from the *human* discard flow. Fixed by calling
  `autoDecl()` from `cpuDiscard()` (pre-discard, mirroring the human
  meld-time correction) and from `cpuDoDiscard()` (post-discard, mirroring
  the human discard-time correction) so all CPU discard paths — conservative
  and gambler alike — get the same correction.
- **"Declare wild card(s)" popup said "Declare joker(s)"** — cosmetic but
  confusing leftover text from before the declare UI was generalized to all
  wild cards; fixed to a generic label.
- **Wild cards melded into a SET never got a `declaredAs`** — `valMeld()`
  returned immediately for sets (`if(trySet(cards)) return{valid:true,type:'set'}`)
  without ever running the declaration-assignment logic, which only existed
  in the run branch. This meant a Joker melded into a set could never be
  redeemed later (nothing to match a redeemer against — reported as "didn't
  ask me what I wanted the joker to be"), and a Deuce melded into a set
  silently scored at a wrong placeholder value instead of the set's actual
  rank. Fixed: `valMeld()` now assigns every wild in a set a concrete
  rank+suit (the set's rank + an unused suit) if it doesn't already have
  one, and `openMeld()` additionally prompts the human to confirm/override
  that suggestion specifically for Jokers (Deuces are handled silently since
  they're never redeemed and only need the rank to be correct for scoring).
- **CPU "calling cards" popup repeated every turn even with no real change**
  — `autoDecl()`'s stale-call invalidation check (`cp.hand.length>cp.declaredCount`)
  didn't tolerate the normal +1 card from that turn's draw, so right after
  every single draw it wrongly invalidated an otherwise-still-valid call;
  `shouldCall()` then immediately re-declared (and re-announced) the exact
  same count, producing a popup every turn instead of only when the call
  actually changed. This bug pre-dated this session but was invisible since
  `autoDecl()` was never called for CPU players until the "stale call count"
  fix above started calling it every turn — exposing the latent flaw. Fixed
  by only invalidating when the hand grows by *more* than one card (i.e. an
  actual abnormal pickup), not on the routine single draw.
- **No way to see the full hand-by-hand scorecard at game end** — the final
  "game over" screen only showed the end-of-game summary table (final
  totals), not the detailed per-hand history. Added a "View full scorecard"
  button on the final screen that opens the existing `showScores()` modal
  (the same one used mid-game via the Scores button), rather than building a
  second copy of that table.
