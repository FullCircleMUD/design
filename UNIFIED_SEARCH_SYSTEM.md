# Unified Search and Targeting System

> Technical design document for MUD-wide target resolution. Every command that takes a keyword-named target resolves that keyword through this system. For spell-specific targeting rules see [SPELL_SKILL_DESIGN.md](SPELL_SKILL_DESIGN.md). For combat targeting see [COMBAT_SYSTEM.md](COMBAT_SYSTEM.md). For exit/door resolution see [EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md).

## What This Document Is (And Isn't)

This document is a **corrective**, not a new subsystem.

An earlier draft of this design treated Evennia's `caller.search()` as inadequate and proposed a multi-layer custom targeting library to "fill the gaps". A careful read of [evennia/objects/objects.py](../src/venv/Lib/site-packages/evennia/objects/objects.py) revealed that most of the supposed gaps don't exist — Evennia already supports scoped searches, tag filters, typeclass filters, nick substitution, numeric disambiguation, lock filtering, dbref lookups, and special-keyword shortcuts. Our messy target-resolution code is largely a case of **not reading the signature** before writing custom helpers that reinvent Evennia's existing machinery.

This document therefore has three jobs:

1. **Catalog what Evennia provides natively** so every future call site uses it correctly the first time.
2. **Name the narrow set of things Evennia genuinely cannot express** — FCM-specific semantic predicates, AoE multi-target shape, and typed-intent-at-call-site.
3. **Specify the small library that wraps Evennia's search with FCM predicates and named resolvers**, and nothing else.

It also stands as a worked example of the Evennia-first rule in [CLAUDE.md](../src/game/CLAUDE.md) § Development Approach: **before building, read what Evennia provides**. This session's earlier drafts are a cautionary tale. The final library is about 200 lines of Python plus tests — not a subsystem.

## The Trigger

A player typed `cast drain life bee-1` in a room called "Under Bee Tree". The spell command called `caller.search("bee-1")` with no `candidates=` argument. Evennia's default candidate list includes the room object itself, so the substring "bee" in "Under Bee Tree" matched the room's key. The spell received the room as its target and crashed in `take_damage()`.

This is not a spell bug. It is a **naming-resolution bug** that reproduces in any command that calls `caller.search()` without thinking about scope. An emergency fix (a `resolve_actor_target` helper that builds a living-actor candidate list) landed earlier to close the crash. That fix is the production stopgap. This document defines the permanent pattern the entire MUD migrates to.

## What Evennia Already Provides

Before specifying any library, catalog the facilities Evennia's [`DefaultObject.search()`](../src/venv/Lib/site-packages/evennia/objects/objects.py#L732) exposes. Every row in this table is something we get for free by calling `caller.search()` with the right kwargs.

### `caller.search()` keyword arguments

| Kwarg | Effect |
|---|---|
| `candidates=[...]` | Restrict to a pre-built list. The escape hatch for any custom filtering. |
| `location=<obj>` | Scope to `obj.contents`. Accepts a list of locations. |
| `location=caller` | **Searches the caller's inventory.** `caller.contents` is the inventory. |
| `location=caller.location` | **Searches the caller's current room only.** Excludes inventory AND the room object itself. |
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
- **`<num>-<string>` numeric disambiguation** — a player types `goblin-2` and Evennia parses it to "the second goblin". Controlled by `settings.SEARCH_MULTIMATCH_REGEX`. No custom parser needed.
- **Multimatch error formatting** — when the player's keyword matches several objects, Evennia formats the `goblin-1 / goblin-2 / goblin-3` disambiguation list via `at_search_result`. Override it to customise, but the default is usually fine.
- **Case-insensitive prefix/substring matching** against both `obj.key` and `obj.aliases.all()`.
- **Input whitespace normalisation.**
- **`#dbref` lookups** routed through the global DB instead of candidate filtering.

### Sub-methods you can override individually

`search()` is cleanly split into overridable sub-methods. Most FCM code should never touch them, but knowing they exist is important for the rare case where a different hook is the cleanest fix:

| Sub-method | Role |
|---|---|
| `get_search_query_replacement` | Preprocess the search string (e.g. custom nick substitution). |
| `get_search_candidates` | Build the default candidate list when none is passed. |
| `get_search_direct_match` | Handle `me`, `self`, `here` keywords. |
| `get_search_result` | Run the DB query. |
| `at_search_result` | Format multimatch / not-found messages. |

### Evennia helpers for predicate bodies

When the library writes predicates, it should lean on existing Evennia primitives rather than reinvent them:

| Question | Evennia primitive |
|---|---|
| Does this object pass a lock? | `obj.access(caller, "get")`, `obj.access(caller, "search")`, etc. |
| Does this object have a tag? | `obj.tags.has(name, category=...)` |
| Is this object of a typeclass? | `obj.is_typeclass(path, exact=False)` |
| Does this object have an alias? | `obj.aliases.all()` |
| Is a script attached? | `obj.scripts.get("key")` |
| What exits does this room have? | `room.exits` (Evennia provides a filtered view) |
| What is in a container? | `obj.contents` |
| What is at a location? | `location.contents` |

## Correct Evennia Idioms for Common Scopes

Almost every target lookup in the MUD fits one of these patterns. Each row is a correct one-line Evennia call. No library required.

| Intent | Idiomatic call |
|---|---|
| Anything the caller can normally reach (room + inventory, room included as target for `look here`) | `caller.search(target)` |
| Inventory only | `caller.search(target, location=caller)` |
| Current room only, room object excluded | `caller.search(target, location=caller.location)` |
| Room + inventory, **room object excluded** | Two calls — see below |
| Global | `caller.search(target, global_search=True)` |
| Only CombatMobs | `caller.search(target, typeclass="typeclasses.actors.mob.CombatMob")` |
| Only exits | `caller.search(target, candidates=list(caller.location.exits))` |
| Only objects with a tag | `caller.search(target, tags=[("peaceful", "npc")])` |
| Accept stacks of identical items | `caller.search(target, stacked=5)` |
| Let the caller handle errors | `caller.search(target, quiet=True)` |

**The "room + inventory without the room object" case**:

```python
room_matches = caller.search(target, location=caller.location, quiet=True) or []
inv_matches  = caller.search(target, location=caller, quiet=True) or []
matches = (room_matches if isinstance(room_matches, list) else [room_matches]) + \
          (inv_matches  if isinstance(inv_matches,  list) else [inv_matches])
```

This is the only common scope that takes more than one Evennia call to express correctly, and it's still plain Evennia — no custom primitive needed. A thin helper in the library codifies this pattern so callers don't repeat the idiom.

## What Evennia Does Not Provide

After the honest catalog above, the gaps shrink to three:

### Gap 1 — FCM-specific semantic predicates

Evennia's `search()` can filter by typeclass, tag, lock, or attribute name — but these are all *structural* filters. They can't express conditions that depend on **runtime state** or **FCM-specific mixins**. Every one of the following needs a custom predicate:

- **`p_living`** — `hp > 0`. Excludes corpses, dead mobs, fixtures, items, and the room itself.
- **`p_not_caster`** — `obj is not caller`. A pure identity compare, trivially cheap.
- **`p_at_height(h)`** — `obj.room_vertical_position == h`. FCM's vertical combat system.
- **`p_at_caster_height`** — same, but bound to the caller's height at call time.
- **`p_is_enemy`** — member of the caller's current combat side (via `combat.combat_utils.get_sides`). Depends on runtime combat state.
- **`p_in_group`** — member of the caller's follow-chain group.
- **`p_pvp_safe`** — in non-PvP rooms, excludes player characters not in the caller's group. Wraps `room.allow_pvp` + `p_in_group`.
- **`p_visible_to`** — respects `HiddenObjectMixin` and `InvisibleObjectMixin` via their existing `is_visible_to()` methods. Evennia's `use_locks` partially overlaps here but does not cover the stealth/invisibility mixins we defined.
- **`p_is_mob`** — `isinstance(obj, CombatMob)`. Distinguishes mobs from PCs and NPCs, used by `beastlore`.

