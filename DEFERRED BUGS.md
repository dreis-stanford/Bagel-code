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
the test was actually run against the pre-fix code. **Update: this exact
caution was independently confirmed a few sessions later** — a "long
feedback message" test came back showing behavior that only matched the
pre-fix code (full untruncated body, no length-check ever firing), even
though the length-check fix had already been delivered. The user's own
conclusion was "maybe it was a cached version that just took a really long
time to replace." So: stale caching of `index.html` (browser-side and/or
GitHub Pages propagation delay) is a real, recurring source of "still
broken after the fix" reports in this project, not just a theoretical
caveat — treat it as the FIRST thing to rule out whenever a just-fixed bug
appears to resurface. First step next session:
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

## OPEN — CPU turn froze (reported, no repro steps)

**Reported:** during a hand, "everything got stuck on a computer player's
turn that it didn't take." Circumstances not captured.

**Mitigation shipped (build 2026-08-08-2), root cause NOT yet confirmed:**
a watchdog (`cpuWatchdogTick`, 2s interval) now detects a CPU turn making no
progress and recovers it, so a freeze no longer ends the hand. It also
records what it saw into `G.stallLog`, which is now included in the Feedback
report. **Next time this happens, hit Feedback — the report will name the
stall kind, the player, the phase, and how long it hung.** That should
identify the cause without needing repro steps.

Two candidate causes were found by inspection:
1. `awaitingCPUAck` set to true while the Continue ▶ button doesn't render
   (it requires `isActive && isCPU && G.awaitingCPUAck` simultaneously) — the
   game would then wait forever for a button nobody can press. Watchdog
   handles this specifically by checking whether the button is in the DOM.
2. A dropped link in the chained `setTimeout`/`alive()` callbacks that drive
   a CPU turn — any escaped exception or mistimed guard leaves nothing to
   advance the turn. Watchdog re-drives `runCPUTurn()`.

Also fixed alongside (definite bug, may or may not relate to the freeze):
**`cpuMeldAsync` removed melded cards from the CPU's hand even when
`pushMeld` had rejected the meld** — silently destroying cards (deck audit
would report them missing). Worse, `cpuFindOneMeld` would keep proposing the
same rejected combo, so the retry loop could spin. Now cards are only removed
on a successful `commitNewMeld`, and a rejected proposal falls through to the
add-to-existing pass instead of retrying.

---

## Resolved — corrected a bad "fix": wild suits in SETS (2026-08-08-13)

Self-inflicted. In 2026-08-08-12 I claimed to fix a "collision" where a deuce
could be auto-assigned the same suit a joker had been declared as. **There was
no such bug**, and the change I made to prevent it broke a legal meld.

Why there was nothing to fix:
- Neither a joker nor a deuce has a suit of its own.
- For a DEUCE, `declaredAs` in a set is meaningless anyway — scoring reads
  only the rank ("Deuces score the value of the card they stand in for"), so a
  deuce in a set of 7s scores 5 whatever suit is stamped on it.
- Sets explicitly permit repeated suits (rules: "3♣ 3♣ 3♦ is valid"), and the
  deck holds TWO of every card — so a repeated suit is never illegal.

What the bad fix broke: reserving declared wild suits made `valMeld` reject
`Q♥ Q♦ + joker(Q♣) + deuce` — a legal 4-card set (2 naturals, 2 wilds) — with
"No valid suit left for the wild in this set."

Corrected behaviour:
- **Deuce in a set** now records the RANK ONLY (`declaredAs='7'`), with no
  meaningless suit. Scoring verified unchanged (5 for a 7-set, 10 for Queens).
- **Joker in a set** still gets a concrete card (redemption needs it), but an
  unused suit is now only a *preference* — it falls back to a repeated suit
  rather than failing. Q♠ remains the sole forbidden choice.
- Same fallback added to the tappable picker.
- Consequence: `Q♥ Q♦ Q♣ + joker` is now VALID with the joker as Q♥. This is
  correct; an earlier test asserting it should be rejected was itself wrong.

---

## Resolved — meld dialog cleanup (2026-08-08-12)

- **Deuce in a SET no longer asks what it stands for.** In a set the rank is
  already fixed and a deuce is never redeemed, so the suit choice changed
  nothing — the prompt asked the player to decide something meaningless.
  `wildAssignmentOptions()`'s set branch now only enumerates for JOKERS
  (which do need a declaration, since a redeemer must produce that exact
  card). Deuces fall through to `valMeld(...,true)`'s silent auto-assignment.
  Preserved: joker-in-set still prompts, deuce-in-RUN still prompts (its rank
  genuinely varies), and a mixed joker+deuce set asks only about the joker.
  (An earlier note here claimed a "suit collision" was also fixed. That was
  wrong and is corrected in the 2026-08-08-13 entry above.)
