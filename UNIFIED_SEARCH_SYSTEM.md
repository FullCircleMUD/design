# Unified Search and Targeting System

> Technical design document for the MUD-wide target-resolution layer. Every command that takes a keyword-named target routes through this system. For spell-specific targeting rules see [SPELL_SKILL_DESIGN.md](SPELL_SKILL_DESIGN.md). For combat targeting see [COMBAT_SYSTEM.md](COMBAT_SYSTEM.md). For exit/door resolution see [EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md).

## The Trigger

A player typed `cast drain life bee-1` in a room called "Under Bee Tree". The spell command called `caller.search("bee-1")` with no `candidates=` argument. Evennia's default candidate list includes the room object itself, so the substring "bee" in "Under Bee Tree" matched the room's key. The spell received the room as its target and crashed in `take_damage()`.

This was not a spell bug. It was a **naming-resolution bug** that could reproduce in any command calling `caller.search()` without thinking carefully about scope. The stopgap fix — a dedicated `resolve_actor_target` helper in `spell_utils.py` — closed the immediate crash, but exposed a deeper problem: every command in FCM that takes a target keyword was implementing its own filter logic around bare `caller.search()`, and the logic drifted from command to command. Some filtered characters. Some didn't. Some respected hidden-object mixins. Some didn't. A single bug in one command hinted at a class of bugs spread across dozens.

## Why a Unified System

The real problem was not any single call site. It was **drift between implementations**. When every command implements its own "find a thing by name" logic, small inconsistencies become silent behavioural differences, and a fix in one command never propagates to the others. A new developer (human or AI) writing a new command starts by copying an existing call site, inheriting whatever filter semantics the donor happened to use, which may or may not be correct.

A unified targeting layer solves this by centralising the question "which object did the player mean?" into a small set of named, tested helpers. Every command that takes a target calls the same library. When a filter rule changes (e.g. hidden-object visibility gets a new mixin), one change to the library propagates to every consumer automatically. Call sites express **intent** (`resolve_container`, `resolve_item_in_source`), not implementation details. A future reader can see at a glance what a command is looking for without reverse-engineering a `candidates=` list.

This is not about building new search machinery. It is about making the machinery we already have **consistent** across the entire codebase.

## First Principle: Evennia-First

Before building anything new, check what Evennia already provides.

The earliest drafts of this design proposed a multi-layer custom targeting library: a `walk_scope` primitive, `scope_*` functions for every common scope, a custom `name_match` helper, and a full resolver catalog. A careful read of [evennia/objects/objects.py](../src/venv/Lib/site-packages/evennia/objects/objects.py) revealed that most of the supposed gaps don't exist. Evennia already supports scoped searches via `location=`, typeclass and tag filters, nick substitution, numeric disambiguation, lock filtering, dbref lookups, and special-keyword shortcuts (`me`/`self`/`here`). FCM's messy target-resolution code was largely a case of **not reading the signature** before writing custom helpers that reinvented Evennia's existing machinery.

This shaped the rule that every new library entry must pass: **does Evennia already do this?** If yes, call Evennia directly and don't wrap. If no, build the thinnest possible layer that adds what Evennia cannot know about — FCM-specific semantics (mixins, combat state, game rules) — and delegate everything else.

The rule lives in [CLAUDE.md](../src/game/CLAUDE.md) § Development Approach as a general development discipline. This library is the first worked example of applying it.

## What Evennia Already Provides

