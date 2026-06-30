# Bagel — Feature Requests

Desired features and enhancements, separate from bug fixes (see
`DEFERRED_BUGS.md` for those). Nothing implemented here yet — this is a
holding pen until we're ready to scope and build each one.

---

## Pending — scoped, ready to design/build next session

### 1. Streamline declaring Jokers and Deuces
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

### 3. Refine CPU strategy realism + add more CPU player types
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
