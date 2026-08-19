# Unified Search and Targeting System

> Technical design document for the MUD-wide target-resolution layer. Every command that takes a keyword-named target routes through this system. For spell-specific targeting rules see [spell-skill-design.md](spell-skill-design.md). For combat targeting see [combat-system.md](combat-system.md). For exit/door resolution see [exit-architecture.md](exit-architecture.md).

## The Trigger

A player typed `cast drain life bee-1` in a room called "Under Bee Tree". The spell command called `caller.search("bee-1")` with no `candidates=` argument. Evennia's default candidate list includes the room object itself, so the substring "bee" in "Under Bee Tree" matched the room's key. The spell received the room as its target and crashed in `take_damage()`.

This was not a spell bug. It was a **naming-resolution bug** that could reproduce in any command calling `caller.search()` without thinking carefully about scope. The stopgap fix — a dedicated `resolve_actor_target` helper in `spell_utils.py` — closed the immediate crash, but exposed a deeper problem: every command in FCM that takes a target keyword was implementing its own filter logic around bare `caller.search()`, and the logic drifted from command to command. Some filtered characters. Some didn't. Some respected hidden-object mixins. Some didn't. A single bug in one command hinted at a class of bugs spread across dozens.

## Why a Unified System

The real problem was not any single call site. It was **drift between implementations**. When every command implements its own "find a thing by name" logic, small inconsistencies become silent behavioural differences, and a fix in one command never propagates to the others. A new developer (human or AI) writing a new command starts by copying an existing call site, inheriting whatever filter semantics the donor happened to use, which may or may not be correct.

A unified targeting layer solves this by centralising the question "which object did the player mean?" into a small set of named, tested helpers. Every command that takes a target calls the same library. When a filter rule changes (e.g. visibility gets a new gate), one change to the library propagates to every consumer automatically. Call sites express **intent** (`items_room_nonexit`, `actor_hostile`), not implementation details. A future reader can see at a glance what a command is looking for without reverse-engineering a `candidates=` list.

This is not about building new search machinery. It is about making the machinery we already have **consistent** across the entire codebase.

## First Principle: Evennia-First

Before building anything new, check what Evennia already provides.

The earliest drafts of this design proposed a multi-layer custom targeting library: a `walk_scope` primitive, `scope_*` functions for every common scope, a custom `name_match` helper, and a full resolver catalog. A careful read of `evennia/objects/objects.py` revealed that most of the supposed gaps don't exist. Evennia already supports scoped searches via `location=`, typeclass and tag filters, nick substitution, numeric disambiguation, lock filtering, dbref lookups, and special-keyword shortcuts (`me`/`self`/`here`). FCM's messy target-resolution code was largely a case of **not reading the signature** before writing custom helpers that reinvented Evennia's existing machinery.

This shaped the rule that every new library entry must pass: **does Evennia already do this?** If yes, call Evennia directly and don't wrap. If no, build the thinnest possible layer that adds what Evennia cannot know about — FCM-specific semantics (mixins, combat state, game rules) — and delegate everything else.

The rule lives in [CLAUDE.md](../src/game/CLAUDE.md) § Development Approach as a general development discipline. This library is the first worked example of applying it.

## What Evennia Already Provides

Cataloging what `DefaultObject.search()` (in `evennia/objects/objects.py`) does natively before specifying any custom layer. Everything in these tables is a one-line call to `caller.search()` with the right kwarg — no new code needed.

### `caller.search()` keyword arguments

| Kwarg | Effect |
|---|---|
| `candidates=[...]` | Restrict to a pre-built list. The escape hatch for any custom filtering. |
| `location=<obj>` | Scope to `obj.contents`. Accepts a list of locations. |
| `location=caller` | **Searches the caller's inventory.** `caller.contents` is the inventory. |
| `location=caller.location` | **Searches the caller's current room only.** Excludes inventory AND the room object itself. |
| `location=container` | **Searches inside a container.** Same mechanism as the caller/room cases. |
| `typeclass=Path` or `typeclass=[...]` | Match only objects of the given typeclass(es). |
| `tags=[("tag", "cat"), ...]` | Match only objects bearing the given tag(s). |
| `attribute_name="foo"` | Search a specific attribute instead of key + aliases. |
| `quiet=True` | Suppress error messages; return a list instead of a single match. Caller owns messaging. |
| `exact=True` | Require full-string match. Default is substring/prefix. |
| `use_nicks=True` (default) | Apply the caller's object nicks before matching — free alias support for every call. |
| `use_locks=True` (default) | Honour the `search` lock — objects with `search:false()` are invisible to non-staff. |
| `stacked=N` | If multiple matches are identical (three stacked potions), return up to N instead of erroring. |
| `global_search=True` | Search the entire DB. |
| `use_dbref=True/False/None` | Allow `#123` lookups. Auto-enabled for Builders. |
| `nofound_string` / `multimatch_string` | Override default error strings on a per-call basis. |

### Behaviour Evennia gives you for free

- **`me` / `self` / `here` keywords** resolve without touching the candidate list, handled by `get_search_direct_match` before any filtering runs.
- **`<num>-<string>` numeric disambiguation** — a player types `goblin-2` and Evennia parses it to "the second goblin" via `settings.SEARCH_MULTIMATCH_REGEX`. No custom parser needed.
- **Multimatch error formatting** — when the player's keyword matches several objects, Evennia formats the `goblin-1 / goblin-2 / goblin-3` disambiguation list automatically.
- **Case-insensitive prefix/substring matching** against both `obj.key` and `obj.aliases.all()`.
- **Input whitespace normalisation.**
- **`#dbref` lookups** routed through the global DB instead of candidate filtering.

### FCMCharacter.search extension (pre-existing)

