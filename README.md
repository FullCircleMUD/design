# FCM Design Documents

System-level design for FullCircleMUD. These documents describe the *what* and *why* — the economic models, spawn algorithms, combat architecture, and world structure. For implementation details and code patterns, see `src/game/CLAUDE.md`.

---

## Document Index

| Document | What It Covers |
|---|---|
| **WORLD.md** | Zone structure, world regions, lore, NPC placement, narrative design |
| **ECONOMY.md** | Gold sinks, AMM pricing, market tiers, resource economics, revenue model |
| **COMBAT_SYSTEM.md** | Real-time combat architecture, weapon hooks, parry/riposte, skills, weapon mastery, stealth, dual-wield |
| **EFFECTS_SYSTEM.md** | EffectsManagerMixin (3-layer effect system), conditions, named effects, damage resistance, damage pipeline |
| **SPELL_SKILL_DESIGN.md** | Spell system architecture, all 12 spell school tables, crafting recipe catalog, spell implementation patterns |
| **CRAFTING_SYSTEM.md** | Crafting/processing system architecture, recipe format, enchanting, gem insetting, room types |
| **NPC_QUEST_SYSTEM.md** | NPC hierarchy, training, shopkeeping, quest engine, implemented quests |
| **INVENTORY_EQUIPMENT.md** | Item system, equipment/wearslots, carrying capacity, NFT ownership, wear effects |
| **INTERZONE_TRAVEL.md** | Zone-to-zone travel — `explore`/`travel`/`sail` commands, cartography mastery gates, ship tiers, route map NFTs, food costs, party mechanics |
| **CARTOGRAPHY.md** | Cartography skill — intra-zone district mapping (`survey`/`map` commands, district map NFTs). For inter-zone route discovery see `INTERZONE_TRAVEL.md` |
| **NEW_PLAYER_EXPERIENCE.md** | Tutorial flow, starter quests, first-hour progression, Millholm onboarding |
| **UNIFIED_ITEM_SPAWN_SYSTEM.md** | Calculator + Distributor architecture for resources, gold, knowledge NFTs, and rare items |
| **SPAWN_COMMODITY_MOBS.md** | Zone-based mob population maintenance via ZoneSpawnScript |
| **SPAWN_BOSS_MOBS.md** | Unique mob placement and delay-based respawn system |
| **ROOM_ARCHITECTURE.md** | Room typeclass hierarchy, terrain types, specialised room types, mixins |
| **EXIT_ARCHITECTURE.md** | Exit typeclass hierarchy, doors, locks, traps, vertical-aware exits |
| **PROCEDURAL_DUNGEONS.md** | Dungeon template system, instance lifecycle, passage dungeons, collapse mechanics |
| **VERTICAL_MOVEMENT.md** | Climbing, flying, swimming, underwater mechanics, vertical exit gates |
| **WEBSITE.md** | Web frontend pages, geo-detection infrastructure, page inventory, implementation status |
| **COMPLIANCE.md** | Legal/regulatory framework (no-redemption model), token classification, game economy management, language policy |
| **SUBSCRIPTIONS.md** | Subscription billing — payment flow, lifecycle, character-entry gating (`ic`/`charcreate`/`chardelete`), trial period |
| **IMPORT_EXPORT.md** | Chain-boundary `import`/`export` commands — gate stack, asymmetric trial gating, Xaman flow, mirror state transitions, planned OFAC SDN screening |
| **DATABASE.md** | Django app layout, three-database architecture, model overview, migrations |
| **DEPLOYMENT.md** | Server deployment, infrastructure, environment configuration |
| **TELEMETRY.md** | Economy telemetry snapshots, player session tracking, saturation snapshots |

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
- Hoarding (high per-capita supply) → supply modifier dampens spawn → new supply flows to actual consumers
- Items junked or destroyed → leave circulation entirely → saturation drops → new copies more likely to drop
- Players withdrawing to their private wallet does NOT reduce saturation — the item still exists and is still player-owned

---

### Mob and Combat Systems

