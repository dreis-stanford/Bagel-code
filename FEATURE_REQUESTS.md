# Bagel — Feature Requests

Desired features and enhancements, separate from bug fixes (see
`DEFERRED_BUGS.md` for those). Nothing implemented here yet — this is a
holding pen until we're ready to scope and build each one.

---

## Pending — scoped, ready to design/build next session

### 1. Streamline declaring Jokers and Deuces

**SHIPPED (build 2026-08-08-6)** — replaced free-text entry with a computed
tappable picker. `wildAssignmentOptions(sel)` enumerates every LEGAL
assignment for the wild(s) in the current selection (all valid run positions,
or all permissible suits for a set), automatically excluding anything that
would put a wild on Q♠. The dialog shows those as tappable card chips with
the option implied by the player's CARD ORDER preselected — so the common
case is now just pressing "Confirm meld" with zero typing, and the player can
never enter something illegal or misspelled. When only one legal option
exists the dialog says so. Verified: Q♥K♥+deuce offers exactly J♥/A♥
(defaulting to A♥); 10♠J♠+deuce offers only 9♠, never Q♠; J♠K♠+deuce
correctly offers nothing; two-wild runs enumerate combined options
(3♥+4♥ | 4♥+7♥ | 7♥+8♥). All 10 offered options across test cases commit as
valid melds with no wild ever landing on Q♠.

Remaining in this area: the ADD-to-existing-meld dialog still has no picker
(it infers the wild's value), and `normDecl()` is now unused by the meld flow.

#### Original notes
Companion to the design-revisit note in `CONTEXT.md`'s Known Issues. The
current flow accreted incrementally and now has inconsistent rules across
contexts (silent auto-detect vs. ambiguous-prompt vs. suggested-default vs.
required input, differing between Jokers and Deuces and between runs and
sets). Goal: redesign as one coherent, predictable flow rather than several
special cases. Revisit `openMeld()`/`confirmMeld()`/`valMeld()` together.
*Awaiting user's concrete before/after examples to scope the actual target
behavior.*

### 2. Easter egg: "The Afikomen" (Grandma Goldie)

**Renamed from "Grandma Goldie" to "The Afikomen"** — named for the hidden
piece of matzah in the Passover seder, which fits Bagel better and plays on
the running joke that bagels aren't kosher for Passover.

**NOTE — content lost once already.** An earlier round of sayings/details was
discussed in a prior session but never got written into this file, and
turned out to be unrecoverable (existed only in that session's chat, not in
any project file). Redone from scratch below. **Lesson: write content into
this file as soon as it's given, not after a "let's finalize this later."**

**Trigger — SHIPPED (build 2026-08-09-16, refined 2026-08-09-19 and
2026-08-09-20).** `isGoldieName(name)` matches, case-insensitively: "Goldie"
alone; "Grandma Goldie" / "Grandma G" / "Grandma G."; the "Gramma" spelling
of the same; and "GG" (her initials) alone or paired with
Goldie/Grandma/Gramma. Does NOT match near-misses (Golda, Goldman, Grandpa
Goldie, bare "Grandma" or "G", "GGG", "G G"). Granny/Nana variants were
removed in 2026-08-09-19 after the user clarified she wasn't called
either — those are different words for a grandmother in general, not
variants of what THIS Grandma Goldie was actually called, and keeping them
risked matching an unrelated player's own grandmother by coincidence.
Checked only for HUMAN players, at game start, setting `player.isGoldie=true`
(case normalization means any capitalization of any matching form works).
Verified against 28 cases (19 matches across case variants, 9 correct
rejections).

**Content bank scaffolded, EMPTY — still needed from the user:**
`AFIKOMEN_SAYINGS` in index.html has keys for `goOut`, `bagel`, `dream`,
`joker`, `win`, `loss`, `general`, each meant to hold a list of things
Grandma Goldie used to say at that moment. `afikomenSaying(event)` picks a
random one from the matching key (falling back to `general`). Nothing is
wired to actually CALL `afikomenSaying()` from game events yet — that's the
next step once there's content to surface, so wiring can be tested against
real sayings rather than placeholders.

**Sayings SHIPPED (build 2026-08-09-17)**, each wired to a real game
moment, not just stored as text:

- **"Deal in Jail"** — fires when Goldie is dealt a lousy opening hand.
  `isRoughHand()`: high average in-hand penalty per card (≥7) AND few
  near-melds available (pairs, adjacent-suit cards). Checked once, right
  after dealing.
- **"You Schmeissed me"** — fires when someone ELSE goes out and Goldie is
  left holding ≥100 points of penalty in hand. Checked in `endHand()` before
  hands get cleared for scoring.
- **"Garbage picker"** — fires when ANOTHER player picks up a pile that's
  either topped by a low-value card or is a genuinely scattered mismatch
  (many different ranks AND many different suits — NOT just "3+ suits",
  which would wrongly flag an ordinary same-rank set). Wired at all three
  pile-pickup commit sites (human `confirmPickup`, both CPU pickup paths in
  `cpuDraw`).
- **"Points is points"** — the SAME garbage-pile heuristic, but said about
  Goldie's own pickup when SHE does it; also fires when Goldie melds a
  low-value set/run (≤15 pts) at ANY point in the hand (no longer limited to her first two turns), per
  "or melds small cards when she could have waited" and the user's
  follow-up that this shouldn't be limited to early turns.
- **"Who's turn is it" / "Play already"** — impatience when the game has
  been idle ≥25s and it is NOT Goldie's own turn (reacting to someone else
  being slow, not her own thinking time). A 20s cooldown prevents spamming.
  Separate lightweight interval, piggybacking on the existing
  `G.lastProgressTs` tracking; wrapped in try/catch so it can never affect
  real gameplay even if something about it misbehaves.