All of these are **pure `(obj, caster) -> bool` functions** and each one is 3–5 lines. None of them exist in Evennia because none of them can exist in Evennia — they are all about FCM rules that Evennia cannot know about.

These predicates are applied by pre-filtering the candidate list and passing it to `caller.search(candidates=...)`. Evennia does the string matching; the predicates provide the FCM-specific semantics.

### Gap 2 — AoE multi-target shape

`caller.search()` is fundamentally single-target: it returns one object (or `None`, or a multimatch list of identically-named candidates). Spells like Fireball, weapons like greatsword cleave, and group buffs like Mass Heal need **"a primary target plus a list of secondary affected objects"**. The shape of that list depends on FCM rules: height of the primary, PvP flag of the room, caster's combat side, caster's group membership.

This is not a string-resolution problem and Evennia has no opinion on it. It's a spatial/combat calculation that takes a primary (already resolved via a normal single-target search) and walks the room to compute who else is affected. It lives in the library but is architecturally distinct from the string-matching helpers.

### Gap 3 — Typed intent at call sites

Every `caller.search()` call returns `Object | None | list[Object]`. From the call site, you cannot tell whether the caller wanted a hostile actor, a held item, a lockable door, or an NPC to talk to. Type errors surface as runtime `AttributeError` downstream — which is exactly how the bee tree crash manifested.

This is not a gap in Evennia. It's a software-engineering gap in **how we use** Evennia. The fix is discipline: every target lookup in the MUD goes through a **named resolver function** whose name describes the intent and whose return value is exactly the promised type. Resolvers are trivial wrappers — the FCM logic that makes them valuable is the predicate stack they apply before calling `caller.search(candidates=...)`.

## The Library

The library is small, focused, and pure-Python on top of Evennia's existing search machinery.

### Module layout

```
src/game/utils/targeting/
    __init__.py            # re-exports the public resolver API
    predicates.py          # FCM-specific (obj, caster) -> bool predicates
    resolvers.py           # named resolver functions (the call-site API)
    aoe.py                 # multi-target (primary + secondaries) computation
    _sets.py               # precomputed set helpers (enemy set, group set, etc.)
```

No `walk_scope` primitive — `caller.search()` with a filtered `candidates` list is the primitive. No `scope_*` functions — Evennia's `location=` handles most cases, and the one union case is a single helper in `resolvers.py`. No `name_match` helper — `caller.search(candidates=..., quiet=True)` is it. No override of Evennia's `search()` — legacy bare calls are a migration item, not something to defend.

### Predicate library

Each predicate is a pure `(obj, caster) -> bool` function. They compose via plain boolean expressions. No decorator magic, no registration, no framework.

```python
# utils/targeting/predicates.py

def p_living(obj, caster):
    hp = getattr(obj, "hp", None)
    if hp is None:
        return False
    try:
        return int(hp) > 0
    except (TypeError, ValueError):
        return False

def p_not_caster(obj, caster):
    return obj is not caster

def p_at_height(height):
    def _pred(obj, caster):
        return getattr(obj, "room_vertical_position", 0) == height
    return _pred

def p_at_caster_height(obj, caster):
    return (
        getattr(obj, "room_vertical_position", 0)
        == getattr(caster, "room_vertical_position", 0)
    )

def p_visible_to(obj, caster):
    if hasattr(obj, "is_visible_to"):
        return obj.is_visible_to(caster)
    return True

def p_is_mob(obj, caster):
    from typeclasses.actors.mob import CombatMob
    return isinstance(obj, CombatMob)

# ... p_is_enemy, p_in_group, p_pvp_safe — see Set Helpers below
```

**Test surface**: one unit test per predicate, using lightweight `SimpleNamespace` stand-ins. No Evennia DB, no fixtures, no room setup. The tests exist to catch regressions in the predicate logic itself; they do not exercise search or combat integration.