[FCMCharacter.search](../src/game/typeclasses/actors/character.py#L495) extends Evennia's search with a **substring fallback** (matches anywhere in the key, not just prefix), and two custom kwargs:

- `exclude_worn=True` — filters equipped items out of results. Used by drop/give/deposit.
- `only_worn=True` — restricts to equipped items. Used by remove.

This override predates the targeting library and is still the right place for those behaviours — they are extensions to the STRING MATCHING layer, not the semantic filter layer. The library's helpers delegate to `caller.search()` and therefore inherit the FCMCharacter extension automatically.

### Evennia helpers for predicate bodies

When the library writes predicates, it leans on existing Evennia primitives rather than reinventing them:

| Question | Evennia primitive |
|---|---|
| Does this object pass a lock? | `obj.access(caller, "get")`, `obj.access(caller, "search")`, etc. |
| Does this object have a tag? | `obj.tags.has(name, category=...)` |
| Is this object of a typeclass? | `obj.is_typeclass(path, exact=False)` |
| Does this object have an alias? | `obj.aliases.all()` |
| Is a script attached? | `obj.scripts.get("key")` |
| What exits does this room have? | `room.exits` (Evennia provides a filtered view) |
| What is in a container? | `obj.contents` |

## What Evennia Does Not Provide

With the native catalog laid out, the honest gap list is small:

1. **FCM-specific semantic predicates** — runtime-state filters Evennia cannot know about. Examples: `hp > 0`, `at this height`, `in this combat side`, `in this follow-chain group`, `visible via HiddenObjectMixin`, `is a ContainerMixin subclass`. Every one of these can be expressed as a tiny `(obj, caller) -> bool` function. None exist in Evennia because they are all FCM rules.

2. **Typed intent at call sites** — Evennia's `search()` always returns `Object | None | list[Object]`. From the call site you cannot tell whether the caller wanted a hostile actor, a held item, a lockable door, or an NPC. Type errors surface as runtime `AttributeError` downstream — which is exactly how the bee tree crash manifested. Named building blocks and composites close this gap at the architecture level.

3. **AoE multi-target shape (shipped)** — `resolve_target` returns `(target, secondaries)` where the secondaries list is built by `_resolve_aoe_secondaries` from the primary target's height, filtered by `aoe` type (unsafe/safe/allies/etc). All five AoE spells migrated. See the AoE section below.

## The Library We Built

### Module layout

```
src/game/utils/targeting/
    __init__.py          — empty exports (no API surface at the package level)
    predicates.py        — (obj, caller) -> bool filters
    helpers.py           — walk_contents, bucket_contents, building blocks,
                           composites, priority-bucketed actor resolvers,
                           resolve_target entry point

src/game/utils/direction_parser.py
    parse_direction()    — splits "door south" → ("door", "south")
                           Used by exit-interacting commands
```

### Predicate library

Each predicate is a pure `(obj, caller) -> bool` function, 3–8 lines long, with its own unit test. Factory predicates return a closure bound at creation time to whatever the check needs and the predicate signature has no room for — a caller, an actor, a lock type, a set of classes.

| Predicate | Purpose |
|---|---|
| `p_not_actor` | `not isinstance(obj, DefaultCharacter)` — excludes all actors (PCs, NPCs, mobs, pets, mounts). Vocabulary: "actor" = any DefaultCharacter subclass. |
| `p_is_character` | `isinstance(obj, FCMCharacter)` — matches player characters only, not NPCs/mobs/pets. |
| `p_not_exit` | `not isinstance(obj, DefaultExit)` — excludes exits from item candidates. |
| `p_object_visible_to` | **Object concealment gate.** Delegates to both object mixins — `obj.is_hidden_visible_to(caller)` (`HiddenObjectMixin`) and `obj.is_invis_visible_to(caller)` (`InvisibleObjectMixin`). An object must clear both. Objects composing neither pass through. |
| `p_actor_visible_to` | **Actor concealment gate.** Two composing condition gates: `HIDDEN` is pierced only by the `true_sight` effect, `INVISIBLE` only by the `DETECT_INVIS` condition. An actor under both requires both counters. Non-actors pass through. |
| `p_height_visible_to` | **Spatial gate.** Delegates to `obj.is_height_visible_to(caller)` — checks the room's visibility barriers against the object's size. Objects small enough to be concealed by a barrier between observer and object are hidden. Same-height objects are always visible. |
| `p_can_perceive` | **Concealment composite.** `p_object_visible_to AND p_height_visible_to AND p_actor_visible_to`. Is the thing there and unconcealed — nothing about whether the observer can see. Use for display, perception, and actions done by touch. Extensible: future concealment gates (ethereal, fog-of-war) get added here and propagate to all consumers, `p_can_see` included. |
| `p_can_see` | **Concealment plus sight.** `p_can_perceive` and the observer is neither `BLINDED` nor in a room dark for them — darkvision, carried light, lit fixtures and the day/night phase all fold into that via `RoomBase.is_dark`. Use where the action needs eyes. Not for display: darkness is meant to redact names, not remove candidates. |
| `p_living` | `hp > 0` — excludes items (hp=None), corpses (hp=0), dead mobs. Defensive on type. |
| `p_in_combat` | Has a `combat_handler` script attached. Combat-specific runtime-state filter. |
| `p_is_container` | `getattr(obj, "is_container", False)` — matches `ContainerMixin`. Corpses are NOT containers. |
| `p_is_lockable` | `hasattr(obj, "is_locked")` — type check for `LockableMixin`. Does NOT check current lock state. |
| `p_is_locked` | `getattr(obj, "is_locked", False)` — state check, True if currently locked. |
| `p_is_openable` | `hasattr(obj, "is_open")` — type check for `CloseableMixin`. Does NOT check current open/closed state. |
| `p_is_open` | `getattr(obj, "is_open", False)` — state check, True if currently open. |
| `p_is_open_exit` | Is this exit barred by its own door? Unlocked *and* open. Defaults to True where `p_is_open` defaults to False — an exit with no door bars nothing. Movement paths only; see Sight lines vs routes below. |
| `p_is_typeclass(*typeclasses)` | **Factory** — `isinstance(obj, typeclasses)`. The general form of the named type predicates, for checks the library has no name for. The caller supplies the classes, so targeting imports no typeclasses and stays a leaf package. Prefer a named predicate where one exists: `p_is_character` carries meaning `p_is_typeclass(FCMCharacter)` does not. |
| `p_not_typeclass(*typeclasses)` | **Factory** — the inverse of `p_is_typeclass`. Its use is carving an exception out of a positive type check, which one positive predicate cannot express: `p_is_typeclass(FCMCharacter, CombatMob)` + `p_not_typeclass(Rabbit)` is "any actor but my own kind". |
| `p_excluding(*excluded)` | **Factory** — rejects specific objects by **identity**, not equality, so a typeclass defining `__eq__` cannot smuggle one past. "Everything but these": the on-kill chain attack excludes the victim it just killed. Dropping the caller is not its job — helpers that must never return their own caller do that themselves. |
| `p_larger_than(actor)` | **Factory** matching objects bigger than `actor` on the `Size` scale. Strict, so equals fail. The prey/predator axis without naming a typeclass — flee what is larger, hunt what is smaller, ignore equals. No `size` counts as `MEDIUM`; an unrecognised size fails rather than guessing. |
| `p_smaller_than(actor)` | **Factory** — the inverse of `p_larger_than`, strict in the same way. |
| `p_fits_through(actor)` | **Factory** wrapping the `max_size` gate `ExitVerticalAware.at_traverse` enforces. Takes the actor *to fit*, not the caller looking — a retreat leader checks the party's largest member, not themselves. |
| `p_passes_lock(lock_type)` | **Factory** returning a predicate wrapping `obj.access(caller, lock_type)`. |
| `p_same_height(caller)` | **Factory** returning a predicate matching objects at the caller's `room_vertical_position`. Used by melee-range spells. |
| `p_different_height(caller)` | **Factory** — inverse of `p_same_height`. For future `ranged_only` spells. |
| `p_same_height_value(height)` | **Factory** matching objects at an explicit height value. Used by AoE secondaries where height comes from the primary target, not the caster. |
| `p_involved_with(caster)` | **Factory** matching actors involved with the caster — combat participants in combat, group members out of combat. Used for bystander protection in non-PvP rooms. |

**Visibility predicate family:**

Three single-axis concealment gates, a composite of them, and a stricter composite that also
requires working sight.

- `p_object_visible_to` — object concealment only (the two object mixins). Used in targeting
  resolvers where height is handled separately by range predicates (`p_same_height`,
  `p_different_height`).
- `p_actor_visible_to` — actor conditions only (`HIDDEN` / `INVISIBLE`). Also used directly by
  `RoomBase.msg_contents` and `msg_contents_with_invis_alt`, which ask it per recipient with the
  operands in messaging order: the sender is the thing seen, the recipient is the observer.
- `p_height_visible_to` — spatial only.
- `p_can_perceive` — composite of all three. Is the thing there, and is it concealed from this
  observer.
- `p_can_see` — `p_can_perceive` plus the observer half: not `BLINDED`, and the room not dark for
  them.

The two concealment systems are independent: an actor can be `HIDDEN` by condition while a chest is
hidden by mixin. Their counters are independent too — `true_sight` pierces physical concealment and
`DETECT_INVIS` pierces invisibility, and neither covers the other.

**Perceiving is not seeing**, and the split is load-bearing because the two halves behave
differently in play:

- **Concealment excludes.** A hidden or invisible actor fails `p_can_perceive`, drops out of the
  candidate list, and the observer never learns they were there.
- **Darkness redacts.** It is not part of `p_can_perceive`, so it never removes a candidate from a
  filtered list. It is asked separately at *render* time by `UnseenNameMixin`, which calls
  `p_can_see` and returns the object's `unseen_name` — "Someone", "something", "Somewhere" — leaving
  the candidate in place. See [room-architecture.md](room-architecture.md) § Name Redaction.

So an unlit room reads as several people you cannot identify rather than an empty room. Folding
darkness into `p_can_perceive` would collapse one into the other.

**Which to reach for.** `p_can_perceive` when the question is "is it there" — room appearance, look,
scan, mob perception, and actions that do not depend on sight, like swinging at the opponent you are
already fighting or finding your own dagger by touch. `p_can_see` when the action genuinely needs
eyes. Sight is a property of the observer, so it is the same answer for every candidate in the room
and costs a room-contents and inventory scan to compute — fine on a targeting path resolving one
command, wrong on a display path rendering every object.

A single-axis predicate is correct only where another mechanism already covers the remaining axes —
for example a resolver that applies its own height predicate. Reaching for `p_object_visible_to` by
default silently skips actor concealment.

**Telling the player why.** A command that needs sight may keep an early `looker_is_blind(caller)`
return alongside the predicate. The predicate is the gate; the early return exists so the refusal
reads honestly. Filtering alone produces "You don't see 'goblin' here", which is misleading when the
goblin is standing in front of them.

**Sight lines vs routes.** Two questions that look alike and take different predicates:

| Question | Composition |
|---|---|
| *May I go this way?* | `p_passes_lock("traverse")` + `p_can_perceive` + `p_is_open_exit`, i.e. `open_exits()` |
| *Can I see this way?* | `is_open` + `p_can_perceive` |

A lock governs passage, never sight — you can see down a corridor you are barred from walking into,
so `p_passes_lock` has no place in a perception path. Nor does `p_is_open_exit`, which folds a lock
check into its answer. Every locked door is a shut one anyway (`lock()` refuses on an open door), so
the plain `is_open` check covers every state that can legitimately occur; an open-and-locked exit is
a data anomaly, not a case to handle.

Concealment cuts across both: an exit the looker cannot perceive blocks sight *and* passage, whether
it stands open or not. Scanning past an undiscovered door would report who is beyond a passage the
character does not know exists.

`cmd_scan` is the worked example — see `_can_scan_through`, and the test that pins a
traverse-locked-but-open exit as still scannable so nobody consolidates it onto `open_exits` later.

Each predicate has a docstring explaining what Evennia handles natively instead. [predicates.py](../src/game/utils/targeting/predicates.py) opens with a detailed comment block listing the dozen-plus filters a developer should implement with a native Evennia kwarg instead of adding a predicate.

### The walk primitive

```python
def walk_contents(caller, source, *predicates):
    """Walk source.contents once and return objects passing every predicate."""
    if source is None:
        return []
    contents = getattr(source, "contents", None)
    if not contents:
        return []
    return [
        obj for obj in contents
        if all(p(obj, caller) for p in predicates)
    ]
```

Universal targeting primitive. Short-circuits per-object via Python's `all()` so the first failing predicate stops evaluation for that object. Returns an empty list if `source` is `None` or has no `.contents`, so callers can iterate unconditionally.

**Predicate order matters**: cheap predicates (identity/attribute checks) should come before expensive ones (method calls, lock checks) so short-circuit eval pays off.

**No shared base constant.** Each call site declares its own predicate stack explicitly. There is no `BASE_ITEM_PREDICATES` — it was removed because it made developers lazy about evaluating what their specific building block actually needs to filter.

### `bucket_contents`

```python
def bucket_contents(caller, source, key_fn, *predicates, order=None):
    """Walk source.contents once, filter via predicates, bucket via key_fn."""
```

Single-pass sibling of `walk_contents`. Objects passing all predicates are classified by `key_fn(obj, caller)` into named buckets. Returns `{bucket_name: [objects]}`. When `order` is supplied, pre-populates those keys (with empty lists) and preserves insertion order for priority-based iteration.

Primary consumer: `combat.combat_utils.get_sides` (partitions living combatants into allies/enemies). Also used by the priority-bucketed actor resolvers and designed for future AI threat bucketing.

### `open_exits`

```python
def open_exits(caller):
    """The exits caller could actually leave the room through."""
```

Composes `p_passes_lock("traverse")`, `p_can_perceive` and `p_is_open_exit` over `room.exits` — Evennia's
own filtered view, which is why there is no "is an exit" predicate.

**Selection, not permission.** The chosen exit's `at_traverse` enforces passage and applies gates
these predicates cannot see — encumbrance, size, traps. A caller picks from this list and then
traverses; membership is a candidate, never a guarantee.

Consumers: `flee_from_combat` and `retreat_group` in `combat/combat_utils.py`, and mob AI via
`AIHandler.get_area_exits`, which every wander, flee, retreat and cornered check reaches. Perception
paths deliberately do **not** use it — see Sight lines vs routes above.

#### `source.contents` is a cache, not a query

Both helpers walk `source.contents`, which Evennia serves from an in-memory cache of
primary keys (`ContentsHandler`), resolved through the idmapper. It is not read from the
database, and `get()` does not verify that a cached object's `.location` still points at
that source. An object whose location has moved on can therefore still be returned — it
will appear in the room's contents while reporting a different `.location`.

This affects every scope in the table below, since they all resolve from `room.contents`.
Combat hardened its own path: the repeating-attack retarget re-checks
`enemy.location == self.obj.location` before committing to a target (see
[combat-system.md](combat-system.md) § Repeating-Attack Retarget).

`[TBD — needs discussion: what corrupts the cache, and whether other resolvers/scopes
should validate location the way combat now does. Observed in play; cause not established.]`

### Direction parser

[`utils/direction_parser.py`](../src/game/utils/direction_parser.py) — `parse_direction(text)` splits player input into `(name, direction)` by checking each word against `ExitVerticalAware.DIRECTION_ALIASES`:

```python
parse_direction("door south") → ("door", "south")
parse_direction("south door") → ("door", "south")
parse_direction("s door")     → ("door", "south")
parse_direction("south")      → ("", "south")
parse_direction("chest")      → ("chest", None)
```

Used by exit-interacting commands (open, close, lock, unlock, picklock, disarm_trap). When a direction is parsed, the command uses `items_room_exit_by_direction` — directional qualification means the player is targeting an exit, so room objects and inventory are not searched.

### Design decisions worth documenting

1. **Identification-only, not action-gating.** The library answers "which object did the player mean?" and stops. Every gate that depends on command intent (open/closed, access locks, hooks, weight checks) stays in the command layer. This keeps the building blocks pure, testable, and reusable across commands with different action semantics.

2. **Delegate string parsing to `caller.search()`.** The library never reimplements nick substitution, dbref lookup, `<num>-<string>` disambiguation, `me`/`self`/`here` shortcuts, `stacked`, or `use_locks`. All of these come for free via `caller.search(candidates=..., **kwargs)`.

3. **No shared base predicates.** Each building block declares its own predicate stack explicitly. This forces every new building block to evaluate what it actually needs to filter, rather than lazily inheriting a "standard" set that may or may not be appropriate.

4. **Fail-soft on missing source.** `None` source or missing `.contents` returns `None` / `[]`, never raises. Lets callers try multiple sources speculatively.

5. **No override of Evennia's `search()`.** An earlier draft proposed overriding `get_search_candidates` to strip the room from the default candidate list. We rejected that approach: the typed building blocks make the footgun structurally impossible for every migrated call site, and overriding `get_search_candidates` fights Evennia's intentional behaviour (`look here` needs the room in candidates).

6. **Predicate extraction is demand-driven.** Every predicate in the library was added when a concrete consumer needed it, not because it might be useful one day.

7. **Base targeting vs filtered targeting — choose per command.** The targeting system provides two levels of resolution:

   - **Base targeting** — "find the visible thing the player named." Uses broad building blocks that don't filter by object type. Every command needs at least this.
   - **Filtered targeting** — "find the visible thing the player named that's ALSO a container / gettable / an enemy / at this height." Uses narrow building blocks. Commands use this when "not found" is the right answer for a type mismatch.

   The choice is **per command, per code path**, based on what error messaging serves the player best:

   - `cmd_put ... in backpack` → **filtered**. If backpack isn't a container, "not found" is fine.
   - `look in sword` → **broad** (find the item, then check `is_container`). "Not a container" is more helpful than "not here."
   - `cmd_get sword` → **broad** (`items_room_nonexit`). If the sword is a fixture, "can't get that" beats "not here."
   - `cmd_attack shopkeeper` → **broad** (`actor_hostile` finds the shopkeeper). Command checks room combat permission.

8. **Predicates are vocabulary, not just filters.** Predicates define *what a concept means* — "visible," "at this height," "is a container." Where a predicate gets evaluated is a separate decision. The same predicate function serves three roles:

   - **Automatic filter** inside `walk_contents` / `bucket_contents` — the resolver never sees objects that fail the predicate.
   - **Explicit check** in a command's `if` branch — the command calls the predicate directly on a resolved object for bespoke decisions.
   - **Display filter** in room appearance / look / scan — the predicate post-filters a contents list for perception purposes.

   Even when a command uses broad (unfiltered) targeting, the predicates are still the consistent, single-source-of-truth way to ask questions about the resolved object.

## Building Blocks

Building blocks are singular target types — each searches one scope with one predicate stack. The command supplies them via `target_type` to `resolve_target`. Building blocks declare only **structural** predicates (what kind of object). Visibility, height, and command-specific checks are the caller's responsibility via `extra_predicates` or post-resolution checks at the command layer.

### Item building blocks

| Building block | Scope | Structural predicates | Notes |
|---|---|---|---|
| `items_inventory` | `caller.contents` | none | Actors/exits/fixtures can't exist in inventory. `exclude_worn=True` via search kwarg. |
| `items_equipped` | `caller.contents` | none | `only_worn=True` via search kwarg. |
| `items_room_all` | `room.contents` | `p_not_actor` | Broadest room scope — includes exits, fixtures, loose items, containers, doors. |
| `items_room_exits` | `room.exits` | none (Evennia's native exit filter) | Name-based exit lookup. Uses Evennia's `room.exits` filtered view. |
| `items_room_exit_by_direction` | `room.exits` filtered by `direction` kwarg | direction match | Direction-based exit lookup. Requires `direction` kwarg from `parse_direction()`. |
| `items_room_nonexit` | `room.contents` | `p_not_actor`, `p_not_exit` | Standard scope for most item commands — everything except actors and exits. |
| `items_room_gettable` | `room.contents` | `p_not_actor`, `p_not_exit`, `p_passes_lock("get")` | Narrowest non-exit scope — only loose, takeable items. |
| `items_room_fixed` | `room.contents` | `p_not_actor`, inverse of `p_passes_lock("get")` | Non-gettable objects including exits/doors. Inverse of `items_room_gettable`. |
| `items_room_fixed_nonexit` | `room.contents` | `p_not_actor`, `p_not_exit`, inverse of `p_passes_lock("get")` | Non-gettable objects excluding exits. Fixtures only. |

### Actor building blocks

| Building block | Scope | Structural predicates | Notes |
|---|---|---|---|
| `actors_in_combat` | `room.contents` | `p_living`, `p_in_combat` | Living actors with combat_handler. No priority bucketing. |
| `actors_not_in_combat` | `room.contents` | `p_living`, inverse of `p_in_combat` | Living idle actors. |

## Composites

Composites chain multiple building blocks with fallback logic. Step 1 searches first; if no match, step 2 searches the fallback scope. Both steps pass `extra_predicates` through.

| Composite | Step 1 | Step 2 | Consumer |
|---|---|---|---|
| `items_room_all_then_inventory` | Room (all non-actor, incl. exits) | Inventory (exclude worn) | Knock spell |
| `items_inventory_then_room_all` | Inventory (exclude worn) | Room (all non-actor, incl. exits) | Identify, Holy Insight |
| `items_inventory_then_room_nonexit` | Inventory (exclude worn) | Room (non-actor, non-exit) | cmd_look "look in" container |
| `items_room_all_then_room` | Room contents (all non-actor) | Room itself (unconditional fallback) | cmd_disarm_trap (pressure plates) |
| `actors_in_combat_then_not_in_combat` | Living in-combat actors | Living idle actors | cmd_join |

## Targeting vs Filtering

An important boundary the library intentionally does not cross:

- **Targeting** answers "which specific object did the player mean?" — returns ONE thing (or a small disambiguation set). Every building block and composite does this.
- **Filtering** answers "which objects in this collection match some criterion?" — returns ALL matches; the caller acts on each in a loop.

Bulk-action patterns like `get all`, `drop all`, and `loot all` are filter-and-act loops, not targeting. They happen to share predicates with the targeting library (a `get all` wants the same living/non-exit/visible/gettable filter as `get sword`), but their loop structure is different. Forcing them into the resolver pattern would add a second pass, blur the boundary, and fragment the library.

For now, bulk-action commands keep their inline filter loops. Predicates from this library are available for reuse, but the loop structure stays with the command.

## `resolve_target` — Universal Entry Point

All targeting in the game routes through a single function:

```python
resolve_target(caller, target_str, target_type, aoe=None,
               extra_predicates=(), stacked=0, direction=None)
```

Lives in `utils/targeting/helpers.py`. Returns `(target, secondaries)` — a tuple of the primary target and a list of AoE secondary targets. For non-AoE callers, `secondaries` is always `[]`. On failure, returns `(None, [])` with an error message already sent to the caller.

### Parameters

| Parameter | Purpose |
|---|---|
| `caller` | The actor doing the targeting. |
| `target_str` | Keyword typed by the player (already parsed). |
| `target_type` | Building block or composite name — routes to the right search logic. |
| `aoe` | AoE cascade type (None, "unsafe", "unsafe_all_heights", "unsafe_self", "safe", "allies"). |
| `extra_predicates` | Additional `(obj, caller) -> bool` predicates appended to the building block's structural stack. Callers supply visibility, height, type checks here. |
| `stacked` | Forwarded to `caller.search(stacked=N)` for quantity operations. |
| `direction` | Canonical direction string from `parse_direction()`. Required by `items_room_exit_by_direction`. |

### Target types — actors

| Type | Resolution | Self policy |
|---|---|---|
| `actor_hostile` | Priority-bucketed (enemy > bystander > ally > self) | Returned — command decides |
| `actor_any` | Same as hostile | Returned — command decides |
| `actor_friendly` | Friendly-priority (self > ally > bystander > enemy), empty defaults to self | Allowed |
| `actors_in_combat` | Flat — living actors with combat_handler | N/A |
| `actors_not_in_combat` | Flat — living idle actors | N/A |
| `actors_in_combat_then_not_in_combat` | Composite — combat first, idle fallback | N/A |
| `self` | Returns caller directly | N/A |
| `none` | Returns None directly | N/A |

Self-rejection is NOT done in `resolve_target` — the caller is returned via the self-bucket fallback so each command can emit its own context-specific error.

### Target types — items

All item building blocks and composites listed in the Building Blocks and Composites sections above are valid `target_type` values.

### Height/range integration

Height filtering has been **removed from `resolve_target`**. There is no `range` parameter. Height and range checks are handled at the command layer:

- `cmd_cast` / `cmd_zap` build `extra_predicates` from `spell.requires_sight` (adding `p_can_see`) and check height/range centrally after resolution using the spell's `range` attribute and overridable messages (`out_of_reach_message`, `too_close_message`).
- Other commands add height predicates via `extra_predicates` as needed.

This keeps the targeting library free of range semantics — it identifies, the command gates.

### Sightlessness — the three shapes

**Darkness without darkvision and blindness are one state.** `looker_is_blind(caller)`
(`utils/visibility.py`) is the whole test: it is true when the caller holds `BLINDED`, *or* when
the room is dark for them. Nothing downstream distinguishes the two causes.

Each command answers one question — **can this be done by touch?** — and the answer puts it in one
of three shapes. Which predicate a command passes to `resolve_target` follows from its shape, which
is why the migrated-command table below is not uniform.

**Fumble — the action works by touch.** Keeps `p_can_perceive`, and charges time instead of
refusing. `check_busy(caller)` at entry, then `start_busy(caller, fumble_seconds(), ...)` with the
command's own wording when sightless (see [busy actions](#busy-actions--the-shared-lock)). The
search runs *before* the outcome is known, so a failed search costs the same as a successful one —
a character who is not carrying what they asked for still spends the time, then hears so. That
matters: an instant failure would tell them what is in their own pack without looking.

*Hold, drink, climb, light, quaff, remove, wear, refuel, drop, open, close, switch, put, search.*

`search` adds the one thing a fumble can carry beyond time: every roll in a sightless search is
made at **disadvantage**, through the dice helper the command already used, so there is no penalty
constant to balance. Note the standing resolution rule — advantage and disadvantage cancel, so a
blind searcher holding `non_combat_advantage` rolls normally.

**Refuse — the action needs eyes.** `p_can_see` on the resolvers, and an early return naming what
was asked for: `"It's too dark to make out 'chest'."` Never `"You don't see it here"`, which reads
as *absent* when the thing is right in front of them.

*Lock, unlock, read, refill, scan, show, zap, trade, give, join, exits, socials with a target, and
every skill command.*

`exits` is the one worth explaining, because `open_exits()` takes the opposite line for the same
doorways: a doorway is *found* by touch, so losing your light must never leave you cornered, but
*reading a list* of them is detail and detail is what darkness takes. You can still flee through a
door whose label you cannot read.

**Reduced fidelity — presence survives, detail does not.** `look` at a specific actor or object
resolves on `p_can_perceive`, then renders at whatever fidelity is available: *"Someone is there,
but it's too dark to make out any detail."* The name comes from `UnseenNameMixin`, so a builder can
set it per mob. Bare `look` is not gated in the command at all — `RoomBase.return_appearance` thins
its own display. Things with no presence to sense (a compass direction, a room detail) answer as if
there were nothing there.

*Look.*

**Untargeted actions are never gated.** You can laugh in the dark; you just cannot laugh *at*
someone.

**Concealment and darkness are not the same lever.** Concealment *excludes* — a hidden or invisible
actor drops out of the candidate list entirely, at every shape. Darkness *redacts* — the thing is
still there, and what changes is how much you can tell about it.

### Busy actions — the shared lock

`utils/busy.py` owns `ndb.is_processing`, the lock held by any action that takes time and takes
both hands — harvesting, and the fumble shape above.

| Function | Purpose |
|---|---|
| `check_busy(caller)` | Entry guard. Messages and returns `True` if the caller is already occupied |
| `start_busy_ticks(caller, ticks, interval, on_complete, progress=None, done_msg=None, self_msg=None, room_msg=None, room_alt_msg=None)` | The general form. Lock, announce, report progress each tick, clear, then run the outcome |
| `start_busy(caller, seconds, on_complete, self_msg=None, room_msg=None, room_alt_msg=None)` | The single-tick case, for actions short enough that progress would be noise |
| `progress_bar(step, total, width=10)` | `####------`, shared by everything that draws one |
| `fumble_seconds()` | The search-by-touch duration, 3–4 seconds |
| `BUSY_MESSAGE` | The refusal wording — busy is busy regardless of cause, so it lives in one place |

`start_busy` is `start_busy_ticks` with one tick and nothing to report, so there is one lock
implementation rather than two. `progress` is a callable `(step, total) -> str` called before each
wait, starting at step 0 so an empty bar appears immediately; every caller words its own. Supplying
`room_alt_msg` switches the announcement to `msg_contents_with_invis_alt`, so observers who cannot
perceive the actor see tools moving by themselves rather than a person using them.

Two different strings, and both are needed: the *refusal* a second command gets is generic and
shared; the *announcement* is per-action and passed in ("You begin gathering...", "You grope about
in the dark, feeling for something to climb...").

Being interrupted does not cancel a busy action — a character jumped mid-harvest finishes the swing
before they can react. Every command that respects the lock refuses while it is held, including
attack, via `CombatMixin._can_start_fight_now()`. The lock is released *before* `on_complete` runs,
so an outcome that starts something else is not blocked by the action that produced it.

Every timed action runs through this: harvesting and the fumble shape, crafting, processing, repair
and insetting, skill and weapon training, and the travel, sail and explore journeys. Nothing outside
`utils/busy.py` touches `ndb.is_processing`.

## Priority-Bucketed Actor Resolvers

The targeting library provides priority-bucketed resolvers for actor targeting. Each resolver uses the same classifier (buckets actors by relationship to the caller) with a parameterised priority order.

**In-combat** — classifier reads `combat_side` from `obj.scripts.get("combat_handler")[0]`:

| Resolver | Order (first wins) |
|---|---|
| `resolve_attack_target_in_combat` | enemy > bystander > ally > self |
| `resolve_friendly_target_in_combat` | self > ally > bystander > enemy |

**Out-of-combat** — classifier reads group membership via `get_group_leader()`:

| Resolver | Order (first wins) |
|---|---|
| `resolve_attack_target_out_of_combat` | stranger > groupmate > self |
| `resolve_friendly_target_out_of_combat` | self > groupmate > stranger |

All four accept `extra_predicates=()` for caller-supplied filters.

The `self` bucket is intentionally last-priority for hostile and first-priority for friendly. The `_is_self_keyword` helper intercepts `"me"` / `"self"` keywords at the top of each resolver to bypass an Evennia direct-match quirk.

**These resolvers still have baked-in `p_object_visible_to`** — the object-concealment predicate is in their `bucket_contents` call, not supplied by the caller. This is a known inconsistency with the "no baked-in predicates" principle. Refactoring to caller-supplied visibility is pending.

It also has a behavioural consequence worth noting: `p_object_visible_to` gates objects, not actors, so **actor concealment is unchecked during resolution** unless the calling command adds `p_can_see` itself. `[TBD — needs discussion: should the actor resolvers default to `p_can_see`?]`

## AoE Secondaries System

The `aoe` parameter on `resolve_target` drives AoE secondary target collection:

| aoe | Secondaries |
|---|---|
| `None` | `[]` — no cascade (default) |
| `"unsafe"` | Everyone at target's height, caster included |
| `"unsafe_all_heights"` | Everyone in room regardless of height, caster included |
| `"unsafe_self"` | Everyone at target's height except caster |
| `"safe"` | Enemies only at target's height |
| `"allies"` | Allies only at target's height, caster included |

**Bystander protection (non-PvP rooms):** For the three unsafe variants, candidates are filtered through `p_involved_with` so that only combat participants (in combat) or the caster's group members (out of combat) can be hit.

The primary target is always excluded from secondaries. Enemy detection for "safe" AoE out of combat uses group membership, matching the priority resolvers.

## Spell Base Class Attributes

Every spell declares targeting-relevant attributes:

```python
class Spell:
    target_type = "actor_hostile"    # which resolve_target branch
    aoe = None                       # AoE cascade type
    medium = "air"                   # environment restriction (not yet enforced)
    requires_sight = True            # False for detect/sense spells
    out_of_reach_message = "That's out of reach."
    too_close_message = "You're too close for that spell."
```

`requires_sight` controls whether `cmd_cast`/`cmd_zap` add `p_can_see` to `extra_predicates`. `out_of_reach_message` and `too_close_message` are overridable per-spell for flavour.

`medium` is declared but not enforced — defaults to `"air"` so nothing accidentally works underwater before the system is designed. Cast-time validation will live in `base_spell.cast()`, not in `resolve_target`.

## Legacy Helpers

These helpers predate the building block architecture and are still used by some code paths:

| Helper | Used by | Notes |
|---|---|---|
| `resolve_item_in_source` | `cmd_get` (some paths) | Pre-filters with `p_not_actor, p_not_exit, p_object_visible_to`. Source-agnostic. |
| `resolve_container` | `cmd_get`, `cmd_put` | Inventory-first, room-fallback container lookup with `p_is_container`. |
| `resolve_character_in_room` | `cmd_give` | PC-only lookup via `p_is_character`. |
| `_resolve_world_item` | Internal | Delegates to `find_exit_target` for directional parsing. |
| `_resolve_all_room` | Internal | Room contents walk for `p_not_actor, p_object_visible_to`. |

These will be retired as remaining consumers migrate to building blocks.

## Commands Migrated to `resolve_target`

Every migrated command follows the same pattern: sightlessness handling → `resolve_target` with
building block + `extra_predicates` → command-layer state/type checks → action.

The **Shape** column is the one from [Sightlessness](#sightlessness--the-three-shapes), and it
determines the predicate: *fumble* commands keep `p_can_perceive` and charge time, *refuse*
commands take `p_can_see` and return early.

| Command | target_type | extra_predicates | Notes |
|---|---|---|---|
| cmd_cast | per spell | `p_can_see` if `requires_sight` | Central range/height check after resolution |
| cmd_zap | per spell | `p_can_see` if `requires_sight` | Same pattern as cmd_cast |
| cmd_attack | `actor_hostile` | | POC — weapon range mapped at call site |
| cmd_hold | `items_inventory` | `p_can_perceive` | Fumble — your own pack is found by feel |
| cmd_wear | `items_inventory` | `p_can_perceive` | Fumble — dressing in the dark takes longer |
| cmd_remove | `items_equipped` | `p_can_perceive` | Fumble — undressing by touch is fiddlier than it sounds |
| cmd_drop | `items_inventory` | `p_can_perceive` | Fumble |
| cmd_give | `items_inventory` + `resolve_character_in_room` | `p_can_see` | Refuse — no point finding the item if you cannot find who to give it to |
| cmd_put | `items_inventory` + `items_inventory_then_room_nonexit` | `p_can_perceive` | Fumble — one search covers both targets. A room container is the same chest `open` works by touch |
| cmd_get | `items_room_nonexit` | `p_object_visible_to` | Broad targeting, get-lock check at command layer. **Object axis only** — not reviewed against the three shapes yet |
| cmd_look (target) | quiet search | `p_can_perceive` | Reduced fidelity — unseen renders as "Someone is there, but it's too dark to make out any detail." |
| cmd_look (container) | `items_inventory_then_room_nonexit` | `p_can_perceive` | "look in" path. The container is sensed; its contents are not listed |
| cmd_quaff | `items_inventory` | `p_can_perceive` | Fumble |
| cmd_drink | `items_inventory_then_room_nonexit` | `p_can_perceive` | Fumble |
| cmd_light | `items_equipped` | `p_can_perceive` | Fumble — an equipped torch is found by feel |
| cmd_extinguish | `items_equipped` | `p_can_perceive` | Not gated — if the source is lit, the room is not dark |
| cmd_refuel | `items_inventory` then `items_equipped` | `p_can_perceive` | Fumble |
| cmd_read | `items_room_fixed_nonexit` | `p_can_see` | Refuse — reading is the one thing with no version done by touch |
| cmd_consider | `actor_hostile` | `p_can_see` | |
| cmd_diagnose | `actor_friendly` | `p_can_see` | |
| cmd_follow | `actor_hostile` | `p_can_see` | Strangers = most likely follow target |
| cmd_join | `actors_in_combat_then_not_in_combat` | `p_can_see` | Refuse, not fumble — a three second grope while your ally takes hits reads wrong |
| cmd_open | `items_room_exit_by_direction` / `items_room_all_then_inventory` | `p_can_perceive` | Fumble — a latch is found by feel. Direction parser splits input |
| cmd_close | `items_room_exit_by_direction` / `items_room_all_then_inventory` | `p_can_perceive` | Fumble — the mirror of `open` in every respect, including `check_busy` |
| cmd_switch | `items_room_nonexit` | `p_can_perceive` | Fumble — a lever is found by running your hands along the wall |
| cmd_lock | `items_room_exit_by_direction` / `items_room_all_then_inventory` | `p_can_see` | Refuse — lining a key up with a keyhole needs eyes |
| cmd_unlock | `items_room_exit_by_direction` / `items_room_all_then_inventory` | `p_can_see` | Refuse — same as lock |
| cmd_picklock | `items_room_exit_by_direction` / `items_room_all_then_inventory` | `p_can_see` | Refuse — eyes on the keyway, same as using a key |
| cmd_disarm_trap | `items_room_exit_by_direction` / `items_room_all_then_room` | `p_can_see` | Room fallback for pressure plates |
| cmd_scan | `room.exits` | `is_open` + `p_can_see` | Sight line, not a route — lock state deliberately unread. Applied at each step outward, judged from the caller at every depth |
| cmd_flee / cmd_retreat | `open_exits` | `p_passes_lock("traverse")` + `p_can_perceive` + `p_is_open_exit` | **Perceive, not see** — a doorway is found by touch, so an unlit room must not leave you cornered. Retreat additionally gates on `p_fits_through` for the party's largest member |

### Commands with darkness/visibility checks (no structural migration)

| Command | What was added |
|---|---|
| cmd_climb | `p_can_perceive` on inline climbable filtering — you climb a wall you can feel. Fumble shape: unseen gropes for the sole climbable, or refuses to tell several apart rather than listing what the climber cannot see |
| cmd_social | `p_can_see` on the **targeted** path only. Untargeted and self-target are never gated |
| cmd_scan | `looker_is_blind` refusal at entry, and `p_can_see` throughout. A dark *destination* room is a separate check — it stops the scan in that direction |
| cmd_survey | `looker_is_blind` refusal at entry; `p_can_see` on mappable exits |
| cmd_search | Fumble shape on its own search loop: `check_busy`, `fumble_seconds`, and every roll at disadvantage when sightless |
| cmd_exits | `looker_is_blind` refusal, plus `p_fits_through` rendered as a `(too small)` label. Size is **labelled, not filtered** — this command lists a locked door and says `(locked)` rather than hiding it, and the room display does no size filtering at all, so hiding a passage would leave the two views disagreeing |
| cmd_inventory | `looker_is_blind` gate: every carried item, every resource line and the gold total collapse to a placeholder, so the count survives and the identity does not. Concealed items do the same individually, via `p_object_visible_to` |
| cmd_say / cmd_whisper / cmd_shout | The speaker names themselves per recipient — see below |

### Who is speaking

A voice carries to people who cannot see its owner, so the line always goes out and only the name
changes. Each of the three speech commands messages its listeners individually rather than through
`msg_contents`, so the per-recipient form is `caller.get_display_name(listener)` — which asks
`p_can_see` and falls back to the speaker's own `unseen_name`.

That single call covers what three hand-rolled copies did not: `HIDDEN` conceals a speaker as
readily as `INVISIBLE`, a blinded listener cannot name anyone, `true_sight` and `DETECT_INVIS` each
pierce their own axis and neither pierces the dark, and the placeholder is content a builder can
set rather than a hardcoded string.

`say` resolves its **target** the same way — being spoken to is not what gives you away. `shout`
needs it in one loop only: adjacent rooms hear a muffled line that names nobody.

## Testing Strategy

Three tiers, each answering one question.

### Tier 1 — predicate unit tests

One test class per predicate. Each test uses a `SimpleNamespace` or `MagicMock(spec=...)` fake — no Evennia DB. Typically 2–3 assertions per predicate covering true case, false case, and one edge case.

[tests/utils_tests/test_targeting_predicates.py](../src/game/tests/utils_tests/test_targeting_predicates.py)

### Tier 2 — helper unit tests

One test class per building block and resolver. Tests cover happy path, None sources, predicate filtering, kwarg forwarding, source fallback precedence (for composites), and non-gating behaviour.

[tests/utils_tests/test_targeting_helpers.py](../src/game/tests/utils_tests/test_targeting_helpers.py)

### Tier 3 — consumer regression tests

Every command migration runs the command's existing test suite as the regression gate. If the migration changes observable behaviour, an existing test fails and we debug.

**Migration discipline**: every cycle follows the same pattern — plan the scope, build the building block(s), test the building block(s), migrate the consumer, rerun the consumer's existing test suite, commit separately. No speculative additions, no building blocks written ahead of concrete consumers.

## Relationship to Other Systems

- **[spell-skill-design.md](spell-skill-design.md)** — all spell target types route through `resolve_target` with `actor_*` / `items_*` flags. `requires_sight`, `out_of_reach_message`, `too_close_message` attributes on spell base class.
- **[combat-system.md](combat-system.md)** — `combat.combat_utils.get_sides()` rebuilt on `bucket_contents`. `cmd_attack` routes through `resolve_target`. Priority-bucketed resolvers used by combat commands and hostile spells.
- **[exit-architecture.md](exit-architecture.md)** — exit commands (open, close, lock, unlock, picklock, disarm_trap) use `parse_direction()` + `items_room_exit_by_direction` for directional input, falling back to `items_room_all_then_inventory` (or `items_room_all_then_room` for disarm_trap).
- **[inventory-equipment.md](inventory-equipment.md)** — `items_inventory` and `items_equipped` building blocks. `exclude_worn` / `only_worn` kwargs forwarded to `caller.search`.
- **[vertical-movement.md](vertical-movement.md)** — height predicates (`p_same_height`, `p_different_height`, `p_same_height_value`) enforce vertical-position rules. Room barrier system (`visibility_up_barrier` / `visibility_down_barrier` on `RoomBase`) + object `size` attribute enforced via `p_height_visible_to`.

---

## Future Considerations

### `_get_all` and bulk-filter patterns

`get all` / `drop all` / `loot all` are filter-and-act loops, not targeting. They share predicates with the targeting library but their loop structure is different. Inline filter loops stay with the commands for now.

**Revisit when**: a second bulk-filter consumer appears with the same shape.

### `_get_by_token_id` and exact-identifier lookups

NFT pickup by integer ID is a different shape from keyword search — no string matching, no disambiguation. Currently inline in `cmd_get`.

**Open question**: should visibility filtering apply? Currently `_get_by_token_id` does NOT check hidden-mixin visibility — if you know the id you can get it. Adding `p_object_visible_to` would be a behaviour change.

**Revisit when**: the visibility question is answered or a third exact-id call site appears.

### Corpse looting

Corpses are NOT containers — they use `FungibleInventoryMixin` and their own `can_loot()` access gate. The `resolve_container` helper deliberately excludes them. A future `loot` command will need its own helper.

### Combat command migration

`cmd_bash`, `cmd_pummel` still call the priority resolvers directly. `cmd_attack` uses `resolve_target` but needs height checks at the command layer. Full migration pending weapon range attributes on weapon classes.

### Actor resolver refactor

`resolve_attack_target_in/out_of_combat` still have baked-in `p_object_visible_to` in their `bucket_contents` calls. This should be moved to `extra_predicates` for consistency with the "no baked-in predicates" principle — and the baked-in axis is the object one, so actor concealment is not checked during resolution today.

### Old helper retirement

`resolve_item_in_source`, `resolve_container`, `resolve_character_in_room`, `_resolve_world_item`, `_resolve_all_room` still exist. Some are still used by `cmd_get`, `cmd_put`, `cmd_give`. Retirement pending as those commands complete their building-block migration.

### `exclude_worn` as a predicate

`exclude_worn=True` is currently a `caller.search()` kwarg, not a predicate. A future `p_not_worn` predicate could make the filter stack visible at call sites. Low priority — the existing mechanism works.

### Broaden give to pets and mounts

`cmd_give` restricts targets to player characters via `p_is_character`. Pets and mounts are `DefaultCharacter` subclasses but not `FCMCharacter`, so they can't receive items. Future shape: `can_receive_gifts` class attribute + `at_pre_receive(giver, item)` hook.

**Revisit when**: pet/mount equipment system lands.

### medium enforcement

`medium = "air"` on spells is declared but not enforced at cast time. Validation will live in `base_spell.cast()` when the underwater casting system is designed.
