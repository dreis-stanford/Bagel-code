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

### 2. Easter egg: "Grandma Goldie" / the Afikoman
If a player is named Goldie, Grandma, Grandma Goldie, etc. (case-insensitive,
likely a fuzzy/substring match rather than exact), trigger a special mode in
her honor — playful nod to the Afikoman (the hidden piece of matzah in the
Passover seder), acknowledging the joke that bagels aren't kosher for
Passover. Pieces to design/build:
- **Name-detection trigger** — needs a matching rule (exact match list vs.
  substring vs. fuzzy) decided with the user; likely checked at player setup
  time (`setPC`/name entry).
- **Special sayings** — a bank of "things Grandma Goldie used to say,"
  surfaced contextually (e.g. on specific game events — going out, drawing a
  joker, getting bageled, winning, etc.). Need the user to supply the actual
  sayings and ideally which circumstance each belongs to.
- **Special "80th birthday" deck** — a themed/alternate deck skin tied to
  this mode. Needs scope clarified: is this purely cosmetic (custom card
  back/face art or color theme) or does it imply different deck composition?
  Default assumption until told otherwise: cosmetic skin only, same 106-card
  structure and rules.
*Awaiting user's specific sayings, trigger circumstances, and confirmation
on deck scope before implementation.*

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