- **"Add to meld" now lists only melds the cards can legally join.** It used
  to render every meld, with rejected ones showing a red ✗ and a reason, so
  the dialog filled with unusable options. Now filtered to targets where
  `v.valid && v.type===m.type` (matching what `confirmAdd` enforces). When
  nothing qualifies it shows a single clear message instead of an empty
  dialog.
- **The add dialog gained the wild picker** the create-meld dialog got in
  2026-08-08-6. Adding a wild to an existing run is genuinely ambiguous (it
  can extend either end); the dialog now shows tappable options per meld when
  more than one placement is legal, and `confirmAdd` applies the choice.

---

## Resolved — CPU turn summary gaps (2026-08-08-11)

- **Melds were missing from the CPU turn summary.** Only 2 of 8 CPU meld
  sites called `cpuLog()` — the two in `cpuMeldAsync`. Everything else was
  silent: the synchronous `cpuMeld()` (which is what runs on the opening
  hand), all four pile-pickup meld paths in `cpuDraw()`, and the gambler's
  bagel lay-down. So the summary reported drawing, calling and discarding but
  never what was actually played to the table. All sites now log, including
  add-to-existing-meld.
- **The opening turn was invisible.** `cpuAfterDiscard()` explicitly excluded
  `phase==='first-discard'` from the acknowledgement pause, so when a CPU
  held the 14-card opening hand it melded and discarded silently before the
  human's first look at the table — no summary, no Continue. It now pauses
  like any other CPU turn. (The exclusion originally guarded against a hang;
  the stall watchdog added in 2026-08-08-2 now covers that risk.)
- **The opening turn also reset its log too late.** `runCPUInstant()`
  returned into the first-discard branch *before* reaching `cpuLogReset()`,
  so that turn either logged nothing or carried stale entries from a previous
  turn. Reset now happens first.

---

## Resolved — hand arrangement & pile-pickup ordering (2026-08-08-9)

- **Pile pickup listed your naturals in the wrong order** — `startPickup()`
  built its list from the raw `cp.hand` array (deal/draw order) rather than
  `cp.handOrder`, so the cards you were choosing from appeared in a different
  order than they sit in your actual hand. Now ordered to match, with any
  stragglers appended.
- **Opening hand is now pre-arranged** — `smartHandOrder()` groups probable
  melds (sets first, then runs in ascending order), tidies the remaining
  singles by suit and rank, and parks wilds at the end as reserves. Replaces
  the old flat wilds→suit→rank sort. Purely a starting point; drag and
  "Reset order" still work. Example: dealt `9♠ 4♥ 2♦ K♦ 9♥ 6♠ 4♣ J♦ ★JK 7♠
  9♦ 4♦ Q♦` → arranged `9♠ 9♥ 9♦ | 4♥ 4♣ 4♦ | J♦ Q♦ K♦ | 6♠ 7♠ | 2♦ ★JK`.
- **Clarified that pile pickup CAN extend an existing meld** — it always
  could (naturals from your table melds are offered alongside hand cards),
  but the confirmation step only mentioned it in small grey print. Step 2 now
  states plainly that the top card will be added to the existing meld rather
  than played as a new one.
- Picked-up cards continue to append at the end of the hand, by preference —
  easy to spot. (An affinity-based auto-insert was built and then removed as
  unwanted.)

---

## Resolved — duplicate character selection & CPU hand interaction (2026-08-08-8)

- **Same character selectable in two slots** — follow-up to the 2026-08-08-7
  fix, which stopped RANDOM picks colliding but still let the user manually
  choose e.g. Poppy in two dropdowns. `onTypeChange` now bounces any other
  slot already holding that character back to "Random CPU" and renames it, so
  a duplicate can't be created at all. (Two same-named CPUs also shared a
  persona, so a table could unknowingly contain two Sharks.)
- **Revealed CPU hands were fully interactive** — with "Show CPU cards" on,
  an active CPU's hand fell through to the human branch, so it rendered with
  drag/drop handlers, tap-to-select, and a "Hold & drag to reorder" hint. You
  could reorder a bot's hand mid-turn, which is meaningless and risked
  desyncing its `handOrder` against `hand`. CPU hands now render face-up but
  strictly read-only, labelled "<name>'s hand (revealed)". Human hands remain
  fully interactive.

---

## Resolved — cut reveal & duplicate CPU names (2026-08-08-7)

- **Perfect cut revealed before committing** — `updateCutSlider()` lit the
  "✓ Exact cut!" badge live as the slider moved, so a player could simply
  hunt for the exact position and collect the Check for free, defeating the
  bonus entirely. The badge no longer appears during adjustment; the result
  is disclosed only after `confirmCut()`, which already announced it.
