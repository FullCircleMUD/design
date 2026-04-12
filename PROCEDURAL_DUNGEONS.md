# Procedural Dungeons

> Technical design document for the procedural dungeon system. For exit architecture and traversal mechanics, see [EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md). For world building and zone design, see [WORLD.md](WORLD.md). For cartography implications, see [CARTOGRAPHY.md](CARTOGRAPHY.md). For economic impact (fungible cleanup, loot), see [ECONOMY.md](ECONOMY.md).

## Design Philosophy

Procedural dungeons exist to create replayable, non-mappable content. Every visit is a fresh experience — different layout, different room order, different branching paths. Players cannot buy maps, memorise routes, or take shortcuts. The procedural interior is genuinely uncertain and dangerous.

The critical design goal: **procedural rooms must be indistinguishable from static world rooms in terms of player experience.** Same exit formatting (`[ Exits: n s ]`), same room display, same lighting and weather behavior, same combat. The only difference is that the layout changes each visit. Players should not realise they've entered a procedural area until, on their third visit, they notice "that wasn't there last time."

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│  World                                       │
│                                              │
│  [Static Room] ──exit──▶ [Procedural Entry]  │
│                                              │
└─────────────────┬───────────────────────────┘
                  │ creates
                  ▼
┌─────────────────────────────────────────────┐
│  DungeonInstanceScript (orchestrator)        │
│  state: active → collapsing → done           │
│                                              │
│  Room (0,0) ──lazy exit──▶ Room (1,0)        │
│      │                        │              │
│      │ return exit        lazy exit           │
│      ▼                        ▼              │
│  [World]              Room (2,0) [boss]       │
│                                              │
└──────────────────────────────────────────────┘
```

### Components

| Component | Role |
|---|---|
| **DungeonTemplate** | Immutable configuration — depth, branching, generators, combat rules |
| **DungeonInstanceScript** | Per-instance orchestrator — lifecycle, room creation, cleanup |
| **DungeonRoom** | Procedural room — extends RoomBase, tracks coordinates and instance |
| **DungeonExit** | Lazy exit inside instances — creates rooms on first traversal |
| **DungeonPassageExit** | Exit from dungeon back to world — cleans up instance tags |
| **ProceduralDungeonMixin** | Reusable mixin providing dungeon entry capability — composed into entry exit types |
| **Entry exits** | World-side exits that create/join instances on traversal |

## Dungeon Types

### Instance Dungeons

Dead-end dungeons with a boss encounter at termination depth. The dungeon ends when the boss is killed — a linger timer gives players time to loot before the instance collapses.

**Flow:** Enter → explore rooms → clear encounters → reach boss room → kill boss → quest complete → collapse timer → evacuated to entrance.

**Example:** Rat Cellar (2 rooms, solo, boss_depth=1).

### Passage Dungeons

Procedural corridors connecting two static world rooms. No boss — the dungeon terminates with an exit to the destination room. Used to make travel between locations feel dangerous and unpredictable.

**Flow:** Enter from Room A → navigate procedural rooms → emerge at Room B.

**Example:** Deep Woods Passage (5 rooms, group, connects entry to clearing).

## Instance Modes

| Mode | Key format | Who enters | Isolation |
|---|---|---|---|
| **Solo** | `{template}_{player_id}` | One player only | Fully isolated |
| **Group** | `{template}_{leader_id}` | Leader + followers | Group shares instance |
| **Shared** | `{template}_shared_{entrance_id}` | Anyone at entrance | Multiple parties, one instance |

Group mode: only the leader can initiate entry. Followers in the same room are collected automatically.

Shared mode: if an active instance exists at this entrance, new arrivals join it. When all players leave, the instance persists for `empty_collapse_delay` seconds before collapsing (allows brief re-entry).

## Room Generation

### Coordinate System

Rooms are placed on a Cartesian grid. The first room is always at (0, 0). Forward exits lead to adjacent coordinates in cardinal directions (north/south/east/west).

**Depth** = Manhattan distance from origin: `abs(x) + abs(y)`. Depth determines room difficulty and when the dungeon terminates.

### Lazy Creation

Rooms are not pre-generated. When a `DungeonExit` is first traversed, it asks the `DungeonInstanceScript` to generate the destination room on the spot. The exit is then permanently linked. This means:

- Memory footprint is proportional to explored rooms, not total dungeon size
- Unexplored branches cost nothing
- The dungeon can be arbitrarily large in theory (bounded only by exit budget)

### Exit Budget

Two parameters control branching:

- `max_unexplored_exits` — total unvisited exits that can exist at any time
- `max_new_exits_per_room` — max forward exits created per room

Low values (1/1) produce winding linear paths. Higher values produce branching mazes. The budget prevents infinite sprawl.

### Encounter Gating

Rooms with hostile mobs are tagged `not_clear` (category `dungeon_room`). Forward exits from these rooms are blocked until the tag is removed. Return exits are always passable — players can always retreat.

The `not_clear` tag is removed when the last mob in a room dies (checked in the mob's `die()` method via `_check_room_cleared()`). Players see "The path forward is blocked!" in the room footer and "The way forward is clear." when the last mob falls.

### Room Properties

The instance script applies template properties to each generated room:

- `allow_combat`, `allow_pvp`, `allow_death` — combat flags
- `defeat_destination` — where defeated players respawn (no-death mode)
- `terrain_type` — terrain tag (controls lighting, weather, forage)
- `always_lit` — override for permanently lit dungeons

Rooms inherit all `RoomBase` functionality: lighting, weather exposure, details, visibility filtering, fungible inventory.

## Entry System

Dungeon entry uses composable exit pieces — a mixin provides the dungeon creation capability, and different exit types decide when to invoke it. See [EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md) for the full exit class hierarchy.

### ProceduralDungeonMixin

**File:** `typeclasses/mixins/procedural_dungeon.py`

A mixin that provides `enter_dungeon(traversing_object)` as a utility method. Handles template lookup, dungeon tag checks, character collection (group/solo/shared), instance resolution, and entry. Does not override `at_traverse` — the host class decides when to call it. Can be mixed into any exit type.

### ProceduralDungeonExit

**File:** `typeclasses/terrain/exits/procedural_dungeon_exit.py`
**Inherits:** `ProceduralDungeonMixin, ExitVerticalAware`

Simple dungeon entry — always enters a procedural dungeon on traversal. Used for unconditional entries.

### ConditionalDungeonExit

**File:** `typeclasses/terrain/exits/conditional_dungeon_exit.py`
**Inherits:** `ProceduralDungeonMixin, ConditionalRoutingExit`

Quest-gated dungeon entry with fallback room. Condition met → `enter_dungeon()`. Condition not met → normal traversal to alternate destination with full vertical checks.

### Compositions

| Class | Inherits | Use case |
|---|---|---|
| `ProceduralDungeonExit` | ProceduralDungeonMixin + ExitVerticalAware | Always enters dungeon (deep woods, cave of trials) |
| `ConditionalDungeonExit` | ProceduralDungeonMixin + ConditionalRoutingExit | Quest-gated dungeon with fallback room (rat cellar) |
| `DungeonDoor` | ProceduralDungeonMixin + ExitDoor | Locked/closeable dungeon entrance (future) |

### Example — Rat Cellar

```python
ConditionalDungeonExit:
  direction: "south"
  condition_type: "quest_active"
  condition_key: "rat_cellar"
  alternate_destination_id: empty_cellar.id
  dungeon_template_id: "rat_cellar"

  Quest active → enter_dungeon() → procedural instance
  No quest    → normal traversal → empty cellar room