### Set helpers (precomputed once per resolver call)

Some predicates depend on expensive-to-compute sets — enemies from the combat handler, the caller's follow-chain group. These are computed **once per resolver call** via helpers in `_sets.py` and captured in closure predicates so the inner loop does O(1) set membership checks.

```python
# utils/targeting/_sets.py

def caster_enemy_set(caster):
    """Frozenset of enemies on the caster's current combat side.
    Returns an empty frozenset if the caster is not in combat.
    """
    from combat.combat_utils import get_sides
    allies, enemies = get_sides(caster)
    return frozenset(enemies)

def caster_ally_set(caster):
    """Frozenset of allies on the caster's combat side, or (out of
    combat) the caster's follow-chain group. Excludes the caster.
    """
    from combat.combat_utils import get_sides
    allies, enemies = get_sides(caster)
    if allies or enemies:
        return frozenset(a for a in allies if a is not caster)
    return caster_group_set(caster) - {caster}

def caster_group_set(caster):
    """Frozenset of the caster's follow-chain members currently in the
    caster's location. Uses the existing FCMCharacter follow helpers.
    """
    # implementation reads FCMCharacter.get_followers / leader walk
    ...
```

`get_sides()` is authoritative for combat sides today. When the combat-get_sides cleanup lands (a separate future task), these helpers become the only call sites that need updating.

### Typed resolvers

Resolvers are the public API. Each one is a 5–15 line function that composes Evennia's `caller.search()` with FCM predicates and returns exactly the type its name promises. No mode flags, no branching on target-type strings, no shared monolithic helper.

The general pattern:

1. Handle empty `target_str` per resolver's contract (return `None`, `caller`, or `[]`).
2. Choose the Evennia scope (`location=caller.location`, `location=caller`, or a candidate union).
3. Pre-filter candidates through predicates.
4. Hand the filtered list to `caller.search(candidates=..., quiet=True)` for string matching.
5. Normalise the return to a single object or `None`.
6. **Never send messages** — caller owns all error wording.