- **Two players could be assigned the same character** (reported feedback
  showed two "Poppy"s at one table). `_usedCPUNames` only tracked names it
  had generated itself, so explicitly selecting a character left that name
  looking unclaimed and a later "Random CPU" could hand out a duplicate.
  Replaced with `namesInUse()`, which reads the live setup inputs (both typed
  names and explicitly-selected characters), making dedup self-correcting.
  `startGame()` additionally rerolls a persona that collides with one already
  claimed and syncs displayed names to the resolved characters. Verified over
  300 trials of the exact failing scenario: zero duplicates.

---

## Resolved — CPU announcements & turn visibility (2026-08-08-5)

- **CPU declarations announced twice** (joker declarations, calling cards,
  redemptions). Callers were BOTH pushing onto `G.jokerNotifications` (marked
  seen only by the acting player) AND immediately popping a modal. In
  single-human mode the human watches CPU turns live, so they saw the modal
  during the CPU's turn — then `checkJokerNotifications` fired it a second
  time on their own turn, since the queue entry was still unseen by them.
  Fixed with a single `announceEvent(msg, actingIdx)` entry point that is
  mode-aware: in single-human mode it shows the modal once and marks the
  entry seen by everyone (can't replay); in all-CPU watch mode it stays
  silent; in pass-and-play it shows for the player at the device and stays
  queued for the others. Verified: single-human 1 modal (was 2), watch mode
  0, pass-and-play 1 live + 1 per other player on their turn.
- **CPU turns were an unexplained blank "Continue"** — the human had no idea
  what the bot had done. Added `cpuLog()` narration recording each action
  (drew from deck / picked up the discard pile with count / melded <cards> /
  added <card> to a meld / called N cards / discarded <card>), reset at the
  start of every CPU turn and displayed as a readable summary panel directly
  above the Continue button.

---

## Resolved — rules-compliance audit (2026-08-08)

Systematic pass comparing `index.html` against `RULES.md`. Found and fixed:

- **Wild substituting for the Queen of Spades in a SET** — the set-branch
  auto-declaration picked "first unused suit" from `SUITS=['♠',...]`, so a
  set of Queens (Q♥ Q♦ + wild) silently stamped the wild as **Q♠**, breaking
  "A wild cannot substitute for the Queen of Spades" / "A joker may never be
  declared as Q♠". Now excluded explicitly; if Q♠ is the only suit left the
  meld is rejected.
- **Wild-on-Q♠ in a spades RUN failed the whole meld instead of backtracking**
  — `tryRun()` returned the first arrangement it found; if that put a wild on
  the Q♠ slot, `valMeld` rejected the entire meld even when a legal
  alternative placement existed. This contradicted the rules' own example:
  "If a wild is at the end of a run and could validly occupy a non-Q♠
  position instead (e.g. 10♠–J♠–wild, where the wild can be 9♠), the meld is
  allowed." The Q♠ constraint now lives inside `tryRun`'s assignment loop, so
  it naturally falls through to the valid alternative.
- **`valMeld()` mutated `declaredAs` as a side effect, but was called for
  previews** (17 call sites: Add dialog rendering, CPU combination searches,
  pickup eligibility...). Merely PREVIEWING a wild against one meld
  permanently stamped it, which then (a) made other legal melds display as
  invalid, and (b) could record a joker with the wrong suit — breaking
  redemption, since the redeemer must match the declaration. `valMeld` is now
  pure by default; only `commitNewMeld`/`commitAddToMeld` pass `assign=true`.
- **200-point end-game bonus went to the wrong player** — awarded to the
  highest scorer over 1,000; rules say "the player who went out on the final
  hand," which is often someone else. Now tracked via `G.finalHandWinnerIdx`.
- **Check bonuses contradicted the rules** — "All Below"/"All Above" require
  only that *naturals* fall in range ("and/or wilds" per the rules) but the
  code required `!hasWilds`, making them nearly unearnable. "No Wilds" was
  also wrongly excluded for Bagel/Dream hands despite the rules stating "A
  Bagel hand can also earn a No Wilds check." Both corrected, plus a guard so
  an empty meld set can't vacuously satisfy both via `[].every()===true`.
- **Nothing prevented melding your entire hand** — rules require always
  ending the turn with a discard ("you cannot go out by playing your last
  card to a meld"), and melding everything also soft-locked the turn since
  discard then had no card available. Guarded in both `confirmMeld` and
  `confirmAdd`.

Verified correct, no change needed: card point values, going-out +100,
Bagel/Dream mutual exclusivity and payouts (10¢ vs 5¢), final settlement
(5¢ per 500, round-down default / round-nearest option), set size 3–4,
aces-high-only, 2-natural wild minimum, pile-pickup excluding wilds from the
2-natural requirement, add-to-own-melds-only, dead-hand handling, and Perfect
Cut (the two push sites are mutually exclusive CPU/human paths, not a double
award).

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
