# spawn-mobs.md — Mob Spawn System

Design doc for how mobs are placed and respawn in the game world. Spawning is driven by the [`evennia-mob-spawner`](https://github.com/FullCircleMUD/evennia-mob-spawner) library reading declarative YAML rules from the [`fcm-mobs`](https://github.com/FullCircleMUD/fcm-mobs) content repo. The library owns the engine (tick loop, population maintenance, room selection); FCM owns the data (rules, typeclasses).

For item spawning (loot, NFTs, scrolls), see **unified-item-spawn-system.md**. For service NPCs (bartenders, shopkeepers, trainers), see **npc-quest-system.md** — those have a different lifecycle (immortal, built once by world-builder, never respawn).

---

## How It Works

```
fcm-mobs/shard0/<zone>/<file>.yaml   ← rules (data)
            │
            ▼
GitHubReader (in evennia-mob-spawner)
            │
            ▼
Validator → Deployer → MobSpawnerScript (one per YAML file)
                            │
                            ▼ every MOB_SPAWNER_TICK_SECONDS (default 15s)
                       observe → cooldown → spawn → tag
```

Each leaf YAML file in `fcm-mobs/shard0/<zone>/` becomes one persistent `MobSpawnerScript` in the game DB after `ms_load`. The script holds that file's rule table and runs a tick loop that:

1. **Observes** living mobs per rule by querying the identity tags stamped at spawn time (`mob_spawner_rule = str(rule_id)` AND `mob_spawner_file = <script.db_key>`).
2. **Detects deaths** via count delta — no callback from the typeclass is required.
3. **Gates on cooldown** — either `respawn_seconds` (clock from last spawn) or `death_cooldown_seconds` (clock from observed kill).
4. **Picks a room** — three-tier fallback: pack-spawn with a leader (`spawn_with_typeclass`) → den room (`den_room_tag`) → random within the rule's `area_tag` pool. All steps respect `max_per_room`.
5. **Spawns one mob** — `create_object(typeclass, key, location)` then stamps tags and applies the rule's `desc`/`attrs`/`tags`.
6. **Invokes the optional hook** — if the typeclass defines `ms_at_post_spawn(self)`, the library calls it for per-spawn behaviour resets (rally-cry flags, AI state, etc.).

Population identity is keyed on `(file, rule_id)`. Two rules sharing the same typeclass and area_tag count as independent populations — this is what enables the indistinguishable-variant loot pattern below without subclass proliferation.

---

## Rule Schema

Every field except the required ones is optional. Mix and match for the encounter you want.

| Field | Required | Type | Purpose |
|---|---|---|---|
| `rule_id` | ✓ | int (≥0) | Author-supplied integer, unique within the file. The script's persistent bookkeeping (cooldown clocks, observed counts) is keyed on it. Stable across YAML reordering. |
| `typeclass` | ✓ | str | Dotted Python path to the mob class. Generic classes (`typeclasses.actors.mob.CombatMob`, `typeclasses.actors.mobs.aggressive_mob.AggressiveMob`) are fine when the mob differs only in stats — drive everything via `attrs`. |
| `key` | ✓ | str | The mob's in-game display name. Multiple rules can share a key (the indistinguishable-variant pattern — see below). |
| `area_tag` | ✓ | str | Tag (under category `mob_area`) defining the rule's room pool. Same tag drives the AI wander container. |
| `target` | ✓ | int (≥1) | How many of this rule's mobs should be alive at once. |
| `max_per_room` | ✓ | int (≥1) | Per-room cap. Enforced per-rule via identity tags, so two rules sharing typeclass+area_tag respect their caps independently. |
| `respawn_seconds` | exactly one of | int | Cooldown from last spawn attempt. Right for populations that should stay near target (wolves, kobolds). |
| `death_cooldown_seconds` | exactly one of | int | Cooldown from observed kill time. Right for "boss respawns N minutes after death" semantics. Validator enforces mutual exclusivity with `respawn_seconds`. |
| `desc` | optional | str | Sets `mob.db.desc`. |
| `attrs` | optional | dict | Per-rule attribute overrides applied via `setattr(mob, k, v)`. Routes through `AttributeProperty` to `mob.db`. **Loot lives here** as `spawn_*_max` attrs. |
| `tags` | optional | list | List of tags to stamp on the spawned mob alongside the library's identity tags. Each entry is a bare string (untyped) or a mapping `{key, category?}`. **Loot eligibility lives here** as `spawn_*` tags. Library-reserved category prefix `mob_spawner_` is refused at validation. |
| `spawn_with_typeclass` | optional | str | Pack-spawn trigger. Library finds a living instance of this typeclass within the rule's `area_tag` and spawns into that mob's room. Falls through to den / random when no leader is found. |
| `den_room_tag` | optional | str | Single-room lair fallback. Used after `spawn_with_typeclass` finds no leader, and as the leader's own spawn room. |

---

## Tags Stamped on Every Spawned Mob

The library applies four tag categories at spawn time:

| Category | Key | Source | Purpose |
|---|---|---|---|
| `mob_area` | `area_tag` value | from rule | AI wander containment + (legacy) other consumer queries |
| `mob_spawner_rule` | `str(rule_id)` | identity | half of the population discriminator |
| `mob_spawner_file` | script's `db_key` (file path) | identity | other half of the discriminator |
| consumer-declared | from YAML `tags:` | data | application-specific (loot eligibility, quest markers, faction flags, ...) |

The identity tags are passive — the library queries them, the typeclass doesn't read or write them. They're how `ms_status` / `ms_spawn_report` and the population-counting code find the right mobs without depending on typeclass.

The consumer-declared `tags:` field is where loot eligibility flags live for FCM:

```yaml
tags:
  - {key: spawn_gold, category: spawn_gold}
  - {key: spawn_resources, category: spawn_resources}
```

---

## Loot Model — Data in YAML

**Loot is data, not typeclass behaviour.** Each rule that wants its mob to drop loot declares the runtime values directly in YAML:

```yaml
- rule_id: 1
  typeclass: typeclasses.actors.mobs.wolf.Wolf
  key: a grey wolf
  area_tag: woods_wolves
  target: 12
  max_per_room: 1
  respawn_seconds: 120
  desc: A grey wolf pads through the undergrowth...
  attrs:
    spawn_resources_max: {8: 1}    # 8 = hide id
    spawn_gold_max: 2
  tags:
    - {key: spawn_resources, category: spawn_resources}
    - {key: spawn_gold, category: spawn_gold}
```

The unified item spawn distributor (see unified-item-spawn-system.md) finds drop-eligible targets by querying the `spawn_*` tag, then reads `spawn_*_max` to size the headroom for proportional allocation.

Typeclass `loot_*` defaults are not used anywhere in the spawn path. The `at_object_creation` derivation that used to translate them to runtime `spawn_*_max` is gone. If a mob has no rule (e.g. test-created via raw `create_object`), it carries no loot tags and the distributor ignores it.

---

## Indistinguishable Variant Pattern

Same look, different loot — without subclass proliferation. Make multiple rules share `key`, `desc`, `area_tag`, `typeclass`, with distinct `rule_id`s and per-rule `attrs` / `tags`:

```yaml
# All produce "a grey wolf" with identical desc — players cannot tell them apart.
- rule_id: 1
  typeclass: typeclasses.actors.mobs.wolf.Wolf
  key: a grey wolf
  area_tag: woods_wolves
  target: 12
  attrs: {spawn_resources_max: {8: 1}, spawn_gold_max: 2}    # hide-only
  tags:
    - {key: spawn_resources, category: spawn_resources}
    - {key: spawn_gold, category: spawn_gold}

- rule_id: 2
  typeclass: typeclasses.actors.mobs.wolf.Wolf
  key: a grey wolf
  area_tag: woods_wolves
  target: 1
  attrs: {spawn_scrolls_max: {basic: 1}}                       # scroll-only
  tags:
    - {key: spawn_scrolls, category: spawn_scrolls}
```

Apparent loot randomness comes from population ratios, not per-kill RNG. Compliance-relevant: deterministic supply governed by population control isn't a gambling mechanic.

The library counts each rule's population independently via the `(file, rule_id)` identity tags, so the rules don't fight each other for `target` or `max_per_room` even though their typeclass + area_tag are identical.

---

## Admin Commands

All ship from the library, auto-installed into `AccountCmdSet`, locked to `cmd:superuser()`. Same scope syntax across all: `all | <level>=<value> [<level>=<value> ...]`. Bare command (no args) prints usage.

Because FCM's levels are `(shard, zone, file)`, every scope starts at `shard=`. Five of the six are gated to the shard they act on: `all` is refused, and so is any shard other than the one this process runs as — which rules out running them from the router. Work one shard at a time. See [evennia-mob-spawner/docs/interoperability.md](../libraries/evennia-mob-spawner/docs/interoperability.md).

| Command | Where it runs | Effect |
|---|---|---|
| `ms_load shard=shard0` | owning shard | Fetch that shard's rule sets via the configured Reader, validate, and deploy. Per-file scripts get upserted in place; cooldown state preserved for rules that survive the swap. |
| `ms_load shard=shard0 zone=millholm file=town` | owning shard | Same but scoped to one file. |
| `ms_status [scope]` | owning shard | Read-only: lists scripts in scope with state, rule count, tick interval, and a next-tick estimate for a script that is ticking. State is colour-coded — green `active`, yellow `paused` / `stopped`, red `stalled`. |
| `ms_spawn_report [scope]` | router only | Live population census: per-rule current vs target, grouped by `area_tag`. Under-target rules marked with `*`. Counts run under the tenancy auto-filter, so only an unscoped process can answer for the whole world. |
| `ms_restart [scope]` | owning shard | Kick the ticker without re-reading YAML. Recovery for stopped, paused and stalled scripts; preserves state. Confirms a tick actually attached before reporting success. |
| `ms_stop [scope]` | owning shard | Pause the ticker. State preserved; resumable via `ms_restart`. Reports a stalled script rather than claiming to have paused it. |
| `ms_delete [scope]` | owning shard | Remove scripts entirely (state lost). Use to clean up orphans whose YAML files have been removed from the manifest. |

The four ticker commands are shard-gated because Evennia keeps a script's `LoopingCall` in `ndb._task`, which exists only in the process running it. Run from elsewhere, `ms_stop` writes nothing at all and still reports success, `ms_delete` removes the shared row while the owning shard keeps ticking a script that no longer exists, and `ms_status` would call every script it does not own stalled.

Each spawner script is also stamped with the shard that owns it, so a restart can't hand it to the wrong process — see [scaling.md](scaling.md#global-scripts--mechanism-and-classification-settled).

### When nothing is respawning

A script can be marked `active` and still have no tick attached — Evennia writes its pause marker only
when a task exists, so a task lost any other way leaves a state that `pause()`, `unpause()` and the
server's own boot walk all skip over. A restart does not clear it. Run
`ms_status shard=shard0` on the shard itself and the state reads `stalled`.

`ms_restart shard=shard0` is the fix, run on the owning shard. It is safe across a whole shard —
healthy scripts are left alone — though it will also wake anything deliberately stopped with
`ms_stop`. `ms_load` refuses while any script in scope is stalled and names them, so reaching for it
first will point you here rather than reporting a deploy onto a dead script. Sightings are logged to
`mob_spawner.log`; what causes the task to be lost is not yet established.

---

## Settings Wiring

```python
# src/game/server/conf/settings.py
MOB_SPAWNER_READER = "evennia_yaml_reader.github.GitHubReader"
MOB_SPAWNER_REPO   = "FullCircleMUD/fcm-mobs"
MOB_SPAWNER_REF    = "main"
MOB_SPAWNER_GITHUB_PAT = os.environ.get("MOB_SPAWNER_GITHUB_PAT", "")  # set in secret_settings.local

# Composed at file bottom (after secret_settings load so PAT override propagates)
MOB_SPAWNER_READER_KWARGS = {
    "repo": MOB_SPAWNER_REPO,
    "ref": MOB_SPAWNER_REF,
    "pat": MOB_SPAWNER_GITHUB_PAT,
}
```

`evennia_mob_spawner` is in `INSTALLED_APPS`. Library auto-installs its admin commands at server start via `AppConfig.ready()`.

Optional tunables (defaults shown):
- `MOB_SPAWNER_TICK_SECONDS = 15` — interval for every `MobSpawnerScript`. Library-level, not per-rule.
- `MOB_SPAWNER_AREA_TAG_CATEGORY = "mob_area"` — tag category for `area_tag` / `den_room_tag` queries.

---

## Death Lifecycle

`CombatMob.die()` runs the common death sequence (corpse, loot transfer, XP, alignment) and deletes the mob. The library observes the death via tick-time count delta — no callback from the typeclass to the script is required or accepted. The next tick's spawn-decision logic sees `current < target` and (if cooldown allows) repopulates.

For rules using `death_cooldown_seconds`, the library stamps `last_death_time` on the rule when it observes the count drop. The cooldown clock for the next spawn starts from that timestamp.

---

## ms_at_post_spawn — Per-Spawn Behaviour Reset

The library calls `mob.ms_at_post_spawn()` after every fresh spawn IF the typeclass defines that method. Use it to reset state that needs to be clean at the start of each life — e.g. a boss's `db.has_rallied` flag that wouldn't be cleared by `at_object_creation` because its default is `None`:

```python
class KoboldChieftain(CombatMob):
    def ms_at_post_spawn(self):
        """Reset per-spawn combat state on every fresh spawn."""
        self.db.has_rallied = False
        # ... any other per-life setup
```

Duck-typed protocol: one optional method name, no inheritance demands, no required base class. Validator's Tier 3 checks the signature is `mob.ms_at_post_spawn()` (zero required args after `self`) so typos like `def ms_at_post_spawn(self, foo):` are caught at load time, not runtime.

Method on the typeclass is the canonical home — it lives with the code it resets state for, not in a sibling module of dotted-path callbacks.

---

## Area Tags — Spawn Pool + Wander Container

Every spawned mob carries the rule's `area_tag` under category `mob_area`. One tag, two purposes:

1. **Spawn room pool** — `_pick_room` queries rooms tagged `mob_area=<area_tag>`. Library's identity-tag filter ensures `max_per_room` is counted per-rule, not pooled across rules sharing the area.
2. **AI wander container** — `ai_wander` only steps into other rooms with the same tag. A wolf tagged `woods_wolves` won't wander into the farms.

**Pinning a mob to one room:** give that room a dedicated tag used only by the rule that should spawn there. Rooms can carry multiple `mob_area` tags — adding a dedicated tag doesn't remove or interfere with broader tags.

**Builder responsibility:** when authoring rooms in the world-builder YAML (fcm-world), tag them with `mob_area:<area_tag>` for any spawn rule that should populate them. Without a tag, no mobs spawn there.

---

## What This System Does Not Handle

- **Service NPCs** — bartenders, shopkeepers, guildmasters, trainers, librarian, townfolk-with-quests, etc. These are authored as YAML entities in **fcm-world** and default to `is_immortal=True` so they never reach `die()`. They have a persistent dbref by virtue of never dying. This system does not respawn them.
- **Procedural dungeon mobs** — dungeon templates (`world/dungeons/templates/`) own their mobs; mobs are created when an instance spawns and never respawn. See `procedural-dungeons.md`.
- **Loot composition** — what items appear on a corpse is handled by the unified item spawn service. See `unified-item-spawn-system.md`. The mob spawn system's only loot-related responsibility is stamping `spawn_*_max` attrs + `spawn_*` tags as declared by each rule.
- **Resource nodes** — see `unified-item-spawn-system.md`.

---

## Operational Notes

**Hot reload.** `ms_load` re-fetches the YAML from GitHub and re-deploys. Scripts that survive the redeploy preserve their cooldown clocks; rules removed from the YAML are purged from the script's bookkeeping.

**Restart-safe.** Population state persists naturally: spawned mobs are real Evennia objects, the script's `db.last_spawn_times` / `db.last_death_times` survive restarts, dead mobs were deleted before the restart.

**Targets are not caps.** A coordinated party can drive a zone below target by clearing faster than the cooldown — that's intended. Cooldowns control how fast the zone refills, not whether players can clear it.

**Cooldown precision.** The audit runs on the tick interval (default 15s), so any cooldown is effectively `cooldown + (0–15)` seconds. Don't design encounters that rely on second-level precision.

**No player-count scaling.** A zone configured for 6 wolves is 6 wolves whether 1 or 100 players are online. Future enhancement if needed.

**Event content.** Special events swap mobs by editing the YAML repo and reloading — "the woods are overrun with corrupted wolves this week" is a YAML diff + `ms_load`.

---

## Files

```
# Library (engine) — separate repo, editable install
libraries/evennia-mob-spawner/
└── src/evennia_mob_spawner/
    ├── script.py        # MobSpawnerScript: tick loop, room selection, spawn-one
    ├── commands.py      # ms_load, ms_status, ms_spawn_report, ms_restart, ms_stop, ms_delete
    ├── validator.py     # rule-schema predicates (rule_id, typeclass, attrs, tags, ...)
    ├── deployer.py      # upsert-with-state-preservation
    ├── loader.py        # YAML → LoadResult
    └── finder.py        # manifest walker (definitions.yaml + per-folder index.yaml)

# Content (rules) — separate repo, fetched via GitHubReader
fcm-mobs/
├── definitions.yaml     # levels: [shard, zone, file], CI-gating flag
├── index.yaml           # top-level shard list
└── shard0/
    └── millholm/
        ├── town.yaml    # one MobSpawnerScript per leaf file
        ├── woods.yaml
        └── ...

# Consumer (typeclasses) — FCM gamedir
src/game/typeclasses/actors/
├── mob.py               # CombatMob base
└── mobs/                # concrete typeclasses (Wolf, Kobold, Skeleton, ...)
```