```
Zone Spawn Rules (JSON)
    │
    ▼
ZoneSpawnScript (15s tick)
    ├── spawns Commodity Mobs ──► Players encounter → Combat
    │       │                          │
    │       ▼                          ▼
    │   [deleted on death]       CombatHandler (per-combatant)
    │       │                          │
    │       └── repopulated by    execute_attack() (14 hooks)
    │           next ZoneSpawnScript tick    │
    │                                   Weapon mastery effects
    │                                   Parry / Riposte
    │                                   Named effects (stun/prone/etc.)
    │                                        │
    └── Boss Mobs (is_unique=True)           ▼
            │                         Mob dies → Corpse (loot)
            ▼                                │
        Legacy _respawn()                    ▼
        (delay-based, 10min)         Quest events fire
                                     XP awarded
```

---

### NFT Item Flow

```
NFT items enter the game via two paths:

PATH A — Common Equipment (Tracker Token AMM — NOT YET BUILT)
    Crafting Room → player crafts sword → NFT spawns → sold to Shopkeeper
    Shopkeeper → Tracker Token AMM (price discovery) → sold to next player

PATH B — Rare/Unique Items (Saturation-based drops)
    Mob dies → loot roll → saturation snapshot consulted
    → undersaturated items weighted higher → item drops to corpse
    → player loots → CHARACTER location
    → player may withdraw to private wallet (ONCHAIN) → item remains in circulation
    → saturation only drops if item is junked or destroyed

KNOWLEDGE ITEMS (Scrolls and Recipes)
    Same saturation mechanism but saturation measures how many active
    players *already know* the spell/recipe, not just circulation count.
    Scrolls consumed (transcribed) → permanently reduce future demand.
    Saturation only rises, never falls naturally.

DAILY SNAPSHOT (NFTSaturationScript)
    ─ runs at midnight
    ─ queries spellbooks, recipe books, NFTGameState
    ─ writes SaturationSnapshot rows
    ─ mob loot selection reads these snapshots (loot selection not yet wired)
```

---

### Resource Spawn in Detail

The three-factor algorithm prevents both inflation and scarcity simultaneously:

```
FACTOR 1: Consumption baseline (what players actually used in last 24h)
              ↓
         × FACTOR 2: Price modifier (AMM price vs target band)
              │  price too high → modifier > 1.0 (spawn more)
              │  price in band  → modifier = 1.0 (stable)
              │  price too low  → modifier < 1.0 (spawn less)
              ↓
         × FACTOR 3: Supply modifier (per-player-hour circulating supply)
              │  too much supply → modifier < 1.0 (dampen)
              │  healthy supply  → modifier = 1.0
              │  too little      → modifier > 1.0 (boost)
              ↓
         = SPAWN_AMOUNT (units this hour)
              │
              ▼
         Allocated to RoomHarvesting rooms by weight (1-5 per room)
              │
              ▼
         Drip-fed across the hour (max 12 ticks, min 5 min apart)
         so early-login players don't harvest everything before others arrive
```

---

### Combat Flow in the Bigger Picture

```
BEFORE COMBAT:
    Equipment effects (wear_effects) → _recalculate_stats() → cached stats
    Spell buffs (named effects) → conditions + stats (also via recalculate)

DURING COMBAT:
    CombatHandler tick (every weapon.speed seconds)
        → execute_attack() (14 hooks)
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
    XP awarded (10 × mob level)
    Corpse dropped with mob's loot
    is_unique=False → mob deleted, ZoneSpawnScript repopulates
    is_unique=True → mob stays in DB, _respawn() scheduled
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

### 90/10 Gold Model

All gold that leaves player hands flows to SINK. Daily reallocation moves SINK → RESERVE (currently 100%). When gold burn is implemented (target: 90% respawn, 10% revenue), the split happens in the reallocation step — no changes to gameplay code needed.

### Population Scaling

Several systems scale with active player count (`active_players_7d` from `PlayerSession`):
- **Resource spawn:** supply modifier uses player-hours as denominator
- **NFT saturation:** targets are expressed as "N copies per X active players"
- **Knowledge saturation:** denominator is eligible players (those with requisite skill/mastery)

A server with 10 players and a server with 1,000 players should feel equally supplied at their respective scales. Population scaling is what makes this automatic.

### Evennia Persistence

Scripts (CombatHandlers, ZoneSpawnScript, ResourceSpawnScript, NFTSaturationScript) survive `evennia restart` via Evennia's built-in script persistence. The game can be restarted mid-combat and handlers resume correctly. Server startup (`at_server_start()`) ensures all global scripts exist and restarts any orphaned timers (corpse decay, purgatory, mob respawn).
