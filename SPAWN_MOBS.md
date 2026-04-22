# SPAWN_MOBS.md — Mob Spawn System

Design doc for how mobs are placed and respawn in the game world. One system, one engine — `ZoneSpawnScript` reading per-zone JSON rules. Mobs differ in their JSON configuration: how many, how often, what room, what AI. There is no "boss vs commodity" distinction in the machinery — only knobs.

For item spawning (loot, NFTs, scrolls), see **UNIFIED_ITEM_SPAWN_SYSTEM.md**. For service NPCs (bartenders, shopkeepers) which use a different lifecycle, see **NPC_QUEST_SYSTEM.md**.

---

## How It Works

Each zone has a JSON file at `world/spawns/<zone_key>.json` containing a list of spawn rules. A `ZoneSpawnScript` for that zone is created at world build time via `ZoneSpawnScript.create_for_zone(zone_key)`, loads the JSON, and ticks every 15 seconds. On each tick, it audits each rule:

1. Count living mobs matching the rule (`typeclass + area_tag`).
2. If count is below `target` and the cooldown has elapsed, pick an eligible room and spawn one.
3. Stamp `last_spawn_times[rule_id] = now` so the next spawn waits for the cooldown.

When a mob dies, `mob.die()` deletes the object and (when configured) notifies the script so cooldowns can restart from kill time rather than from the last spawn attempt. The script's next tick repopulates.

Rule identity is `f"{typeclass}:{area_tag}"` — same typeclass in different areas counts as different rules.

---

## JSON Rule Fields

Every field is optional except `typeclass`, `key`, and `area_tag`. Mix and match for the encounter you want.

| Field | Type | Purpose |
|---|---|---|
| `typeclass` | str | Dotted Python path to the mob class. Use a generic class (`typeclasses.actors.mob.CombatMob`) when the mob differs only in stats — see `attrs`. |
| `key` | str | The mob's in-game name. Any string — unique names like `"the Heffalump"` or `"Rabbit"` work fine. |
| `area_tag` | str | Tag that scopes the rule. Rooms tagged `mob_area=<area_tag>` form the spawn pool *and* the AI wander container. To pin to a single room, give that room a dedicated tag. |
| `target` | int | How many of this rule should be alive at once. |
| `max_per_room` | int | Cap per room. `0` means no cap. Use `1` to prevent stacking and to pin solitary mobs alongside `target=1`. |
| `respawn_seconds` | int | Cooldown measured from the **last spawn attempt**. Right for populations that should always be near target — wolves, kobolds. |
| `death_cooldown_seconds` | int | Cooldown measured from **kill time** (the script is notified by `mob.die()`). Right for "respawn N minutes after death" semantics. Overrides `respawn_seconds` if both are set. |
| `desc` | str | Sets `mob.db.desc`. |
| `attrs` | dict | Per-rule stat/attribute overrides applied via `setattr(mob, k, v)` after creation. Routes through `AttributeProperty` to `.db`. Use this to give one generic typeclass many distinct named instances. |
| `post_spawn_hook` | str (dotted path) | Module-level `fn(mob) -> None` invoked after `start_ai()` on every fresh spawn. Used to set AI state, reset per-life flags, etc. |

A rule's `attrs` dict can also override loot fields (`loot_gold_max`, `loot_resources`, `spawn_recipes_max`, `spawn_scrolls_max`). The script re-reads these after applying `attrs` and updates spawn-tag categories so the unified item spawn service can find the mob.

---

## Examples

These illustrate the full range — same engine, different configurations.

### Wolf pack — many mobs, wide area, spawn-attempt cooldown

```json
{
    "typeclass": "typeclasses.actors.mobs.wolf.Wolf",
    "key": "a grey wolf",
    "area_tag": "millholm_woods",
    "target": 6, "max_per_room": 1,
    "respawn_seconds": 180,
    "desc": "A grey wolf prowls through the undergrowth."
}
```

