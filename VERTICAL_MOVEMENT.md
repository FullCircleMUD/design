# Vertical Movement System

> Technical design document for height, depth, and vertical transitions. For exit architecture, see [EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md). For world building, see [WORLD.md](WORLD.md).

## Current System

Characters have a `room_vertical_position` (integer) that tracks height/depth within a room:
- `0` = ground level
- `> 0` = flying (1, 2, etc.)
- `< 0` = underwater (-1, -2, etc.)

Rooms define vertical bounds:
- `max_height` (default 1) — maximum flying level. 0 = no flying (indoor rooms).
- `max_depth` (default 0) — maximum swimming depth. 0 = no water.

`ExitVerticalAware.at_traverse()` blocks movement when the character's height exceeds the destination room's bounds. Characters retain their vertical position when moving between rooms.

**Commands:** `fly up`/`fly down` (requires FLY condition), `swim up`/`swim down`, `climb up/down <target>` (requires ClimbableMixin fixture in room).

**Consequences:** Losing FLY while airborne triggers fall damage (10 HP per height level) — UNLESS a ClimbableMixin fixture in the room supports the character's current height, in which case the character grabs on and slides safely to ground (no damage). Water cushions falls — rooms with `max_depth < 0` absorb the first 20 HP of fall damage (`WATER_FALL_ABSORB`), so a 1-2 level fall into water is harmless (splash message) while high falls still hurt (e.g. height 5 into water = 30 damage instead of 50). Underwater without WATER_BREATHING starts a breath timer (CON-based duration: `20 + CON_mod × 5` seconds, min 10s) that leads to drowning (~5% max HP per tick). Breath timer starts on any transition to underwater — diving within a room (`swim down`), moving between rooms at underwater depth, or losing WATER_BREATHING while submerged. Timer stops when surfacing (height ≥ 0) or gaining WATER_BREATHING. Entering an air pocket (e.g. underwater cave at height 0) stops the timer; leaving back into water restarts it.

## HeightAwareMixin — Vertical Position Tracking

All objects that exist within a room's vertical space share a single mixin: `HeightAwareMixin` (`typeclasses/mixins/height_aware_mixin.py`).

```python
class HeightAwareMixin:
    room_vertical_position = AttributeProperty(0)

    def is_height_visible_to(self, looker):
        ...
```

**Composed into:** `BaseActor`, `Corpse`, `BaseNFTItem`, `WorldFixture`, `WorldItem`.

**NOT composed into:** Rooms and exits — they *define* vertical space, they don't exist within it.

`room_vertical_position` is the single source of truth for an object's vertical position. All height checks (combat reachability, exit gating, room display filtering) read this attribute.

### Height-Gated Visibility (Room Barrier System)

Height-gated visibility uses a **room + object** model:

- **Rooms** define visibility barriers: `visibility_up_barrier` and `visibility_down_barrier`, each a tuple of `(barrier_height, max_concealed_size)` or None.
- **Objects** define their `size` attribute (from the `Size` enum in `enums/size.py`).

`is_height_visible_to(looker)` checks whether a barrier lies between the observer and the object, and whether the object is small enough to be concealed. Same-height objects are always visible regardless of barriers.

Used by `p_height_visible_to` predicate (wraps `is_height_visible_to`) and the composite `p_can_see` predicate (stealth + height). Room display methods (`get_display_characters`, `get_display_things`) and `cmd_scan` use `p_height_visible_to` for filtering.

Examples:
- A forest canopy room sets `visibility_up_barrier = (1, "small")` — tiny/small creatures in the canopy are hidden from ground-level observers.
- A deep water room sets `visibility_down_barrier = (-1, "small")` — small objects at depth are hidden from surface observers.
- A gargantuan dragon at height 1 is NOT concealed by a canopy barrier hiding "small" — too large.

Item sizes: `BaseNFTItem`/`WorldItem` default to `"tiny"`, `WorldFixture` defaults to `"medium"`. Mobs have explicit sizes (bee=tiny, kobold=small, wolf=large).

### Death-Height Behavior

When an actor dies, their corpse inherits (or adapts) their vertical position:

| Actor Position | Corpse Position | Message |
|---|---|---|
| Flying (height > 0) | Ground (0) | "The corpse of X falls to the ground." |
| Underwater (height < 0) | Same depth | (no message — sinks where it is) |
| Ground (0) | Ground (0) | (no message) |

**Implementation:** Height is captured at the very top of `_real_death()` / `_defeat()` / `_create_corpse()` — BEFORE `clear_all_effects()` or `exit_combat()` run. This is critical because equipment-granted FLY (e.g. Skydancer's Ring) persists through `clear_all_effects()` (which only strips named effects). The height must be read before any state changes.

**Tests:** `tests/typeclass_tests/test_height_aware_mixin.py`.

---

## The Relative Height Problem

Height is room-relative, not world-relative. A character at height 1 outside a castle wall and a character at height 1 on top of the wall are at completely different real-world elevations, but the system treats them identically. This means:

- A character can't fly over a castle wall (both rooms have height 1, but the wall top is conceptually higher)
- An underwater cave entrance at depth -2 can't lead to a room where ground level is the cave floor
- There's no way to represent exits that only exist at certain heights (a window halfway up a tower, a ledge you can only reach by flying)

## Exit as Height Adapter (Implemented)

Rather than introducing absolute room elevation (which would touch every room and every height check), make the **exit** the height transition point. Two new mechanisms on `ExitVerticalAware`:

### Height Gating — `required_height` range

**Attributes:** `required_min_height` and `required_max_height` (both `AttributeProperty(None)`)
**Default:** `None` / `None` (no height gate — current behavior)

When set, the exit is only traversable AND visible to characters whose `room_vertical_position` falls within the range (inclusive). Characters outside the range cannot see or use the exit.

A single height can be expressed as `required_min_height=X, required_max_height=X`. An open range uses only one bound (e.g. `required_min_height=-3` with no max = "depth -3 or deeper").

### Height Transition — `arrival_heights` dict

**Attribute:** `arrival_heights` (`AttributeProperty(None)`)
**Default:** `None` (keep current height — current behavior)

A dict mapping the character's current height to their arrival height in the destination room. This is the height adapter — it handles the conceptual elevation change between rooms.

```python
# Castle wall exit (outside → same room on other side):
arrival_heights = {
    1: 0,   # flying at height 1 → land on wall top (ground level)
    2: 1,   # flying at height 2 → still airborne over the wall
}
```

Heights not in the map cannot use the exit (acts as an implicit height gate in addition to `required_min/max_height`). If `arrival_heights` is None, the character keeps their current height (current behavior).

### Fall Warning

When an exit would set `arrival_height > 0` and the character does NOT have the FLY condition, traversal will trigger a fall. The exit displays a warning message before proceeding — movement is not blocked, you simply fall (matching MUD convention: walk off a cliff, you fall).

**Attribute:** `fall_warning` (`AttributeProperty(None)`)

If set and the traversal would cause a fall, display this message before movement:

> "You are about to step off the wall. This will hurt."

If not set, use a generic default: "You brace yourself for the fall..."

Fall damage is applied after the movement completes.

## Two Patterns

The height system supports two distinct gameplay patterns using the same building blocks:

### Pattern 1 — Height Adapter (same destination, different arrival height)

One exit, one direction, one destination room. The character's arrival height depends on how high they were when they went through. Used when the same physical space is reached from different elevations.

**Building blocks:** `arrival_heights` dict on the exit. Heights not in the dict can't use the exit.

**Example:** Castle wall — flying at height 1 lands you on the wall (arrival 0). Flying at height 2 carries you over the wall (arrival 1, still airborne). Ground level can't use the exit at all.

### Pattern 2 — Height Routing (same direction, different destination)

Multiple exits sharing the same direction in one room, each with a non-overlapping `required_height` range and a different destination. The player always sees the same direction abbreviation but which exit they traverse depends on their height.

**Building blocks:** `required_min_height` / `required_max_height` on each exit. `get_display_exits()` filters to show only the one matching the character's current height.

**Example:** Underwater — two exits both keyed "south". One has `required_min_height=-2, required_max_height=2` (surface to shallow) → ocean room. The other has `required_min_height=-3, required_max_height=-3` (deep only) → underwater cave. The player sees `[Exits: s]` at every depth but goes to different rooms.

## Height-Dependent Room Descriptions

RoomBase already has an unused `vert_descriptions` dict stub. Wire it up:

```python
vert_descriptions = {
    0: "A cobblestone courtyard surrounded by high walls...",
    1: "From above, the courtyard spreads out below you. Guards patrol the walls...",
    -1: "The water is waist deep here. The courtyard stones are slippery underfoot...",
}
```

`get_display_desc()` checks `looker.room_vertical_position`:
- If a matching key exists in `vert_descriptions` → use that description
- If no match → fall back to `db.desc` with the current additive prefix ("Flying you can see below you:" / "Swimming underwater you can dimly perceive above you:")

This gives builders full control over what players see at each height, while the default behavior (additive prefix + standard description) works for rooms that don't need custom descriptions.

## Height-Dependent Exit Visibility

Exits with height requirements are filtered from `get_display_exits()` based on the looker's vertical position. A ground-level character doesn't see the castle wall flyover exit. A flying character sees it.

When multiple exits share the same direction at different heights, only the one matching the character's current height appears. The player sees one `s` in `[Exits: s]` regardless — the system picks the right exit.

The filtering integrates with the existing visibility system — `get_display_exits()` already filters by `is_visible_to()` and closed door state. Height filtering is an additional gate.

The `exits` command and `look <direction>` also respect this — you can't look through an exit you can't see at your current height.

## Implementation

All changes are backward-compatible additions to existing classes (all defaults None = current behavior):

1. **ExitVerticalAware** (`typeclasses/terrain/exits/exit_vertical_aware.py`) — `required_min_height`, `required_max_height`, `arrival_heights`, `fall_warning` AttributeProperties. `is_height_accessible(height)` query method. Height gate + arrival adaptation + fall warning in `at_traverse()`.

2. **RoomBase.get_display_exits()** — height filter using `is_height_accessible()`.

3. **RoomBase.get_display_desc()** — `vert_descriptions` AttributeProperty (replaces class-level stub). Height-specific descriptions when available, falls back to `db.desc`.

4. **CmdExits** + **CmdLook direction handler** — height filtering integrated.

**Tests:** `tests/typeclass_tests/test_height_adapter.py`.

## Gameplay Examples

### Castle Wall (Pattern 1 — same destination, different arrival height)

One exit north from outside the wall. Both flying heights reach the same
room (Wall Top / other side), but arrive at different vertical positions.
Ground-level characters cannot use the exit at all.

```
Outside Castle Wall (max_height=2):
  Exit "n" → Wall Top (arrival_heights={1: 0, 2: 1})
    → height 1: fly onto wall top, land (arrival 0)
    → height 2: fly over the wall, still airborne (arrival 1)
    → height 0: cannot use exit (not in arrival_heights)

  Ground: sees [ Exits: ]       (no north — can't reach the wall)
  Flying: sees [ Exits: n ]

Wall Top (max_height=1):
  Exit "s" → Outside Castle Wall (arrival_heights={0: 1, 1: 2})
    → height 0: step off wall, arrive at height 1 outside
       → with FLY: no problem, you're flying
       → without FLY: fall warning → "You are about to step off
         the wall. This will hurt." → fall damage (10 HP)
    → height 1: fly south, arrive at height 2 outside (above the wall)
```

### Underwater Cave (Pattern 2 — same direction, different destination)

Two exits both keyed "south" in the same room, each with a non-overlapping
height range. The player always sees `[Exits: s]` but goes to a different
room depending on their depth.

```
Coastal Shallows (max_depth=-3):
  Exit 1 "s" → Ocean Room (required_min_height=-2, required_max_height=2)
  Exit 2 "s" → Underwater Cave (required_min_height=-3, required_max_height=-3)

  Flying:   sees [ Exits: s ] → Ocean Room
  Surface:  sees [ Exits: s ] → Ocean Room
  Depth -1: sees [ Exits: s ] → Ocean Room
  Depth -2: sees [ Exits: s ] → Ocean Room
  Depth -3: sees [ Exits: s ] → Underwater Cave

Underwater Cave (max_height=0, max_depth=0):
  Air pocket — ground level inside, no water
  Exit "up" → Coastal Shallows (arrival_heights={0: -3})
    → emerge back underwater at depth -3
```

### Tower Window (Pattern 1 — same destination, different arrival height)

A single exit north from an indoor room. The room has max_height=0 (indoor,
can't fly), so the only possible height is 0. The exit puts you in the air
outside — fall warning if you can't fly.

```
Tower Interior Floor 2 (max_height=0):
  Exit "d" → Floor 1 (stairs, normal)
  Exit "u" → Floor 3 (stairs, normal)
  Exit "n" → Outside (arrival_heights={0: 2})
    → arrive at height 2 outside
    → without FLY: "You climb through the window. The ground is far
      below..." → 20 HP fall damage
    → with FLY: fly out the window, no problem
```

## ClimbableMixin — Climbing Without FLY

Characters can reach elevated heights without the FLY condition by climbing fixtures — drainpipes, ladders, ropes, vines, trees. This enables height-routed exits for non-flying characters and opens up vertical content (rooftops, tree canopies, cliff faces).

### Data Mixin (`typeclasses/mixins/climbable_mixin.py`)

Pure data — no methods. Global `CmdClimb` and `_check_fall()` inspect these attributes.

| Attribute | Default | Purpose |
|-----------|---------|---------|
| `climbable_heights` | None | Set of supported heights, e.g. `{0, 1}` |
| `climb_dc` | 0 | DEX check DC. 0 = auto-succeed |
| `climb_up_msg` | "You climb upwards." | First-person success message |
| `climb_down_msg` | "You climb downwards." | First-person success message |
| `climb_fail_msg` | "You fail to get a grip..." | First-person failure message |

### Concrete Typeclass (`typeclasses/world_objects/climbable_fixture.py`)

```python
class ClimbableFixture(ClimbableMixin, WorldFixture):
    pass
```

### CmdClimb (`commands/all_char_cmds/cmd_climb.py`)

Global character command: `climb up [target]` / `climb down [target]`

- Searches room for objects with `climbable_heights` attribute
- Auto-targets if only one climbable fixture in room
- Moves `room_vertical_position` ±1 within fixture's `climbable_heights`
- If `climb_dc > 0`: DEX check via `dice.roll_with_advantage_or_disadvantage()`
- Guards: must be standing, not encumbered, not in combat

### Fall Safety

`BaseActor._check_fall()` scans the room for any ClimbableMixin fixture whose `climbable_heights` includes the character's current height. If found, the character slides down safely (height → 0, no damage, flavour message). This covers all fall triggers: FLY removal, encumbrance, and exit traversal.

### First Instance — Millholm Rooftop Entry

- **Back Alley** behind vacant workshop on Artisan's Way (hidden door, find_dc 16)
- **Drainpipe** ClimbableFixture in the alley (`climbable_heights={0, 1}`, dc=0)
- **Height-routed exit:** at height 1, `up` leads to "Rooftops Above Artisan's Way" (arriving at height 0)
- Entry point to a future rooftop world above Millholm Town

---

## Future Considerations

- ~~**Height-dependent mob AI**~~ — **Done.** See `design/COMBAT_SYSTEM.md` § Height Combat and `design/NPC_MOB_ARCHITECTURE.md` for mob height-matching, retargeting, and flee behavior.
- ~~**Climbing**~~ — **Done.** ClimbableMixin + CmdClimb. See above.
- ~~**Height-gated object visibility**~~ — **Done.** Room barrier system (`visibility_up_barrier` / `visibility_down_barrier` on `RoomBase`) + object `size` attribute. Enforced via `p_height_visible_to` / `p_can_see` predicates. See § Height-Gated Visibility above.
- **`look up` / `look down` commands** — explicit commands to inspect what's above or below the character's current height (would use the same `is_height_visible_to` filter but display other heights, not just the current one)
- **Vertical fog of war** — atmospheric/distance effects when looking at things several heights away
- **Get/drop/loot height gating** — items at different heights, fishing from surface, etc.
- **Gradual descent** — falling could be a multi-tick event rather than instant, allowing reactions