```python
# utils/targeting/resolvers.py

def _apply(candidates, predicates, caster):
    """Run a candidate list through a sequence of predicates."""
    return [o for o in candidates if all(p(o, caster) for p in predicates)]

def _first_match(caller, target_str, candidates, use_nicks=True):
    """Delegate string matching to Evennia's caller.search."""
    if not candidates or not target_str:
        return None
    result = caller.search(
        target_str,
        candidates=candidates,
        quiet=True,
        use_nicks=use_nicks,
    )
    if isinstance(result, list):
        return result[0] if result else None
    return result

# ── Actor resolvers ─────────────────────────────────────────────────

def resolve_hostile_actor(caller, target_str):
    """A living enemy of the caller in the current room.

    Excludes: the caller themselves, dead actors, non-actors, bystander
    PCs in non-PvP rooms.

    Fast path: if caller.ndb.combat_target is set, in the room, and
    matches the keyword, return it without filtering.
    """
    current = _combat_target_if_valid(caller)
    if current is not None:
        if not target_str:
            return current
        if _name_hits(current, target_str):
            return current

    if not target_str:
        return None

    group = caster_group_set(caller)
    room = caller.location
    pvp = getattr(room, "allow_pvp", False) if room else False

    def p_pvp_safe(obj, _c):
        if pvp or not getattr(obj, "account", None):
            return True
        return obj in group

    room_candidates = list(room.contents) if room else []
    candidates = _apply(
        room_candidates,
        [p_living, p_not_caster, p_pvp_safe],
        caller,
    )
    return _first_match(caller, target_str, candidates)


def resolve_allied_actor(caller, target_str):
    """A living ally in the room (caller themselves OK)."""
    if not target_str:
        return caller
    room_candidates = list(caller.location.contents) if caller.location else []
    candidates = _apply(room_candidates, [p_living], caller)
    return _first_match(caller, target_str, candidates)


def resolve_any_actor(caller, target_str):
    """Any living actor in the room. For spells with target_type 'any'."""
    if not target_str:
        return None
    room_candidates = list(caller.location.contents) if caller.location else []
    candidates = _apply(room_candidates, [p_living], caller)
    return _first_match(caller, target_str, candidates)

# ── Item resolvers ──────────────────────────────────────────────────

def resolve_inventory_item(caller, target_str):
    if not target_str:
        return None
    return caller.search(target_str, location=caller, quiet=True) or None

def resolve_held_item(caller, target_str=None):
    held = caller.get_slot(HumanoidWearSlot.HOLD)
    if held is None:
        return None
    if not target_str:
        return held
    return held if _name_hits(held, target_str) else None

def resolve_room_item(caller, target_str):
    """Any item in the room (excludes the room object)."""
    if not target_str:
        return None
    return caller.search(target_str, location=caller.location, quiet=True) or None

def resolve_gettable_room_item(caller, target_str):
    """A room item the caller has permission to pick up."""
    if not target_str:
        return None
    room_candidates = list(caller.location.contents) if caller.location else []
    candidates = [
        o for o in room_candidates
        if o is not caller
        and p_visible_to(o, caller)
        and o.access(caller, "get")
    ]
    return _first_match(caller, target_str, candidates)

# ── Exit resolvers ──────────────────────────────────────────────────

def resolve_exit(caller, target_str):
    if not target_str or not caller.location:
        return None
    candidates = [e for e in caller.location.exits if p_visible_to(e, caller)]
    return _first_match(caller, target_str, candidates)

def resolve_unlockable_exit(caller, target_str):
    from typeclasses.mixins.lockable_mixin import LockableMixin
    if not target_str or not caller.location:
        return None
    candidates = [
        e for e in caller.location.exits
        if p_visible_to(e, caller) and isinstance(e, LockableMixin)
    ]
    return _first_match(caller, target_str, candidates)

# ── NPC resolvers ───────────────────────────────────────────────────

def resolve_npc_speaker(caller, target_str):
    """An NPC in the room the caller can talk to."""
    if not target_str:
        return None
    return caller.search(
        target_str,
        location=caller.location,
        typeclass="typeclasses.actors.npc.BaseNPC",
        quiet=True,
    ) or None

def resolve_trade_partner(caller, target_str):
    """A PC in the room (for give / trade)."""
    if not target_str:
        return None
    room_candidates = list(caller.location.contents) if caller.location else []
    candidates = [
        o for o in room_candidates
        if o is not caller
        and p_living(o, caller)
        and getattr(o, "account", None)
    ]
    return _first_match(caller, target_str, candidates)
```

Notice what's missing compared to the earlier draft:

- **No `walk_scope` primitive.** Inline list comprehensions with `_apply` are plenty. Evennia's `search()` is the real iteration primitive when `candidates=` is passed.
- **No `scope_*` functions.** `location=caller`, `location=caller.location`, and a couple of inlined room-content lists handle every case.
- **No custom `name_match`.** `caller.search(candidates=..., quiet=True)` is it.
- **No `get_search_candidates` override.** Not needed — resolvers always pass explicit candidates or a `location=` scope.

What stays, because each of these is a genuine FCM addition:

- **Predicates** for runtime state (`p_living`, `p_at_height`, `p_is_enemy`, `p_in_group`, `p_pvp_safe`, `p_is_mob`, `p_visible_to`) and the set helpers they compose with.
- **Named resolvers** expressing typed intent at call sites.
- **AoE multi-target** (next section).

### AoE: primary + secondaries

AoE is the one genuine multi-target case. The library exposes it as a separate module because it's architecturally distinct — given a primary (resolved via a normal single-target resolver), compute the secondaries.

