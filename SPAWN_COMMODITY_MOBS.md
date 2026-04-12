# SPAWN_COMMODITY_MOBS.md — Zone-Based Mob Population System

> Design document for how common (commodity) mobs are spawned and maintained in static game zones. For boss mob spawning, see **SPAWN_BOSS_MOBS.md**. For technical implementation, see **src/game/CLAUDE.md** § Zone Spawn Script.

---

## Purpose

The game world must feel alive. Mobs killed by players need to reappear. Zones need a stable, manageable population — not too sparse (boring), not too crowded (overwhelming).

This system handles **commodity mobs** — the common, repeatable encounters that make up the bulk of the world's danger: wolves, kobolds, gnolls, cellar rats, bandits. These are not unique or rare encounters. They exist in fixed populations that self-maintain.

---

## Two Spawn Systems — By Design

FCM deliberately uses **two separate mob spawn systems** for different purposes:

| System | Handles | Mechanism |
|---|---|---|
| **ZoneSpawnScript** (this document) | Common, repeatable mobs | Population maintenance — always near target count |
| **Boss Mob Spawn** (see SPAWN_BOSS_MOBS.md) | Rare, unique encounters | Spawn chance, time/weather gating, one-at-a-time enforcement |

Mixing these into one system would create a mess. Wolves need to respawn reliably after every player kill. A boss dragon needs spawn conditions, a cooldown, and "already alive" detection. Different problems, different tools.

---

## How It Works

### JSON Spawn Rules

Each zone has a JSON file (`world/spawns/<zone_key>.json`) that defines what lives there:

```json
[
  {
    "typeclass": "typeclasses.actors.mobs.wolf.Wolf",
    "key": "a grey wolf",
    "area_tag": "millholm_woods",
    "target": 6,
    "max_per_room": 1,
    "respawn_seconds": 180,
    "desc": "A grey wolf prowls through the undergrowth."
  },
  {
    "typeclass": "typeclasses.actors.mobs.kobold.Kobold",
    "key": "a kobold",
    "area_tag": "mine_kobolds",
    "target": 9,
    "max_per_room": 3,
    "respawn_seconds": 120,
    "desc": "A scrawny, reptilian creature crouches in the shadows."
  }
]
```

**Rule identity:** Each rule is identified by `typeclass + area_tag`. The same mob type can have different rules in different zones. "There are wolves in the woods" and "there are wolves in the highlands" are two independent rules.

### Population Audit (Every 15 Seconds)

`ZoneSpawnScript` ticks every 15 seconds (`self.interval = 15`). On each tick:

1. Count living mobs matching each rule (`typeclass + area_tag` tag)
2. If count < `target` and the respawn cooldown has elapsed: spawn one replacement
3. Repeat until count meets target (one spawn per tick per rule to avoid bursts)

**Why 15 seconds?** Fast enough that players don't notice a dead mob zone after a group clears it. Slow enough that a single kill doesn't immediately respawn a replacement in view.

### Mob Death → Deletion

Common mobs (`is_unique = False`) are **deleted** on death, not just dropped to the floor waiting for cleanup. The `ZoneSpawnScript` then spawns a fresh object when the population audit next runs.

**Why delete instead of persist?** Fresh spawns reset stats, position, and AI state cleanly. A partially looted corpse or a mob stuck mid-flee is just noise. For common mobs, a fresh copy is always preferable to a recycled one.

### Respawn Cooldown

Each rule has `respawn_seconds` — minimum time before a replacement spawns. This is tracked per-rule, not per-room.

**Design intent:** After a group clears an area, they get a window of safety before mobs return. A 3-minute respawn means clearing the mine entrance gives the party ~3 minutes of breathing room. Use longer cooldowns for more dangerous mobs to reward clearing; use shorter cooldowns for ambient wildlife that should feel persistent.

### max_per_room

Each rule can set `max_per_room`. The spawn system won't place a mob in a room that already contains `max_per_room` of that mob type (checking by typeclass).

**Use case:** Wolves use `max_per_room = 1` so they don't pack up in a single room while wandering. Kobolds might allow 3 per room to enable the pack-courage group behaviour. Bank guards should use `max_per_room = 1` so doorways aren't gridlocked.

---

## Area Tags: Spawn Zone + Wander Containment

Every spawned mob is tagged with the rule's `area_tag`. This tag serves **two purposes**:

### 1. Spawn Room Pool

When spawning a replacement, the system picks a random room tagged `category="spawn_zone"` with the matching `area_tag`. This means mobs distribute naturally across the zone rather than all spawning in the same room.

**Builder responsibility:** Tag rooms with `set_zone()` / zone-specific area tags when building. Every room in the mine entrance area should have `area_tag = "millholm_mine"` for kobolds to spawn across the whole area.

### 2. AI Wander Containment

Mob AI uses the same `area_tag` to restrict wandering. A wolf tagged `millholm_woods` only wanders into rooms also tagged `millholm_woods`. It won't wander into the farm district or the town.

