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

### Tag taxonomy

The system uses three tag categories to track participation, pending recovery, and corpse-driven instance keepalive. See [Death and Corpse Recovery](#death-and-corpse-recovery) for the full state machine.

| Tag category | Set on | Meaning |
|---|---|---|
| `dungeon_character` | Player / pet | Currently physically inside an instance |
| `dungeon_pending` | Player | Has a corpse to recover in a specific instance |
| `dungeon_corpse` | Corpse | Marks a corpse as keeping its instance alive |
| `dungeon_room` | Room | Belongs to an instance — used for collapse cleanup |
| `dungeon_exit` | Exit | Belongs to an instance — used for collapse cleanup |
| `dungeon_mob` | Mob | Belongs to an instance — used for collapse cleanup |

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
| **Group** | `{template}_{leader_id}` | Leader + followers + pets | Group shares instance |
| **Shared** | `{template}_shared_{entrance_id}` | Anyone at entrance + followers + pets | Multiple parties, one instance |

**Solo mode:** the player must enter alone. Entry is blocked if the player has any followers (group members or pets) in the room — they must ungroup and leave pets outside first.

**Group mode:** only the leader can initiate entry. Followers and pets in the same room are collected automatically and tagged as instance participants.

**Shared mode:** if an active instance exists at this entrance, new arrivals join it. Followers and pets in the same room are collected and tagged alongside the player. When all players and pets leave, the instance persists for `empty_collapse_delay` seconds before collapsing (allows brief re-entry).

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

Simple dungeon entry — always enters a procedural dungeon on traversal. Used for unconditional entries. Pets that independently traverse this exit are stopped by an invisible barrier.

### ConditionalDungeonExit

**File:** `typeclasses/terrain/exits/conditional_dungeon_exit.py`
**Inherits:** `ProceduralDungeonMixin, ConditionalRoutingExit`

Quest-gated dungeon entry with fallback room. Condition met → `enter_dungeon()`. Condition not met → normal traversal to alternate destination with full vertical checks. Pets that independently traverse this exit are stopped by an invisible barrier.

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

**A dungeon instance never collapses while any character or pet is physically inside, and never collapses while any corpse tagged with its `instance_key` exists.** This is hardcoded — there is no template setting to override it. The design principle is that players should never receive overt signals that they are in a procedural area, and a forced evacuation would break that immersion. The corpse rule additionally guarantees that defeated players always have a chance to recover their gear.

| Trigger | Condition |
|---|---|
| **No participants present and no corpses** | No tagged characters or pets are physically in any dungeon room of this instance, AND no `dungeon_corpse` tags exist for this instance — collapses immediately, or after `empty_collapse_delay` seconds if set |
| **Server restart** | All instances collapsed unconditionally on boot (safety cleanup) |

`instance_lifetime_seconds` exists on templates for reference but does **not** trigger forced collapse.

**Pending recovery does not prop instances up.** A `dungeon_pending` tag on a player out in the world means *"I have a corpse to recover here"* — but the corpse itself is what keeps the instance alive, not the pending tag. When the corpse despawns (standard 10-minute timer), the instance becomes eligible for collapse and the pending tag is scrubbed during cleanup. See [Death and Corpse Recovery](#death-and-corpse-recovery).

**Physical presence, not just tagging.** Because a defeated player keeps a `dungeon_character` tag temporarily during the death-flow transition (before it is converted to `dungeon_pending`), the collapse check considers physical location: only characters whose `location` is a `dungeon_room`-tagged room of this instance count. This avoids false-positives during teleport-out and false-negatives if a character is somehow stranded outside a dungeon room while still tagged.

**Note on boss kills:** Killing the boss does not trigger collapse. `on_boss_defeated()` sets a `boss_defeated` flag used by quest progression, but the instance only ends when all participants have left and no corpses remain.

**Note on pets:** Pets tagged as `dungeon_character` (group/shared modes) count as participants while physically present. A pet left behind with `pet stay` will keep the instance alive until the pet is retrieved, leaves, or starves. In solo mode, pets cannot enter — the player must leave them outside. Pets do not receive `dungeon_pending` tags — when a pet dies its NFT transitions to craft_input and there is no corpse to recover.

### Cleanup

13. State set to "collapsing"
14. All characters and pets physically inside the instance evacuated to entrance room (teleport)
15. `dungeon_character` tags removed from all evacuated participants
16. `dungeon_pending` tags scrubbed from any tagged characters out in the world; connected players notified that their pending recovery is gone
17. All `dungeon_corpse`-tagged corpses deleted (defence in depth — collapse normally cannot fire while corpses exist, but covers race conditions)
18. All dungeon mobs deleted
19. All dungeon exits deleted
20. All fungibles returned to RESERVE from dungeon rooms (this includes any items that remain in corpses being deleted at step 17 — they are returned to RESERVE rather than dropped to limbo)
21. All dungeon rooms deleted
22. Instance script deleted

The `exit dungeon` command also evacuates the caller's owned pets — any pet tagged with the same instance and `owner_key == caller.key` is untagged and teleported to the entrance alongside the player.

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
| `instance_lifetime_seconds` | int | 7200 | Reference only — does not trigger forced collapse |
| `room_generator` | Callable | None | `func(instance, depth, coords) → DungeonRoom` |
| `boss_generator` | Callable | None | `func(instance, room) → boss NPC` |
| `room_typeclass` | str | DungeonRoom path | Dotted typeclass path |
| `allow_combat` | bool | True | Combat enabled in rooms |
| `allow_pvp` | bool | False | PvP enabled in rooms |
| `allow_death` | bool | False | True death or defeat mode |
| `defeat_destination_key` | str | None | Room key for defeat respawn |
| `empty_collapse_delay` | int | 0 | Seconds to wait after the last participant leaves before collapsing (0 = immediate). Useful for shared mode where another party may arrive shortly. Independent of corpse keepalive — corpses extend lifetime by their own standard 10-minute timer. |
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
| `rat_cellar` | instance | solo | 1 | 2 rooms (rats + boss), no pets allowed | Harvest Moon cellar |
| `deep_woods_passage` | passage | group | 5 | Winding forest trail, pets enter with group | Deep woods (4 routes) |
| `lake_passage` | passage | shared | 5 | Plains terrain, low branching, pets enter with owner | Lake crossing |
| `cave_of_trials` | instance | group | 5 | Branching cave, pets enter with group | Test world only |

Templates are registered at server startup via `at_server_startstop.py` which imports the template modules, triggering their module-level `register_dungeon()` calls.

## Pet Handling

Pets interact with dungeons differently depending on instance mode:

| Mode | Pets at entrance | Behaviour |
|---|---|---|
| **Solo** | Blocked | Entry refused: "You can only enter here alone, ungroup and leave your pets outside." |
| **Group** | Collected | Pets following the leader are tagged `dungeon_character` and teleported in with the group |
| **Shared** | Collected | Pets following the player are tagged `dungeon_character` and teleported in with the player |

**Pet guard on entry exits:** If a pet independently traverses a `ProceduralDungeonExit` or `ConditionalDungeonExit` (e.g. via a future `pet north` command), it is stopped: "An invisible barrier stops \<name\> from entering." This prevents pets from creating their own dungeon instances.

**Pets inside dungeons:** Tagged pets are full participants — they count towards the "anyone still inside?" check that prevents collapse. A pet told to `stay` retains its `dungeon_character` tag and keeps the instance alive.

**Leaving with pets:** The `exit dungeon` command evacuates the caller's owned pets (matched by `owner_key`) alongside the player. The `DungeonPassageExit` already handles follower untagging — pets that are actively following will cascade out with their owner. Pets on `stay` keep their tag and remain in the dungeon.

**Collapse evacuation:** On collapse, all tagged participants (characters and pets) are teleported to the entrance. This is a safety net — under normal play, participants leave voluntarily.

## Cartography Integration

Procedural dungeon interiors are **not mappable**. Their rooms are created fresh each instance and do not persist. Cartographers can mark the entrance on a district map, but the interior cannot be surveyed or mapped.

This is a core design decision: procedural areas are permanently dangerous. No shortcuts, no bought maps, no memorised routes. See [CARTOGRAPHY.md](CARTOGRAPHY.md) § Procedural Areas for details.

## Economic Integration

When a dungeon instance collapses, all fungibles (gold, resources) in dungeon rooms are returned to RESERVE via `return_gold_to_reserve()` and `return_resource_to_reserve()`. This prevents gold/resources from being lost when rooms are deleted.

Player corpses created on defeat in a dungeon room are loot-bearing — they hold the defeated player's full inventory, gold, and resources. The dungeon instance stays alive until the corpse is recovered or its standard 10-minute despawn timer elapses (see [Death and Corpse Recovery](#death-and-corpse-recovery)). If the corpse despawns or the instance collapses while it still contains items, fungibles are returned to RESERVE alongside other dungeon fungibles. NFT items inside an unrecovered corpse follow the standard `Corpse.at_object_delete` cleanup — see [INVENTORY_EQUIPMENT.md](INVENTORY_EQUIPMENT.md) for the unrecovered-NFT path.

## Combat Integration

Dungeon rooms inherit `allow_combat`, `allow_pvp`, and `allow_death` from the template via `_set_room_properties()`. All current dungeon templates set `allow_death=False`, routing player defeat through the `_defeat()` flow with dungeon-specific corpse-recovery semantics — see [Death and Corpse Recovery](#death-and-corpse-recovery) for the full design.

Encounter gating (`not_clear` tag) is set by room generators when mobs are present, and cleared by mob `die()` methods when the last mob in a room dies. The check is in a shared `_check_room_cleared()` function that any dungeon mob type can call.

## Death and Corpse Recovery

> **Status: planned design.** The current implementation creates an empty flavour corpse and teleports the defeated player out with their inventory intact. The design described in this section makes corpses loot-bearing and adds a re-entry flow so players can recover their gear, with the dungeon instance staying alive while the corpse exists. Implementation pending — see [Implementation touch-points](#implementation-touch-points) at the end of this section.

### Rationale

`allow_death=False` rooms produce a "soft defeat" — the player is rescued rather than killed. The current behaviour strips no items, since the corpse and surrounding rooms are deleted on instance collapse and a loot-bearing corpse would be unrecoverable.

This creates a hidden inconsistency that players cannot perceive: the same death animation, message, and HP-to-1 result, but with completely different mechanical consequences depending on whether the room they died in was a procedural-dungeon room or a static world room. Procedural rooms are deliberately indistinguishable from static rooms (a core design principle), so the divergent gear-loss outcome feels arbitrary.

The corpse-recovery design closes the gap: defeat in a dungeon strips gear onto a corpse like normal death, but the dungeon instance remains accessible for the duration of the corpse's existence, allowing the player to walk back in and recover.

### State machine

The interaction between three tag categories drives the entire system:

```
       death                                         walk in
        in            recover                        through
     dungeon          corpse                         entrance
        │                │                              │
        ▼                ▼                              ▼
   ┌─────────┐    ┌───────────┐    walk out      ┌──────────┐
   │ pending │◀───│ recovering │◀────────────────│  active  │
   └─────────┘    └───────────┘   (corpse gone)  └──────────┘
        │              ▲                              ▲
        │              │ walk through entrance        │
        └──────────────┘ matching pending template    │
        │                                             │
        │ corpse despawns →                           │
        │ instance collapses → tag scrubbed           │
        ▼                                             │
    ┌─────────┐                                       │
    │  free   │───────────────────────────────────────┘
    └─────────┘   walk in through any entrance
```

Each tag has a specific role:

| Tag | On | Meaning | Set when | Cleared when |
|---|---|---|---|---|
| `dungeon_character` | Player / pet | Currently physically inside an instance | Walking through entry exit | Walking through passage exit / `exit dungeon` / collapse evacuation |
| `dungeon_pending` | Player | Has a corpse to recover in this instance | Dying inside a dungeon | Corpse fully looted (emptiness check) / corpse despawned / instance collapsed |
| `dungeon_corpse` | Corpse | This corpse keeps an instance alive | Created during defeat in a dungeon room | Corpse deleted (corpse-side marker, removed with the corpse) |

A character may hold **at most one** `dungeon_character` tag (they cannot be in two dungeons at once) and **any number** of `dungeon_pending` tags (independent claim tickets across different templates).

The pending tag scrubs the moment the corpse becomes empty (no NFTs, no gold, no resources) — the recovery ticket is consumed. The corpse itself continues to decay on its standard 10-minute timer; an empty looted corpse is just a flavour artifact that vanishes naturally. This means a player who fully recovers can immediately attempt other runs of the same dungeon template without being bounced into the now-empty old instance. Partial loot keeps the pending tag set so the player can return for the rest until either fully looted or natural decay.

### Death flow

When a player drops to 0 HP in a dungeon room (`allow_death=False`):

1. `exit_combat()` — detach combat handler.
2. `clear_all_effects()` — strip DoTs, buffs, debuffs.
3. Create a `Corpse` in the current dungeon room.
4. **Transfer all carried items, equipped gear, gold, and resources onto the corpse** (mirroring the inventory-transfer step of `_real_death()`).
5. Tag the corpse with `dungeon_corpse=instance_key`.
6. Remove the player's `dungeon_character` tag.
7. Add a `dungeon_pending=instance_key` tag to the player.
8. Reset HP to 1.
9. Start the corpse's standard timers (`start_timers()` — 5 minutes owner-only, 10 minutes total).
10. Broadcast defeat to the room.
11. Send the player a clear, prominent message:
    > **You have been defeated! Your body lies in [Dungeon Name] — return through its entrance to reclaim what you carried.**
12. Teleport to `defeat_destination_key` (template field, falls back to `self.home`).

Pets defeated inside the dungeon use their existing `BasePet.die()` path: NFT transitions to `craft_input`, no recovery — pets are permanently destroyed on death regardless of dungeon context, with no `dungeon_pending` tag and no corpse keepalive.

### Re-entry flow

When a player traverses any `ProceduralDungeonExit` or `ConditionalDungeonExit`:

```
if player has dungeon_character tag:
    reject — "You are already in a dungeon."

if player has dungeon_pending tag:
    look up the instance by the tag's instance_key
    if instance is active and its template_id matches THIS entrance's template:
        teleport player to room (0,0) of the existing instance
        add dungeon_character tag (keep dungeon_pending)
        return
    if instance is gone (stale tag):
        scrub the stale dungeon_pending tag
        proceed to normal entry
    otherwise (different template):
        proceed to normal entry — pending tag does not interfere

normal new-instance flow
```

The pending tag is **template-scoped**, not entrance-scoped. A `southern_woods_passage` pending tag will redirect from any of the six southern woods entrances; a `rat_cellar` pending tag is inert when entering southern woods.

### Collapse rule

The instance's `at_repeat()` collapse gate becomes:

> Collapse if no character or pet is **physically present in any dungeon room of this instance**, AND no `dungeon_corpse` tag exists for this `instance_key`.

The `dungeon_pending` tag on a player out in the world does **not** prop the instance up — only the corpse does. This means when a player dies and walks away without returning, the instance's lifetime is bounded by the corpse's despawn timer, not by the player's behaviour. Tag lookups via `search_tag(instance_key, category=...)` are indexed Django queries — efficient regardless of how many characters exist in the world.

### Corpse lifecycle and the "ticket consumed" moment

Dungeon defeat-corpses use the standard `Corpse` 10-minute despawn timer — same as any other player corpse. There is no per-template override. A player who can't make it back in 10 minutes is in the same boat as a player who dies in an outdoor zone they can't reach in time.

The pending-tag claim ticket has two end states:

- **Successful recovery (ticket consumed):** when the corpse becomes empty — no NFTs in contents, no gold, no resources — the `Corpse.at_object_leave` hook detects emptiness and scrubs the owner's matching `dungeon_pending` tag. The owner gets a positive-feedback message: *"You have recovered everything from your remains."* The corpse itself remains as flavour and decays naturally on its 10-minute timer.
- **Unrecovered decay (ticket lost):** when the corpse despawns naturally with items still inside, `Corpse.at_object_delete` scrubs the owner's matching `dungeon_pending` tag and sends a closure message: *"Your remains have decayed — anything left behind is lost."* Items return to RESERVE via the standard despawn cleanup. The instance — now empty of both characters and corpses — collapses on the next `at_repeat` tick.

Both hooks are implemented universally on `Corpse` (not gated to dungeons) so non-dungeon corpse recovery gets the same UX — owner-driven full loot of any player corpse triggers the recovery confirmation, and natural decay of any owned corpse triggers the closure message. The dungeon-specific behaviour (pending-tag scrub) layers on top via a `dungeon_corpse` tag check.

### Edge case matrix

| Scenario | Behaviour |
|---|---|
| Player dies, returns within timer, loots corpse fully, walks out | Last item leaves corpse → `at_object_leave` detects emptiness → owner gets "recovered everything" message → `dungeon_pending` cleared → walks out → `dungeon_character` cleared by passage exit → empty corpse decays on its remaining timer → instance collapses on next tick after corpse decay |
| Player dies, returns, partial loot, walks out, returns again, finishes loot | First trip: pending tag retained because corpse not empty. Re-entry redirected to same instance (pending tag still active). Final loot empties corpse → tag scrubs → walks out → instance collapses after corpse decay |
| Player dies, never returns, corpse despawns | Corpse deletes (still has items) → `at_object_delete` scrubs `dungeon_pending`, sends "decayed" message → instance has no characters and no corpses → collapses next tick |
| Player dies in dungeon A, walks through unrelated dungeon B to reach A's entrance | Pending tag for A doesn't block entry to B (different template). Player carries `dungeon_character=B` and `dungeon_pending=A` simultaneously. Walks out of B → `dungeon_character` cleared, `dungeon_pending=A` retained. Enters A → redirected to existing instance |
| Player has pending corpse from southern_woods_passage NE→clearing run, tries to enter southern_woods_passage SE→clearing entrance | Same template — pending tag fires the redirect. Player teleported into the original NE instance, not the SE entrance they tried. Design feature: must recover before doing another run of the same dungeon type |
| Player has pending corpses in two different dungeon templates | Both `dungeon_pending` tags coexist. Either dungeon's entrance redirects only to its matching pending instance |
| Server restart with pending corpses in flight | Existing startup cleanup collapses all stale instances. Each collapse scrubs its pending tags via the extended `collapse_instance` cleanup. Items in surviving corpses are returned to RESERVE alongside other dungeon fungibles. Connected players see the loss message; offline players see it silently scrubbed by the time they next attempt dungeon entry |
| Player tries to enter a fresh dungeon while pending corpse exists in a *different* template | Allowed — pending tag is template-scoped, doesn't interfere |
| Player tries to enter a fresh dungeon while `dungeon_character` is set | Rejected — "You are already in a dungeon" |
| Pending instance corrupted / deleted out from under the player | Stale pending tag detected on entry attempt — scrubbed silently, normal new-instance flow proceeds |
| Pet dies in dungeon | Standard pet death — NFT to craft_input, no corpse, no keepalive contribution. The instance can collapse normally even if a player's pet died inside it |
| Player disconnects after dying, reconnects later | Reconnects in `defeat_destination`. Pending tag still set if instance is alive — they can walk back in. Otherwise stale tag, scrubbed silently on next entry attempt or on next login if the cleanup pass detects stale tags |
| Multiple players in a group, one dies | Only the dying player gets `dungeon_pending` and is teleported out. Remaining group members continue. The corpse keeps the instance alive even after the rest of the group leaves, until the dead player returns or the timer expires |
| Two group members die in succession | Both get `dungeon_pending` for the same instance. Both corpses tagged `dungeon_corpse`. Either player returning redirects them into the shared instance — they may meet at the entrance or recover separately. Last corpse cleared triggers eligibility to collapse |
| Player dies in dungeon, server restarts before they return | All instances collapsed on boot. The collapse cleanup scrubs pending tags via the extended `collapse_instance` path. If anything escapes, the entry-side redirect detects stale tags (instance not found) and scrubs them silently |
| Corpse moved out of a dungeon room (e.g. dragged through an exit) | Tag still attached to the corpse object. Wherever the corpse is, the instance stays alive while it exists. Edge case — not a normal play action |

### Implementation touch-points

| File | Change |
|---|---|
| `typeclasses/world_objects/corpse.py` | New `is_empty()` helper. New `at_object_leave` hook: universal "recovered everything" message to owner on emptiness + dungeon-specific pending-tag scrub. New `at_object_delete` hook: universal "decayed" message to owner + dungeon-specific pending-tag scrub. |
| `typeclasses/actors/character.py` (`_defeat`) | Detect dungeon room (presence of `dungeon_room` tag); transfer inventory to corpse via shared `_transfer_inventory_to_corpse()` helper; tag corpse `dungeon_corpse`; replace player's `dungeon_character` tag with `dungeon_pending`; revised player message. Non-dungeon `allow_death=False` rooms keep the existing empty-corpse-keep-gear flow unchanged. |
| `typeclasses/actors/character.py` (`_real_death`) | Refactor inventory transfer into the shared `_transfer_inventory_to_corpse()` helper called from both `_real_death` and `_defeat`'s dungeon branch. |
| `typeclasses/mixins/procedural_dungeon.py` (`enter_dungeon`) | Replace blanket `dungeon_character` rejection with: reject if active, redirect-if-pending-matches-template (via new `_try_pending_recovery_redirect`), scrub-if-stale, fall through otherwise. |
| `typeclasses/scripts/dungeon_instance.py` | New `get_present_characters()` and `get_dungeon_corpses()` helpers. `at_repeat` collapse gate considers physical presence AND corpse presence; `dungeon_pending` does not prop instance up. `collapse_instance` scrubs `dungeon_pending` tags with notification, scrubs leftover `dungeon_character` tags from absent characters as defence in depth, defensively despawns any corpses still present (idempotent with the universal corpse hooks). |

## Future Work

- **More templates** — larger dungeons with multiple encounter types, treasure rooms, environmental hazards
- **Mob spawn tables** — configurable per-depth mob spawning instead of hardcoded in room generators
- **Dungeon-specific quest hooks** — generalised patterns for quest completion triggers
- **Treasure/loot system** — reward procedural dungeon completion with items
- **Puzzle elements** — interactive objects in dungeon rooms (doors, switches, environmental hazards)
- **Encounter variety** — traps, locked doors, skill challenges within procedural rooms