```python
# utils/targeting/aoe.py

def resolve_hostile_aoe_safe(caller, target_str):
    """Primary + other enemies at the primary's height.
    No friendly fire. For safe AoE spells.
    """
    primary = resolve_hostile_actor(caller, target_str)
    if primary is None:
        return []

    enemy_set = caster_enemy_set(caller)
    height = getattr(primary, "room_vertical_position", 0)

    secondaries = [
        o for o in (caller.location.contents if caller.location else [])
        if p_living(o, caller)
        and o is not caller
        and o in enemy_set
        and getattr(o, "room_vertical_position", 0) == height
        and o is not primary
    ]
    return [primary] + secondaries


def resolve_hostile_aoe_unsafe(caller, target_str):
    """Primary + enemies + allies at the primary's height.
    Bystanders in non-PvP rooms are excluded (they're not in either set).
    For unsafe AoE spells (Fireball).
    """
    primary = resolve_hostile_actor(caller, target_str)
    if primary is None:
        return []

    enemy_set = caster_enemy_set(caller)
    ally_set = caster_ally_set(caller)
    affected = enemy_set | ally_set
    height = getattr(primary, "room_vertical_position", 0)

    secondaries = [
        o for o in (caller.location.contents if caller.location else [])
        if p_living(o, caller)
        and o is not caller
        and o in affected
        and getattr(o, "room_vertical_position", 0) == height
        and o is not primary
    ]
    return [primary] + secondaries


def resolve_allied_aoe(caller):
    """All allies at the caller's height. Used by group heal, mass bless."""
    ally_set = caster_ally_set(caller) | {caller}
    caller_height = getattr(caller, "room_vertical_position", 0)
    return [
        o for o in (caller.location.contents if caller.location else [])
        if p_living(o, caller)
        and o in ally_set
        and getattr(o, "room_vertical_position", 0) == caller_height
    ]
```

**Cost profile**: every AoE resolver does at most two walks of `room.contents` — one by the single-target resolver (fast-path to zero walks if the caster is already fighting the primary) and one for the secondary filter. Precomputed sets make the filter O(1) per object. For a 15-actor room, the full cost is a handful of microseconds.

**Two walks is the floor for fresh-target AoE**, not a smell. The secondary filter depends on the primary's runtime height, which is only known after the primary is resolved.

## Usage Catalog

Concrete examples of how commands across the MUD use the system. These double as the **contract tests** for the resolver API — if any of them feels contorted, the resolver catalog is wrong.

### Spell targeting (`cmd_cast`)

```python
# dispatching on spell.target_type
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
    target = resolve_room_item(caller, target_str)

if target is None and spell.target_type not in ("none", "self"):
    caller.msg(f"Cast {spell.name} at whom?")
    return
```

### AoE spells (Fireball)

```python
targets = resolve_hostile_aoe_unsafe(caller, target_str)
if not targets:
    caller.msg("You don't see a valid target for Fireball here.")
    return False, None
primary, *secondaries = targets
# apply damage
```

### Combat (`cmd_attack`, class skills)

```python
target = resolve_hostile_actor(caller, target_str)
if target is None:
    caller.msg("Attack whom?")
    return
```

### Item manipulation

```python
# cmd_get
item = resolve_gettable_room_item(caller, target_str)

# cmd_drop / cmd_wear / cmd_hold
item = resolve_inventory_item(caller, target_str)

# cmd_remove
item = resolve_worn_item(caller, target_str)
```

### Giving

```python
item = resolve_inventory_item(caller, item_str)
recipient = resolve_trade_partner(caller, recipient_str)
```

### NPC dialogue / shops / quests

```python
npc = resolve_npc_speaker(caller, npc_str)
```

### Movement / exits

```python
# cmd_unlock / cmd_open / cmd_close
exit_ = resolve_unlockable_exit(caller, target_str)
```

### Skills

```python
# cmd_beastlore — mobs only
target = resolve_mob(caller, target_str)

# cmd_appraise — inventory items only
item = resolve_inventory_item(caller, target_str)

# cmd_picklock
target = resolve_unlockable_exit(caller, target_str)
```

### Look / examine

```python
# cmd_look — keep Evennia default, room IS a valid target
target = caller.search(target_str)
```

