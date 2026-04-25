# FCM Design Documents

System-level design for FullCircleMUD. These documents describe the *what* and *why* — the economic models, spawn algorithms, combat architecture, and world structure. For implementation details and code patterns, see `src/game/CLAUDE.md`.

---

## Document Index

### World & Narrative

| Document | What It Covers |
|---|---|
| **WORLD.md** | Zone structure, world regions, lore, NPC placement, narrative design |
| **NEW_PLAYER_EXPERIENCE.md** | Tutorial flow, starter quests, first-hour progression, Millholm onboarding |
| **INTERZONE_TRAVEL.md** | Zone-to-zone travel — `explore`/`travel`/`sail` commands, cartography mastery gates, ship tiers, route map NFTs, food costs, party mechanics |
| **CARTOGRAPHY.md** | Cartography skill — intra-zone district mapping (`survey`/`map` commands, district map NFTs). For inter-zone route discovery see `INTERZONE_TRAVEL.md` |
| **PROCEDURAL_DUNGEONS.md** | Dungeon template system, instance lifecycle, passage dungeons, collapse mechanics |
| **VERTICAL_MOVEMENT.md** | Climbing, flying, swimming, underwater mechanics, vertical exit gates |

### Combat, Spells, Effects

| Document | What It Covers |
|---|---|
| **COMBAT_SYSTEM.md** | Real-time combat architecture, weapon hooks, parry/riposte, skills, weapon mastery, stealth, dual-wield |
| **WEAPON_DAMAGE_SCALING.md** | Material-tier × mastery damage tables, base die lookup, weapon balance and identities |
| **EFFECTS_SYSTEM.md** | EffectsManagerMixin (3-layer effect system), conditions, named effects, damage resistance, damage pipeline |
| **SPELL_SKILL_DESIGN.md** | Spell system architecture, per-school tables, spell implementation patterns, reactive spells |
| **ALIGNMENT_SYSTEM.md** | Alignment score, alignment-gated items, shifts on player action |

### NPCs, Mobs, Pets

| Document | What It Covers |
|---|---|
| **LLM_VISION.md** | Narrative entry point for FCM's LLM layer — how the three memory systems compose to make the world come alive and drive emergent gameplay. Read this first, then drill into the specs below |
| **NPC_MOB_ARCHITECTURE.md** | BaseNPC / CombatMob composition hierarchy, mixin system, mob AI tiers, hybrid LLM combat mobs |
| **NPC_QUEST_SYSTEM.md** | NPC hierarchy, training, shopkeeping, quest engine, implemented quests |
| **LORE_MEMORY.md** | Embedded world knowledge for NPCs — scope-tagged lore entries, semantic retrieval, multi-tag AND semantics |
| **COMBAT_AI_MEMORY.md** | Combat memory and strategy bot for Tier 4 bosses — pre-encounter briefing, post-combat logging |
| **LANGUAGE_SYSTEM.md** | Languages, garble engine, NPC/mob/animal language integration, Speak With Animals |
| **PETS_AND_MOUNTS.md** | Persistent pet NFTs, familiars, mounts, taming, stabling |

### Items & Rooms

| Document | What It Covers |
|---|---|
| **INVENTORY_EQUIPMENT.md** | Item system, equipment/wearslots, carrying capacity, NFT ownership, wear effects |
| **CRAFTING_SYSTEM.md** | Crafting/processing system architecture, recipe format, enchanting, gem insetting, room types |
| **ROOM_ARCHITECTURE.md** | Room typeclass hierarchy, terrain types, specialised room types, mixins |
| **EXIT_ARCHITECTURE.md** | Exit typeclass hierarchy, doors, locks, traps, vertical-aware exits |
| **WORLD_OBJECTS.md** | WorldFixture / WorldItem base classes, interaction mixins (climbable, switch, lit, trap), builder patterns |

### Economy & Spawning