Display: a small toast (`#goldie-quip`, sidebar) reading `👵 "<line>"`,
auto-hiding after 5 seconds. Non-blocking — never gates play the way
warnings/errors do.

Heuristics were bugged on first pass and caught by testing before shipping:
the garbage-pile detector originally flagged any pile touching 3+ suits as
"mismatched," which incorrectly caught ordinary same-rank sets (a set of
Kings necessarily spans multiple suits) — fixed to require BOTH scattered
ranks AND scattered suits together.

**Still open:**
1. Confirmation the "100th birthday" deck (corrected from an earlier "80th" guess — the actual photo is dated 2015, "Celebrate 100!") is a cosmetic skin only (default
   assumption) rather than a different card composition — not yet built.
2. All five heuristic thresholds (7 avg-penalty, ≤2 near-melds, 40pt stuck
   threshold, ≤15pt meld, 25s/20s impatience timing) are first-guess tuning,
   not validated against real play — expect to adjust once these are seen
   firing (or not firing) in an actual game.
3. **RESOLVED (build 2026-08-09-21).** Goldie can now play as a CPU. When
   her name is typed into a setup slot, an inline prompt appears asking
   Human or CPU (`checkGoldiePrompt`/`setGoldiePlayMode`, fires on input,
   only while unresolved). Choosing CPU selects a hidden `cpu:goldie`
   dropdown option; `startGame()` special-cases it: she keeps her own typed
   name (never renamed to "Everything" the way an ordinary CPU-character
   pick would), and borrows Everything's persona weights as a starting
   point per the user — "make her play like Everything for now... maybe
   later we'll give her her own strategy." `isGoldie` is now purely
   identity-based (`isGoldieName(name)`, no longer gated on being human), so
   all five Afikomen quips work identically whether she's played by a human
   or the CPU. Verified: she keeps her name, resolves to type `cpu`, gets
   the Dreamer archetype's weights, and a plain human player elsewhere is
   unaffected.

**Intro popup — SHIPPED (build 2026-08-09-18).** The user's photo of Grandma
Goldie holding a custom "Celebrate 100!" deck (Sept 5, 2015 — corrects the
earlier 80th-birthday guess) is now embedded directly in index.html as a
base64 data URI (resized to 900px max, ~118KB, so the single-file deploy
model keeps working with no separate asset upload needed). Shows once,
right when `startGame()` confirms a Goldie player is actually in the game —
not on every hand, just that first moment of discovery. Dismissible via
"Deal me in."

