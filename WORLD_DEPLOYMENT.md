# World Deployment

> Technical design document for how FullCircleMUD world content is authored, validated, and applied to the running game. The pipeline is owned by the [`evennia-world-builder`](https://github.com/FullCircleMUD/evennia-world-builder) library; world content lives in the [`fcm-world`](https://github.com/FullCircleMUD/fcm-world) YAML repository. This document is FCM-specific — it covers what the library means *for FCM*, FCM's authoring conventions, the operator workflow, and how the YAML pipeline composes with FCM's runtime systems (spawn scripts, evacuation, sharding). For library internals (pipeline stages, validator predicates, cleanup model) follow the links into the library's own DESIGN docs. For room and exit internals see [ROOM_ARCHITECTURE.md](ROOM_ARCHITECTURE.md) and [EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md). For zone-to-zone travel see [INTERZONE_TRAVEL.md](INTERZONE_TRAVEL.md). For multi-shard sharding see [SCALING.md](SCALING.md). This document is about deploying **content into the running game**, not about deploying the game process to infrastructure.

## Design Philosophy

World content is data, and data changes more often than the systems that interpret it. As game systems stabilise, the dominant ongoing work shifts to authoring new zones, balancing existing districts, and patching bugs in specific rooms. The pipeline follows that shift:

- **World content is authored as YAML**, not as imperative Python builders. Rooms, exits, fixtures, NPCs, contents, locks, attributes — all declared as data.
- **A library applies the data** to Evennia's database. The library is `evennia-world-builder`, a separate repo with its own version, its own tests, and a stable contract with the consumer game.
- **A file is the atomic redeploy unit.** Authors choose granularity by how they chunk YAML — one room per file means one-room redeploys; thirty rooms in one file means thirty-room redeploys. The library does no finer-grained partial clean within a file.
- **Operator-driven, no orchestrator.** Operators broadcast a warning, wait for players to clear, then manually run the rebuild command. No scheduler, no lock state, no automated rollback.

The pre-YAML era used Python `build_<zone>()` / `build_<district>()` functions, a planned `find_room` resolver, a district manifest, and an `@rebuild_district` command. That whole approach is superseded — `deploy_world.py` and every per-district Python builder have been retired. Git history preserves the prior thinking if a reader ever needs it.

## Three Repositories

```
src/game                      consumer game (this repo)
  typeclasses/                rooms, exits, NPCs, items — game systems, not content
  utils/exit_helpers.py       runtime exit helpers (procedural dungeons,
                              conditional rebinds); not the authoring surface
  commands/, services/, …     everything else FCM-specific

libraries/evennia-world-builder    pipeline library (separate git repo)
  Reader → Definitions → Finder → Loader → Validator → Builder
  Ships wb_build (in-game) + wb-validate (CLI)

fcm-world                     world content (separate git repo)
  definitions.yaml            level scheme: (shard, zone, district, file)
  index.yaml                  top-level manifest pointer
  shard0/scaffold/            system rooms (Limbo, Purgatory, recycle bin)
  shard0/<zone>/<district>/   the world, as data
```

Mob populations are managed by a parallel pipeline — the [`evennia-mob-spawner`](https://github.com/FullCircleMUD/evennia-mob-spawner) library against the [`fcm-mobs`](https://github.com/FullCircleMUD/fcm-mobs) content repo, with its own operator commands (`ms_load` / `ms_status` / `ms_restart` / `ms_stop` / `ms_delete`). The two pipelines are deliberately separate: static room/exit/fixture/NPC content lives in `fcm-world` and is applied via `wb_build`; respawning mob populations live in `fcm-mobs` and are applied via `ms_load`. They meet only at the `mob_area` tag, which world-builder authors place on rooms and mob-spawner uses to find where to spawn.

The library installs into the game's venv as a normal Python package; the YAML repo is fetched at build time by the library's Reader (GitHub by default, local filesystem for development). FCM points the Reader at `fcm-world` via `WORLDBUILDER_READER_KWARGS` in [server/conf/settings.py](../src/game/server/conf/settings.py).

## Deployment Identity

Every Evennia object the library creates is tagged with two values: **`wb_deployment_file`** (the YAML file path, set by the Builder) and **`wb_deployment_id`** (an author-supplied integer, mandatory, unique within its file). The pair is globally unique across the world.

This identity is load-bearing:

- Cleanup on rebuild is a one-tag sweep: delete every object tagged `wb_deployment_file=<file>`, then recreate from the current YAML. Entities added, removed, or changed are all handled by the same primitive.
- Cross-references between entities use `{deployment_file, deployment_id}` dicts in YAML — `location:`, `destination:`, `home:`. The Builder resolves these against (a) the in-build map for entities being built right now, and (b) DB tag-search for entities in already-built files.
- Folder reorganisation changes `deployment_file` for affected entities and will require migration tooling. **Treat the fcm-world folder layout as a stable structural choice.**

For the full identity contract see [DESIGN/deployment-identity.md](../../libraries/evennia-world-builder/DESIGN/deployment-identity.md) in the library.

### Tags Authored on Rooms

Every room YAML must carry exactly one `zone` tag and one `district` tag. Other tag categories are additive.

| Category   | Value              | Applied to                              |
|------------|--------------------|------------------------------------------|
| `zone`     | `<zone_key>`       | every room in the zone                  |
| `zone`     | `system_zone`      | system rooms (Limbo, Purgatory, recycle bin) |
| `district` | `<district_key>`   | every room in the district              |
| `district` | `system_district`  | system rooms                            |
| `mob_area` | `<spawn_area_key>` | rooms participating in an evennia-mob-spawner population pool |
| `map_cell` | `<terrain>:<point>`| cartography survey grid                 |

These tags are author-controlled in YAML; the library is content-agnostic about what tags mean. FCM systems read them: `wb_build` cleans by `wb_deployment_file` (per-file scoped), evennia-mob-spawner finds spawn rooms by `mob_area`, and cartography composes a district map from `map_cell`. (The legacy zone-tag-scoped `clean_zone()` helper has been retired; `wb_build`'s file-scoped cleanup is the production primitive.)

Gateway rooms today are tagged with their owning zone's `zone` / `district` keys (e.g. `zone:port_shadowmere` + `district:port_shadowmere_travel`) — not a unified `world_base` meta-zone. System rooms today are tagged `zone:system_zone` + `district:system_district`. A future unification under `zone:world_base` is possible; not on the near-term roadmap.

## Operator Workflow

Two commands cover the whole pipeline:

| Command                 | Surface          | What it runs                                |
|-------------------------|------------------|---------------------------------------------|
| `wb_build <scope>`      | In-game (OOC/IC) | Full pipeline incl. Builder; superuser-only |
| `wb-validate --root <repo>` | Shell / CI    | Everything *up to* the Builder              |

### `wb_build` scopes

The library's `definitions.yaml` declares FCM's level scheme as `(shard, zone, district, file)`. `wb_build` accepts scope queries against those levels:

- `wb_build all` — every YAML file in the manifest. Required keyword; bare `wb_build` refuses (safety guard).
- `wb_build zone=millholm` — every file under `shard0/millholm/`.
- `wb_build zone=millholm district=town` — every file under `shard0/millholm/town/`.
- `wb_build zone=millholm district=town file=bakery.yaml` — that one file.
- Append `--force-validate` to force whole-repo pre-validation regardless of the `repo-ci-pre-validation` flag.

The smallest sensible redeploy is a single file. Author files at the granularity you want to redeploy — a single bakery, a single forest path, a 30-room dungeon — whatever maps best to how that content is iterated on.

### Redeploy protocol

1. Operator broadcasts a warning to affected players: *"The bakery is being rebuilt in 5 minutes; finish what you're doing."*
2. Operator waits ~5 minutes.
3. Operator runs `wb_build <scope>`.

`wb_build` defers the entire pipeline to a Twisted worker thread via `evennia.utils.utils.run_async`. **Gameplay continues while the build runs** — players outside the scope are unaffected. Operator-facing output (pre-validation summary, validator findings, cleanup count, created objects) is collected during the worker pass and flushed to the caller via the success callback when the build completes.

### What `wb_build` does, step by step

For the canonical version see [DESIGN/library-commands.md](../../libraries/evennia-world-builder/DESIGN/library-commands.md) in the library. Summarised:

1. **Read** YAML via the configured `Reader` (GitHub by default; local filesystem in development).
2. **Walk** the `index.yaml` manifest tree to find entities matching the scope query.
3. **Pre-validate** (whole repo, unless gated off by `repo-ci-pre-validation` in `definitions.yaml`).
4. **Validate** the in-scope entities through the full predicate pipeline. Any finding refuses the build with a complete report; no DB mutation.
5. **Clean** every object tagged with the in-scope `wb_deployment_file` values. (Players in a deleted room are relocated to their `home` by Evennia's own pipeline — see [Evacuation](#evacuation).)
6. **Build** in four passes:
   - **Pass 1** — non-exits (rooms, fixtures, NPCs, items).
   - **Pass 2** — exits. Destinations resolve against the just-built rooms or via DB tag-search for cross-file targets.
   - **Pass 3** — `incoming_exits:` file-level registrations. If a cross-file exit's home file is *not* in scope and the exit was cascade-deleted by step 5, the library fetches the canonical file and rebuilds just the missing exit, tagged with its true home file so future cleanups handle it correctly.
   - **Pass 4** — `links:` file-level declarations. Resolves cross-entity attribute references (e.g. paired door `other_side`).

### Cross-file rebuild safety

When a single file is redeployed, cross-file exits pointing at rooms in *other* files survive — they were never in scope, never cleaned. Cross-file exits pointing *into* a redeployed file are the load-bearing case: the destination room is deleted and recreated, and the inbound exits live in a different file. FCM authors register these via `incoming_exits:` at the bottom of the destination file, and pass 3 restores any exit the cleanup cascade took out. The pattern is documented per file in fcm-world.

## Authoring Workflow

The YAML side of FCM. For an author landing here, the loop is:

1. **Edit YAML** in `fcm-world/shard0/<zone>/<district>/<file>.yaml`.
2. **Validate locally**: from `FCM/src/game/` with the venv active:
   ```
   wb-validate --reader local --root ../../fcm-world
   ```
   This runs Reader → Definitions → Finder → Loader → Validator and prints any findings. No DB mutation. Run before committing.
3. **Commit + push** in fcm-world.
4. **Deploy in-game**: `wb_build zone=<zone> district=<district> file=<file>.yaml` against the running server (which fetches the new content via the configured Reader).

`wb-validate` is also wired as a pre-commit hook / CI gate for fcm-world. A repo with `repo-ci-pre-validation: true` in `definitions.yaml` trusts that gate and skips whole-repo pre-validation inside `wb_build`. FCM currently keeps the flag off — every `wb_build` re-validates the whole repo. Cheap insurance, kept until the CI gate is more battle-tested.

For YAML shape, file structure, and per-entity dimensions (typeclass, contents, exits, attributes, locks, aliases, descriptions), see the library's design docs:

- [discovery-and-loading.md](../../libraries/evennia-world-builder/DESIGN/discovery-and-loading.md) — manifests, `entities:` shape, nested `contents:` / `exits:`.
- [deployment-identity.md](../../libraries/evennia-world-builder/DESIGN/deployment-identity.md) — `deployment_id`, `home:`, cross-refs.
- [builder.md](../../libraries/evennia-world-builder/DESIGN/builder.md) — per-entity build pass; what's applied to each object.
- [validator.md](../../libraries/evennia-world-builder/DESIGN/validator.md) — predicate tiers and what gets caught.
- [links.md](../../libraries/evennia-world-builder/DESIGN/links.md) — cross-entity attribute references.

## System Rooms

Limbo, Purgatory, and the NFT recycle bin live as one YAML file each under `fcm-world/shard0/scaffold/`:

- `shard0/scaffold/purgatory.yaml`
- `shard0/scaffold/nft_recycle_bin.yaml`
- (Limbo follows the same pattern when its YAML file is added — see the scaffold folder for the current state.)

Each file owns exactly one room, so rebuilding one system room is a one-file `wb_build`. They are tagged `zone:system_zone` + `district:system_district` and protected from accidental destruction by `wb_build`'s file-scoped cleanup model — system rooms only enter a delete set if `shard0/scaffold/` is explicitly named in the build scope. (The previous zone-tag `clean_zone()` sweep with an `_is_system_room()` name-match guard has been retired alongside the rest of the legacy zone-build infrastructure.)

`RoomRecycleBin` extends `evennia.DefaultRoom` directly (not `RoomBase`) so the catch-all bin doesn't drag in combat / lighting / weather / inventory mechanics. See [ROOM_ARCHITECTURE.md](ROOM_ARCHITECTURE.md) for the room class hierarchy.

## Evacuation

Players and objects in a room being deleted by the cleanup pass are handled by Evennia's own object lifecycle, not by world-builder primitives:

- **Characters** in a deleted room → moved to their `home` attribute (typically Limbo for default characters).
- **NPCs** with `home: { deployment_file: …, deployment_id: … }` in YAML → moved to that home room.
- **NFT items** with `home: shard0/scaffold/nft_recycle_bin.yaml#1` → moved to the recycle bin, which deletes them on arrival (handled by `RoomRecycleBin.at_object_receive`).
- **Untagged objects** without an explicit `home:` → fall through to `settings.DEFAULT_HOME` (Limbo).

The operator's pre-build broadcast is what keeps players from being yanked mid-action; the home-relocation is the safety net for stragglers, AFK characters, and unattended NPCs. Because rebuilds are scoped (single file or single district most of the time), the relocation footprint is small.

## Spawn Lifecycle

Mob spawning lives in its own pipeline: the [`evennia-mob-spawner`](https://github.com/FullCircleMUD/evennia-mob-spawner) library reading the [`fcm-mobs`](https://github.com/FullCircleMUD/fcm-mobs) content repo. From the world-builder's point of view it is opaque — the only contact surface is the `mob_area` tag that world-builder authors place on rooms, which the spawner uses as the population's room set.

Operator surface mirrors `wb_build` shape:

| Command         | What it does                                                       |
|-----------------|--------------------------------------------------------------------|
| `ms_load`       | Load (or reload) spawn definitions for a scope from `fcm-mobs`     |
| `ms_status`     | Inspect live spawn population vs. target                           |
| `ms_restart`    | Tear down and re-create a spawner                                  |
| `ms_stop`       | Stop a spawner without deleting its mobs                           |
| `ms_delete`     | Stop and delete a spawner (its mobs go through normal lifecycle)   |

A `wb_build` does not touch any mob spawner; an `ms_load` does not touch any room. The two operate on disjoint object sets. Mobs deleted as collateral during a `wb_build` cleanup (e.g. a tagged NPC inside a redeployed room) are repopulated by the spawner's own self-healing loop on the next tick. Boss death cooldowns live on the spawner side and survive `wb_build` redeploys.

The two pipelines are deliberately separated:

- **Static content** (rooms, exits, fixtures, persistent NPCs that don't respawn) → `fcm-world`, deployed via `wb_build`.
- **Respawning mob populations** → `fcm-mobs`, deployed via `ms_load`, finding rooms by `mob_area` tags that the world-builder side placed.

The legacy `ZoneSpawnScript` + `world/spawns/*.json` pipeline that previously did this job has been retired; its class is dormant in the source tree pending future deletion.

## What's Left in Python

World content is no longer built from Python. The previous orchestrator `deploy_world.py` was deleted; the per-zone `soft_deploy.py` shells that used to handle `clean_zone()` and spawn-script creation were commented out in place once `wb_build` plus evennia-mob-spawner covered every previous use case. Those files (`world/game_world/zone_utils.py`, `world/game_world/zones/millholm/soft_deploy.py`, `world/game_world/zones/book_zones/hundred_acre_wood.py`) sit in the source tree behind dated `DEPRECATION NOTICE` docstrings, marked for future deletion pending live verification. `ZoneSpawnScript` is dormant in the same way.

What's still Python and load-bearing:

- **Typeclasses** in `src/game/typeclasses/` — rooms, exits, NPCs, items, services. Game systems, not content; not touched by YAML deploys.
- **Internal exit-creation helpers** in `src/game/utils/exit_helpers.py` (`connect_bidirectional_exit`, `connect_bidirectional_door_exit`, etc.). The library's Loader and FCM's exit typeclasses call these internally, and runtime exit creators (procedural dungeons, conditional rebinds) still go through them — but they are not the authoring surface. Authors author `exits:` blocks in YAML.
- **Per-player / runtime room builders** in `src/game/world/tutorial/` and `src/game/world/test_world/`. Tutorials are per-player ephemeral instances, not authored static content; test_world is dev/QA staging. Neither fits the `wb_build` model.

No human edits zone-build Python anymore. New zones are added by creating a folder under `fcm-world/shard0/<zone>/` and writing the YAML.

## Sharding Interaction

The world deployment model extends naturally to the multi-shard architecture in [SCALING.md](SCALING.md). At `shard_count == 1` (today) every claim below collapses to a no-op.

- **Each shard runs its own `wb_build`** against its own slice of the YAML repo. Today the slice is the whole repo; multi-shard splits it by `shard` level — the top-level `definitions.yaml` already declares `shard` as the outermost manifest level.
- **System rooms are per-shard.** Each shard builds its own Limbo / Purgatory / recycle bin from `shard0/scaffold/*.yaml`. Different shards may reach different revisions of the scaffold YAML; that's fine, the rooms are independent.
- **Gateway rooms are paired across shards.** A gateway between zone A (shard X) and zone B (shard Y) lives as two `RoomGateway` rows — one owned by each side, each authored in the YAML of the shard that owns it. Each side's `destinations:` list stores composite `(target_zone, target_district, target_key)` triples; cross-shard traversal hands off via the SCALING protocol.
- **Shard reshuffles are planned downtime events.** Bring the game offline, update the `shard` level assignment, rerun `wb_build` per shard, bring the game online.

## What's Implemented vs Planned

**Implemented today:**

- Full world content authored as YAML in `fcm-world`. Every static zone, including book zones, is on the YAML pipeline.
- `evennia-world-builder` library — Reader (GitHub + local), Definitions, Finder, Loader, Validator, Builder, four-pass build, cross-file refs, `incoming_exits:`, `links:`, surgical rebuilds, async pipeline.
- `wb_build` in-game admin command and `wb-validate` CLI.
- `RoomGateway` model with declarative `destinations:` lists, authored per zone in YAML.
- System room protection via `wb_build`'s file-scoped cleanup (system rooms in `shard0/scaffold/` only enter the delete set if explicitly scoped).
- Parallel mob-spawn pipeline via [`evennia-mob-spawner`](https://github.com/FullCircleMUD/evennia-mob-spawner) reading [`fcm-mobs`](https://github.com/FullCircleMUD/fcm-mobs); operator-driven via `ms_load`.
- Legacy infrastructure (`deploy_world.py`, per-zone `soft_deploy.py`, `zone_utils.clean_zone()`, `ZoneSpawnScript`, `world/spawns/*.json`) retired or commented out behind dated DEPRECATION NOTICE blocks pending future deletion.

**Possible future work (not on the immediate roadmap):**

- Migration tooling for fcm-world folder reorganisation (today, moving a file invalidates its `deployment_file` tag and orphans previously-built objects).
- Codification of a `world_base` meta-zone unifying gateway rooms and system rooms under one tag namespace.
- A second Reader implementation if the GitHub Reader's rate-limit or auth model proves limiting in production.
- Cross-repo references (referring from one YAML repo to another). Out of scope for v0; single content-repo per build.

The pipeline is in production use today. Most ongoing work happens in `fcm-world` (content) rather than in the library (infrastructure).