| Document | What It Covers |
|---|---|
| **ECONOMY.md** | Gold sinks, AMM pricing, market tiers, resource economics, revenue model |
| **TREASURY.md** | Three-wallet architecture (issuer/vault/operating), subscription processing, issuance discipline |
| **UNIFIED_ITEM_SPAWN_SYSTEM.md** | Calculator + Distributor architecture for resources, gold, knowledge NFTs, and rare items |
| **SPAWN_MOBS.md** | Mob spawning — JSON rules, population, respawn semantics (`respawn_seconds` vs `death_cooldown_seconds`), area tags, post-spawn hooks |
| **TELEMETRY.md** | Economy telemetry snapshots, player session tracking, saturation snapshots |

### Compliance, Billing, Chain Boundary

| Document | What It Covers |
|---|---|
| **COMPLIANCE.md** | Legal/regulatory framework (no-redemption model), token classification, game economy management, language policy |
| **SUBSCRIPTIONS.md** | Subscription billing — payment flow, lifecycle, character-entry gating (`ic`/`charcreate`/`chardelete`), trial period |
| **IMPORT_EXPORT.md** | Chain-boundary `import`/`export` commands — gate stack, asymmetric trial gating, Xaman flow, mirror state transitions, planned OFAC SDN screening |

### Infrastructure

| Document | What It Covers |
|---|---|
| **DATABASE.md** | Django app layout, four-database architecture, model overview, migrations |
| **WEBSITE.md** | Web frontend pages, geo-detection infrastructure, page inventory, implementation status |
| **CONNECTION_TRANSPORT.md** | Why FCM is WebSocket-only — Cloudflare, `cf-ipcountry`, telnet/SSH limitations, decision rationale |

> Deployment, infrastructure, recovery runbooks, and Railway/CI configuration live in the private `ops/` repository, not in `design/`.

---

## System Interaction Map

### The Economic Loop

Everything in FCM ultimately connects to the economy. Here is how the major systems relate:

```
XRPL AMM Pools
    │  (price signals)
    ▼
Resource Spawn Algorithm ──────── feeds ──────────► Harvest Rooms
    │  (floor/ceiling bands                              │
    │   vs live AMM price)                               ▼
    │                                               Players harvest
    │                                                    │
    ▼                                                    ▼
Economy Telemetry ◄──── consumption data ──── Processing Rooms
    │  (hourly snapshots)                        (wheat→flour→bread)
    │                                                    │
    ▼                                            Crafting Rooms
AMM Buy/Sell Prices                            (ore+ingot→iron sword)
    │                                                    │
    └──► Shopkeeper NPCs ◄────────────────── Players sell/buy items
              │
              └──► Gold Circulation ──► Gold Sink (fees, repair, crafting)
                                                    │
                                                    ▼
                                           ReallocationScript
                                         (daily SINK → RESERVE)
```

**Key feedback loops:**
- Price rises → spawn rate increases → supply recovers → price normalises
- Hoarding drives AMM price up → the price modifier responds → spawn rate rises until new supply flows to non-hoarders
- Items junked or destroyed → leave circulation entirely → saturation drops → new copies more likely to drop
- Players withdrawing to their private wallet does NOT reduce saturation — the item still exists and is still player-owned

---

### Mob and Combat Systems

```
Zone Spawn Rules (JSON)
    │
    ▼
ZoneSpawnScript (15s tick)
    │
    ├── spawns mobs into rooms      ──► Players encounter → Combat
    │   per their JSON rule                    │
    │   (target, area_tag, respawn,            ▼
    │    optional post_spawn_hook)        CombatHandler (per-combatant)
    │                                          │
    │                                     execute_attack() — full weapon
    │                                     hook pipeline (mastery, parry,
    │                                     riposte, named effects)
    │                                          │
    │                                          ▼
    │                                     Mob dies → Corpse (loot)
    │                                          │
    │                                          ├──► Quest events fire
    │                                          │
    │                                          ├──► XP awarded
    │                                          │
    │                                          └──► die() deletes mob and
    │                                               (if rule sets
    │                                                death_cooldown_seconds)
    │                                               notifies the script
    │                                               so the cooldown clock
    │                                               restarts at kill time
    │
    └── next tick: rule below target + cooldown elapsed → spawn replacement
```