Six wolves spread across every room tagged `mob_area=millholm_woods`, no more than one per room. After a kill the next replacement spawns 3 minutes after the *previous* spawn (or sooner if the previous spawn was already > 3 min ago).

### Stationary atmospheric NPC — generic typeclass, full stat overrides, idle hook

```json
{
    "typeclass": "typeclasses.actors.mob.CombatMob",
    "key": "Rabbit",
    "area_tag": "haw_rabbit_house",
    "target": 1, "max_per_room": 1,
    "death_cooldown_seconds": 300,
    "post_spawn_hook": "typeclasses.scripts.spawn_hooks.set_ai_idle",
    "desc": "A fussy-looking rabbit with a permanent expression of mild irritation.",
    "attrs": {
        "room_description": "gives you a dirty look, you must have stepped in his garden.",
        "damage_dice": "2d4+1", "level": 5,
        "base_hp_max": 55, "hp_max": 55, "hp": 55,
        "base_strength": 17, "strength": 17,
        "base_armor_class": 15, "armor_class": 15,
        "loot_gold_max": 10,
        "spawn_recipes_max": {"skilled": 1}
    }
}
```

`area_tag` belongs to a single room (`rabbit_house`), so `target=1` pins Rabbit there. `set_ai_idle` keeps him stationary — without it, base CombatMob's `ai_wander` would move him 30% of ticks. Death cooldown of 5 minutes gives a brief consequence window after a kill.

### Single-room unique encounter with state reset

```json
{
    "typeclass": "typeclasses.actors.mobs.kobold_chieftain.KoboldChieftain",
    "key": "the Kobold Chieftain",
    "area_tag": "mine_kobold_warren",
    "target": 1, "max_per_room": 1,
    "death_cooldown_seconds": 1800,
    "post_spawn_hook": "typeclasses.scripts.spawn_hooks.reset_chieftain_state",
    "desc": "A squat but powerfully built kobold wearing a crude crown…"
}
```

A bespoke typeclass (`KoboldChieftain`) handles combat-tick behaviours like the rally cry. The post-spawn hook clears `db.has_rallied` on every fresh spawn. `mine_kobold_warren` is a tag attached to exactly one room, alongside the broader `mine_kobolds` tag used by the surrounding kobold population.

### Loot variants — same look, different drops

```json
[
    {"typeclass": "...mobs.kobold.Kobold",         "key": "a kobold", "area_tag": "mine_kobolds", "target": 9, "max_per_room": 3, "respawn_seconds": 120, "desc": "..."},
    {"typeclass": "...mobs.kobold.KoboldRecipeLoad","key": "a kobold", "area_tag": "mine_kobolds", "target": 3, "max_per_room": 3, "respawn_seconds": 120, "desc": "..."}
]
```

12 kobolds in the area, but 25% drop a recipe — controlled by spawn-table targets, not runtime RNG. Both rules share `key`, `desc`, and `area_tag` so players can't tell them apart. See § Pseudo-Random Loot below.

### Mastery override on a generic mob class

```json
{
    "typeclass": "typeclasses.actors.mobs.kobold.KoboldWarrior",
    "key": "a kobold warrior",
    "area_tag": "mine_kobolds",
    "target": 2, "max_per_room": 1,
    "respawn_seconds": 240,
    "attrs": {
        "class_skill_mastery_levels": {"bash": 3},
        "weapon_skill_mastery_levels": {"dagger": 3}
    }
}
```

`attrs` can promote any mob's mastery levels per-rule without subclassing.

---

## Area Tags — Spawn Pool + Wander Container

Every spawned mob is tagged with the rule's `area_tag` (category `mob_area`). One tag, two purposes:

1. **Spawn room pool** — `_pick_spawn_room` queries rooms tagged `mob_area=<area_tag>`, picks one that isn't at `max_per_room` capacity, and spawns there.
2. **AI wander container** — `ai_wander` only walks into other rooms with the same tag. A wolf tagged `millholm_woods` won't wander into the farms.