**Note**: `look` and `examine` are the one class of commands where passing the target_str straight to `caller.search()` (with no `location=`) is correct. The room object is a legitimate match for `look here`, and the `here` / `me` / `self` keywords are handled by Evennia before any filtering. No resolver needed.

## Migration Strategy

### Phase 1 — land the library

1. Implement `predicates.py`, `_sets.py`, `resolvers.py`, `aoe.py`.
2. Unit tests for every predicate (lightweight mock objects, no DB).
3. Integration tests for every resolver using a deliberate fixture room (see Testing).
4. Contract tests for every call site in the Usage Catalog above — the tests exist even though the call sites haven't been migrated yet.
5. Land as one PR. Production code unchanged. Zero runtime risk.

### Phase 2 — migrate consumers

Each PR handles one command category, runs its existing test suite, and replaces bare `caller.search()` with a named resolver.

- **PR 2a**: spells — `cmd_cast`, `cmd_zap`, AoE spells (Fireball first)
- **PR 2b**: combat — `cmd_attack`, `cmd_bash`, `cmd_taunt`, other class skills
- **PR 2c**: movement and exits — `cmd_unlock`, `cmd_open`, `cmd_close`, `cmd_smash`
- **PR 2d**: items — `cmd_get`, `cmd_drop`, `cmd_give`, `cmd_hold`, `cmd_wear`, `cmd_remove`
- **PR 2e**: NPC interaction — `cmd_say_to`, `cmd_ask`, `cmd_buy`, `cmd_sell`, quest turn-ins
- **PR 2f**: skills — `cmd_beastlore`, `cmd_appraise`, `cmd_picklock`, `cmd_pickpocket`

`cmd_look` and `cmd_examine` are **not migrated** — they correctly use bare `caller.search()` because they legitimately want the room as a target.

### Phase 3 — retire legacy helpers

- Delete `utils/find_exit_target.py`, `world/spells/spell_utils.py::resolve_actor_target` / `resolve_item_target`, `get_room_enemies`, `get_room_all`, `get_room_enemies_at_height`, `get_room_all_at_height`.
- Grep for remaining bare `caller.search(` calls in game code. Every remaining call must justify why it isn't a resolver — the `look`/`examine` pair are the only expected survivors.
- Add a CI grep check or ruff rule that flags new bare `caller.search(` calls in `commands/`, `world/spells/`, `combat/`, and `typeclasses/` outside a small allowlist. This is cheap prevention against future regressions.

### Phase 4 — unify combat side detection

`combat.combat_utils.get_sides()` is the authoritative source for combat sides and is called in 27+ places. Once the targeting library is proven in Phases 1–3, the targeting predicates and set helpers can replace `get_sides`'s internal walk — both systems converge on one iteration pattern. Task for a separate future plan; not part of this work.

## Testing Strategy

Three tiers, each answering one question.

### Tier 1 — predicate unit tests

One test per predicate. Each test uses a `SimpleNamespace` with only the attributes the predicate reads. No Evennia DB, no fixtures. Example:

```python
def test_p_living_rejects_hp_none():
    obj = SimpleNamespace(hp=None)
    assert p_living(obj, caster=None) is False

def test_p_living_accepts_positive_hp():
    obj = SimpleNamespace(hp=10)
    assert p_living(obj, caster=None) is True

def test_p_living_rejects_zero_hp():
    obj = SimpleNamespace(hp=0)
    assert p_living(obj, caster=None) is False
```

Fast, exhaustive, no setup cost. Every predicate gets the true case, the false case, and one or two edge cases (missing attribute, wrong type).

### Tier 2 — resolver integration tests

Each resolver gets tested against a deliberate fixture room. The fixture is the single most important test artefact in the library because it forces every edge case into the same scenario:

**Fixture room** (`key="Under Goblin Tree"` — deliberate ambiguous-room-name to catch the bee tree class of bugs):