**Why the same tag?** Single source of truth. The area that defines "where this mob lives" is the same area that constrains "where this mob can go". No separate wander-zone configuration.

---

## Per-Zone Configuration

### Creating a New Zone's Spawn Rules

1. Create `world/spawns/<zone_key>.json` with the rules array
2. Tag all zone rooms with the appropriate `area_tag` for both spawn pool and AI wander containment
3. Call `ZoneSpawnScript.create_for_zone("zone_key")` in the world builder or server init

### Hot Reload

JSON files can be reloaded without restarting the server. `ZoneSpawnScript` re-reads its JSON on demand. Useful during world building — adjust numbers and reload without a full `evennia restart`.

---

## What This System Does NOT Handle

- **Boss mobs** — see SPAWN_BOSS_MOBS.md
- **Unique/named NPCs** — these are placed once by world builders and use legacy delay-based `_respawn()` if they need to respawn at all
- **Procedural dungeon mobs** — dungeons manage their own mob populations internally via `DungeonTemplate`
- **Mob loot tables** — item drops are handled by the unified spawn system (see UNIFIED_ITEM_SPAWN_SYSTEM.md)
- **Resource nodes** — see UNIFIED_ITEM_SPAWN_SYSTEM.md

---

## System Interactions

| System | How It Relates |
|---|---|
| **AI Mob System** | Spawned mobs use `area_tag` for wander containment via the AI state machine |
| **Combat System** | Mob death triggers deletion; ZoneSpawnScript replenishes after respawn_seconds |
| **Dungeon Templates** | Procedural dungeons spawn their own mobs — ZoneSpawnScript covers only static zones |
| **SPAWN_BOSS_MOBS** | Companion system for rare encounters — different trigger logic, same tagged room pools |
| **World Builder scripts** | `create_for_zone()` called during `build_game_world.py` to initialise zone populations |
| **Server restart** | Scripts persist via Evennia's persistence — population is maintained across restarts |

---

## Pseudo-Random Loot via Spawn Variants

To make loot feel randomised without adding runtime RNG to the spawn system, use **class variants** — multiple typeclasses that are identical in appearance, stats, and behaviour but differ only in what loot the unified spawn service loads onto them.

### Pattern

1. **Base class** — the "common" variant. Drops gold, no knowledge loot.
2. **Loot variant(s)** — child classes that override `loot_gold_max = 0` and add `spawn_recipes_max`, `spawn_scrolls_max`, or other loot attributes.

All variants share the same `key`, `desc`, `area_tag`, and `max_per_room` in the spawn JSON so players cannot distinguish them. They wander and mix freely within the same area.

### Example: Kobolds (Millholm Mine)

```
Kobold (base)          — loot_gold_max=2, no knowledge
KoboldRecipeLoad       — loot_gold_max=0, spawn_recipes_max={"basic": 1}
```

Spawn split: 9 base + 3 recipe = 12 total. A player killing kobolds sees ~75% drop gold, ~25% drop a recipe — but the ratio is controlled entirely by the spawn rule targets, not by a loot table roll.

### Example: Gnolls (Millholm Southern)

```
Gnoll (base)           — loot_gold_max=3, no knowledge
GnollRecipeLoad        — loot_gold_max=0, spawn_recipes_max={"basic": 1}
GnollScrollLoad        — loot_gold_max=0, spawn_scrolls_max={"basic": 1}
```

Spawn split: 5 base + 2 recipe + 1 scroll = 8 total.

### Why This Approach

- **No code changes** — uses existing ZoneSpawnScript and unified spawn service as-is.
- **Tunable** — adjust ratios by changing JSON target counts, no recompile.
- **Transparent** — the "loot table" is the spawn rule file, readable at a glance.
- **Indistinguishable** — same key, desc, and behaviour means players experience apparent randomness.
- **Composable** — works for any mob type. Add a `WolfHideLoad` variant, a `SkeletonScrollLoad`, etc.

### Guidelines

- Variant classes should be minimal — inherit everything from the base, override only loot attributes.
- Keep all variants in the same file as the base class for discoverability.
- Always use the same `key` and `desc` across variants so players can't tell them apart.
- The `area_tag` must match across variants so they share the same room pool and wander together.

---

## Design Notes

**Target counts are not firm caps.** They're targets. If players clear faster than the respawn timer, the zone will temporarily drop below target. This is intentional — coordinated groups should be able to "clear" an area.

**Scaling with player count:** Current system does not scale target_count with server population. A zone with 6 wolves is 6 wolves whether there are 1 or 100 players. This may need revisiting if the server scales significantly — a future enhancement could multiply targets by an active player ratio.

**Event content:** During special events, JSON files can temporarily swap in different mobs ("the forest is overrun with corrupted wolves this week") by editing the JSON and reloading. No code changes needed.