Single source of truth — define an area once, mobs spawn and roam within it.

**Pinning a mob to one room:** give that room a dedicated tag used only by the rule that should spawn there. Rooms can carry multiple `mob_area` tags — adding a dedicated tag doesn't remove or interfere with broader tags. The Kobold Warren has `mine_kobolds` (for the surrounding kobold population) and `mine_kobold_warren` (used only by the chieftain rule).

**Builder responsibility:** when adding rooms to a zone, tag them with `set_zone()` and any appropriate `mob_area` tags. Without tags, no mobs will spawn there.

---

## Death Lifecycle

`CombatMob.die()` runs the common death sequence (corpse, loot transfer, XP, alignment) and then:

```python
# Notify ZoneSpawnScript when this mob carried death-cooldown metadata
rule_id = self.db.spawn_rule_id
zone_key = self.db.spawn_zone_key
if rule_id and zone_key:
    script = ScriptDB.objects.filter(db_key=f"zone_spawn_{zone_key}").first()
    if script:
        script.on_mob_death(rule_id)

# Branch on is_unique
if self.is_unique:
    # Service NPC path (bartenders, shopkeepers, pets) — see below
    self.location = None
    delay(self.respawn_delay, self._respawn)
else:
    # All combat mobs — delete; ZoneSpawnScript spawns a fresh replacement
    self.delete()
```

The `db.spawn_rule_id` and `db.spawn_zone_key` attributes are stamped on the mob at spawn time when its rule sets `death_cooldown_seconds`. `on_mob_death(rule_id)` updates `last_spawn_times[rule_id] = now`, so the existing cooldown check in `_check_rule` then naturally measures from death.

`last_spawn_times` is persisted on the script's `db`, so cooldowns survive restarts.

### Post-spawn hooks

`at_object_creation()` runs on every fresh spawn and sets up equipment, base stats, etc. `post_spawn_hook` is the place for state that needs explicit per-life reset — flags that wouldn't be cleared by `at_object_creation` because they default to `None` (Evennia's `db` returns `None` for unset attrs, but a typeclass might assume a different default).

Two homes for hooks:
- **Generic, reusable** — `typeclasses/scripts/spawn_hooks.py` (e.g. `set_ai_idle(mob)`, `reset_chieftain_state(mob)`).
- **Specific to one mob** — alongside the typeclass file.

A bad hook path logs an error via `logger.log_err` and the spawn proceeds. Hooks fail safe.

---

## Pseudo-Random Loot via Spawn Variants

To make loot feel randomised without per-kill RNG, use **class variants** — multiple typeclasses identical in appearance, stats, and behaviour, differing only in the loot the unified spawn service loads onto them.

**Pattern:**
1. **Base class** — drops gold, no knowledge.
2. **Loot variant(s)** — child classes with `loot_gold_max = 0` and `spawn_recipes_max` / `spawn_scrolls_max` populated.

All variants share the same `key`, `desc`, `area_tag`, and `max_per_room` in the spawn JSON. Players cannot tell them apart; the apparent randomness is the population ratio.

Examples in the codebase: `Kobold` + `KoboldRecipeLoad`; `Gnoll` + `GnollRecipeLoad` + `GnollScrollLoad`; `Jagular` + `JagularRecipeLoad` + `JagularScrollLoad`; `Woozle` + `WoozleRecipeLoad` + `WoozleScrollLoad`.

**Guidelines:**
- Variant classes inherit everything from the base, override only loot attributes.
- Keep variants in the same file as the base for discoverability.
- Same `key` and `desc` across variants is mandatory — players must see one creature, not three.
- Same `area_tag` across variants means they share the same room pool and wander together.
- Adjust ratios by changing JSON target counts, no recompile.

---

## What This System Does Not Handle