- Caster (PC, mage)
- Follower (PC, in caster's group)
- Bystander (PC, not in group)
- 3 ground-level goblins (enemies)
- 1 dead goblin (`hp=0`)
- 1 flying goblin (`room_vertical_position=2`)
- Bartender NPC (peaceful)
- Rabbit (unaggroed mob)
- Iron chest (lockable, not gettable — scenery)
- Glowing wand (in caster's inventory)
- Iron door exit (lockable, visible)
- Open archway exit (non-lockable, visible)

Per-resolver test cases:

- Normal match (named target exists and is valid)
- Ambiguous-room-name case (the keyword substring-matches the room's key) — **the bee tree regression test**
- Dead-target case (excluded by `p_living`)
- Wrong-type case (item keyword when an actor was expected)
- PvP-bystander case (non-PvP room, bystander excluded from hostile)
- Height case (flying target excluded from ground-level AoE secondaries)
- Multi-match case (3 goblins → first or `combat_target` wins)
- `<num>-string` disambiguation (`2-goblin` picks the second — Evennia handles this for free; the test just verifies the library doesn't break it)
- Empty-string case (resolver-specific default)
- No-match case (returns `None` / `[]`)
- Nick substitution case (caller has nick `g -> goblin`; `resolve_hostile_actor(caller, "g")` resolves)

### Tier 3 — contract tests

One test per call site listed in the Usage Catalog, even though those call sites haven't been migrated yet. Each test wires the fixture room, invokes the resolver, and asserts the return shape and content match what the future migration PR will need. If any contract test feels contorted, the resolver API is wrong and the library is revised before shipping.

## Open Questions

**1. Resolver error messaging**: resolvers return `None`/`[]` silently and never message. The caller owns all wording. This is deliberate — different commands want different error text ("Cast at whom?" vs "Who do you want to target?" vs "You don't see that here"). If the pattern feels wrong after a few migration PRs, optional `error_msg` kwargs can be added without breaking the API.

**2. `resolve_anything_visible` for `look` / `examine`**: do we wrap these at all, or leave them on bare `caller.search()`? Lean: leave bare. Evennia's default is correct for this exact use case, and wrapping adds no value.

**3. Future combat-sides unification**: when Phase 4 lands, `get_sides()` becomes the one place that reads from `caller.ndb._combat_handler` and the targeting library becomes the one place that walks rooms with predicates. The two converge. Not in scope for this document.

## Relationship to Other Systems

- **[SPELL_SKILL_DESIGN.md](SPELL_SKILL_DESIGN.md)** — spell `target_type` values map 1:1 onto resolvers. `cmd_cast` becomes a thin dispatcher.
- **[COMBAT_SYSTEM.md](COMBAT_SYSTEM.md)** — `get_sides()` stays authoritative for combat sides until Phase 4. The library reads it via `caster_enemy_set` / `caster_ally_set`.
- **[EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md)** — `utils/find_exit_target.py` is retired by `resolve_exit` / `resolve_unlockable_exit` in Phase 3.
- **[INVENTORY_EQUIPMENT.md](INVENTORY_EQUIPMENT.md)** — `resolve_held_item` / `resolve_worn_item` delegate to existing wearslot helpers on `BaseActor`.
- **[VERTICAL_MOVEMENT.md](VERTICAL_MOVEMENT.md)** — height predicates enforce the vertical-position rules defined there without requiring every caller to re-check.

## Summary

This system is a **thin FCM-specific layer over Evennia's existing `caller.search()`**. It adds what Evennia cannot know about (FCM semantic predicates, AoE multi-target shape) and codifies typed intent at every call site. It is not a custom search engine, not a replacement for `search`, not an override of any Evennia sub-method. The library is small enough to read in one sitting.

The deeper lesson — and the reason this document exists in its corrected form — is that the messy targeting code we set out to fix was **not a case of Evennia being inadequate**. It was a case of not reading Evennia's API before writing around it. The Evennia-first rule in [CLAUDE.md](../src/game/CLAUDE.md) exists to prevent the next session from making the same mistake.