Cataloging what [`DefaultObject.search()`](../src/venv/Lib/site-packages/evennia/objects/objects.py#L732) does natively before specifying any custom layer. Everything in these tables is a one-line call to `caller.search()` with the right kwarg — no new code needed.

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

2. **Typed intent at call sites** — Evennia's `search()` always returns `Object | None | list[Object]`. From the call site you cannot tell whether the caller wanted a hostile actor, a held item, a lockable door, or an NPC. Type errors surface as runtime `AttributeError` downstream — which is exactly how the bee tree crash manifested. Named resolver functions close this gap at the architecture level.

3. **AoE multi-target shape (deferred)** — spells like Fireball and weapon moves like cleave need "a primary target plus a list of secondary affected objects" shaped by FCM rules (height, combat sides, PvP, friendly fire). Evennia's `search` is fundamentally single-target. This was in the original gap list; it is still a gap, but this cycle deferred it until the first AoE consumer justifies the shape. See Future Considerations.

## The Library We Built

### Module layout

```
src/game/utils/targeting/
    __init__.py          — empty exports (no API surface at the package level)
    predicates.py        — (obj, caller) -> bool filters
    helpers.py           — walk_contents + resolvers
```

### Predicate library

Twelve predicates. Each is a pure `(obj, caller) -> bool` function, 3–8 lines long, with its own unit test. Factory predicates return a closure bound to caller-specific state at creation time.

| Predicate | Purpose |
|---|---|
| `p_not_caller` | `obj is not caller` — identity compare. |
| `p_not_actor` | `not isinstance(obj, DefaultCharacter)` — excludes all actors (PCs, NPCs, mobs, pets, mounts). Vocabulary: "actor" = any DefaultCharacter subclass. |
| `p_is_character` | `isinstance(obj, FCMCharacter)` — matches player characters only, not NPCs/mobs/pets. |
| `p_not_exit` | `not isinstance(obj, DefaultExit)` — excludes exits from item candidates. |
| `p_visible_to` | Delegates to `obj.is_hidden_visible_to(caller)` when the `HiddenObjectMixin` method exists; returns True otherwise. |
| `p_living` | `hp > 0` — excludes items (hp=None), corpses (hp=0), dead mobs. Defensive on type. |
| `p_in_combat` | Has a `combat_handler` script attached. Combat-specific runtime-state filter. |
| `p_is_container` | `getattr(obj, "is_container", False)` — matches `ContainerMixin`. Corpses are NOT containers. |
| `p_passes_lock(lock_type)` | **Factory** returning a predicate wrapping `obj.access(caller, lock_type)`. |
| `p_same_height(caller)` | **Factory** returning a predicate matching actors at the caller's `room_vertical_position`. Used by melee-range spells so the resolver only considers actors the caster can physically reach. |
| `p_different_height(caller)` | **Factory** — inverse of `p_same_height`. For future `ranged_only` spells that can only target actors at a different height. |

Each predicate has a docstring explaining what Evennia handles natively instead. [predicates.py](../src/game/utils/targeting/predicates.py) also opens with a detailed comment block listing the dozen-plus filters a developer should implement with a native Evennia kwarg instead of adding a predicate.

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

### `BASE_ITEM_PREDICATES` constant

```python
BASE_ITEM_PREDICATES = (p_not_character, p_not_exit, p_visible_to)
```

Documents the universal "item-like" filter stack. Any lookup that wants "things in source.contents that could be items the caller might act on" combines this tuple with additional filters via `*BASE_ITEM_PREDICATES`. The constant has two real consumers today and documents the shared intent so future additions don't diverge.

### `resolve_item_in_source`

```python
def resolve_item_in_source(caller, source, search_term, **kwargs):
    """Find an item keyword inside a source object's contents."""
    candidates = walk_contents(caller, source, *BASE_ITEM_PREDICATES)
    if not candidates:
        return None
    return caller.search(search_term, candidates=candidates, **kwargs)
```

The primary item-identification helper. Source-agnostic — works for any object with `.contents` (room, container, caller's inventory, corpse, or any future source). Delegates string matching to `caller.search(candidates=..., **kwargs)`, inheriting all of Evennia's nick/dbref/multimatch/stacked/lock handling for free.

**Does NOT check `get` access lock**. That stays in the calling command so it can emit custom per-item error messages (`obj.db.get_err_msg`) instead of a generic "you don't see that". Same principle applies to any command-specific state gate: the library identifies; the command decides.

### `resolve_container`

```python
def resolve_container(caller, name):
    """Find a container by name — inventory first, then room."""
    for source in (caller, caller.location):
        if source is None:
            continue
        candidates = walk_contents(
            caller, source, *BASE_ITEM_PREDICATES, p_is_container,
        )
        if not candidates:
            continue
        result = caller.search(name, candidates=candidates, quiet=True)
        if result:
            if isinstance(result, list):
                result = result[0] if result else None
            if result is not None:
                return result
    return None
```

Container identification with a two-step source fallback: inventory first (the common case — "get sword from pack"), then the caller's current room (chests, trap chests, open world containers). Composes `BASE_ITEM_PREDICATES` with `p_is_container` to produce the full filter stack in one line.

**Does NOT check open/closed state.** `is_open` is a command-layer policy decision that varies per consumer:

- `get from chest` — needs the chest to be open
- `picklock chest` — thief is picking the lock; chest is closed by definition
- `smash chest` — barbarian; state irrelevant
- `look in chest` — maybe open, maybe not

Same identification-only principle as `resolve_item_in_source`'s `get` lock decision. Commands that need `is_open` check it themselves after calling the helper.

### Design decisions worth documenting

1. **Identification-only, not action-gating.** The library answers "which object did the player mean?" and stops. Every gate that depends on command intent (open/closed, access locks, hooks, weight checks) stays in the command layer. This keeps the helpers pure, testable, and reusable across commands with different action semantics.

2. **Delegate string parsing to `caller.search()`.** The library never reimplements nick substitution, dbref lookup, `<num>-<string>` disambiguation, `me`/`self`/`here` shortcuts, `stacked`, or `use_locks`. All of these come for free via `caller.search(candidates=..., **kwargs)`.

3. **`**kwargs` passthrough.** Helpers accept `**kwargs` and forward them to `caller.search`. Callers pass whatever Evennia kwargs they need (`stacked`, `quiet`, `nofound_string`, `multimatch_string`, `exclude_worn`, etc.) without the library needing to know about them.

4. **Fail-soft on missing source.** `None` source or missing `.contents` returns `None` / `[]`, never raises. Lets callers try multiple sources speculatively.

5. **No override of Evennia's `search()`.** An earlier draft proposed overriding `get_search_candidates` to strip the room from the default candidate list (closing the bee-tree footgun at the source). We rejected that approach: the typed resolvers make the footgun structurally impossible for every migrated call site, migration discipline is auditable via grep, and overriding `get_search_candidates` fights Evennia's intentional behaviour (`look here` needs the room in candidates).

6. **Predicate extraction is demand-driven.** Every predicate in the library was added when a concrete consumer needed it, not because it might be useful one day. The predicates.py docstring calls this out explicitly so future sessions maintain the discipline.

## Targeting vs Filtering

An important boundary the library intentionally does not cross:

- **Targeting** answers "which specific object did the player mean?" — returns ONE thing (or a small disambiguation set). Every resolver in this library does this.
- **Filtering** answers "which objects in this collection match some criterion?" — returns ALL matches; the caller acts on each in a loop.

Bulk-action patterns like `get all`, `drop all`, and `loot all` are filter-and-act loops, not targeting. They happen to share predicates with the targeting library (a `get all` wants the same living/non-exit/visible/gettable filter as `get sword`), but their loop structure is different. Forcing them into the resolver pattern would:

- Add a second pass (filter-then-act instead of filter-and-act-in-one)
- Blur the line between "identify a target" and "apply an action to everything matching"
- Fragment the library by adding "bulk" variants of every resolver

For now, bulk-action commands keep their inline filter loops. Predicates from this library are available for reuse if a command wants to compose the same filter stack, but the loop structure stays with the command. If a future second bulk-filter consumer appears with the same shape, the pattern gets extracted — probably into a sibling `utils/targeting/bulk.py` module to keep the target/filter boundary clear.

See **Future Considerations** at the end of this document for the specific deferred cases (`_get_all`, `_get_by_token_id`).

## Usage Catalog

Concrete call sites across the codebase today and their resolver mapping. This is the current reality, not an aspirational catalog — every row is either shipped or in flight.

### Priority-bucketed actor resolvers (shipped)

The targeting library provides priority-bucketed resolvers for actor targeting. Each resolver uses the same classifier (buckets actors by relationship to the caller) with a parameterised priority order. The `order` kwarg defaults to hostile priority; friendly wrappers pass a reversed tuple.

**In-combat** — classifier reads `combat_side` from `obj.scripts.get("combat_handler")[0]`:

| Resolver | Order (first wins) | Consumers |
|---|---|---|
| `resolve_attack_target_in_combat` | enemy > bystander > ally > self | cmd_attack, cmd_bash, cmd_pummel, hostile/any_actor spells |
| `resolve_friendly_target_in_combat` | self > ally > bystander > enemy | friendly spells |

**Out-of-combat** — classifier reads group membership via `get_group_leader()`:

| Resolver | Order (first wins) | Consumers |
|---|---|---|
| `resolve_attack_target_out_of_combat` | stranger > groupmate > self | same as above |
| `resolve_friendly_target_out_of_combat` | self > groupmate > stranger | same as above |

Both resolver pairs accept `extra_predicates=()` — additional predicates appended to the `walk_contents` predicate stack. Used by `resolve_spell_target` to add height filtering based on `spell_range` (see below).

The `self` bucket is intentionally last-priority for hostile and first-priority for friendly. The `_is_self_keyword` helper intercepts `"me"` / `"self"` keywords at the top of each resolver to bypass an Evennia direct-match quirk that corrupts `searchdata` when the caller is not in the candidate list.

### Universal target resolver — `resolve_target` (shipped)

All targeting in the game routes through a single universal entry point:

```python
resolve_target(caller, target_str, target_type, range="ranged", aoe=None)
```

Lives in `utils/targeting/helpers.py`. Returns `(target, secondaries)` — a tuple of the primary target and a list of AoE secondary targets. For non-AoE callers, `secondaries` is always `[]`. On failure, returns `(None, [])` with an error message already sent to the caller.

**Consumers**: `cmd_cast`, `cmd_zap` (all spells), `cmd_attack` (combat POC), `cmd_get` (item pickup POC). Any future consumer — weapon abilities, mob AI, non-spell commands — can use the same function with the same flags.

**Target types — actors** (`actor_` prefix):

| Type | Resolution | Self policy | Consumers |
|---|---|---|---|
| `actor_hostile` | Attack-priority resolvers (enemy > bystander > ally > self) | Returned — command decides | cmd_attack, cmd_bash, cmd_pummel, hostile/any_actor spells |
| `actor_any` | Same as hostile | Returned — command decides | augur, divine_scrutiny |
| `actor_friendly` | Friendly-priority resolvers (self > ally > bystander > enemy) | Allowed (empty target defaults to self) | friendly spells |

Self-rejection is NOT done in `resolve_target` — the caller is returned via the self-bucket fallback so each command can emit its own context-specific error ("You can't attack yourself" vs "You can't target yourself with that spell").

**Target types — items** (`items_` prefix, composable naming):

| Type | Resolution | Consumers |
|---|---|---|
| `items_inventory` | Inventory only | Create Water |
| `items_gettable_room` | Room items (BASE_ITEM_PREDICATES — excludes actors, exits, hidden) with height filtering | cmd_get |
| `items_all_room_then_inventory` | Room (all visible objects + exits) first, inventory fallback | Knock |
| `items_inventory_then_all_room` | Inventory first (exclude worn), room (objects + exits) fallback | Identify, Holy Insight |

**Convention-defined, not yet implemented** (raise `NotImplementedError`): `items_all_room`, `items_fixed_room`, `items_gettable_room_then_inventory`, `items_inventory_then_gettable_room`.

**Meta types**: `self` (returns caller), `none` (returns None — used by targetless spells like teleport, summons).

### Height integration via `range` parameter (shipped)

The `range` parameter on `resolve_target` controls vertical-position filtering for both actor AND item target_types:

| range | Height filter | Current consumers |
|---|---|---|
| `"melee"` | `p_same_height` — same height only | melee spells (cure_wounds, vampiric_touch), cmd_get (items), cmd_attack (melee weapons) |
| `"ranged"` | none — any height | ranged spells (default), cmd_attack (ranged weapons) |
| `"ranged_only"` | `p_different_height` — different height only | no current consumer (built for future use) |
| `"self"` | N/A — no target resolution | self-buff spells |

Height filtering is enforced at resolution time via `extra_predicates` on the priority resolvers AND on item resolution branches. A melee spell or melee weapon attack with two same-named targets at different heights picks the one at the caller's height. A player on the ground can't pick up items at flying height.

For combat commands, weapon range is mapped at the call site: `weapon_type == "missile"` → `"ranged"`, everything else → `"melee"`. TODO: when weapons gain a universal `range` attribute (matching spells), this mapping goes away.

### AoE secondaries system (shipped)

The `aoe` parameter on `resolve_target` drives AoE secondary target collection from the primary target's height:

| aoe | Secondaries | Consumers |
|---|---|---|
| `None` | `[]` — no cascade (default) | all single-target spells, combat commands, item commands |
| `"unsafe"` | Everyone at target's height, including caster | Fireball |
| `"unsafe_all_heights"` | Everyone in room regardless of height, including caster | Call Lightning |
| `"unsafe_self"` | Everyone at target's height except caster | Soul Harvest |
| `"safe"` | Enemies only at target's height (get_sides in combat, group membership out of combat) | Cone of Cold, Flame Burst |
| `"allies"` | Allies only at target's height, caster included | future Mass Heal |

The primary target is always excluded from secondaries (they're already the primary). AoE spells receive `(target, secondaries)` and iterate the list with spell-specific logic (saves, diminishing accuracy, damage application, condition effects).

The enemy detection for "safe" AoE out of combat uses group membership (same as the priority resolvers) — NOT the broken `not isinstance(FCMCharacter)` typeclass proxy. Mobs casting safe AoE spells against player characters use the exact same targeting path.

### Spell base class attributes (shipped)

Every spell declares four targeting-relevant attributes:

```python
class Spell:
    target_type = "actor_hostile"   # which resolve_target branch
    range = "ranged"                # height filtering
    aoe = None                      # AoE cascade type
    medium = "air"                  # environment restriction (not yet enforced)
```

`medium` is declared but not enforced — defaults to `"air"` so nothing accidentally works underwater before the underwater system is designed. Cast-time validation will live in `base_spell.cast()`, not in `resolve_target` (medium gates whether the spell fires at all, not who the target is).

### Item pickup (shipped — cmd_override_get)

```python
# _get_object — standard NFT pickup by keyword
objs = resolve_item_in_source(
    caller, caller.location, search_term, stacked=self.number
)

# Multi-word probe — detect "is this an item name or a container split?"
obj = resolve_item_in_source(
    caller, caller.location, self.args.strip(), quiet=True
)

# _find_container — identify a container by name, handles error messages
container = resolve_container(caller, name)
# ... command-layer is_open check and fallback disambiguation ...

# _try_split_container — silent probe for the container-split parse path
container = resolve_container(caller, container_word)
# ... command-layer is_open check ...

# _get_object_from_container — find an item inside a resolved container
obj = resolve_item_in_source(
    caller, container, search_term,
    nofound_string=f"{container.key} doesn't contain '{search_term}'.",
)
```

Five distinct call sites, one library, zero duplication of filter logic. `_find_container` even composes three library calls for a single command-layer task (primary lookup + two error-disambiguation fallbacks).

### Item drop (shipped — cmd_override_drop)

```python
# _drop_object — inventory keyword search, exclude_worn forwarded
# via **kwargs to FCMCharacter.search
objs = resolve_item_in_source(
    caller, caller, search_term,
    nofound_string=..., multimatch_string=...,
    stacked=self.number,
    exclude_worn=True,
)
```

### Item give (shipped — cmd_override_give)

```python
# Target lookup — find a player character in the room
target = resolve_character_in_room(caller, self.rhs)
if not target:
    caller.msg(f"You don't see a character called '{self.rhs}' here.")
    return
if target == caller:
    caller.msg("You can't give things to yourself.")
    return

# Item lookup — identical to cmd_drop, including exclude_worn
to_give = resolve_item_in_source(
    caller, caller, search_term,
    nofound_string=..., multimatch_string=...,
    stacked=self.number,
    exclude_worn=True,
)
```

`resolve_character_in_room` is the first consumer of the new `p_is_character` predicate — the library's entry-point for character-targeting resolvers. Scope-suffixed name (`_in_room`) distinguishes it from future scope variants (`_in_game`, `_in_zone`, `_in_district`) which will use different execution models.

### Commands migrated to `resolve_target`

| Command | target_type | range | aoe | Notes |
|---|---|---|---|---|
| cmd_cast (all spells) | per spell | per spell | per spell | Full migration — every spell routes through resolve_target |
| cmd_zap (wand spells) | per spell | per spell | per spell | Same path as cmd_cast |
| cmd_attack | `actor_hostile` | weapon-dependent | `None` | POC — weapon range mapped at call site |
| cmd_bash | `actor_hostile` (via direct resolvers) | N/A | N/A | Uses priority resolvers directly, pending resolve_target migration |
| cmd_pummel | `actor_hostile` (via direct resolvers) | N/A | N/A | Same as cmd_bash |
| cmd_get | `items_gettable_room` | `melee` | `None` | POC — height filtering on item pickup |

### Commands using targeting primitives directly (not yet via resolve_target)

- **cmd_bash, cmd_pummel** — call `resolve_attack_target_in/out_of_combat` directly. Will migrate to `resolve_target` when weapon range attributes land on weapon classes.
- **cmd_hold, cmd_wear, cmd_remove** — call `resolve_item_in_source` directly with `exclude_worn` / `only_worn` kwargs.
- **cmd_drop** — calls `resolve_item_in_source` directly.
- **cmd_give** — calls `resolve_character_in_room` + `resolve_item_in_source` directly.
- **cmd_put** — calls `resolve_container` + `resolve_item_in_source` directly.

### Commands on legacy paths (not yet using targeting library)

- **Exits and doors** — `cmd_unlock`, `cmd_open`, `cmd_close`, `cmd_smash`, `cmd_picklock`, `cmd_disarm_trap` use `utils/find_exit_target.py`. Migration candidate for a future exit-resolution primitive with directional parsing.

## Migration Progress

| Commit | What shipped |
|---|---|
| `9baec37` | **Stopgap** — `resolve_actor_target` in spell_utils, wired into `cmd_cast` + `cmd_zap`. Closes the bee-tree crash immediately. |
| `c423a15` | **Library foundation** — `utils/targeting/` package with 5 predicates, `resolve_item_in_source` helper, and 24 unit tests. Migrates `cmd_override_get._get_object` and its multi-word probe. |
| `33a4053` | **`walk_contents` primitive + `resolve_container`** — generalises the filter stack into a varargs primitive, introduces `BASE_ITEM_PREDICATES`, adds `p_is_container`, and ships `resolve_container` with inventory-first fallback. |
| `19c4829` | **`cmd_get` from-container migration** — wires `_find_container`, `_try_split_container`, and `_get_object_from_container` to the new helpers. |
| `a8f2ed7` | **`cmd_drop._drop_object` migration** — one-line swap to `resolve_item_in_source` with `exclude_worn` forwarded via `**kwargs`. |
| `db40670` | **Vocabulary cleanup + `resolve_character_in_room`** — renames `p_not_character` → `p_not_actor` (the predicate filters any DefaultCharacter, which is an "actor", not a "character"). Adds `p_is_character` for FCMCharacter specifically. Adds `resolve_character_in_room` helper. Deletes `p_not_caller` (YAGNI — no consumer). |
| `7a867cb` | **`cmd_give` target + item migration** — replaces the worst footgun shape yet (bare `caller.search(self.rhs)` with no scope). Uses `resolve_character_in_room` for the target and `resolve_item_in_source` for the item. Drops the now-redundant `isinstance(target, FCMCharacter)` check. |
| `8d5bf33` | **Direct `walk_contents` tests + predicate boundary tests** — test-coverage audit found that the core primitive had no direct tests and that `p_is_character` had no test against `DefaultCharacter` (the critical boundary). 9 `TestWalkContents` tests + 2 boundary tests added. |
| `7dc83aa` | **cmd_give target-resolution gap-fill tests** — 3 new tests in `TestCmdGiveObject` for self-give on the object path, target-not-found, and non-character target (e.g. room scenery object). Locks the new error wording and the `p_is_character` boundary at the command level. |

**Test coverage**: 166/166 green across the full `tests.utils_tests` suite. 20/20 green in `test_cmd_give`, 23/23 in `test_cmd_get`, 15/15 in `test_cmd_drop`. No regressions introduced by any migration.

**Migration discipline**: every cycle follows the same pattern — plan the scope, build the helper(s), test the helper(s), migrate the consumer, rerun the consumer's existing test suite, commit separately. No speculative additions, no resolvers written ahead of concrete consumers. After each consumer migration, audit the consumer's test suite for gaps the migration exposed and fill them before moving on.

## Testing Strategy

Three tiers, each answering one question.

### Tier 1 — predicate unit tests

One test class per predicate. Each test uses a `SimpleNamespace` or `MagicMock(spec=...)` fake — no Evennia DB, no fixtures beyond the `EvenniaTest` base class (needed for the `isinstance` check against `DefaultCharacter` / `DefaultExit`). Typically 2–3 assertions per predicate covering true case, false case, and one edge case (missing attribute, None, etc.).

[tests/utils_tests/test_targeting_predicates.py](../src/game/tests/utils_tests/test_targeting_predicates.py) — 13 tests across the 6 predicates.

### Tier 2 — helper unit tests

One test class per resolver. Tests cover:

- Happy path (matching item in source → returned)
- None sources (missing contents, empty contents, None source → None)
- Each predicate filter (caller, character, exit, hidden → excluded)
- Each source variant (container, inventory, room)
- Kwarg forwarding (`stacked`, `quiet`, `nofound_string`)
- Source fallback precedence (for `resolve_container`: inventory wins over room)
- Non-gating behaviour (for `resolve_container`: closed containers still returned)

[tests/utils_tests/test_targeting_helpers.py](../src/game/tests/utils_tests/test_targeting_helpers.py) — 22 tests across `resolve_item_in_source` and `resolve_container`.

### Tier 3 — consumer regression tests

Every command migration runs the command's existing test suite as the regression gate. If the migration changes observable behaviour, an existing test fails and we debug. If every test stays green, the migration preserved player-facing semantics. [tests/command_tests/test_cmd_get.py](../src/game/tests/command_tests/test_cmd_get.py) — 23 tests covering all migrated code paths in `cmd_override_get.py`.

## Relationship to Other Systems

- **[SPELL_SKILL_DESIGN.md](SPELL_SKILL_DESIGN.md)** — all spell target types now route through `resolve_target` with `actor_*` / `items_*` flags. The old `resolve_actor_target` stopgap is retired.
- **[COMBAT_SYSTEM.md](COMBAT_SYSTEM.md)** — `combat.combat_utils.get_sides()` was rebuilt on `bucket_contents` and is used by `_resolve_aoe_secondaries` for in-combat safe/allies AoE. `cmd_attack` routes through `resolve_target`.
- **[EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md)** — `utils/find_exit_target.py` is still used by the `items_all_room_then_inventory` branch (Knock) via `_resolve_world_item`. Migration candidate for a future exit-resolution primitive.
- **[INVENTORY_EQUIPMENT.md](INVENTORY_EQUIPMENT.md)** — wearslot-aware resolvers (`resolve_held_item`, `resolve_worn_item`) are deferred until first concrete consumer.
- **[VERTICAL_MOVEMENT.md](VERTICAL_MOVEMENT.md)** — height predicates (`p_same_height`, `p_different_height`, `p_same_height_value`) enforce the vertical-position rules. Height filtering is structural for both actor targeting (spells + combat) and item targeting (cmd_get).

---

## Future Considerations

This section captures deferred work known to the current design. Each item was considered during the current migration cycle and consciously not built yet, with a note explaining why. These are candidates for future sessions to revisit when a concrete consumer justifies the work.

### `_get_all` and bulk-filter patterns

[cmd_override_get.py:_get_all](../src/game/commands/all_char_cmds/cmd_override_get.py) walks `room.contents` with an inline filter stack (not caller, not exit, not character, visible, has get access, passes `at_pre_get`, passes weight check) and moves each passing object into the caller's inventory. The first four filters are exactly `BASE_ITEM_PREDICATES + p_passes_lock("get")`, so the migration would look clean on paper.

It was deferred this cycle because **this is filtering, not targeting**. The Targeting vs Filtering section above explains the distinction: `get all` does not identify a target; it applies an action to every object matching a criterion. Forcing it through a targeting resolver would add a second pass (filter-then-act instead of filter-and-act-in-one), blur the library's boundary, and not meaningfully improve reuse because the only similar consumers today (`_drop_all`, `_get_all_from_container`) need different filter stacks.

**Revisit when**: a second bulk-filter consumer appears with the same shape (likely `_drop_all` if its `exclude_worn` logic can be unified with the predicate stack). At that point a sibling `utils/targeting/bulk.py` module with a `filter_contents` function (or similar) might earn its place. Until then, `_get_all`'s inline loop is fine where it is.

### `_get_by_token_id` and exact-identifier lookups

[cmd_override_get.py:_get_by_token_id](../src/game/commands/all_char_cmds/cmd_override_get.py) (and its container sibling `_get_by_token_id_from_container`) walks `source.contents` looking for an NFT with a specific integer ID. Currently inline, ~8 lines in two places.

This is a different shape from keyword search: there is no string to match, no disambiguation, no fuzzy matching. A dedicated helper might look like:

```python
def resolve_by_id(caller, source, item_id, typeclass=None):
    for obj in source.contents:
        if isinstance(obj, typeclass or BaseNFTItem) and obj.id == item_id:
            return obj
    return None
```

Or, more in line with the library's style:

```python
def p_has_id(item_id):
    def _pred(obj, caller):
        return getattr(obj, "id", None) == item_id
    return _pred

# Consumer:
matches = walk_contents(caller, source, p_is_typeclass(BaseNFTItem), p_has_id(42))
obj = matches[0] if matches else None
```

**Open questions before migrating**:

1. **Should visibility filtering apply?** Currently `_get_by_token_id` does NOT check hidden-mixin visibility — if you know the id you can get it. Is that intentional (you saw the id via some inspection command, so you know it exists) or an oversight? Adding `p_visible_to` to a future `resolve_by_id` would be a behaviour change that needs a design call, not just a refactor.

2. **Does exact-id lookup even belong in a string-matching-adjacent library?** The current library's identity is "unified keyword → object resolution". Exact-ID lookup is a different problem and might belong in its own tiny module. Or it might be fine as a `walk_contents` user with a custom predicate. Deferred pending a conversation.

**Revisit when**: either the visibility question is answered (ideally via a game design decision, not a refactor), or a third exact-id call site appears and forces the pattern into existence.

### AoE multi-target (shipped)

All five implemented AoE spells now route through `resolve_target` with a primary target + `_resolve_aoe_secondaries` for the cascade:

| Spell | aoe type | Behavior |
|---|---|---|
| Fireball | `"unsafe"` | Everyone at target's height, caster included. DEX save for half. |
| Call Lightning | `"unsafe_all_heights"` | Everyone in room regardless of height. DEX save for half. |
| Soul Harvest | `"unsafe_self"` | Everyone at target's height except caster. CON save for no damage. Undead filtered in _execute. Caster heals for total damage. |
| Cone of Cold | `"safe"` | Enemies only at target's height. Primary guaranteed hit, secondaries have diminishing accuracy (80/60/40/20%). SLOWED condition. |
| Flame Burst | `"safe"` | Same pattern as Cone of Cold but fire damage, no condition. |

The legacy `get_room_all` / `get_room_enemies` helpers are no longer called by any migrated spell. They remain in `spell_utils.py` referenced by stub comments (Mass Confusion, Dimensional Lock, Antimagic Field) and can be deleted when those stubs are implemented against the AoE framework.

The broken out-of-combat enemy heuristic (`not isinstance(FCMCharacter)`) is fixed for AoE: `_resolve_aoe_secondaries` uses group membership for safe AoE out of combat, matching the priority resolvers. Mobs casting safe AoE against player characters use the exact same code path.

### `get_sides` rebuild (shipped)

`combat.combat_utils.get_sides(combatant)` was rebuilt to use `bucket_contents` with a classify closure that reads `combat_side` from each actor's combat handler. Single-loop implementation: R + C script-handler lookups (down from R + 3C in the original 3-loop version). Signature unchanged — 27+ consumers still get `(allies, enemies)` tuples with zero code changes.

### Corpse looting

Corpses are NOT containers in FCM — [Corpse](../src/game/typeclasses/world_objects/corpse.py) uses `FungibleInventoryMixin` and its own `can_loot()` access gate, not `ContainerMixin` / `is_open`. The `resolve_container` helper deliberately excludes them.

When a `loot` command is migrated, it will need its own helper (possibly `resolve_lootable`) that understands the corpse-specific gate. The `walk_contents` primitive and `BASE_ITEM_PREDICATES` stack can both be reused — only a new predicate (`p_can_loot`?) and a new resolver are needed.

**Revisit when**: the first corpse-looting command is touched.

### `exclude_worn` as a predicate

FCMCharacter.search currently handles `exclude_worn=True` as a kwarg filter inside the string-matching pass. Both `cmd_override_drop._drop_object` and `cmd_override_give._give_object` forward this kwarg unchanged through `resolve_item_in_source`'s `**kwargs` to `caller.search`. It works but hides the semantic intent — a reader of either command has to know about the FCMCharacter.search extension to understand what the filter does.

A future cleanup could add a `p_not_worn` predicate to the library, letting drop and give commands compose `*BASE_ITEM_PREDICATES, p_not_worn` explicitly at the call site. This would make the filter stack visible in the command code rather than hidden in a search kwarg. Low priority — the existing mechanism works, has two consumers, and both forward the kwarg correctly.

**Revisit when**: the filter becomes complex enough that call-site visibility matters, or the FCMCharacter.search extension is otherwise refactored.

### Broaden give to pets and mounts

`cmd_give` restricts targets to player characters only via `resolve_character_in_room` + `p_is_character`, which matches `FCMCharacter` exclusively. This correctly excludes NPCs, mobs, quest NPCs, shopkeepers, and hostile creatures — you can't give a sword to a goblin or lose a quest item by handing it to the wrong NPC outside the quest flow.

But this also blocks legitimate future use cases: **putting a saddle on a horse, panniers on a mule, a coat on a dog, barding on a warhorse**. Pets and mounts are `DefaultCharacter` subclasses but NOT `FCMCharacter`, so they currently fail `p_is_character` and can't receive items via give.

The likely future shape: add an opt-in mechanism — probably a `can_receive_gifts` class attribute defaulting to `False`, overridden to `True` on specific pet/mount base classes, plus an optional `at_pre_receive(giver, item)` hook for item-specific refusal (e.g. a horse accepts a saddle but rejects a longsword). A new helper — perhaps `resolve_gift_recipient_in_room(caller, name)` — composes a new `p_can_receive_gifts` predicate with `p_not_actor` inversion, and `cmd_give` calls it instead of `resolve_character_in_room`. Quest NPCs explicitly stay off the opt-in so their quest commands remain the canonical turn-in path.

**Revisit when**: the first pet or mount equipment system lands and needs give-command integration. Needs a game-design decision before implementation:

- Which existing NPC types should default to `can_receive_gifts = True`? (Probably none initially.)
- How do shopkeepers interact — should `give` to a shopkeeper route to `sell`, emit a hint, or reject silently?
- How do quest NPCs communicate their acceptance rules — one-size-fits-all attribute, or per-quest-state check via `at_pre_receive`?
- Is there a world-wide audit command worth shipping so devs can see which NPCs currently accept gifts?

None of these need answering until the first real consumer forces the question.
