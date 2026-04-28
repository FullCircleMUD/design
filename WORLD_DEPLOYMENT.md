# World Deployment

> Technical design document for the world build, redeploy, and hot-reload pipeline. Covers how zones and districts are loaded into the running database, how individual districts can be redeployed without disrupting players in other parts of the world, and the tag and identity conventions that make this safe. For room and exit internals, see [ROOM_ARCHITECTURE.md](ROOM_ARCHITECTURE.md) and [EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md). For zone-to-zone travel, see [INTERZONE_TRAVEL.md](INTERZONE_TRAVEL.md). For multi-shard sharding (which uses the same zone seam), see [SCALING.md](SCALING.md). This document is about deploying **content into the running game**, not about deploying the game process to infrastructure.

## Design Philosophy

The world is content, and content changes more often than systems. As game systems stabilise, the dominant ongoing work shifts to authoring new zones, balancing existing districts, and patching bugs in specific rooms. The deployment pipeline must follow that shift:

- **Locally** the burden of rebuilding the entire world is small, and the simplest path is fine.
- **In production** rebuilding the entire world for a one-room fix is unacceptable. We need to redeploy content at the smallest sensible granularity — a district — without disturbing players in unrelated parts of the world.

The pipeline is therefore tiered. A small **world base** of connective tissue is built once and effectively never touched. **Zones** are the unit of authorship. **Districts** are the unit of redeploy. Each tier is rebuildable in isolation by the tier above it, and the design refuses to introduce parallel systems where Evennia or the existing FCM helpers already cover the need.

**This is a manual-operator design, not an automation layer.** Operators broadcast a warning to a zone's players, wait for them to evacuate, then manually run the redeploy command. There is no scheduler, no lock-state machinery, no automated rollback, no router-dispatched job orchestration. District redeploys are localised enough and infrequent enough that operator judgement plus a five-minute heads-up to players is the right level of control. New-shard deployment is rarer still — a planned downtime event, not a hot operation.

## Three-Tier Architecture

```
World base   (zone:world_base)            never rebuilt in production
├── Gateways    (district:gateways)       cross-zone transition rooms
└── System      (district:system)         Limbo, Purgatory, recycle bin

Zones        (zone:<x>)                   rebuildable
└── Districts   (district:<y>)            rebuildable, the redeploy unit

Spawn rules  (JSON in world/spawns/)      hot-reloadable, self-healing
```

### World Base

The world base is the connective scaffolding the rest of the world hangs off. It contains:

- **Gateway rooms** — every cross-zone transition room (`RoomGateway` instances), tagged `zone:world_base` + `district:gateways`. Their `destinations` lists are populated by zone builders during their build pass; the rooms themselves are not destroyed.
- **System rooms** — Limbo, Purgatory, the NFT recycle bin. Tagged `zone:world_base` + `district:system`. Already protected today via `_is_system_room()` (`world/game_world/zone_utils.py`).

The world base is built by a one-shot setup function and is not part of any zone or district redeploy path. Its rooms have stable dbrefs that other code may rely on.

### Zones

A zone is a coherent geographic region (Millholm, Aethenveil, Shadowsward, etc.) with its own identity, tagged `zone:<zone_key>`. A zone owns:

- One or more districts.
- Zone-level build orchestration (calling each district's builder in dependency order).
- Zone-scoped configuration (spawn JSON file paths, zone-wide hooks).

Zones are listed in `ACTIVE_ZONES` in `world/game_world/deploy_world.py`. Each has a `soft_deploy.py` that exposes `build_zone()` and `clean_zone()`.

### Districts

A district is the **redeploy unit** — the smallest chunk of the world we redeploy in production. Tagged `zone:<zone_key>` + `district:<district_key>`. A district has:

- A builder function (`build_<district>()`) that creates rooms, places mobs/items via spawn rules, wires intra-district exits, and attaches inbound edges from gateway rooms or neighbouring districts via the resolver.
- A declared **evacuation target** — the room players in the district are sent to during redeploy (typically the zone's gateway room, or `DEFAULT_HOME` for self-contained districts).
- A list of **dependencies** — other districts whose rooms it links to. Used by the resolver's tolerant mode (a missing dependency mid-rebuild logs and retries rather than crashing).

Today's zones already use district tags consistently; what's new is making the per-district builder the public surface (factored out of the monolithic `build_zone()`) and registering it in a manifest.

## Stable Identity

Every persistent room has a natural composite identity: **`(zone_tag, district_tag, room.key)`**. This triple is stable across rebuilds — the dbref is not. All cross-district and cross-zone references resolve through this triple, never via stored dbrefs.

### `find_room(zone, district, key)`

A thin wrapper around Evennia's `search_tag` that intersects the zone tag, district tag, and room key. Returns the live room or `None`. This is the only resolver primitive in the system; everything else composes from it.

```python
# Pseudocode — wrapper around evennia.search_tag.
def find_room(zone, district, key):
    rooms = search_tag(zone, category="zone")
    return next(
        (r for r in rooms
         if r.tags.get(district, category="district") and r.key == key),
        None,
    )
```

District builders use it to obtain neighbour rooms before passing them to the existing exit helpers in `utils/exit_helpers.py` (`connect_bidirectional_exit`, `connect_bidirectional_door_exit`, etc.). There is no separate "spec-based" exit API — the resolver composes with the existing helpers.

### Exit Wiring Convention

When a district builds an exit into a room owned by another district or by the world base, it must use the resolver:

```python
# Inside build_<district>() in some zone's soft_deploy.py
from utils.world import find_room
from utils.exit_helpers import connect_bidirectional_exit

target = find_room("aethenveil", "aethenveil_travel", "border_post")
if target is None:
    # Operator sequencing error or genuinely missing dependency.
    log.warning("build aborted: target room not found")
    raise BuildError("border_post not in aethenveil_travel — build that first")

connect_bidirectional_exit(local_room, target, "north")
```

If `find_room` returns `None`, the operator has run builds out of order or a referenced district is missing. The right response is to fail loudly so the operator sees it, fixes the sequencing, and reruns. There is no deferred-link queue.

### Why Composite, Not Extra Tags

A composite-key resolver is cheaper than introducing a per-room semantic tag (`room:border_post`) and avoids cluttering the tag namespace. The exception is **named landmarks** referenced from many zones (a god's temple, a great library) — those may carry a single semantic landmark tag in addition to their zone/district tags. This is opt-in and documented per landmark.

## Tag Conventions

Tags are already applied consistently across all 21 active zones; this section codifies the convention so future builders stay aligned.

| Category   | Value                  | Applied to                          |
|------------|------------------------|-------------------------------------|
| `zone`     | `<zone_key>`           | every room in the zone              |
| `zone`     | `world_base`           | gateways, system rooms              |
| `district` | `<district_key>`       | every room in the district          |
| `district` | `gateways`             | cross-zone transition rooms         |
| `district` | `system`               | Limbo, Purgatory, recycle bin       |
| `mob_area` | `<spawn_area_key>`     | rooms participating in a spawn pool |
| `map_cell` | `<terrain>:<point>`    | cartography survey grid             |

Every room must carry exactly one `zone` tag and one `district` tag. Other tag categories are additive and orthogonal.

## Build & Clean Operations

The pipeline mirrors Evennia's own clean-by-tag pattern:

| Operation                       | Filter                              | Status     |
|---------------------------------|-------------------------------------|------------|
| `clean_zone(zone)`              | tag `zone == zone`                  | implemented |
| `clean_district(zone, district)`| tag `zone == zone` AND `district == district` | planned   |
| `clean_world_base()`            | tag `zone == world_base`            | not exposed; behind interactive confirm |
| `build_zone()` per zone         | calls each `build_<district>()`     | partly implemented (inline today) |
| `build_district(zone, district)`| creates rooms with both tags        | planned (factor out) |
| `build_world_base()`            | one-shot setup                      | partly implemented (`_ensure_system_room` etc.) |

Clean steps mirror today's `clean_zone()`:

1. **Evacuate players** in the affected scope to the declared evac target.
2. **Delete mobs / NPCs**, returning their gold and resources to vault.
3. **Delete orphaned items** (NFTs follow recycle-bin rules).
4. **Delete exits** whose location is in the affected scope.
5. **Delete rooms** in the affected scope.

System rooms are protected by their `zone:world_base` tag and never enter the delete set.

## Redeploy Protocol

Per-district redeploy is the canonical production operation. Per-zone redeploy is the same protocol applied at zone scope; full-world rebuild is reserved for local development. The whole sequence is operator-driven — there is no orchestrator.

**Operator workflow:**

1. Operator broadcasts a warning to the affected zone: *"the inn district will be rebuilt in 5 minutes; please move to the town square."*
2. Operator waits ~5 minutes for players to clear out.
3. Operator runs `@rebuild_district <zone> <district>`.

**What `@rebuild_district` does:**

1. `clean_district(zone, district)` — evacuate any stragglers to the district's evac target, then delete by composite tag (mobs → items → exits → rooms).
2. `build_<district>()` — rebuild rooms, intra-district exits, and inbound edges from gateways or neighbouring districts via `find_room`.
3. `ZoneSpawnScript.create_for_zone(zone)` — reload the zone's spawn JSON. The next tick (≤15s) repopulates mobs.

The command runs in a background thread so the server stays responsive, but there's no orchestration layer — it's just three function calls in sequence, with exceptions propagating to the operator's terminal.

### Failure Handling

If `build_<district>()` raises, the exception is logged and the operator sees it in their terminal. The district has been deleted (step 1) but not rebuilt — players entering would find nothing there. The operator fixes the builder and reruns `@rebuild_district`. There is no automatic rollback and no lock state — the operator is the lock.

This is acceptable because: builders are version-controlled, the operator is present and watching, and the affected scope is one district. A failed rebuild is a "five more minutes, sorry" event, not an outage.

### Cross-District References During Redeploy

A district being redeployed temporarily disappears between clean and build. Other districts holding live exits into it find their `destination` references invalidated — Evennia handles this gracefully (deleted-destination exits report sensibly to anyone trying to traverse). When the rebuild completes, the destination is recreated with the same `(zone, district, key)` identity, but the dangling exits are still pointing at the now-deleted dbref. The fix is for the rebuilt district's `build_<district>()` to **re-attach the inbound side of any cross-district exits it owns**. Each district owns the rooms on its own side of every inbound edge, so this is a natural part of every district builder, not a special protocol.

`RoomGateway.destinations` lists are populated separately at world-base / zone-build time and are not touched by district redeploys — cross-zone wiring is unaffected.

## Player Evacuation

Player evacuation is scoped to the unit being rebuilt:

- **District redeploy** — players in rooms tagged `district:<y>` are moved to the district's declared evac target.
- **Zone redeploy** — players in rooms tagged `zone:<x>` are moved to Limbo (matching today's behaviour).
- **World base rebuild** — never executed in production.

The evacuation target for each district is part of its manifest entry. Sensible defaults:

- Most districts within a zone that has a gateway → that zone's outbound gateway room.
- Self-contained districts (deep dungeons, cut-off islands) → `DEFAULT_HOME`.
- A district being redeployed cannot be its own evac target.

Evacuation happens at the start of `clean_district`, before any deletes. The existing `clean_zone()` already does this for the zone scope; `clean_district` extends the same pattern. Because the operator has broadcast a warning and waited five minutes, in practice almost all players have already left voluntarily — the evacuation step is mainly a safety net for stragglers and AFK players.

## Spawn Lifecycle

`ZoneSpawnScript` already self-heals across rebuilds. Its lifecycle is:

- One persistent script per zone (`zone_spawn_<zone_key>`).
- On rebuild, the script's `db.spawn_table` is updated in place from the JSON.
- Mob deletions during clean are absorbed — the next 15-second tick repopulates from the (possibly updated) table.

District redeploy does **not** delete the zone's spawn script. It simply triggers a JSON reload via `ZoneSpawnScript.create_for_zone()` and lets the next tick refill missing population. Boss death cooldowns (the `death_cooldown_seconds` JSON field, see [SPAWN_MOBS.md](SPAWN_MOBS.md)) are preserved across rebuilds because they live on the script, not on the room.

## District Manifest

A small registry maps each district to its builder and evac target — what `@rebuild_district` needs to run a redeploy. Conceptually:

```python
DISTRICTS = {
    ("millholm", "millholm_town"): District(
        builder="world.game_world.zones.millholm.town:build_town",
        evac_target=("millholm", "millholm_town", "town_gate"),
    ),
    ...
}
```

The manifest lives alongside `ACTIVE_ZONES` in `world/game_world/deploy_world.py` (or a sibling module). A district is registered exactly once. Each district that wants to be redeployable must be registered here; districts not in the manifest can only be rebuilt as part of a full zone rebuild.

The manifest is intentionally minimal — no dependency graph, no automatic ordering. Operators sequence builds correctly; if they don't, `find_room` returns `None` and the build fails loudly so they can fix it and rerun.

## Admin Commands

| Command                                  | Scope          | Status     |
|------------------------------------------|----------------|------------|
| `@rebuild_world`                         | All zones      | implemented |
| `@rebuild_zone <zone>`                   | One zone       | implemented |
| `@rebuild_district <zone> <district>`    | One district   | planned     |
| `@rebuild_world_base`                    | World base     | planned, gated by interactive confirm — almost never used |

All commands run in background threads, return control to the caller, and report completion (or failure) via `caller.msg`.

The Evennia batch processors (`@batchcommand`, `@batchcode`) are explicitly **not** part of this pipeline. They are designed for one-shot offline initial setup, run synchronously in the reactor, have no idempotency, and require text-`exec` semantics that are incompatible with our importable, testable, async-rebuildable Python builders. We use Evennia's built-in command framework and our own builders, not its batch tooling.

## Builder Conventions

For consistency across all zones — current and future — district builders must:

1. **Tag every room** with `zone:<zone_key>` (category `zone`) and `district:<district_key>` (category `district`) at creation time. Use the per-zone `ZONE_KEY` and `DISTRICT` constants already established in `soft_deploy.py` files.
2. **Look up cross-district neighbours** via `find_room(zone, district, key)`. Never store or pass dbrefs across district boundaries.
3. **Use `utils/exit_helpers.py`** for all exit creation — never instantiate exit typeclasses directly. Pass live room objects, obtained via `find_room` for cross-boundary cases.
4. **Register the district** in the manifest. A district that exists only as inline code inside `build_zone()` cannot be redeployed individually.
5. **Declare an evac target** in the manifest. The target must not be inside the district being declared.
6. **Fail loudly on missing neighbours** during build. If `find_room` returns `None`, raise — do not silently skip. The operator needs to see the sequencing error so they can build the missing district first and rerun.
7. **Avoid global state** (module-level dicts, class attributes) that would survive across rebuilds. All persistent state belongs in the database and is reachable via tags.
8. **Be idempotent against a clean DB** — running a build function on a clean DB must produce a complete, playable district. No reliance on prior runs.

These conventions are enforced by code review for now. When a builder linter is worth writing, it will check these rules statically.

## Sharding Interaction

The world deployment model extends naturally to the multi-shard architecture in [SCALING.md](SCALING.md). At `shard_count == 1` (today's reality) every claim below collapses to a no-op.

**Districts are shard-local by construction.** A district lives entirely within one zone, and a zone lives entirely on one shard. Per-district redeploy never crosses a shard boundary — the operator runs `@rebuild_district` on the shard that owns the zone, and that's it. The frequent operation has zero multi-shard complexity.

**World base is built on every shard via the same script.** `build_world_base()` reads a `SHARD_ID` environment variable and creates only the gateway rooms relevant to that shard, plus the system rooms (Limbo, Purgatory, recycle bin) which exist independently on every shard. The shard → gateways mapping is hard-coded as branches inside the script — no registry, no config layer. At `shard_count == 1` the branching is a no-op and the script builds every gateway.

**Gateway pairing is two physical rooms, one per side.** A gateway between zone A (shard X) and zone B (shard Y) lives as two `RoomGateway` rows — one owned by X (the outbound side from A), one owned by Y (the outbound side from B). Each side's `destinations` list stores the composite triple `(target_zone, target_district, target_key)` instead of a live cross-shard object reference. When a player traverses an outbound gateway whose target zone is on a different shard, the SCALING handoff protocol fires.

**`find_room` is shard-local.** It returns rooms tagged within the current shard's zone scope. Cross-shard lookups are not a builder operation — they happen inside the handoff protocol, not inside district builders. Builders that need to wire an exit to a room on another shard are doing something wrong; that exit belongs on a `RoomGateway`.

**Deploying a new shard is a planned downtime event.** Deferred from the everyday operator workflow:

1. Bring the game offline.
2. Update the hard-coded branches in `build_world_base()` to reflect the new gateway distribution.
3. Spin up the new instance with its `SHARD_ID`.
4. Run `build_world_base()` on every shard (each picks up its responsibilities from the env var).
5. On each shard, manually run `@rebuild_zone` for each zone now assigned to it. Zone content is the operator's responsibility to deploy in the right place.
6. Bring the game online.

If the shard reshuffle happens while there is live player content (placed loot, in-flight quests, account balances), zone ObjectDB rows stay in Postgres untouched — what changes is which shard's `ACTIVE_ZONES` includes them and therefore which idmapper caches them. Step 5 is then a cache-attribution flip, not a content rebuild. This is the production reshuffle path; pre-launch shuffles can wipe and rebuild without ceremony.

New-shard deployment is anticipated to be a roughly yearly event at most, so none of this is automated. An operator with the runbook does it by hand.

## What's Implemented vs Planned

**Implemented today:**

- Zone tag + district tag on every room (consistent across all 21 zones).
- `clean_zone()` with tag-based delete and player evacuation.
- `connect_bidirectional_*` exit helper family.
- `RoomGateway` model with declarative `destinations` list, repopulated by `deploy_world()`.
- `ZoneSpawnScript` with hot-reload of JSON spawn rules and self-healing population on tick.
- `@rebuild_world` and `@rebuild_zone` commands running in background threads.
- System room protection via `_is_system_room()` in `zone_utils.py`.

**Planned (this design):**

- `find_room(zone, district, key)` resolver — thin wrapper around `search_tag`.
- `clean_district(zone, district)` — `clean_zone` with two-tag filter and district-scoped evacuation.
- District manifest in `deploy_world.py` (or sibling module).
- Per-district `build_<district>()` factored out of monolithic `build_zone()` orchestrators.
- `@rebuild_district <zone> <district>` admin command.
- Codification of the world base layer (gateways, system rooms) under `zone:world_base`.
- `SHARD_ID` env var read by `build_world_base()` so the same script runs everywhere (when sharding lands; today the script just builds everything).

The implementation is small in surface area; most of the discipline is convention rather than code. The single largest engineering task is factoring existing zone builders into per-district functions and registering them in the manifest — work that can proceed zone by zone without breaking anything.