**Still needed:** more images (user mentioned more may come later — the
photo is saved at `assets/goldie-popup-1.jpg` in the working session, not
yet a gallery/rotation, just the one). Also still need actual card face/back
art for the "100th birthday" deck skin itself — the photo shows her holding
the custom deck at an angle, which isn't usable as print-ready card art
directly; either a flatter photo of a single card, or a description of the
design to build a similar-spirited digital version, would unblock that.

### 4. UI redesign — make it look like a real card game (DEFERRED)

Current UI is functional but clunky and doesn't read as a card game. Decision
deliberately deferred; options preserved below so we can pick one up later.

**The core difficulty is melds, not seating.** A round table works well for
≤4 players and gets awkward at 5 — but melds are what actually breaks table
layouts at any player count: they grow unboundedly, and a player with five
melds needs far more room than a fixed seat can give.

**Option A — Seats + shared meld area.** Players around the edge showing only
name, card count, status. ALL melds live in a central zone, colour-coded or
badged by owner. Handles 5 players cleanly and scales with meld count.
Trade-off: loses the immediate visual "these are *her* melds" grouping.

**Option B — Seats with expandable melds.** Each seat shows a compact summary
("3 melds, 85 pts") that expands on tap. Keeps ownership obvious and the
table tidy; costs a tap to inspect an opponent.

**Option C — Focus view.** Your hand and melds prominent at the bottom;
opponents a compact strip you tap to bring into focus. Least "card table"-ish,
but most robust on a small screen and least likely to break at 5 players.

**Recommended method: a separate FORK, not an option flag.** A real layout
change touches nearly every render path; maintaining two layouts behind a
toggle would roughly double the surface area for exactly the class of bug
we've spent this project chasing. Fork it, play both, keep the winner.

**Recommended timing: AFTER the CPU memory model (item 3).** The CPU work is
confined to decision logic and won't conflict with layout. Doing an AI change
and a UI rewrite simultaneously would make regressions hard to attribute.