Service NPCs (bartenders, shopkeepers, etc.) bypass this loop — they are placed once by world builders and default to `is_immortal=True`, so they never reach `die()` and the spawn system never has to replace them. See SPAWN_MOBS.md § What This System Does Not Handle.

---

### NFT Item Flow

```
NFT items enter the game via two paths:

PATH A — Common Equipment (Tracker Token AMM — NOT YET BUILT)
    Crafting Room → player crafts sword → NFT spawns → sold to Shopkeeper
    Shopkeeper → Tracker Token AMM (price discovery) → sold to next player

PATH B — Knowledge and Rare NFTs (Pre-placed by unified spawn system)
    Each hour, UnifiedSpawnScript asks per-item calculators for a
    budget, then ScrollDistributor / RecipeDistributor / RareNFTDistributor
    pre-place the NFTs onto tagged targets (mobs, containers, rooms)
    via drip-feed ticks. Players encounter the mob, kill it, and loot
    the corpse normally — no loot-roll-on-death step.

    Player loots → CHARACTER location → player may withdraw to
    private wallet (ONCHAIN) → item remains in circulation.

KNOWLEDGE ITEMS (Scrolls and Recipes)
    Pre-placed via ScrollDistributor / RecipeDistributor exactly as
    above. The calculator uses a gap-based budget:
        budget = max(0, eligible_players - known_by - unlearned_copies)
    Scrolls consumed (transcribed) permanently reduce future demand.
    See UNIFIED_ITEM_SPAWN_SYSTEM.md § KnowledgeCalculator.

HOURLY SATURATION SNAPSHOT (NFTSaturationScript)
    ─ runs hourly in the pipeline (60s after telemetry, 60s before spawn)
    ─ queries spellbooks, recipe books, NFTGameState, SPELL_REGISTRY, RECIPES
    ─ writes SaturationSnapshot rows (one per tracked item per day,
      rewritten each hour with the latest state)
    ─ KnowledgeCalculator reads these snapshots when the next spawn
      cycle fires
```

---

### Resource Spawn in Detail

The two-factor algorithm self-corrects for inflation and scarcity:

```
FACTOR 1: Consumption baseline (24h rolling average of actual
          consumption — crafting, processing, eating, repair)
              ↓
         × FACTOR 2: Price modifier (AMM buy price vs target band)
              │  price too high → modifier > 1.0 (spawn more)
              │  price in band  → modifier = 1.0 (stable)
              │  price too low  → modifier < 1.0 (spawn less)
              ↓
         = SPAWN_AMOUNT (units this hour)
              │
              ▼
         Distributed across tagged targets (rooms, mobs, containers)
         proportionally by headroom, with alternating sort direction
              │
              ▼
         Drip-fed across the hour (max 12 ticks, min 5 min apart)
         so early-login players don't harvest everything before others arrive
```

**Note:** An earlier design included a third factor (circulating supply per player-hour) intended as an anti-hoarding defence. It was removed — consumption already captures demand and AMM price already captures market conditions, so a third factor with a manually-configured target was redundant guesswork. See UNIFIED_ITEM_SPAWN_SYSTEM.md § Why Not a Supply Modifier for the full rationale.

---

### Combat Flow in the Bigger Picture

```
BEFORE COMBAT:
    Equipment effects (wear_effects) → _recalculate_stats() → cached stats
    Spell buffs (named effects) → conditions + stats (also via recalculate)

DURING COMBAT:
    CombatHandler tick (fixed COMBAT_TICK_INTERVAL, default 4.0s)
        → weapon speed affects INITIATIVE, not tick rate (speed + DEX
          modifier determines turn order within the tick)
        → execute_attack() runs the full weapon hook pipeline
        → weapon mastery effects (at_hit, at_crit, at_kill, etc.)
        → named effects applied/ticked (stun, slow, prone, etc.)
        → reactive spells (Shield on hit, Smite on kill)
        → durability loss (weapons + armour)

AFTER COMBAT (combat ends):
    clear_combat_effects() → all combat_round named effects removed
    position reset to "standing"
    stances (offence/defence) cleared

POST-KILL:
    at_kill() on weapon → mob special abilities (Rampage, Cleave)
    XP awarded to the killer (formula in BaseActor.die / _award_xp)
    Corpse dropped with mob's loot
    Mob deleted; ZoneSpawnScript repopulates after the rule's
        death_cooldown_seconds (or respawn_seconds) elapses
```

