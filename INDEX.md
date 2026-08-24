# docs — index

The technical / knowledge wiki for the **FCM umbrella** — the single shared documentation surface for
the umbrella and every nested repo. One focused topic per file; this is the entry point. **Read this
every session and load the relevant doc before related work** (it is not auto-loaded like `MEMORY`).

## Project / infrastructure

| Document | Summary |
|---|---|
| [doco-structure.md](doco-structure.md) | The four documentation surfaces (`README` / `CLAUDE.md` / `MEMORY` / `design/`) — what belongs where, the `CLAUDE.md`⇄`MEMORY` split, and the conventions that keep them coherent. |
| [library-standards.md](library-standards.md) | Conventions for the reusable libraries under `libraries/` — naming, src layout, `pyproject.toml`, test framework, the reduced sub-repo documentation set, and the bootstrap checklist. |
| [new-machine-setup.md](new-machine-setup.md) | How to reconstitute a working FCM environment on a fresh machine: the nested-repo re-clone manifest (incl. which repos are private + the `FullCircleMUD` gh account), portable-memory first-launch, and git-crypt unlock for the game secrets. |
| [progress.md](progress.md) | Running log of validated milestones — what has actually been proven, under what conditions, and what remains unproven. Keeps status out of the design docs; not a changelog (that's `git log`) and not a roadmap (that's `ops/DEVELOPMENT/`). |

## Self-documenting sub-repos

Some sub-repos — notably the reusable Evennia **libraries** — are meant to be understood **in
isolation**: a third party reading just that repo should be able to learn how it works. By design
**these document themselves internally** (their own `docs/` wiki) rather than here. This is a
**deliberate exception** to the "one wiki surface" rule. Each follows the same conventions
as this wiki (an `INDEX.md` entry point, one kebab-case topic per file).

| Sub-repo | Internal docs | What it is |
|---|---|---|
| [evennia-world-builder](../libraries/evennia-world-builder/docs/INDEX.md) | `docs/` | Declarative YAML world authoring for Evennia. |
| [evennia-shards](../libraries/evennia-shards/docs/INDEX.md) | `docs/` | Split deployment & horizontal sharding for Evennia. |
| [evennia-mob-spawner](../libraries/evennia-mob-spawner/docs/INDEX.md) | `docs/` | Declarative YAML mob spawn system for Evennia. |
| [evennia-targeting](../libraries/evennia-targeting/docs/INDEX.md) | `docs/` | Composable targeting predicates & content filters. |
| [evennia-yaml-reader](../libraries/evennia-yaml-reader/docs/INDEX.md) | `docs/` | Source-agnostic YAML reader for the Evennia ecosystem. |

> These links point into the cloned sub-repos (gitignored by the umbrella) — they resolve when you work
> from the umbrella root with the libraries cloned. Each library's own `CLAUDE.md` also points at its
> `docs/` wiki.

## Game design

System-level design for FullCircleMUD — the *what* and *why*. **Start with [design-overview.md](design-overview.md)** — the
cross-system narrative (economic loop, mob/combat flow, NFT item flow, progression, world structure,
and the cross-cutting invariants) that no single doc holds.

### World & narrative

| Document | Summary |
|---|---|
| [world.md](world.md) | Zone structure, world regions, lore, NPC placement, narrative design. |
| [new-player-experience.md](new-player-experience.md) | Tutorial flow, starter quests, first-hour progression, Millholm onboarding. |
| [interzone-travel.md](interzone-travel.md) | Zone-to-zone travel — `explore`/`travel`/`sail`, cartography mastery gates, ship tiers, route map NFTs, food costs, party mechanics. |
| [cartography.md](cartography.md) | Cartography skill — intra-zone district mapping (`survey`/`map`, district map NFTs). |
| [procedural-dungeons.md](procedural-dungeons.md) | Dungeon template system, instance lifecycle, passage dungeons, collapse mechanics. |
| [vertical-movement.md](vertical-movement.md) | Climbing, flying, swimming, underwater mechanics, vertical exit gates. |
| [movement-messages.md](movement-messages.md) | What a room is told when someone arrives or leaves — the single announce seam, verb resolution by movement mode, direction phrasing. |

### Progression & survival

| Document | Summary |
|---|---|
| [character-progression.md](character-progression.md) | Races/classes, point-buy ability scores, levelling, skill/weapon mastery tiers, remort. |
| [knowledge-grants.md](knowledge-grants.md) | How mastery-derived spells and recipes reach a character — the learned/granted split, the reconcile engine, and its chargen/training/login triggers. |
| [survival-system.md](survival-system.md) | Survival mechanics — hunger & thirst. |

### Combat, spells & effects