**QUESTION TO ASK THE USER BEFORE STARTING:** is the primary device the iPad,
or should this also work well on a phone? That one answer eliminates roughly
half the design space (notably, Option C becomes much more attractive if
phone matters; Options A/B are more natural if it's iPad-first).

Related smaller item: the in-game UI still says "Draw" / "Draw pile" rather
than the "fresh card" / "deck (stack)" vocabulary standardised in RULES.md.
Worth folding into whichever redesign happens.

---

### 3. Refine CPU strategy realism + add more CPU player types

**BAGEL APPETITE FIX (build 2026-08-09-6)** — user suspected CPUs melded too
eagerly and rarely held out for a bagel, and wondered whether they'd simply
under-sampled. Measured it: the behaviour was *binary*, not under-sampled.
`isBagelChasing()` used a hard threshold (`bagelAmbition > 0.7`), so only
Everything (0.95) ever qualified — and it then chased on 100% of hands, while
Garlic (0.60), Poppy (0.45) and everyone else chased on 0%. A second hard gate
in `cpuFindOneMeld` repeated the same threshold.

Fixed by making the decision a per-HAND roll with probability equal to
`bagelAmbition` (`rollBagelIntent()`, called for every CPU at deal). Risk
appetite now varies both between characters and from hand to hand. Measured
over 1500 hands: Plain 4%, Egg 6%, Cinnamon-Raisin 20%, Onion 24%,
Pumpernickel 34%, Poppy 44%, Garlic 60%, Everything ~100% (its 0.95 ambition
plus the strong negative meld score keeps the Dreamer almost always dreaming).
The choice is sticky within a hand and clears once the CPU melds, since a
bagel is impossible after that.

**STRATEGY REFINEMENTS (build 2026-08-09-2)** — three weaknesses reported
from real play, all fixed:
- *Two-card trap.* CPUs kept leaving themselves exactly 2 cards. That is a
  dead spot: next turn you draw one, hold three, and must discard one and play
  the other two — but two cards can never form a meld alone (minimum 3), so
  BOTH must be addable to existing melds or you cannot go out at all,
  whatever you draw. `twoCardTrapPenalty()` now penalises a discard leaving 2
  cards: 0 if both remaining cards extend a meld, 30 if one does, 55 if
  neither. One card and three-plus are unpenalised.
- *Over-long runs.* CPUs built one long run where two shorter ones were
  available. Two 3-card runs give four ends to extend; one 6-card run gives
  two. `scoreMeld` now adds +18 for a 3-card meld — deliberately more than the
  extra card's face value, otherwise a 4-card meld TIES on score and the
  preference becomes an accident of enumeration order. Measured: Shark splits
  100%, Grinder 98%, Scatterbrain 84% (personality still differentiates).
- *Wasteful wild placement.* A deuce scores whatever it stands in for, so with
  a free choice it should be an Ace (20) not a 4 (5). `tryRun` returns the
  first valid arrangement, which is arbitrary in value terms. New
  `cpuCommitMeld()` enumerates legal placements via `wildAssignmentOptions()`
  and picks the highest-scoring one before committing. Verified: 7♥ 8♥ + deuce
  now takes 9♥ (10pts) over 6♥ (5pts); Q♥ K♥ + deuce takes A♥ (20) over J♥ (10).

**STAGE 1 SHIPPED (build 2026-08-08-3)** — scoring/selection framework +
personalities. Still TODO: **Stage 2, the memory model** (`memDepth`,
`memRetention`, `memCorruption` traits are defined on each persona but NOT
yet consulted — CPUs currently read game state directly). Also still TODO:
personality-aware pile-pickup (`pileGreed` is defined but `cpuDraw` still
uses the old greedy logic) and gambler-path unification.

Architecture built:
- `PERSONAS` — eight named archetypes, each a weight vector. Tied to NAMES so
  the family gets to know the characters, with ±5% per-game jitter so they're
  recognizable but not perfectly predictable. Deliberately NOT surfaced in
  the UI — discovered through play.
  Egg=Grinder, Cinnamon-Raisin=Vulture, Everything=Dreamer, Onion=Hawk,
  Pumpernickel=Scatterbrain, Plain=Plodder, Poppy=Shark, Garlic=Improviser.
  ("Goldie" is reserved for HUMAN players and triggers the easter egg — it is
  not a CPU persona.)
- `pickByScore(items,scoreFn,temp)` — softmax selection. `temperature` is the
  difficulty dial AND the variety source: a weak player isn't scripted to
  blunder, it just has fuzzier judgement. Verified response: temp 0.12→0%
  suboptimal, 0.95→17%, 1.30→25%.
- `scoreMeld` / `scoreDiscard` replace the old first-match and
  highest-penalty-wins logic.
- `cardUsefulness` — near-meld detection (pairs, adjacent suits, gap-bridging,
  extends-own-meld) so CPUs stop discarding cards that nearly complete a run.
- `discardDanger` — weighs whether a discard feeds an opponent, doubled for
  players who have called.
- `callDiscipline` — low-discipline personas genuinely forget to call and eat
  the real rules penalty, which reads far more human than a random bad move.

Measured differentiation (1000 trials, choice between a 14.5-pt and 29.5-pt
meld): Shark 100% optimal, Grinder/Hawk 99%, Plodder 89%, Scatterbrain 84%,
Improviser 77% (+14% declines), Dreamer declines 96% hoarding for a Bagel.

#### Original notes
Currently only two CPU archetypes exist: Conservative (melds greedily, smart
pile pickup, discards highest penalty) and Gambler (holds for bagel, bails
under specific conditions). Two threads here:
- **Refine existing strategies** — make Conservative and Gambler play more
  realistically (e.g. smarter discard risk assessment, better awareness of
  what opponents might need/be holding back for, more nuanced calling
  decisions, gambler bail-condition tuning — this last one is already flagged
  separately in `CONTEXT.md` known issue #2).
- **Add new CPU archetypes** — e.g. a more aggressive/defensive blocker type,
  a card-counting type, a "balanced" hybrid, etc. Needs the user's input on
  what archetypes/personalities they want and what distinguishes their
  decision-making (draw/pickup logic, meld timing, discard risk tolerance,
  calling behavior, joker redemption usage — currently NO CPU type uses
  redemption at all, see `CONTEXT.md` known issue #8).
*Awaiting user's examples/preferences for new archetype personalities and
which specific behaviors feel "unrealistic" today before scoping changes.*

---

## Pending — awaiting examples/details

*(Reserved for future items not yet scoped.)*
