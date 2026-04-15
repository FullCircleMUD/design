# Unified Search and Targeting System

> Technical design document for the MUD-wide target-resolution architecture. Every command that takes a keyword-named target — spells, combat, movement, item manipulation, NPC interaction, skill use — resolves that keyword through this system. For spell-specific targeting rules see [SPELL_SKILL_DESIGN.md](SPELL_SKILL_DESIGN.md). For combat targeting see [COMBAT_SYSTEM.md](COMBAT_SYSTEM.md). For exit/door resolution see [EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md).

## Design Philosophy

Every interactive MUD command eventually has to answer the same question: **"the player typed this keyword — which game object did they mean?"** Today the codebase answers that question in dozens of different places, each with its own ad-hoc combination of `caller.search()`, manual candidate lists, typeclass checks, and downstream error handling. The inconsistencies cause real bugs (the `cast drain life bee-1` crash resolved a wand target to the room object named "Under Bee Tree") and make it hard to reason about what "targeting" actually means for a given command.

This document defines a **single unified targeting layer** that:

1. **Leans on Evennia's built-in `search` infrastructure** for everything it already does well — string parsing, multimatch disambiguation, dbref lookups, nick substitution, special keywords (`me`/`self`/`here`), lock filtering, and stacking.
2. **Adds a thin composable layer on top** for everything Evennia doesn't cover — semantic predicates, scope selection, typed resolvers, and multi-target (AoE-shaped) result lists.
3. **Closes Evennia's one footgun** (the default candidate list includes the room itself) with a single narrow override of `get_search_candidates()`.
4. **Exposes typed, named resolvers** that every command uses. `resolve_hostile_actor(caller, "goblin")` returns an enemy actor or `None`. `resolve_held_item(caller, "wand")` returns an item held in the caller's HOLD slot or `None`. `resolve_unlockable_door(caller, "iron door")` returns an `ExitDoor` with a lock or `None`. Call sites express intent; the library guarantees the return shape.
5. **Guarantees O(n) or better** in the number of candidates for every resolution, with short-circuit predicate evaluation and at most two room walks for AoE-shaped queries.
6. **Is open to extension.** New predicates, new scopes, and new resolvers can be added by any subsystem without touching the core primitive.

The end state is that **no game command ever calls `caller.search()` directly**. Every target resolution flows through a named resolver whose return type is obvious from its name.

## Problem Statement

### The immediate trigger: the bee tree crash

A player in a room called "Under Bee Tree" typed `cast drain life bee-1`. The `cmd_cast` handler called `caller.search("bee-1")`. With no explicit `candidates=` list, Evennia's default candidate builder included the room itself — and when no bee matched the exact string, the room's key "Under Bee Tree" matched the substring "bee" and won the search. `cmd_cast` passed the room object to `spell.cast()`, the spell called `target.take_damage()`, and the game crashed with `AttributeError: 'RoomBase' object has no attribute 'take_damage'`.

This is not a spell bug. It is a **naming-resolution bug** that can reproduce in any command that trusts `caller.search()` to return "a thing the player can meaningfully act on". Every command that takes a target keyword is potentially affected.

### The shape of the broader problem

Every interactive command in the MUD solves the same four-step puzzle:

1. **Scope** — what pool of objects should be considered? (The room? The caller's inventory? The caller's equipped gear? Visible exits? Everyone in the room including the room itself? A union of those?)
2. **Filter** — which objects in that pool are semantically valid for this command? (Living actors for `attack`, lockable objects for `unlock`, gettable items for `get`, items in the caller's HOLD slot for `zap`, etc.)
3. **Match** — which of the filtered candidates best matches the player's typed keyword? (Substring, alias, multimatch disambiguation, `<num>-<string>`, `#dbref`, `me`/`self`/`here`.)
4. **Consequence** — what happens on success, failure, or ambiguity? (Execute the action; tell the player "you don't see X here"; tell the player "which X did you mean?")

Evennia's `caller.search()` collapses steps 1–3 into a single call, but does so with one fixed scope (caller + contents + location + location.contents) and zero semantic filtering. Commands that need something different — which is almost all of them — must either build their own candidate list and pass it as `candidates=`, or inherit the default and risk the footgun.

Today the codebase does both. Some commands (the ones we've fixed) build explicit candidate lists. Most commands don't. The ones that don't are one unlucky room name away from the bee tree crash.

### Specific failure modes we want to eliminate

- **Room-as-target**: `caller.search("name")` returning the room because the room's key substring-matches the keyword.
- **Fixture-as-target**: `caller.search("chest")` returning a scenery object with `hp=None` when the command expected a living actor.
- **Corpse-as-target**: combat commands targeting the corpse left over from a previous kill because the corpse still has the mob's key.
- **Silent cross-type success**: `cmd_give character_name item` where `character_name` matches an item in the room, which then gets "given" to the item (or crashes).
- **Bypassed visibility**: `caller.search` returning hidden or invisible objects that the player shouldn't be able to see.
- **Inconsistent multi-match behavior**: some commands use `caller.search(quiet=True)` and auto-pick the first result; others prompt the player; others error. No consistency across the codebase.
- **Bypassed PvP/grief protection**: unsafe AoE spells hit every actor in the room with `hp > 0`, including innocent bystanders, because there is no shared abstraction for "who should be hittable by AoE from this caster in this room".
- **Bypassed height/vertical rules**: a ground-level ranged attack can hit a flying target with no logic to filter by vertical position.

These are not unrelated bugs. They are symptoms of the same missing abstraction.

## What Evennia Already Provides

Evennia's `DefaultObject.search()` ([`evennia/objects/objects.py`](../src/venv/Lib/site-packages/evennia/objects/objects.py), line 732) is a surprisingly capable primitive. Before building anything new, this design fully catalogs what it offers so we only build what Evennia doesn't.

### `caller.search()` capabilities

| Feature | What it does |
|---|---|
| `candidates=[...]` | Restrict search to a pre-built list instead of the default scope. The primary escape hatch for semantic filtering. |
| `location=<obj>` | Restrict to contents of a given location. |
| `typeclass=...` | Match only objects of a given typeclass (or list). |
| `attribute_name=...` | Search a specific attribute instead of key + aliases. |
| `tags=[...]` | Match only objects bearing one or more tags. |
| `quiet=True` | Suppress default error messages; return a list instead of a single object. Caller handles all messaging. |
| `exact=False` (default) | Prefix / substring matching. `exact=True` requires a full-string match. |
| `use_nicks=True` | Apply the player's object-level nick substitutions before matching. Free alias support. |
| `use_locks=True` | Apply the `search` lock — objects with `search:false()` are hidden from non-staff. |
| `stacked=N` | If multiple matches are identical (3 copies of the same potion), treat as a stack and return up to N matches. |
| `global_search=True` | Search the entire DB. |
| `use_dbref=True/False/None` | Allow `#123` dbref lookups; auto-enabled for Builders. |
| `nofound_string` / `multimatch_string` | Override the default error message strings. |

And free-of-charge, not as parameters but as built-in behavior:

- **Special keywords**: `me`, `self`, and `here` resolve without hitting the candidate list.
- **Multimatch disambiguation**: `<num>-<string>` parsing via `settings.SEARCH_MULTIMATCH_REGEX`. A player can type `2-goblin` to select the second goblin in a list. Evennia parses this for free.
- **Multimatch error formatting**: the `at_search_result` hook builds the `goblin-1 / goblin-2 / goblin-3` disambiguation list when the player's keyword matches multiple objects.
- **Input normalization**: whitespace stripping, case-insensitivity.

### Overridable sub-methods

Critically, Evennia split `search()` into clean overridable sub-methods ([`evennia/objects/objects.py`](../src/venv/Lib/site-packages/evennia/objects/objects.py)):

| Sub-method | Role |
|---|---|
| `get_search_query_replacement` | Preprocessing hook for the search string. |
| `get_search_candidates` | Builds the default candidate list when none is passed. **This is where the footgun lives.** |
| `get_search_direct_match` | Handles `me`, `self`, `here` keywords before hitting candidates. |
| `get_search_result` | Runs the actual DB query against the candidate list. |
| `at_search_result` | Formats multimatch / not-found messages for the caller. |

Because these are split cleanly, we can fix the footgun with a **single narrow override of `get_search_candidates`** without touching `search()` itself, without changing any signature, and without affecting any call site that passes `candidates=` explicitly.

### Evennia helpers the system should lean on inside predicates and scopes

Where it makes sense, predicates and scopes should delegate to Evennia primitives instead of reinventing them:

| Task | Evennia primitive |
|---|---|
| Lock check (get, use, examine, search) | `obj.access(searcher, "get")`, `obj.access(searcher, "search")`, etc. |
| Tag membership | `obj.tags.has(tagname, category=...)` |
| Alias lookup | `obj.aliases.all()`, `obj.aliases.get(...)` |
| Type check | `obj.is_typeclass(path, exact=False)` |
| Script lookup (combat handler, etc.) | `obj.scripts.get("script_key")` |
| Attribute read | `obj.attributes.get(...)`, `obj.db.something` |
| Lock check, generic | `obj.locks.check(searcher, "lock_type")` |
| Exits of a room | `room.exits` (Evennia provides a filtered view of `contents`) |
| Contents of a container | `obj.contents` |
| Nick substitution | `use_nicks=True` on the underlying `search` call — free |

The targeting library **never reimplements any of these**. It wraps them in named predicates so call sites read like English.

## The Gaps

With Evennia's capabilities catalogued, the gaps fall out cleanly:

### Gap 1: The footgun

`get_search_candidates` explicitly includes `self.location` (the room) in the default candidate list ([line 608](../src/venv/Lib/site-packages/evennia/objects/objects.py#L608)) so that `look here` can resolve the room as a target. This is deliberate for the `look`/`examine` use case. It is dangerous for every other use case.

### Gap 2: No semantic predicates

`caller.search` can filter by typeclass, tags, attribute name, or lock, but it cannot express:

- "Living (hp > 0) actors"
- "Actors not including the searcher themselves"
- "Actors at the searcher's current vertical height"
- "Objects visible to the searcher given hidden/invisible mixin rules"
- "Items in the caller's HOLD wearslot"
- "Lockable exits in the current room"
- "Enemies from the caller's combat side"
- "Allies from the caller's follow-chain group"
- "Targets that pass the PvP rules for this room"

Every one of these can be expressed as a tiny predicate. None of them exist in Evennia.

### Gap 3: No typed resolvers

`caller.search()` always returns `Object | None | list[Object]`. There is no way to say "this call will return a `LivingActor` or `None`" at the call site. Type errors surface as runtime `AttributeError` downstream — exactly how the bee tree crash manifested. A typed resolver layer closes this gap at the architecture level, not just through discipline.

### Gap 4: No multi-target shape

AoE spells, cleave attacks, mass heals, and group buffs all need "a primary target plus a list of secondary affected objects". `caller.search()` is fundamentally single-target. The current codebase builds multi-target lists via parallel helpers (`get_room_enemies`, `get_room_all`, `get_room_enemies_at_height`) that duplicate iteration logic and bypass single-target validation entirely.

### Gap 5: No shared scoping abstraction

Commands today express scope through whatever combination of Evennia helpers fits: `caller.contents`, `caller.location.contents`, `caller.location.exits`, manual unions of these, or passing `location=` and hoping for the best. There is no shared vocabulary for "this scope" vs "that scope", and no way to compose scopes (e.g. "room contents OR inventory") without reimplementing it every time.

### Gap 6: FCM-specific rules are scattered

Height filtering lives in [spell_utils.py](../src/game/world/spells/spell_utils.py) AoE helpers. PvP rules live implicitly in [combat_utils.py:get_sides](../src/game/combat/combat_utils.py). Combat side detection lives in `get_sides`. Group membership lives on `FCMCharacter`. None of these are available to commands that don't already know about the right helper. A player casting `fireball bystander` in a non-PvP inn has no shared enforcement point.

## Architecture

### Five layers

The system is intentionally layered so that each layer has one responsibility and can be tested in isolation.

```
Layer 5 — Typed resolvers
         resolve_hostile_actor(), resolve_held_item(), resolve_unlockable_door(), ...
         Named by intent; return exactly the type they promise.

Layer 4 — Predicate library (core + domain)
         p_living, p_visible, p_at_height, p_is_enemy, p_in_group,
         p_lockable, p_wieldable, p_gettable, p_has_account, ...
         Pure (obj, searcher) -> bool functions.

Layer 3 — Scope functions
         scope_room(searcher), scope_inventory(searcher), scope_exits(searcher),
         scope_room_and_inventory(searcher), scope_equipped(searcher), ...
         Pure (searcher) -> iterable[obj] functions.

Layer 2 — The walk primitive
         walk_scope(searcher, scope_fn, *predicates) -> list[obj]
         The only function in the library that iterates candidates.

Layer 1 — Name matching
         name_match(searcher, target_str, candidates) -> obj | None
         Thin wrapper over caller.search(candidates=..., quiet=True, use_nicks=True).

Layer 0 — Evennia primitives (unchanged) + one narrow override
         caller.search, obj.access, obj.tags, obj.aliases, obj.scripts, ...
         + override of get_search_candidates that excludes self.location.
```

Each layer depends only on the layer below it. Layer 5 resolvers read like English because they compose Layers 1–4 by name.

### Layer 0: the narrow `get_search_candidates` override

On our base object class (probably `FCMObject` or directly on `FCMCharacter` / `RoomBase`), override one sub-method of `search()`:

```python
def get_search_candidates(self, searchdata, **kwargs):
    """Evennia's default includes self.location in the candidate list,
    which lets a substring-match against the room's key win the search.
    Strip it. Named keyword 'here' is still handled by
    get_search_direct_match, which runs before this method, so the
    'look here' use case is unaffected.
    """
    candidates = super().get_search_candidates(searchdata, **kwargs)
    if candidates is None:
        return None
    location = self.location
    if location is not None:
        candidates = [c for c in candidates if c is not location]
    return candidates
```

**Why this is safe:**

- The special keywords `me` / `self` / `here` are handled by `get_search_direct_match`, which runs **before** `get_search_candidates`. The `here` shortcut still works.
- Only the default candidate-building path is affected. Any code that passes `candidates=[...]` explicitly bypasses this override entirely.
- No signature changes, no return type changes, no behavior changes for any caller that isn't currently hitting the footgun.
- The override closes the footgun globally — every existing `caller.search()` call across the codebase is fixed without migration, including third-party contribs and default Evennia commands.

**This is the one production-touching change the system requires**. Everything else is parallel infrastructure.

### Layer 1: `name_match` — the thin string-parsing wrapper

```python
def name_match(searcher, target_str, candidates, *, use_nicks=True, stacked=0):
    """Delegate string parsing to Evennia's caller.search().

    Takes a pre-filtered candidate list (built by Layer 2+ with
    semantic predicates applied) and hands it to caller.search with
    quiet=True. We get substring matching, alias lookup, dbref
    resolution, multimatch disambiguation (<num>-<string>), me/self/here
    handling, and nick substitution for free.

    Args:
        searcher: The object initiating the lookup (usually caller).
        target_str: The raw keyword typed by the player.
        candidates: Pre-filtered list of semantically valid objects.
            MUST be a concrete list (not None) — passing None would
            reintroduce the footgun.
        use_nicks: Whether to apply the searcher's object nicks.
        stacked: Pass through to caller.search for identical stacks.

    Returns:
        The matched object or None. Never returns a list.
    """
    if candidates is None:
        raise ValueError("name_match requires an explicit candidate list")
    if not candidates:
        return None
    if not target_str:
        return None

    result = searcher.search(
        target_str,
        candidates=candidates,
        quiet=True,
        use_nicks=use_nicks,
        stacked=stacked,
    )
    if not result:
        return None
    if isinstance(result, list):
        return result[0] if result else None
    return result
```

This is the only function in the library that touches string parsing. Everything else operates on object references.

### Layer 2: `walk_scope` — the one iteration primitive

```python
def walk_scope(searcher, scope_fn, *predicates):
    """Iterate scope_fn(searcher) once and return objects that pass
    every predicate.

    Short-circuit evaluation via all() — predicates are checked in
    argument order and iteration stops at the first False per object.
    Cheap predicates should come first.

    This is the ONLY function in the library that iterates candidates.
    Every scope and every resolver ultimately funnels through here.
    """
    return [
        obj for obj in scope_fn(searcher)
        if all(p(obj, searcher) for p in predicates)
    ]
```

### Layer 3: scope functions

Each scope is a tiny `(searcher) -> iterable` function. None of them filter — filtering is the predicates' job. Scopes only decide **which universe of objects to consider**.

```python
def scope_room(searcher):
    """Everything in the searcher's current location (contents).
    Excludes the room itself — Evennia's contents already does this.
    """
    room = searcher.location
    return room.contents if room is not None else []

def scope_inventory(searcher):
    """Everything the searcher is carrying."""
    return searcher.contents

def scope_room_and_inventory(searcher):
    """Union of room contents and inventory. Used by 'examine', 'look',
    'drop' (to find the item in inventory) and 'get' (to find the item
    in the room).
    """
    room = searcher.location
    return list(searcher.contents) + (list(room.contents) if room else [])

def scope_exits(searcher):
    """Exits in the searcher's current room. Evennia's room.exits
    provides this as a filtered view of contents.
    """
    room = searcher.location
    return room.exits if room is not None else []

def scope_equipped(searcher):
    """Everything in a wearslot on the searcher. Uses the existing
    wearslot iteration on BaseActor.
    """
    return searcher.get_all_equipped()

def scope_held(searcher):
    """Whatever is in the HOLD wearslot — normally at most one item."""
    held = searcher.get_slot(HumanoidWearSlot.HOLD)
    return [held] if held is not None else []

def scope_worn(searcher):
    """Equipped items excluding HOLD (armour, trinkets, rings, etc.)."""
    return [
        item for slot, item in searcher.get_all_wearslot_items()
        if slot is not HumanoidWearSlot.HOLD and item is not None
    ]

def scope_global(*_):
    """Not a room walk — for cross-room queries like 'where'.
    Use sparingly; every call is a DB query.
    """
    from evennia.objects.models import ObjectDB
    return ObjectDB.objects.all()
```

New scopes are added by writing one function. The rest of the library does not care.

### Layer 4: predicate library

Each predicate is a pure `(obj, searcher) -> bool` function. They split into two tiers:

**Tier A — core predicates** (domain-neutral, no FCM-specific knowledge):

```python
def p_not_searcher(obj, searcher):
    return obj is not searcher

def p_living(obj, searcher):
    hp = getattr(obj, "hp", None)
    if hp is None:
        return False
    try:
        return int(hp) > 0
    except (TypeError, ValueError):
        return False

def p_has_account(obj, searcher):
    return bool(getattr(obj, "account", None))

def p_is_typeclass(typeclass_path):
    def _pred(obj, searcher):
        return obj.is_typeclass(typeclass_path, exact=False)
    return _pred

def p_has_tag(tag_name, category=None):
    def _pred(obj, searcher):
        return obj.tags.has(tag_name, category=category)
    return _pred

def p_accessible(access_type="search"):
    def _pred(obj, searcher):
        return obj.access(searcher, access_type)
    return _pred

def p_visible_to(obj, searcher):
    """Respects HiddenObjectMixin / InvisibleObjectMixin. Delegates
    to the mixin's is_visible_to() helper when present.
    """
    if hasattr(obj, "is_visible_to"):
        return obj.is_visible_to(searcher)
    return True
```

**Tier B — FCM domain predicates** (require project-specific knowledge):

```python
def p_at_height(height):
    def _pred(obj, searcher):
        return getattr(obj, "room_vertical_position", 0) == height
    return _pred

def p_at_searcher_height(obj, searcher):
    return (
        getattr(obj, "room_vertical_position", 0)
        == getattr(searcher, "room_vertical_position", 0)
    )

def p_is_enemy(obj, searcher):
    """Enemy from the searcher's current combat side. Reads
    combat_utils.get_sides() once per resolver call and closes
    over the frozenset.
    """
    # Implementation uses a precomputed set passed via closure —
    # see Resolver Pattern below.
    ...

def p_in_group(obj, searcher):
    """Member of the searcher's follow-chain group."""
    ...

def p_pvp_safe(obj, searcher):
    """In non-PvP rooms, exclude player characters not in the
    searcher's group. In PvP rooms, always True.
    """
    room = searcher.location
    if getattr(room, "allow_pvp", False):
        return True
    if not p_has_account(obj, searcher):
        return True
    return p_in_group(obj, searcher)

def p_is_mob(obj, searcher):
    """Is a CombatMob (not an NPC, not a PC). Used by beastlore."""
    from typeclasses.actors.mob import CombatMob
    return isinstance(obj, CombatMob)

def p_lockable(obj, searcher):
    """Has a Lock from LockableMixin."""
    from typeclasses.mixins.lockable_mixin import LockableMixin
    return isinstance(obj, LockableMixin)

def p_gettable(obj, searcher):
    """Passes the 'get' lock AND is not fixed-in-place scenery."""
    return obj.access(searcher, "get") and not getattr(obj, "db.no_pickup", False)

def p_wieldable(obj, searcher):
    """Can be wielded as a weapon — has a WIELD-compatible wearslot."""
    ...

def p_has_spell_bound(obj, searcher):
    """Wand with a bound spell (for zap resolution)."""
    from typeclasses.items.holdables.wand_nft_item import WandNFTItem
    return isinstance(obj, WandNFTItem) and bool(obj.spell_key)
```

**Predicate factories**: `p_at_height(5)`, `p_is_typeclass("typeclasses.actors.mob.CombatMob")`, `p_has_tag("peaceful", category="npc")`, `p_accessible("get")` return closures that capture their parameter. This keeps the library extensible — any subsystem can build one-off predicates without polluting the shared library.

**Closure predicates inside resolvers**: when a resolver needs a precomputed set (enemies, group members), it builds the set **once** at the top of the function and defines a local closure predicate that captures the set. The walk primitive then does O(1) set-membership checks inside the iteration. See the Resolver Pattern section.

### Layer 5: typed resolvers

Resolvers are short recipes — usually 5–15 lines — that compose scopes, predicates, and `name_match` into named functions. Each resolver has one job and a descriptive name. No mode flags, no branching on target_type strings, no shared monolithic helper.

```python
def resolve_hostile_actor(searcher, target_str):
    """A living enemy of the searcher in the current room.

    Excludes: the searcher themselves, dead actors, non-actors,
    bystander PCs in non-PvP rooms.

    Fast path: if searcher.ndb.combat_target is set and matches, return
    it without walking the room.
    """
    current = combat_target_of(searcher)
    if current is not None:
        if not target_str:
            return current
        if _name_hit(current, target_str.strip().lower()):
            return current

    group_set = _searcher_group_set(searcher)
    def p_pvp_safe_closure(obj, _s):
        room = searcher.location
        if getattr(room, "allow_pvp", False):
            return True
        if not p_has_account(obj, searcher):
            return True
        return obj in group_set

    candidates = walk_scope(
        searcher,
        scope_room,
        p_living, p_not_searcher, p_pvp_safe_closure,
    )
    if not target_str:
        return None
    return name_match(searcher, target_str, candidates)


def resolve_allied_actor(searcher, target_str):
    """A living ally (searcher themselves OK). Empty target_str returns
    the searcher.
    """
    if not target_str:
        return searcher
    candidates = walk_scope(searcher, scope_room, p_living)
    return name_match(searcher, target_str, candidates)


def resolve_any_actor(searcher, target_str):
    """Any living actor in the room. Used by spells with target_type 'any'."""
    if not target_str:
        return None
    candidates = walk_scope(searcher, scope_room, p_living)
    return name_match(searcher, target_str, candidates)


def resolve_inventory_item(searcher, target_str):
    """Any item the searcher is carrying."""
    if not target_str:
        return None
    candidates = walk_scope(searcher, scope_inventory, p_not_searcher)
    return name_match(searcher, target_str, candidates)


def resolve_held_item(searcher, target_str=None):
    """The item in the HOLD wearslot. Keyword optional — if given,
    must match the held item's key.
    """
    held = searcher.get_slot(HumanoidWearSlot.HOLD)
    if held is None:
        return None
    if not target_str:
        return held
    return held if _name_hit(held, target_str.strip().lower()) else None


def resolve_worn_item(searcher, target_str):
    """Any item in a non-HOLD wearslot."""
    if not target_str:
        return None
    candidates = walk_scope(searcher, scope_worn)
    return name_match(searcher, target_str, candidates)


def resolve_gettable_room_item(searcher, target_str):
    """An item in the room that the searcher has permission to pick up."""
    if not target_str:
        return None
    candidates = walk_scope(
        searcher,
        scope_room,
        p_not_searcher,
        p_visible_to,
        p_accessible("get"),
    )
    return name_match(searcher, target_str, candidates)


def resolve_exit(searcher, target_str):
    """An exit in the searcher's current room."""
    if not target_str:
        return None
    candidates = walk_scope(searcher, scope_exits, p_visible_to)
    return name_match(searcher, target_str, candidates)


def resolve_unlockable_exit(searcher, target_str):
    """A LockableMixin-bearing exit in the current room."""
    if not target_str:
        return None
    candidates = walk_scope(searcher, scope_exits, p_visible_to, p_lockable)
    return name_match(searcher, target_str, candidates)


def resolve_npc_speaker(searcher, target_str):
    """An NPC the searcher can talk to — has a dialogue cmdset."""
    if not target_str:
        return None
    candidates = walk_scope(
        searcher,
        scope_room,
        p_not_searcher,
        p_living,
        p_is_typeclass("typeclasses.actors.npc.BaseNPC"),
    )
    return name_match(searcher, target_str, candidates)


def resolve_trade_partner(searcher, target_str):
    """A PC the searcher can hand items to in their current room."""
    if not target_str:
        return None
    candidates = walk_scope(
        searcher,
        scope_room,
        p_not_searcher,
        p_living,
        p_has_account,
    )
    return name_match(searcher, target_str, candidates)
```

**Multi-target resolvers** return lists instead of single objects:

```python
def resolve_hostile_actors_safe_aoe(searcher, target_str):
    """Primary enemy + other enemies at the primary's height.
    No friendly fire. For safe AoE spells.
    """
    primary = resolve_hostile_actor(searcher, target_str)
    if primary is None:
        return []

    enemy_set = _searcher_enemy_set(searcher)
    def p_enemy_closure(obj, _s):
        return obj in enemy_set

    height = getattr(primary, "room_vertical_position", 0)
    secondaries = walk_scope(
        searcher,
        scope_room,
        p_living, p_not_searcher, p_enemy_closure, p_at_height(height),
    )
    return [primary] + [o for o in secondaries if o is not primary]


def resolve_hostile_actors_unsafe_aoe(searcher, target_str):
    """Primary + enemies + allies at the primary's height.
    Bystanders in non-PvP rooms are excluded. For unsafe AoE.
    """
    primary = resolve_hostile_actor(searcher, target_str)
    if primary is None:
        return []

    enemy_set = _searcher_enemy_set(searcher)
    ally_set = _searcher_ally_set(searcher)
    def p_damage_eligible(obj, _s):
        return obj in enemy_set or obj in ally_set

    height = getattr(primary, "room_vertical_position", 0)
    secondaries = walk_scope(
        searcher,
        scope_room,
        p_living, p_not_searcher, p_damage_eligible, p_at_height(height),
    )
    return [primary] + [o for o in secondaries if o is not primary]


def resolve_allied_actors_aoe(searcher):
    """All allies at the searcher's height. Used by group heal, mass bless."""
    ally_set = _searcher_ally_set(searcher)
    def p_ally_closure(obj, _s):
        return obj in ally_set or obj is searcher
    return walk_scope(
        searcher,
        scope_room,
        p_living, p_ally_closure, p_at_searcher_height,
    )
```

### The Resolver Pattern

Every resolver follows the same shape:

1. **Handle empty target_str** up front. Some return `None`, some return `searcher`, some return `[]`. The rule is explicit per resolver.
2. **Precompute any set(s)** needed by predicates (enemy set, ally set, group set). Call these helpers exactly once per resolver call.
3. **Build a closure predicate** if needed, capturing the precomputed sets.
4. **Call `walk_scope`** once (or twice for AoE) with the chosen scope and predicate stack.
5. **Call `name_match`** on the filtered candidate list to pick a single result. Or, for multi-target resolvers, return the list directly.
6. **Never send messages.** Resolvers are pure query functions. Messaging is always the caller's responsibility. This keeps the library testable and keeps call-site error wording specific to the command.

### Cost profile

| Scenario | Walks |
|---|---|
| Single-target, searcher's ndb.combat_target already matches | **0** |
| Single-target, empty target_str with combat_target set | **0** |
| Single-target, new target | 1 |
| Single-target, empty target_str without combat_target | 0 (returns None or searcher) |
| AoE, searcher already fighting the primary | 1 (only the secondary walk) |
| AoE, fresh primary | 2 |
| Allied AoE (no primary) | 1 |
| Hostile AoE around searcher (no primary) | 1 |

**Two walks is the AoE floor** when targeting a new primary: the second walk's height filter depends on the primary's runtime vertical position, which is only known after the first walk resolves. Short-circuit predicate eval cuts the constant factor for each walk to "the cheapest filter fails fast".

**Closures are precomputed once per resolver call**, so `obj in enemy_set` is O(1) inside the walk. No per-predicate recomputation, no per-object recomputation.

### Why caching doesn't help

Targeting is inherently fresh per command. Between ticks actors can die, move, change height, enter or leave groups, or flip combat sides. Caching results is only valid for the duration of a single command execution, at which point it's not a cache — it's just "don't call the resolver twice in the same frame". The library is designed to be cheap enough that "fresh every command" is the right pragma.

## Usage Catalog

Concrete examples of how the system is applied across every major command category. These are the **contract tests** for the library's API — if any of these feels contorted, the resolver catalog is wrong and needs adjustment before shipping.

### Spell targeting

```python
# cmd_cast.py — dispatching on spell.target_type
if spell.target_type == "hostile":
    target = resolve_hostile_actor(caller, target_str)
elif spell.target_type == "friendly":
    target = resolve_allied_actor(caller, target_str)
elif spell.target_type == "any":
    target = resolve_any_actor(caller, target_str)
elif spell.target_type == "self":
    target = caller
elif spell.target_type == "none":
    target = None
elif spell.target_type == "inventory_item":
    target = resolve_inventory_item(caller, target_str)
elif spell.target_type == "world_item":
    target = resolve_gettable_room_item(caller, target_str)
# ... etc

if target is None and spell.target_type not in ("none", "self"):
    caller.msg(f"Cast {spell.name} at whom?")
    return
```

### AoE spell targeting

```python
# inside a Fireball (unsafe AoE) spell
targets = resolve_hostile_actors_unsafe_aoe(caller, target_str)
if not targets:
    caller.msg("You don't see a valid target for Fireball here.")
    return False, None

primary, *secondaries = targets
# apply damage to each; message primary, secondaries, and bystanders
# differently
```

### Combat targeting (follow-up work, not Task 1)

```python
# cmd_attack.py
target = resolve_hostile_actor(caller, target_str)
if target is None:
    caller.msg("Attack whom?")
    return
# ... existing enter_combat flow

# cmd_bash.py (class skill)
target = resolve_hostile_actor(caller, target_str)
# ... existing bash logic
```

### Movement and exits

```python
# cmd_unlock.py
exit_ = resolve_unlockable_exit(caller, target_str)
if exit_ is None:
    caller.msg(f"You don't see a lockable '{target_str}' here.")
    return
# ... existing unlock flow

# cmd_open.py / cmd_close.py
door = resolve_closeable_exit(caller, target_str)
```

### Item manipulation

```python
# cmd_get.py
item = resolve_gettable_room_item(caller, target_str)
if item is None:
    caller.msg(f"You don't see '{target_str}' here.")
    return
# ... existing get flow

# cmd_drop.py
item = resolve_inventory_item(caller, target_str)

# cmd_hold.py
item = resolve_inventory_item(caller, target_str)
if not p_wieldable(item, caller):
    caller.msg("You can't hold that.")
    return

# cmd_remove.py
item = resolve_worn_item(caller, target_str)

# cmd_wear.py
item = resolve_inventory_item(caller, target_str)
```

### Giving and trading

```python
# cmd_give.py — two targets, two resolvers
item = resolve_inventory_item(caller, item_str)
recipient = resolve_trade_partner(caller, recipient_str)
if item is None:
    caller.msg(f"You don't have '{item_str}'.")
    return
if recipient is None:
    caller.msg(f"You don't see '{recipient_str}' here.")
    return
```

### NPC dialogue

```python
# cmd_say_to.py / cmd_ask.py
npc = resolve_npc_speaker(caller, npc_str)
if npc is None:
    caller.msg(f"You don't see '{npc_str}' to talk to.")
    return
# ... dialogue dispatch
```

### Skill use

```python
# cmd_beastlore.py — CombatMob only
target = resolve_mob(caller, target_str)  # uses p_is_mob

# cmd_appraise.py — items only
target = resolve_inventory_item(caller, target_str) or resolve_gettable_room_item(caller, target_str)

# cmd_picklock.py — lockable exits or containers
target = resolve_unlockable_exit(caller, target_str) or resolve_unlockable_container(caller, target_str)

# cmd_pickpocket.py — NPCs and PCs, but not mobs
target = resolve_pickpocketable_actor(caller, target_str)

# cmd_stealth.py (no target)
# no resolver needed
```

### Wand usage

```python
# cmd_zap.py
wand = resolve_held_item(caller)  # the held item, whatever it is
if wand is None or not isinstance(wand, WandNFTItem):
    caller.msg("You aren't holding a wand.")
    return
# ... existing zap flow uses the wand's bound spell's target_type
# to call the appropriate actor or item resolver
```

### Searching / examining

```python
# cmd_look.py (with target)
target = resolve_anything_visible(caller, target_str)
# resolves against scope_room_and_inventory with p_visible_to

# cmd_examine.py
target = resolve_anything_visible(caller, target_str)
```

### Crafting and recipes

```python
# cmd_craft.py
recipe_item = resolve_inventory_item(caller, recipe_str)

# cmd_enchant.py
wand = resolve_held_item(caller)
scroll = resolve_inventory_item(caller, scroll_str)
```

### Quest commands

```python
# cmd_give.py to quest NPC
npc = resolve_npc_speaker(caller, npc_str)
item = resolve_inventory_item(caller, item_str)
```

### Shop and economy

```python
# cmd_buy.py
shopkeeper = resolve_npc_speaker(caller, npc_str)

# cmd_sell.py
shopkeeper = resolve_npc_speaker(caller, npc_str)
item = resolve_inventory_item(caller, item_str)
```

## Extension Points

The library is designed to be extended without touching the core primitive. Three ways a subsystem can add behavior:

### Add a new predicate

Any `(obj, searcher) -> bool` function is a valid predicate. Add it to the appropriate module:

- **Core predicate** → `utils/targeting/predicates.py`
- **FCM-specific predicate** → `utils/targeting/predicates_fcm.py`
- **Subsystem-specific predicate** → keep it local to the subsystem and pass it into `walk_scope` directly as a closure

Example — a new dungeon system needs "objects tagged as quest-relevant":

```python
# in dungeon/dungeon_targeting.py
from utils.targeting import walk_scope, scope_room, p_living

def p_is_quest_relevant(obj, searcher):
    return obj.tags.has("quest_relevant", category="dungeon")

def resolve_quest_target(searcher, target_str):
    candidates = walk_scope(searcher, scope_room, p_living, p_is_quest_relevant)
    return name_match(searcher, target_str, candidates)
```

No core library changes needed. The dungeon module owns its own predicate and resolver and composes them via the shared primitive.

### Add a new scope

Any `(searcher) -> iterable` function is a valid scope. Same extension pattern — add to `utils/targeting/scopes.py` if generally useful, or keep it local otherwise. Examples of scopes a new subsystem might add:

- `scope_pet_inventory(searcher)` — the searcher's active pet's inventory
- `scope_guild_hall(searcher)` — everyone in the searcher's guild's current hall room
- `scope_procedurally_visible(searcher)` — objects revealed by the searcher's `reveal_mask`

### Add a new resolver

Resolvers are just Python functions that compose scopes, predicates, and name matching. Adding one is a five-minute change:

```python
def resolve_pet_item(searcher, target_str):
    """Item in the active pet's inventory."""
    if not target_str:
        return None
    pet = searcher.active_pet
    if pet is None:
        return None
    candidates = walk_scope(pet, scope_inventory)
    return name_match(searcher, target_str, candidates)
```

New resolvers go in `utils/targeting/resolvers.py` if generally useful, or in subsystem-specific modules otherwise. Every new resolver comes with a test — see Testing below.

## Module Layout

```
src/game/utils/targeting/
    __init__.py            # re-exports the public resolver API
    walk.py                # walk_scope primitive (the only iterator)
    scopes.py              # scope_room, scope_inventory, scope_exits, ...
    predicates.py          # core predicates (p_living, p_visible, ...)
    predicates_fcm.py      # FCM-specific predicates (p_is_enemy, p_at_height, ...)
    match.py               # name_match + _name_hit helpers
    resolvers.py           # typed resolver catalog
    _utils.py              # _searcher_group_set, _searcher_enemy_set, combat_target_of
```

And one narrow override:

```
src/game/typeclasses/.../fcm_object.py  # or similar base class
    get_search_candidates()  # strips self.location from default candidates
```

The subpackage lives under `utils/` because it is MUD-wide infrastructure, not spell- or combat-specific. Existing callers of `find_exit_target`, `resolve_actor_target`, `resolve_item_target`, `get_room_enemies`, `get_room_all`, and bare `caller.search()` are migration candidates — but **migration is a separate effort**, not part of landing the library.

## Migration Strategy

The library lands in one PR, unused by any game command, with comprehensive tests. This is deliberate: proving correctness in isolation before migrating production commands keeps risk low and keeps the library honest about its API.

**Phase 1 — land the library.**
- Implement `walk_scope`, scopes, predicates, `name_match`, and the typed resolver catalog.
- Override `get_search_candidates` on the base object class (closes the bee tree footgun globally for every existing call site).
- Write unit tests for every predicate, scope, and resolver.
- Write contract tests for every call site listed in the Usage Catalog above. These prove the API shape is right without wiring anything.
- Ship.

**Phase 2 — migrate consumer categories, one PR per category.**
- PR 2a: spell targeting — `cmd_cast`, `cmd_zap`, AoE spells (Fireball first)
- PR 2b: combat commands — `cmd_attack`, `cmd_bash`, `cmd_taunt`, other class skills
- PR 2c: movement and exits — `cmd_unlock`, `cmd_open`, `cmd_close`, `cmd_smash`
- PR 2d: item manipulation — `cmd_get`, `cmd_drop`, `cmd_give`, `cmd_hold`, `cmd_wear`, `cmd_remove`
- PR 2e: NPC interaction — `cmd_say_to`, `cmd_ask`, `cmd_buy`, `cmd_sell`, quest turn-ins
- PR 2f: skill commands — `cmd_beastlore`, `cmd_appraise`, `cmd_picklock`, `cmd_pickpocket`
- PR 2g: look/examine — `cmd_look`, `cmd_examine`

Each migration PR:
- Replaces the legacy `caller.search()` / `resolve_*_target()` / `get_room_*()` call with a named resolver
- Runs the existing command-level tests
- Adds any missing resolvers (and their tests) before wiring

**Phase 3 — retire legacy helpers.**
- Delete `utils/find_exit_target.py`, fold its logic into `resolve_exit` / `resolve_door`
- Delete the old `resolve_actor_target` / `resolve_item_target` in `spell_utils.py`
- Delete `get_room_enemies`, `get_room_all`, `get_room_enemies_at_height`, `get_room_all_at_height`
- Grep for any remaining bare `caller.search()` in game code; every remaining call must justify why it isn't a resolver

**Phase 4 — unify combat targeting.**
- Rebuild `get_sides()` on top of the targeting primitives. Predicates `p_in_combat` and `p_on_side(n)` + a tiny specialized resolver reproduce the pair-of-lists contract. Combat internals become another consumer of the same library, closing the architectural loop.

Phase 4 is the final convergence — at that point the MUD has **one** iteration primitive over room contents, used by every command, spell, and combat tick. The value of the system is measured by how few different places in the codebase walk `room.contents` directly.

## Testing Strategy

The library is tested in three tiers, each answering a different question.

### Tier 1 — predicate unit tests

One test per predicate. Each test uses a minimal mock object with only the attributes the predicate reads. No Evennia DB, no fixtures, no room setup — just `class FakeObj: pass` with the relevant attributes set. Example:

```python
def test_p_living_rejects_hp_none():
    obj = SimpleNamespace(hp=None)
    assert p_living(obj, searcher=None) is False

def test_p_living_accepts_positive_hp():
    obj = SimpleNamespace(hp=10)
    assert p_living(obj, searcher=None) is True

def test_p_living_rejects_zero_hp():
    obj = SimpleNamespace(hp=0)
    assert p_living(obj, searcher=None) is False
```

Lightweight, fast, exhaustive. Every predicate gets ~3–5 tests covering the true case, the false case, the edge case (None, missing attribute, wrong type).

### Tier 2 — scope unit tests

One test per scope. Each test stubs `searcher.location` or `searcher.contents` and verifies the scope yields the right subset.

### Tier 3 — resolver integration tests

Every resolver gets tested in a realistic fixture room containing a deliberate mix of actors and items. The fixture catches all the edge cases the resolver is supposed to handle:

**Fixture room** ("Under Goblin Tree" — deliberate ambiguous-room-name test):

- 1 caster (PC)
- 1 follower (PC, in caster's group)
- 1 bystander (PC, not in group)
- 3 ground-level goblins (enemies)
- 1 dead goblin (hp=0)
- 1 flying goblin (`room_vertical_position=2`)
- 1 bartender (peaceful NPC)
- 1 rabbit (unaggroed mob)
- 1 iron chest (lockable, not gettable)
- 1 glowing wand (in caster's inventory)
- 1 iron door exit (lockable, hidden mixin)
- 1 open archway exit (visible)

Test cases cover, for each resolver:

- The normal match (named target exists and is valid)
- The ambiguous-room-name case (the room's key substring-matches the keyword)
- The dead-target case (hp=0 excluded)
- The wrong-type case (item keyword when an actor was expected)
- The PvP-bystander case (non-PvP room, bystander PC excluded from hostile resolution)
- The height case (flying target excluded from ground-level AoE secondaries)
- The multi-match case (3 goblins → first or combat_target wins)
- The `<num>-string` disambiguation (`2-goblin` picks the second goblin via Evennia's built-in parser)
- The empty-string case (resolver-specific default)
- The no-match case (returns None / `[]`)
- The nick substitution case (player has a nick `g -> goblin`; `resolve_hostile_actor(caller, "g")` resolves)

Tier 3 tests are the **contract tests** mentioned in the architecture section — they double as acceptance tests for "the API is fit for every call site listed in the Usage Catalog".

### Footgun regression test

One dedicated test confirms the bee tree crash cannot recur. Build a fixture room with `key="Under Bee Tree"` and a `bee` mob inside it. Call `resolve_hostile_actor(caller, "bee")`. Assert the return is the bee, not the room. Call with `quiet=True` against the raw `caller.search()` to confirm the override excluded the room from the default candidate list. This test documents the fix and prevents regression.

## Risks and Open Questions

**Risk: Evennia upgrade breaks the `get_search_candidates` override.** Mitigation — the override is a thin wrapper that calls `super().get_search_candidates()` first and filters the result. If Evennia's sub-method signature changes, the override breaks loudly in tests, not silently in production.

**Risk: the resolver catalog grows unbounded.** Mitigation — resolvers are cheap to write and free to test. If the catalog hits 100+ names, that's evidence the MUD has 100+ distinct target resolution scenarios, which is a feature, not a bug. The alternative is every command inventing its own.

**Risk: closures make predicates harder to test.** Mitigation — closure predicates are used only **inside resolvers** where a precomputed set needs to be captured. The module-level `p_*` predicates stay closure-free and are the primary test surface. Closures inside resolvers are exercised by Tier 3 resolver tests.

**Open question: resolver error messaging.** Resolvers return `None` / `[]` silently and never message the caller. The caller composes its own wording ("Cast at whom?" vs "Who do you want to target?" vs "You don't see that here"). This is deliberate — the alternative centralizes error wording and loses command-specific context. If the pattern feels wrong after a few migration PRs, we can add optional `error_msg_hook` kwargs per resolver without breaking the core API.

**Open question: should AoE resolvers include the primary in secondary walks?** Currently `resolve_hostile_actors_safe_aoe` prepends the primary to the secondary list. An alternative is to return `(primary, [secondaries])` as a tuple. Lean: flat list, because most callers iterate uniformly and the primary is already in the list at position 0.

**Open question: should `name_match` return a list or a single object?** Currently returns a single object (the first match). If the caller wants all matches, they can iterate the filtered candidate list directly — `name_match` is about "pick one", not "return all that match". Lean: keep single-object return, add a separate `name_match_all` helper if needed by a future consumer.

**Open question: does every resolver need to handle the `<num>-<string>` multimatch syntax?** Yes, for free, via `caller.search(candidates=..., quiet=True)`. Evennia's built-in parser handles it regardless of which candidate list we pass. Our only job is not to break it — which we don't, because we delegate string parsing entirely.

## Relationship to Other Systems

- **[EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md)** — `utils/find_exit_target.py` is an early precursor of this system, scoped to exits only. It is a migration candidate for `resolve_exit` / `resolve_unlockable_exit`.
- **[SPELL_SKILL_DESIGN.md](SPELL_SKILL_DESIGN.md)** — spell `target_type` values (`hostile`, `friendly`, `any`, `self`, `none`, `inventory_item`, `world_item`, `any_item`) map 1:1 onto resolvers. `cmd_cast` becomes a thin dispatcher.
- **[COMBAT_SYSTEM.md](COMBAT_SYSTEM.md)** — `get_sides()` is the authoritative source for "who is on whose side" during combat. The targeting library reads it via `_searcher_enemy_set` / `_searcher_ally_set`. Phase 4 rebuilds `get_sides` on top of the targeting primitives, closing the architectural loop.
- **[INVENTORY_EQUIPMENT.md](INVENTORY_EQUIPMENT.md)** — wearslot-aware scopes (`scope_held`, `scope_worn`, `scope_equipped`) delegate to existing wearslot helpers on `BaseActor`.
- **[NPC_QUEST_SYSTEM.md](NPC_QUEST_SYSTEM.md)** — `resolve_npc_speaker`, `resolve_trade_partner`, and quest turn-in resolvers unify NPC target resolution across dialogue, shops, and quests.
- **[VERTICAL_MOVEMENT.md](VERTICAL_MOVEMENT.md)** — height filtering predicates (`p_at_height`, `p_at_searcher_height`) enforce the vertical-position rules defined there without requiring every caller to re-check.

## Summary

This system answers **"what did the player mean by that keyword"** in one place, consistently, for every command in the MUD. It leans on Evennia's string-parsing infrastructure (multimatch, nicks, dbref, locks, stacking) for everything Evennia already does well, closes Evennia's one footgun with a narrow override, and adds a thin composable layer of scopes + predicates + typed resolvers for everything Evennia doesn't express. The result is:

- **One iteration primitive** (`walk_scope`) across the entire MUD
- **One string-matching call** (`name_match` → `caller.search`) across the entire MUD
- **One place to add** new target-resolution rules (predicates) or new candidate universes (scopes)
- **Typed resolvers** at every call site so intent is visible in the command code
- **No way** to reproduce the bee tree crash — the footgun is closed at the source
- **O(n) bounded** cost per resolution with short-circuit eval
- **A clear migration path** that replaces dozens of ad-hoc helpers with a single tested library
