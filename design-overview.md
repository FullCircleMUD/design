# FCM Design Documents

System-level design for FullCircleMUD. These documents describe the *what* and *why* — the economic models, spawn algorithms, combat architecture, and world structure. For implementation details and code patterns, see `src/game/CLAUDE.md`.

---

## The design docs

The full catalogue of design documents is in **[INDEX.md](INDEX.md)** (the *Game design* section),
organised by category with a one-line summary each. This overview keeps the **cross-system narrative**
below — how the major systems interact — which no single doc holds.

> Deployment, infrastructure, recovery runbooks, and Railway/CI configuration live in the private
> `ops/` repository, not in these design docs.

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
Mob spawn rules (fcm-mobs YAML)
    │
    ▼
evennia-mob-spawner (per-rule tick)
    │
    ├── spawns mobs into rooms      ──► Players encounter → Combat
    │   per their rule                         │
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

Service NPCs (bartenders, shopkeepers, etc.) bypass this loop — they are authored once in fcm-world YAML and default to `is_immortal=True`, so they never reach `die()` and the spawn system never has to replace them. See spawn-mobs.md § What This System Does Not Handle.

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
    See unified-item-spawn-system.md § KnowledgeCalculator.

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

**Note:** An earlier design included a third factor (circulating supply per player-hour) intended as an anti-hoarding defence. It was removed — consumption already captures demand and AMM price already captures market conditions, so a third factor with a manually-configured target was redundant guesswork. See unified-item-spawn-system.md § Why Not a Supply Modifier for the full rationale.

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
    Mob deleted; evennia-mob-spawner repopulates after the rule's
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

Knowledge — learned (permanent, survives remort)
    Spell scrolls (mage) → transcribe → db.spellbook
    Recipe scrolls / trainer purchase → learn → db.recipe_book

Knowledge — granted (mastery-derived, cleared by remort)
    Cleric & paladin spells → every spell at or below divine-school mastery
    Enchanting recipes → every recipe at or below enchanting mastery
    Reconciled at chargen, on mastery training, and at login

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

All gold that leaves player hands flows to SINK. `ReallocationServiceScript` drains SINK → RESERVE daily (currently 100%). A future evolution will burn a percentage of gold as a deflationary counterweight to active player provisioning — the burn step belongs inside the existing reallocation path, not a new system. See treasury.md § Gold Sink Integration for the planned split.

### Population Scaling

Several systems scale with active player count (`active_players_7d` from `PlayerSession`):
- **Resource spawn:** consumption baseline and price modifier are measured against live player activity — the 24h rolling consumption window naturally scales with how many players are actually using the resource
- **NFT saturation:** targets are expressed as "N copies per X active players"
- **Knowledge saturation:** denominator is eligible players (those with requisite skill/mastery)

A server with 10 players and a server with 1,000 players should feel equally supplied at their respective scales. Population scaling is what makes this automatic.

### Evennia Persistence

Scripts (CombatHandlers, evennia-mob-spawner population scripts, UnifiedSpawnScript, NFTSaturationScript) survive `evennia restart` via Evennia's built-in script persistence. The game can be restarted mid-combat and handlers resume correctly. Server startup (`at_server_start()`) ensures all global scripts exist and restarts any orphaned timers (corpse decay, purgatory, mob respawn).