- **Service NPCs with `is_unique=True`** — bartenders, shopkeepers, guildmasters, trainers, librarian, town guards, townfolk, and pets. These are placed once by world builders in `npcs.py` files and use `delay(self.respawn_delay, self._respawn)` to come back with the same dbref after death. They live a different lifecycle (placed once, persistent identity, dbref preserved). Don't migrate these — the legacy path is correct for them.
- **Procedural dungeon mobs** — dungeon templates (`world/dungeons/templates/`) own their mobs; mobs are created when an instance spawns and never respawn. See `PROCEDURAL_DUNGEONS.md`.
- **Mob loot tables** — what items appear on a corpse is handled by the unified item spawn service. See `UNIFIED_ITEM_SPAWN_SYSTEM.md`.
- **Resource nodes** — see `UNIFIED_ITEM_SPAWN_SYSTEM.md`.

---

## Operational Notes

**Hot reload.** JSON files can be re-read without restarting. `ZoneSpawnScript` re-reads its rules on demand — useful while building.

**Restart-safe.** Population state persists naturally: spawned mobs are real Evennia objects, the script's `db.last_spawn_times` is persisted, and dead mobs were already deleted before the restart. Cooldowns survive.

**Targets are not caps.** A coordinated party can drive a zone below target by clearing faster than the cooldown — that's intended behaviour. Cooldowns control how fast the zone refills, not whether players can clear it.

**Cooldown precision.** The audit runs on the 15-second tick, so any cooldown is effectively `cooldown + (0–15)` seconds. Don't design encounters that rely on second-level precision.

**No player-count scaling.** A zone configured for 6 wolves is 6 wolves whether 1 or 100 players are online. Future enhancement if needed; not currently a problem.

**Event content.** Special events can swap mobs by editing JSON and reloading — "the woods are overrun with corrupted wolves this week" is a one-line change.

---

## Future Work

The current engine handles unconditional respawn (with cooldown semantics chosen per rule). Conditional spawning is unbuilt:

- **Time gating** — bosses that only appear at night, in winter, during a storm.
- **Spawn chance** — % per respawn window rather than guaranteed-after-cooldown.
- **Population prerequisite** — a chieftain that only spawns when the surrounding pack is intact.
- **Global event flags** — world state toggles to enable/disable spawn rules (seasonal, quest-driven).

Each of these is implementable as an extra optional JSON field plus a check in `_check_rule`. They'll be added when the first encounter that needs them is designed.

---

## System Interactions

| System | How It Relates |
|---|---|
| **AI handlers** | Spawned mobs use `area_tag` for wander containment via the AI state machine; `post_spawn_hook` can flip the initial AI state (e.g. `set_ai_idle`). |
| **Combat** | Mob death triggers deletion (or the legacy NPC respawn path). The script repopulates after the cooldown. |
| **Procedural dungeons** | Dungeons spawn their own mobs — outside the zone-script scope. |
| **Quest system** | Quest progress matches kills by typeclass string, not dbref. JSON create/delete on each respawn is safe — quests still tick. |
| **Combat memory** | `CombatMemory` is keyed on mob_type / mob_name / level. Persists across the dbref churn from JSON respawn. |
| **Unified item spawn service** | Loot drops are placed onto mobs by the unified spawn service tagging. The zone script re-syncs `spawn_resources` / `spawn_gold` / `spawn_scrolls` / `spawn_recipes` tag categories after applying `attrs`. |
| **World builder scripts** | Call `ZoneSpawnScript.create_for_zone(zone_key)` after rooms exist and tags are placed. |

---

## Files

```
typeclasses/scripts/
├── zone_spawn_script.py      ← engine: tick, populate, _check_rule, _spawn_mob, on_mob_death
└── spawn_hooks.py            ← reusable post_spawn_hook callables (set_ai_idle, reset_chieftain_state)

world/spawns/
└── <zone_key>.json           ← per-zone rule lists

typeclasses/actors/
├── mob.py                    ← CombatMob.die() (deletion + death notification + legacy is_unique branch)
└── mobs/                     ← mob typeclasses (Wolf, Kobold, KoboldChieftain, GnollWarlord, …)
```