| Document | Summary |
|---|---|
| [combat-system.md](combat-system.md) | Real-time combat architecture, weapon hooks, parry/riposte, skills, weapon mastery, stealth, dual-wield. |
| [weapon-damage-scaling.md](weapon-damage-scaling.md) | Material-tier × mastery damage tables, base die lookup, weapon balance and identities. |
| [effects-system.md](effects-system.md) | EffectsManagerMixin (3-layer effect system), conditions, named effects, damage resistance, damage pipeline. |
| [spell-skill-design.md](spell-skill-design.md) | Spell system architecture, per-school tables, implementation patterns, reactive spells. |
| [alignment-system.md](alignment-system.md) | Alignment score, alignment-gated items, shifts on player action. |
| [unified-search-system.md](unified-search-system.md) | Unified search and targeting system — how search/targeting resolves across objects. |

### NPCs, mobs & pets

| Document | Summary |
|---|---|
| [llm-vision.md](llm-vision.md) | Narrative entry point for FCM's LLM layer — how the three memory systems compose to make the world come alive. Read this first, then drill into the specs. |
| [npc-mob-architecture.md](npc-mob-architecture.md) | BaseNPC / CombatMob composition hierarchy, mixin system, mob AI tiers, hybrid LLM combat mobs. |
| [npc-quest-system.md](npc-quest-system.md) | NPC hierarchy, training, shopkeeping, quest engine, implemented quests. |
| [lore-memory.md](lore-memory.md) | Embedded world knowledge for NPCs — scope-tagged lore entries, semantic retrieval, multi-tag AND semantics. |
| [combat-ai-memory.md](combat-ai-memory.md) | Combat memory and strategy bot for Tier 4 bosses — pre-encounter briefing, post-combat logging. |
| [language-system.md](language-system.md) | Languages, garble engine, NPC/mob/animal language integration, Speak With Animals. |
| [pets-and-mounts.md](pets-and-mounts.md) | Persistent pet NFTs, familiars, mounts, taming, stabling. |

### Items & rooms

| Document | Summary |
|---|---|
| [inventory-equipment.md](inventory-equipment.md) | Item system, equipment/wearslots, carrying capacity, NFT ownership, wear effects. |
| [crafting-system.md](crafting-system.md) | Crafting/processing architecture, recipe format, enchanting, gem insetting, room types. |
| [room-architecture.md](room-architecture.md) | Room typeclass hierarchy, terrain types, specialised room types, mixins. |
| [exit-architecture.md](exit-architecture.md) | Exit typeclass hierarchy, doors, locks, traps, vertical-aware exits. |
| [world-objects.md](world-objects.md) | WorldFixture / WorldItem base classes, interaction mixins (climbable, switch, lit, trap), builder patterns. |

### Economy & spawning

| Document | Summary |
|---|---|
| [economy.md](economy.md) | Gold sinks, AMM pricing, market tiers, resource economics, revenue model. |
| [treasury.md](treasury.md) | Three-wallet architecture (issuer/vault/operating), subscription processing, issuance discipline. |
| [unified-item-spawn-system.md](unified-item-spawn-system.md) | Calculator + Distributor architecture for resources, gold, knowledge NFTs, and rare items. |
| [spawn-mobs.md](spawn-mobs.md) | Mob spawning — JSON rules, population, respawn semantics, area tags, post-spawn hooks. |
| [telemetry.md](telemetry.md) | Economy telemetry snapshots, player session tracking, saturation snapshots. |

### Compliance, billing & chain boundary

| Document | Summary |
|---|---|
| [compliance.md](compliance.md) | Legal/regulatory framework (no-redemption model), token classification, game economy management, language policy. |
| [subscriptions.md](subscriptions.md) | Subscription billing — payment flow, lifecycle, character-entry gating, trial period. |
| [import-export.md](import-export.md) | Chain-boundary `import`/`export` — gate stack, asymmetric trial gating, Xaman flow, mirror state transitions, planned OFAC SDN screening. |

### Infrastructure

| Document | Summary |
|---|---|
| [database.md](database.md) | Django app layout, five-database architecture, transactions across aliases, model overview, migrations. |
| [website.md](website.md) | Web frontend pages, geo-detection infrastructure, page inventory, implementation status. |
| [connection-transport.md](connection-transport.md) | Why FCM is WebSocket-only — Cloudflare, `cf-ipcountry`, telnet/SSH limitations, decision rationale. |
| [scaling.md](scaling.md) | Scaling strategy for the initial era — capacity and growth approach. |
| [world-deployment.md](world-deployment.md) | World deployment — building and deploying world content into the running game. |

> Deployment, infrastructure, recovery runbooks, and Railway/CI config live in the private `ops/` repo,
> not in these design docs.