---

### Player Progression and the Knowledge Economy

```
Character Creation
    Race + class selection → racial ability bonuses, class cmdset
    Point buy (27 points) → ability scores
    Weapon/skill selection → initial mastery (UNSKILLED)

Levelling Up (XP → levels_to_spend)
    Guildmaster.advance → class level increases
    Skill training (TrainerNPC) → mastery tiers BASIC → GM
    Weapon training (TrainerNPC) → weapon mastery tiers

Knowledge (one-way acquisition)
    Spell scrolls (mage) → transcribe → spellbook permanent
    Recipe scrolls → learn → recipe book permanent
    Enchanting → auto-granted at mastery tier (no scrolls)
    Cleric spells → auto-granted at divine skill tier

Guild Quests (one-time unlock)
    Warrior Initiation → rat cellar clearance
    Thief Initiation → navigate Thieves' Gauntlet, retrieve guild token
    Mage Initiation → deliver 1 Ruby
    Cleric Initiation → feed bread to beggar

Remort (level 40 reset)
    Resets to level 1, keeps accumulated advantages
    Perks: extra point buy, bonus HP/Mana/Move
    Unlocks min_remort-gated content (races, classes, items)
```

---

### World Structure

```
Millholm (hub town)
    ├── millholm_town    ← shops, guilds, bank, inn, post office, cemetery
    ├── millholm_farms   ← wheat fields, cotton farm, windmill
    ├── millholm_woods   ← sawmill, smelter, deep woods passage (procedural)
    ├── millholm_mine    ← copper/tin ore, kobold warren (boss: Chieftain)
    ├── millholm_sewers  ← Thieves' Lair (hidden, find_dc=20)
    ├── millholm_southern ← Gnoll territory (boss: Warlord), barrow, moon fields
    └── millholm_faerie_hollow ← arcane dust harvest (invisible entrance)

Procedural Zones (DungeonTemplate system)
    ├── Rat Cellar (instance, solo, quest-gated)  ← boss: RatKing
    ├── Deep Woods Passage (passage, group)
    └── Cave of Trials (instance, group)

Future zones expand the same pattern: static district rooms + optional procedural deepening.
```

---

## Cross-Cutting Concerns

### The Blockchain Invariant

Every resource and item movement in the game ultimately maps to an XRPL ledger state. The invariant that must always hold:

```
RESERVE + SPAWNED + ACCOUNT + CHARACTER + SINK = vault on-chain balance
```

All service calls (GoldService, ResourceService, NFTService) are accessed through encapsulation layers (FungibleInventoryMixin, BaseNFTItem hooks) — game code never calls services directly. This ensures every in-game state change is paired with the correct DB operation.

### Gold Sink Model

All gold that leaves player hands flows to SINK. `ReallocationServiceScript` drains SINK → RESERVE daily (currently 100%). A future evolution will burn a percentage of gold as a deflationary counterweight to active player provisioning — the burn step belongs inside the existing reallocation path, not a new system. See TREASURY.md § Gold Sink Integration for the planned split.

### Population Scaling

Several systems scale with active player count (`active_players_7d` from `PlayerSession`):
- **Resource spawn:** consumption baseline and price modifier are measured against live player activity — the 24h rolling consumption window naturally scales with how many players are actually using the resource
- **NFT saturation:** targets are expressed as "N copies per X active players"
- **Knowledge saturation:** denominator is eligible players (those with requisite skill/mastery)

A server with 10 players and a server with 1,000 players should feel equally supplied at their respective scales. Population scaling is what makes this automatic.

### Evennia Persistence

Scripts (CombatHandlers, ZoneSpawnScript, ResourceSpawnScript, NFTSaturationScript) survive `evennia restart` via Evennia's built-in script persistence. The game can be restarted mid-combat and handlers resume correctly. Server startup (`at_server_start()`) ensures all global scripts exist and restarts any orphaned timers (corpse decay, purgatory, mob respawn).