```

### Example — Deep Woods Passage

```python
ProceduralDungeonExit:
  direction: "north"
  dungeon_template_id: "deep_woods_passage"
  dungeon_destination_room_id: clearing.id

  Always → enter_dungeon() → procedural passage
```

## Lifecycle

### Instance Creation

1. Player traverses a dungeon entry exit
2. Instance key determined by mode (solo/group/shared)
3. For shared mode: check for existing active instance at this entrance
4. `DungeonInstanceScript` created, template and entrance info stored
5. `start_dungeon(characters)` called:
   - First room generated at (0, 0)
   - Room tagged and properties applied
   - Forward exits created from first room
   - Return exit created back to entrance
   - Characters tagged and teleported in

### Exploration

6. Player moves through a forward exit (DungeonExit)
7. DungeonExit detects lazy placeholder (destination == self.location)
8. Asks instance script to generate the destination room
9. New room created, tagged, properties applied
10. Return exit created back to source room
11. If at termination depth: boss spawned (instance) or world exit created (passage)
12. If not terminal: forward exits created, respecting exit budget

### Collapse Triggers

| Trigger | Condition |
|---|---|
| **Lifetime expired** | `elapsed >= instance_lifetime_seconds` (skipped if `persistent_until_empty=True`) |
| **All characters left** | No tagged characters in the instance — collapses immediately, or after `empty_collapse_delay` seconds if set |

**Note on boss kills:** Killing the boss does not directly trigger collapse. `on_boss_defeated()` sets a `boss_defeated` flag used by quest progression and the room's `not_clear` clearing logic, but the instance only ends via the two triggers above. Templates that want the player to walk out at their own pace after the boss kill set `persistent_until_empty=True` (e.g. `rat_cellar`); this disables the lifetime expiry and lets the dungeon stay alive until the last player leaves the instance, at which point the "all characters left" trigger fires.

### Cleanup

13. State set to "collapsing"
14. All characters evacuated to entrance room (teleport)
15. Character instance tags removed
16. All dungeon mobs deleted
17. All dungeon exits deleted
18. All fungibles returned to RESERVE from dungeon rooms
19. All dungeon rooms deleted
20. Instance script deleted

### Server Restart

Stale dungeon instances are collapsed on every server boot via `at_server_startstop.py`. This prevents orphaned instances from accumulating after crashes.

## Templates

### DungeonTemplate (frozen dataclass)

| Field | Type | Default | Purpose |
|---|---|---|---|
| `template_id` | str | — | Unique identifier |
| `name` | str | — | Display name shown to players |
| `dungeon_type` | str | "instance" | "instance" (boss dead-end) or "passage" (corridor) |
| `instance_mode` | str | "solo" | "solo", "group", or "shared" |
| `boss_depth` | int | 5 | Manhattan distance for termination |
| `max_unexplored_exits` | int | 2 | Exit budget cap |
| `max_new_exits_per_room` | int | 2 | Branching factor |
| `instance_lifetime_seconds` | int | 7200 | Max instance lifetime (2 hours) |
| `room_generator` | Callable | None | `func(instance, depth, coords) → DungeonRoom` |
| `boss_generator` | Callable | None | `func(instance, room) → boss NPC` |
| `room_typeclass` | str | DungeonRoom path | Dotted typeclass path |
| `allow_combat` | bool | True | Combat enabled in rooms |
| `allow_pvp` | bool | False | PvP enabled in rooms |
| `allow_death` | bool | False | True death or defeat mode |
| `defeat_destination_key` | str | None | Room key for defeat respawn |
| `persistent_until_empty` | bool | False | If True, disable lifetime expiry — instance only collapses when all players leave (used by templates where the player should set their own pace, e.g. `rat_cellar` after boss kill) |
| `empty_collapse_delay` | int | 0 | Seconds to wait after the last player leaves before collapsing (0 = immediate). Useful for shared mode where another party may arrive shortly. |
| `terrain_type` | str | "dungeon" | Terrain tag for generated rooms |
| `always_lit` | bool | False | Permanent lighting override |

### Room Generators

Room generators are plain functions that receive the instance, depth, and coordinates and return a `DungeonRoom` object. They are responsible for:

- Creating the room with `create_object(DungeonRoom, key=...)`
- Setting the description (`room.db.desc`)
- Spawning mobs in the room (tagging them with `instance.instance_key`, category `"dungeon_mob"`)
- Setting the `not_clear` tag if mobs are present
- Setting quest tags if applicable

The instance script handles everything else: tagging the room, storing it in the grid, applying template properties (combat flags, terrain, lighting), creating exits.

### Boss Generators

Boss generators receive the instance and the boss room. They spawn the boss mob and return it. The instance script tags the boss as a dungeon mob.

For the Rat Cellar, the boss generator is separate from the room generator because the boss room is created by the standard room generator, then the boss is spawned into it at termination depth.

### Registered Templates

| Template | Type | Mode | Depth | Layout | Location |
|---|---|---|---|---|---|
| `rat_cellar` | instance | solo | 1 | 2 rooms (rats + boss), `persistent_until_empty` | Harvest Moon cellar |
| `deep_woods_passage` | passage | group | 5 | Winding forest trail | Deep woods (4 routes) |
| `lake_passage` | passage | shared | 5 | Plains terrain, low branching, 1h lifetime | Lake crossing |
| `cave_of_trials` | instance | group | 5 | Branching cave | Test world only |

Templates are registered at server startup via `at_server_startstop.py` which imports the template modules, triggering their module-level `register_dungeon()` calls.

## Cartography Integration

Procedural dungeon interiors are **not mappable**. Their rooms are created fresh each instance and do not persist. Cartographers can mark the entrance on a district map, but the interior cannot be surveyed or mapped.

This is a core design decision: procedural areas are permanently dangerous. No shortcuts, no bought maps, no memorised routes. See [CARTOGRAPHY.md](CARTOGRAPHY.md) § Procedural Areas for details.

## Economic Integration

When a dungeon instance collapses, all fungibles (gold, resources) in dungeon rooms are returned to RESERVE via `return_gold_to_reserve()` and `return_resource_to_reserve()`. This prevents gold/resources from being lost when rooms are deleted.

Player corpses created in no-death dungeons are inside the instance and will be deleted on collapse. Since defeated players are teleported to the defeat destination (e.g. the inn), their items are safe — corpses in no-death dungeons are empty (items stay on the character in defeat mode).

## Combat Integration

Dungeon rooms inherit `allow_combat`, `allow_pvp`, and `allow_death` from the template via `_set_room_properties()`. The no-death defeat flow teleports players to `defeat_destination_key` with 1 HP — the same `_defeat()` path used by all rooms with `allow_death=False`.

Encounter gating (`not_clear` tag) is set by room generators when mobs are present, and cleared by mob `die()` methods when the last mob in a room dies. The check is in a shared `_check_room_cleared()` function that any dungeon mob type can call.

## Future Work

- **More templates** — larger dungeons with multiple encounter types, treasure rooms, environmental hazards
- **Mob spawn tables** — configurable per-depth mob spawning instead of hardcoded in room generators
- **Dungeon-specific quest hooks** — generalised patterns for quest completion triggers
- **Treasure/loot system** — reward procedural dungeon completion with items
- **Puzzle elements** — interactive objects in dungeon rooms (doors, switches, environmental hazards)
- **Encounter variety** — traps, locked doors, skill challenges within procedural rooms
