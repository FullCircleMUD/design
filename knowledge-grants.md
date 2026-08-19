# Knowledge Grants

How characters acquire spells and recipes they did not buy, find, or transcribe. Some knowledge is
**earned** — a mage transcribes a scroll, a smith buys a recipe — and some is **granted**: it follows
automatically from skill mastery, and every character who holds that mastery is entitled to it. This
document defines the single grant engine that keeps granted knowledge correct: what auto-grants, when
the entitlement is reconciled against a character, and how the same mechanism serves character
creation, mastery training, and content added to the game after a character already exists.

## The two acquisition models

Every spell and every recipe reaches a character by exactly one of two routes.

**Learned** knowledge is acquired by a deliberate act and is permanent. A mage transcribes a spell
scroll NFT; a crafter consumes a recipe NFT or buys one from a trainer; a character picks one BASIC
recipe per general skill during creation. Learned knowledge survives remort — the character paid for
it, and the mastery it requires is incidental to keeping it.

**Granted** knowledge is a consequence of mastery, not an acquisition. A cleric who reaches SKILLED in
Divine Healing *is* a cleric who can cast Purify; there is no scroll, no gold, and no separate step.
Because the entitlement derives entirely from mastery, granted knowledge is lost when the mastery is
lost — remort clears it along with the mastery that justified it.

The distinction is what makes a single engine possible. Granted knowledge is a **pure function of
character state**: given a character's mastery, the set of spells and recipes they are owed is fully
determined. Nothing needs to remember which grants were issued, or when. The engine's whole job is to make
storage agree with that function.

## What grants

Two independent rules decide whether something auto-grants, because the two subsystems key off
different things.

**Spells grant by class.** A character class declares `grants_spells` (`CharClassBase`). Classes with
it set — cleric and paladin — receive every spell in their spell schools at or below their mastery in
that school, automatically. Classes without it, notably mage, learn spells by transcribing scrolls
instead. The property is on the class because it is a statement about how that class relates to its
magic: divine casters are given their spells, arcane casters study for them.

**Recipes grant by skill.** A crafting skill is listed in `AUTO_GRANT_RECIPE_SKILLS` in the grant
module. Enchanting is the only member: it has no recipe scrolls, and its recipes unlock purely by
mastery tier. Every other crafting skill uses scrolls and trainers, and is absent from the set. The
declaration is on the skill rather than the class because it is a property of how that craft is
distributed, not of who practises it.

Adding a druid, a new divine school, or a second scroll-less craft is a declaration in one of those
two places. No engine change.

## Storage

Four stores, one axis: learned knowledge is permanent, granted knowledge is mastery-derived.

| | Learned (permanent) | Granted (mastery-derived) |
|---|---|---|
| Spells | `db.spellbook` | `db.granted_spells` |
| Recipes | `db.recipe_book` | `db.granted_recipes` |

All four are `{key: True}` dicts. Queries consult both stores for their kind: `knows_spell()` checks
spellbook and granted spells, `knows_recipe()` checks recipe book and granted recipes.

The two stores for a kind never hold the same key. `learn_spell()` and `learn_recipe()` refuse
anything already known by either route, and the reconcile skips anything already learned — so a key
sits in exactly one store, and which one records how it was acquired.

The separate granted store is what makes remort correct. Remort clears mastery, so it must also clear
what mastery justified, while leaving purchased and transcribed knowledge intact. With granted
knowledge in its own store this is a wipe of two dicts, not a reconstruction of provenance.

Granted knowledge is excluded from the `known_by` counts that drive knowledge-NFT spawn budgets (see
[unified-item-spawn-system.md](unified-item-spawn-system.md)). Those budgets exist to size the supply
of scrolls; knowledge that has no scroll has no supply to size.

## The reconcile

The engine lives in `world/grants.py`. One function is the whole interface:

```python
reconcile_grants(character) -> dict          # {"spells": [...], "recipes": [...]}
```

It calls two providers — `grant_spells(character)` and `grant_recipes(character)` — and returns the
keys newly added by each. `format_gains()` turns that result into the player-facing lines, so the
wording lives in one place rather than at each trigger. Both providers run the same engine over a
different description of where entitlement comes from:

| | Spells | Recipes |
|---|---|---|
| Eligible skills | spell schools available to a class the character holds with `grants_spells` | skills in `AUTO_GRANT_RECIPE_SKILLS` |
| Catalogue | `get_spells_for_school(skill)` | `get_recipes_for_skill(skill)` |
| Destination | `db.granted_spells` | `db.granted_recipes` |

For each eligible skill the engine reads the character's mastery in it, takes every catalogue entry
whose `min_mastery` is at or below that mastery, and adds the ones the character doesn't already
have — skipping both what is already granted and what they learned the hard way. Mastery is read
through one helper that resolves general skills against
`db.general_skill_mastery_levels` and class skills against `db.class_skill_mastery_levels`, so
providers never care which pool a skill lives in.

Two properties make the engine safe to call from anywhere:

**It is idempotent.** Running it against an already-correct character changes nothing and returns
empty lists. There is no cost to calling it more often than strictly necessary, which is what lets the
triggers below be generous rather than surgical.

**It is additive.** It never removes. Downgrade is handled by remort clearing the granted stores
outright, not by the reconcile computing a difference. This is deliberate: `db.granted_spells` holds
mastery grants alongside future quest rewards and racial innates with nothing to distinguish them, so
a subtractive pass would delete knowledge it did not issue.

`[TBD — needs discussion: whether granted stores should record provenance (e.g. `"mastery"` /
`"quest"`) instead of `True`. Nothing today needs it, but revoking a single mastery grant without
touching a quest grant is impossible until the stores can tell them apart. Deciding now is cheaper
than migrating the storage contract later.]`

## Where it runs

Three triggers, each for a different reason.

**Character creation**, at the end of `_apply_chargen_to_character()`. A new cleric has BASIC mastery
in the schools they selected, so the reconcile derives exactly the starting spell set. Chargen holds
no auto-grant logic of its own; it sets mastery and calls the engine. What chargen keeps is the mage's
interactive one-of-N spell pick and the one-BASIC-recipe-per-general-skill pick — those are player
*choices*, not entitlements, and they write to the learned stores.

**Mastery training**, in `_resolve_skill_training()` after the new mastery is written. This is the
trigger that produces the player-visible moment: the returned keys become
`*** You have gained the spell Purify! ***` alongside the existing mastery-advance message. Any future
route to a mastery increase calls the same function in the same place in its own flow.

**Login**, in `at_post_puppet()`. This is the trigger that makes the system self-healing. A character
who missed a grant — because they were created before a spell existed, or because a code path advanced
mastery without reconciling — is corrected the next time they connect. It is also how new content
reaches existing characters: adding a spell to a school hands it to every character with the mastery
for it, with no migration script and no per-character bookkeeping — announced on arrival, the same
way training announces it. A routine login grants nothing and so says nothing. Because the entitlement is a
function of current state rather than a record of past grants, "what has this character already been
processed for" is a question the design never has to answer.

## Extending it

- **A new spell in an existing school** — add the spell module. Existing characters receive it at next
  login if their mastery covers it.
- **A new divine school** — add the skill to the `skills` enum and map it to its classes. Spells in it
  grant automatically for classes with `grants_spells`.
- **A new granting class** (druid) — set `grants_spells=True` on the class.
- **A new scroll-less craft** — add the skill to `AUTO_GRANT_RECIPE_SKILLS`.

## What this does not cover

Learned knowledge is untouched: spell scrolls, recipe scrolls, trainer recipe purchases, and the
chargen picks all continue to write to `db.spellbook` and `db.recipe_book` through `learn_spell()` and
`learn_recipe()`. Memorisation is also unaffected — a granted spell still has to be memorised before
it can be cast, and the memory-slot cap is a function of class level and ability score, not mastery
(see [spell-skill-design.md](spell-skill-design.md)).

## Related

- [spell-skill-design.md](spell-skill-design.md) — spell schools, per-school spell tables, the
  spellbook and memorisation system.
- [crafting-system.md](crafting-system.md) — recipe format, the enchanting system, crafting rooms.
- [character-progression.md](character-progression.md) — classes, mastery tiers, trainers, remort.
