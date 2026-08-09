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

## Resolved — massive card loss across hands after someone goes out (2026-08-09-14)

**Reported:** after Poppy dreamed, no cards were visible for anyone
(including Poppy's own melded cards). The NEXT hand also showed empty hands
for everyone but the human (who had 1 card), the human was offered a pile
pickup on what should have been a fresh 14-card opening turn, and the deck
audit showed only 15/106 cards accounted for anywhere — draw pile, discard
pile, or any hand. ~90 cards had simply vanished. The feedback report's
stall log showed two CPUs (Plain and Poppy) stalled with `hand=0, drew=true`.

**Root cause:** the watchdog only ever skipped its stall-check for
`phase==='cut'` or `'over'`. But from the moment `endHand()` is called (the
instant someone goes out) until the NEXT hand's cut screen actually appears,
`G.phase` is still whatever it was during live play, and `G.currentIdx`
still points at whoever just went out — with an EMPTY hand. This window can
last as long as the human spends looking at the "X went out" pause (added in
2026-08-09-9) and the following scores screen, both of which are entirely
normal, unhurried moments — not a stalled turn.

If the watchdog fired during this window, it saw a "CPU" with no real turn
in progress and tried to recover it: with `G.drewCard` still stale-true from
the winner's actual last turn, the recovery path skipped drawing and called
`cpuDiscard()` directly on an EMPTY hand. That function's own empty-hand
fallback is `if(!toDiscard){endHand(G.currentIdx);return;}` — calling
`endHand()` a SECOND TIME for the SAME winner, re-entering the entire
scoring/dealing cascade while the first pass might still be pending, or
after the human had already moved past it. Two stalled players in the report
(Plain and Poppy) is consistent with this repeating as `currentIdx` got
shuffled around inside the broken recovery attempts.

**Fix:** new `G._handEnding` flag, checked alongside the existing
`phase==='cut'`/`'over'` early-return in `cpuWatchdogTick()`. Set at the very
first line of `endHand()` and `endHandDead()` — before anything else runs —
and cleared at the very first line of `dealHand()`, so the watchdog is
completely inert for the ENTIRE span from "someone went out" through both
the final-play pause and the end-of-hand scores screen, only resuming once a
genuinely new hand has begun forming. Verified directly: with the flag set,
45+ seconds of "staleness" produces zero watchdog action; once cleared for a
real new hand, normal stall protection resumes exactly as before.

---

## Resolved — adding a joker to an existing meld offered no/wrong suit choices (2026-08-09-13)

**Reported:** adding a joker to an existing meld didn't let the player
choose the suit.

**Root cause:** `wildAssignmentOptions()` cleared the `declaredAs` of EVERY
wild in the combined card set before enumerating — including wilds ALREADY
committed to the existing meld, which have no business being reassigned
(melds can never be rearranged; a wild permanently owns the position it was
played into, per the 2026-08-08-17 fix). Two failures resulted:
1. Reproduced directly: adding a joker to a run `8♠ 9♠ 2♣(=10♠, already
   committed)` generated THREE raw options, two of which required silently
   MOVING the already-fixed deuce (one even offered "10♠" as a choice for
   the joker — a straight collision with the deuce's own fixed position).
2. After a first fix pass correctly excluded the illegal options, only ONE
   legitimate option (J♠, extending high) was found instead of the actual
   two (7♠ extending low, or J♠ extending high) — because the run-branch's
   gap-filling loop assigned wilds to gaps strictly by ARRAY POSITION, so
   the fixed deuce (first in the array) was always tried against whichever
   gap came first spatially, rather than being matched to its own specific
   required rank wherever that fell in the range.

**Fix:** `wildAssignmentOptions(sel, fixedIds)` takes an optional set of
already-committed wild ids. Only non-fixed wilds have their declaration
cleared for enumeration. The run branch was rewritten to match every fixed
wild to its EXACT declared rank (via `declRank()`, mirroring the same-session
fix already applied to `tryRun()`) regardless of array position, then fills
remaining gaps with free wilds; a candidate start is only valid if every
fixed wild's rank actually falls within it. The set branch similarly now
only asks about non-fixed jokers and reserves a fixed wild's already-used
suit. `openAdd()` passes the existing meld's wild ids as `fixedIds`;
`openMeld()`/`cpuCommitMeld()` (brand-new melds, nothing pre-existing) are
unaffected — call with no second argument, as before.

Verified: the exact reported scenario now offers both legitimate options
(7♠, J♠), the deuce is never re-offered, both options commit correctly via
the real `commitAddToMeld` path with the deuce staying at 10♠, and a set
variant (joker added to a set with an already-fixed deuce) correctly offers
the one remaining unused suit. Regression-checked: brand-new melds
(`openMeld`) and deuce-only/joker-only set behavior unchanged.

---

## Resolved — final tally invisible after leaving the game-over screen (2026-08-09-13)

**Reported:** after leaving the end-of-game scoreboard, opening it again
didn't show the full tally (bonuses and settlement).

**Root cause:** the +200 end-game bonus and money-settlement breakdown were
only ever rendered by `showFinal()`, directly into the dedicated final
screen. `showScores()` — the general "Scores" modal reachable from anywhere,
including after leaving that screen — only ever showed the per-hand history
table. That information was simply never written anywhere else.

**Fix:** `showScores()` now appends the same bonus/settlement breakdown
whenever the game has been tallied (`G._tallied`, reusing the exact stored
`G._settlement`/`G._endBonusTo` values `showFinal()` already computed once —
no recalculation, so this can't double-charge anyone). Silent and unchanged
mid-game. (One self-caught slip while implementing: the section was
initially spliced INSIDE the per-hand table's own `<tbody>`, before its
closing tag, which is invalid HTML nesting — a `<div>`/nested `<table>` can't
legally sit inside a `<tbody>` without a `<tr><td>` wrapper. Moved outside
the outer table's close before shipping, and verified via direct nesting
inspection of the rendered output.)

---

## Resolved — call counts silently corrupted after EVERY discard (2026-08-09-12)

**Reported:** human called 3, next turn showed 2; a CPU showed "called 1"
while actually holding 2 cards; another CPU's call didn't appear until a
turn later than expected.

**Root cause:** `autoDecl()`'s correction formula, `should = cp.hand.length-1`,
is only correct when a discard is still UPCOMING (called mid-turn, e.g. right
after melding, before discarding — as in `confirmMeld`/`confirmAdd`, and the
pre-discard check at the top of `cpuDiscard`). But it was ALSO called from
`doDiscard()` and `cpuDoDiscard()` — both AFTER the card had already been
removed from hand. At that point `hand.length` already equals the true final
count for the turn, so subtracting 1 again silently double-decremented the
call on every single discard: a call of 3 became 2, a CPU's call of 2 became
1 while the player still visibly held 2 cards. This also explains the
"delayed" call — a CPU that DID call correctly mid-turn had it corrupted
before the next render, so the correct value was never actually seen.

**Fix:** `autoDecl(finalCount)` now takes an explicit parameter. Omitted
(mid-turn call sites, unchanged): defaults to `hand.length-1`, predicting the
count after an upcoming discard. Passed explicitly (all four post-discard
call sites, in `doDiscard` and `cpuDoDiscard`): callers now pass
`cp.hand.length` directly, since the discard has already happened and that
IS the final count — no second subtraction.

Verified against all three reported symptoms directly (call stays at 3 after
a matching discard; call stays at 2 matching an actual 2-card hand) plus a
regression check confirming the pre-discard/mid-turn correction (used when a
CPU or human melds further after already having called) still works exactly
as before.

---

## Resolved — regression from the previous fix: humans couldn't discard (2026-08-09-11)

**The 2026-08-09-10 guard broke ordinary play.** It re-armed `G._turnEnded`
only inside `runCPUInstant`/`runCPUTurn` — but a HUMAN's turn never runs
through either of those. So the very first CPU-to-human handoff left the
guard permanently latched from the CPU's turn, and every subsequent
`endTurnHO()` call the human made (i.e., every discard) was silently
blocked: the discard itself still happened (nothing gated that part), but
the turn never advanced, so clicking discard repeatedly kept discarding
extra cards with no visible error. This is also why the Summary
button/last-turn panel vanished — both live inside `renderLastTurn()`,
populated by `flushTurnLog()`, which lives inside the same blocked
`endTurnHO()` call.

**Fix:** re-arm the guard unconditionally, immediately after `currentIdx`
advances inside `endTurnHO()` itself — i.e., revert to the SIMPLE approach,
which works correctly for every player type including humans. This does
mean the boolean guard is no longer airtight against a genuinely
back-to-back SAME-TICK duplicate call (verified: two synchronous calls in a
row without any real turn processing between them will still both succeed).
That tradeoff is acceptable because it isn't actually needed: the diagnosed
race requires the watchdog to fire ASYNCHRONOUSLY after being blocked by a
long computation, and the true fix for that — `noteProgress()` firing at the
exact moment of discard, added in the previous build — already prevents the
watchdog from attempting a call at all once it finally gets to run, since it
sees freshly-updated progress. Directly verified: given a discard timestamp
that's fresh (as it now always will be), the watchdog's own logic produces
ZERO stall-log entries and takes no action; with the OLD (unfixed) stale
timing it correctly reproduces the original false trigger. So the two
defenses divide responsibility cleanly — noteProgress prevents the real
async race, the boolean guard is a lightweight backstop that no longer needs
to (and doesn't) survive a contrived same-tick double-call.

Lesson for next time noted in CONTEXT.md: a "defense in depth" fix that
changes normal-path behavior needs the normal path tested just as
rigorously as the bug path before shipping.

---

## Resolved — CPU turn freeze fully diagnosed: root cause found (2026-08-09-10)

**The round summary evidence (Poppy, Everything, Garlic each appearing
TWICE before the human's turn) pinpointed the exact mechanism.** JS is
single-threaded: a long SYNCHRONOUS computation (the opening turn's meld
search over a 14-card hand — the largest, most expensive hand size in the
game) blocks the watchdog's timer from firing AT ALL while it runs. The
moment that computation finally finishes, the discard has already happened
and `cpuAfterDiscard()` has already scheduled its own `endTurnHO()` via
`setTimeout`. But the watchdog — having been completely blocked the whole
time — gets its first chance to run right as control returns to the event
loop, sees a huge apparent "idle" gap (the entire blocking duration, since
nothing had updated the progress timestamp during it), and because
`cp._discardedThisTurn` is now true, calls `endTurnHO()` DIRECTLY. That
produces TWO pending calls to `endTurnHO()` for one turn — the watchdog's
immediate one and the originally-scheduled one — both of which fire in quick
succession, advancing `currentIdx` by 2 instead of 1 and silently skipping
whoever was next. Exactly matches the reported doubled CPU turns.

**Two-layer fix:**
1. **Root cause** — `noteProgress()` is now called the moment a discard is
   recorded (inside `cpuDoDiscard`), which happens synchronously as part of
   the SAME blocking computation. Since the watchdog cannot run until that
   computation returns control to the event loop, by the time it finally
   checks, the timestamp is already fresh — it correctly sees "not actually
   idle" and takes no action. A slow-but-successful turn no longer looks like
   a stalled one.
2. **Defense in depth** — `endTurnHO()` now refuses to advance `currentIdx` a
   second time for the same turn (`G._turnEnded` guard), re-armed only when
   real CPU turn processing genuinely begins (`runCPUInstant`/`runCPUTurn`),
   not merely after the previous turn ends — an earlier version of this
   guard reset itself immediately after succeeding, which reopened the
   window right away and was caught by testing before shipping. This
   protects against ANY future duplicate-call path, not just this specific
   race.

Verified by directly simulating the exact race (two `endTurnHO()` calls with
no real turn processing between them → second call blocked, nobody skipped)
and the legitimate case (a real turn processes, then ends → advances
normally). Also fixed in the same pass: a duplicate `cpuLog('discarded...')`
call in the human's first-discard path (cosmetic double-line in the summary).

---

## Resolved — stale call counts & invisible final settlement (2026-08-09-9)

- **A call didn't update when the card count changed.** `autoDecl()` compared
  `hand.length` against `declaredCount`, which is off by one: a call means
  "this many AFTER I discard". A player who called 3, then next turn drew to 4
  and melded one card back down to 3, sat at `hand.length === declaredCount`
  and so never updated — yet after discarding they held 2 while still
  advertising 3. Now compares `hand.length-1` (the true post-discard count).
  Also extended to human players, who had the identical problem.
- **The final money settlement was invisible, and could be applied twice.**
  The maths was correct (5¢ per 500 points, rounded down by default) but it
  silently folded into the Money column with no indication anything had been
  collected — and in close games where every gap is under 500 the correct
  result is genuinely 0¢, which looked like a bug. The final screen now shows
  a settlement breakdown (who pays whom, the point gap, how many 500s, the
  amount), states plainly when nothing changed hands and why, and calls out
  the +200 end-game bonus recipient. Separately, `showFinal()` MUTATES state
  (bonus + money transfers) and was reachable both automatically at 1,000
  points and via the "End game & tally" button — running it twice
  double-counted both. Now guarded by `G._tallied`.
  The status line also now names the points winner and the money winner
  separately, since the rules make money the actual win condition.

---

## Resolved — bagel chaser melded anyway; melding into a stuck position (2026-08-09-7)

- **"Everything" (the Dreamer) melded on the opening hand** despite being
  committed to the bagel line. The opening (first-discard) turn runs through
  `cpuMeld()`, which used `cpuFindBestMeld()` — a plain first-match search
  with NO bagel check and no persona scoring whatsoever. So the bagel intent,
  the meld scoring, the split preference and the wild-value logic were all
  bypassed on that turn. `cpuMeld()` now returns immediately when
  `isBagelChasing(cp)` and otherwise uses the scored `cpuFindOneMeld()`.
  `cpuFindBestMeld()` deleted so nothing reaches for it again.
- **A player could meld down to one card without being able to go out.**
  Reported: melded everything but the discard, discarded, and the hand simply
  continued — leaving an empty hand having achieved nothing. The existing
  guard only blocked melding ALL cards (leaving zero). But leaving exactly ONE
  is equally broken when you haven't called: discarding it empties your hand,
  which IS going out, and going out is blocked without a prior call. New
  shared `meldLeavesYouStuck(cp,count)` blocks both cases in `confirmMeld`
  and `confirmAdd`, and CPU meld selection filters candidates through it too.
  Correctly still ALLOWS melding to one card when the player has called on an
  earlier turn, or is on a bagel/dream hand (calling never applied).

---

## Resolved — hidden opening turn & free Perfect Cut (2026-08-09-5)

- **A CPU's turn showed a "🤖 X is thinking..." cover instead of the table.**
  `showHandoff()` handled CPU turns only inside the `allCPUMode` branch;
  every other CPU turn fell through to the pass-the-device cover, which shows
  nothing but card counts. Most visible on the opening turn of a hand, where
  the first player is often a CPU and its whole turn happened behind that
  screen. A CPU now NEVER gets the cover — its turn always plays out on the
  game screen (visibly, or instantly when "Show CPU cards" is off). The cover
  is reserved for passing between multiple humans, which is its actual job.
- **The cut defaulted exactly onto the Perfect Cut with 4 players.** The
  slider started at `deckLen/2` = 53, and with 4 players the perfect cut is
  13×4+1 = 53 cards below — the midpoint of a 106-card deck. So the default
  position WAS the bonus, handing out a free Check every hand if the optional
  rule was on. The CPU auto-cut was centred on the same value. New
  `suggestCutPos(deckLen,n)` randomises the starting position and explicitly
  steers at least 3 cards clear of the perfect cut. Verified over 20,000
  trials at 3, 4 and 5 players: never lands on it, while still varying.

---

## Resolved — melds could be illegally rearranged (2026-08-08-17)

**Reported:** Egg picked up the pile with 8♠ and added it to its meld
`2♣ 9♠ 10♠ J♠`, producing `2♣ 8♠ 9♠ 10♠ J♠`. The 2♣ had been standing in for
8♠; the addition silently re-declared it as 7♠ so the natural 8♠ could take
its place. Illegal — melds cannot be rearranged, and a wild owns its slot
permanently. The correct outcome is to reject the addition.

**Root cause (self-inflicted).** `commitAddToMeld` cleared the declarations of
EVERY wild in the combined meld before revalidating — including wilds already
committed. That clearing was added earlier this session to fix "stale wild
declarations after extending a run", but that reasoning was itself wrong: a
committed wild's rank never legitimately changes. The real cause of those
stale values was premature declaration assignment, fixed properly in
2026-08-08-14. The clearing then remained as pure harm.

**Fixes:**
- `commitAddToMeld` now clears declarations ONLY on the cards being added.
  Wilds already in the meld keep their rank permanently.
- `tryRun` reworked to place already-declared wilds at their fixed ranks
  FIRST, then fill remaining gaps with undeclared wilds. It previously
  assigned wilds to gaps in array order, which wrongly rejected legal
  additions — e.g. adding a fresh wild at 7♠ to a run whose committed wild
  sits at 8♠ failed purely because of card ordering. The Q♠ guard now applies
  to the final placement rather than mid-loop.

Verified: adding 8♠ to `2♣(=8♠) 9♠ 10♠ J♠` is rejected; adding 7♠ or Q♠ is
accepted with the deuce unmoved; adding a fresh wild is accepted and correctly
takes 7♠ while the committed deuce stays at 8♠. Q♠ guard and set/run
validation unaffected.

---

## Resolved — CPU took the pile without melding the top card (2026-08-08-16)

**Reported:** Egg picked up the pile but melded nothing — no top card, no
2 naturals from hand or table. A direct rules violation.

**Root cause — TWO flaws, confirmed by reproduction.**

*(1) The candidate search was illegal.* It pooled hand naturals and table
naturals into one list and tested pairs from that combined pool, so it could
"find" a meld built from cards locked inside DIFFERENT existing melds — cards
that can never leave to form a new meld. Reproduced on Egg's exact board
(melds `2♣ 9♠ 10♠ J♠`, `8♥ 9♥ 10♥`, `J♣ Q♣ K♣ A♣`): with **J♦** on top the
old search returned `J♦ + J♠[meld1] + J♣[meld3]` — a meld that was never
possible. The CPU took the pile on the strength of that phantom. This also
explains the phantom "J♦" in the 2026-08-08-15 report: the log line was real,
it just described a meld that could not exist.

*(2) An ordering bug.* `cpuDraw()` took the pile FIRST and only
then attempted to commit the meld. Worse, the two steps asked *different
questions*: `bestMeld` was validated as a STANDALONE meld (top card + 2
naturals), but when those naturals lived on the table the code then called
`commitAddToMeld` — i.e. "can the top card join this existing meld?", a
different validity test entirely. That routinely failed, and because a fix
earlier in this session had (correctly) stopped it force-merging invalid
melds, the failure path now simply left the cards in hand. Net effect: pile
taken, nothing melded. The earlier fix turned an illegal *meld* into an
illegal *pickup*.

**Fix — decide before touching any cards.** New `planPilePickup(cp,top)`
computes a plan up front and returns it only if it is guaranteed to commit:
  • `'extend'` — the top card joins one of our existing melds (that meld must
    already hold 2+ naturals, per the rules), verified with the new pure
    helper `canAddToMeld()`
  • `'new'` — the top card plus 2-3 naturals FROM HAND form a valid new meld
If no plan exists the CPU simply draws a fresh card instead. The pile is only
taken once a verified plan is in hand, then executed. A defensive
console.warn plus visible log line covers the now-unreachable failure case
rather than failing silently.

The pooling flaw is closed because a `'new'` plan now considers ONLY hand
naturals, and an `'extend'` plan only ever adds the top card to ONE actual
existing meld — cards can never be borrowed across melds.

Verified on Egg's exact board: top card J♦ now yields `plan = null`, so the
CPU draws a fresh card instead of taking the pile. Legitimate plans still
work (J♥ correctly extends the heart run 8♥ 9♥ 10♥), and every plan produced
commits successfully.

---

## Resolved — CPU summary omitted discards & could misattribute actions (2026-08-08-15)

**Reported:** Egg's summary claimed it added J♦ (Jack of diamonds) to a meld —
a card that appeared nowhere, in no hand and no meld — and separately omitted
that it discarded a 5♠.

**Missing discard — confirmed and fixed.** There are three CPU discard sites
and only ONE was logging:
  • `cpuDoDiscard()` main path — was logged ✓
  • `cpuDoDiscard()` missedDeclaration path — logged nothing ✗
  • `cpuDiscard()` all-specials go-out path — logged nothing ✗
So any CPU discarding via the latter two produced a summary with no discard
line at all. Both now log.

**Phantom J♦ — mitigated by attribution.** `G.cpuTurnLog` was a single shared
list of bare strings with no record of WHOSE turn produced each entry, and the
CPU zone rendered the whole list. Any entry surviving from another player's
turn would therefore be displayed as though the current CPU had done it —
which fits a summary naming a card that player never held. Entries are now
tagged with the acting player's id and the zone renders only its own via
`cpuLogFor(pid)`, so cross-player contamination is structurally impossible.
(Note: this was a plausible cause but not directly reproduced. If a phantom
action is ever reported again, it is now provably that player's own action.)

---

## Resolved — stale wild declarations leaking between melds (2026-08-08-14)

**Reported:** Egg's meld showed a joker declared as a DIAMOND while sitting in
a HEARTS run (`4♥ 2♠ 6♥ 2♠ ★JK`), and the turn summary described adding it to
a meld that didn't appear to exist.

**Root cause:** three CPU code paths stamped `card.declaredAs` onto wilds
*before* knowing whether that meld would actually be committed — leftovers
from before `commitNewMeld`/`commitAddToMeld` handled assignment:
  • `cpuDraw()` pile-pickup candidate (the likely culprit here)
  • `cpuMeld()` new-meld path
  • the gambler's bagel lay-down
If the candidate was then abandoned (pickup not committed, meld rejected), the
wild kept a declaration describing a run it never joined — while still sitting
in hand. When it was later melded into a genuinely different suit,
`valMeld(...,true)` skipped it, because it only assigns when `!declaredAs`.
Hence a diamond-labelled joker inside a hearts run.

**Fixes:**
- Removed all three premature assignments; the commit helpers do it, once, and
  only on success.
- `commitNewMeld` now clears and recomputes wild declarations before
  validating (matching `commitAddToMeld`), with a `keepDecl` id list so the
  human's explicitly tapped choice is preserved. Restores prior values if the
  meld turns out invalid.
- Safety net in `endTurnHO()`: a wild sitting in a HAND represents nothing, so
  any lingering declaration is cleared at end of turn (skipped while a
  redeemed joker is pending, since the rules say it retains its declaration).

Verified: a joker carrying a stale `J♦` now recomputes to `7♥` when melded into
a hearts run; an explicit human choice survives; failures roll back cleanly.

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
