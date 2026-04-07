# ECONOMY.md

> **THIS FILE is for ECONOMIC DESIGN only** — pricing models, market structures, spawn algorithms, trade mechanics, gold sinks, revenue models, and economic balance. For technical architecture, code patterns, and implementation details, see **src/game/CLAUDE.md**. For world building, lore, and creative direction, see **design/WORLD.md**. For economic implementation backlog, see **ops/ECONOMY_BACKLOG.md**. Do not put technical implementation details here. Do not put world building content here.

---

## Purpose

This is the economic bible for FullCircleMUD. Everything that shapes how value flows through the game — from gold sinks to AMM pricing to spawn algorithms to market structures — lives here. When designing new economic systems, balancing existing ones, or making decisions about what things should cost, this is the source of truth.

---

## Table of Contents

- [Core Economic Philosophy](#core-economic-philosophy)
- [Fee Structure & Value Circulation](#fee-structure--value-circulation)
- [Currency & Asset Types](#currency--asset-types)
- [AMM Trade Accounting](#amm-trade-accounting)
- [Gold Sink Model (90/10 Respawn/Revenue)](#gold-sink-model-9010-respawnrevenue)
- [Market Structure — Three Tiers](#market-structure--three-tiers)
- [Tier 1: Resource Shopkeepers (Fungible AMM)](#tier-1-resource-shopkeepers-fungible-amm)
- [Tier 2: Equipment Shopkeepers (Tracker Token AMM)](#tier-2-equipment-shopkeepers-tracker-token-amm)
- [Tier 3: Auction House (Player-Driven)](#tier-3-auction-house-player-driven)
- [Spawn Algorithms](#spawn-algorithms) (Resources, NFT Items)
- [Supply-Side Levers](#supply-side-levers)
- [Demand-Side Factors](#demand-side-factors)
- [Anti-Manipulation Mechanisms](#anti-manipulation-mechanisms)
- [Item Classification — What's Tradeable Where](#item-classification--whats-tradeable-where)
- [Enchanting Economics](#enchanting-economics)
- [Crafting Tier Progression & AMM Cutoff](#crafting-tier-progression--amm-cutoff)
- [Player Economic Roles](#player-economic-roles)
- [Telemetry Requirements](#telemetry-requirements)
- [Economic Invariants](#economic-invariants)

---

## Core Economic Philosophy

The economy is **on-chain** — gold and resources are issued currencies on the XRPL, items are NFTs. Everything is transparent and auditable. This means:

- **No free loot** — everything that enters the game is drawn from a finite, managed supply
- **Supply and demand are real** — AMM pools drive prices based on actual scarcity
- **Players are rational economic actors** — they will act in their own self-interest, and we design for that
- **Every transaction serves the economy** — fees prevent inflation, recirculate value back into the player reward pool, and cover operational costs (hosting, LLM API fees, development)
- **Withdrawal expands the market, not the supply** — assets withdrawn to private wallets remain in active supply and can be traded on AMMs or external NFT markets; total supply is unchanged

---

## Fee Structure & Value Circulation

Every value flow path serves three purposes: (1) deflationary pressure to prevent asset inflation and maintain item value, (2) recirculation of value back into the player reward pool, and (3) revenue to cover hosting, LLM API fees, and ongoing development:

| Path | Economic Purpose |
|---|---|
| In-game buy (resource) | Ceil-rounding dust (gold) → SINK → recirculated as rewards / operational costs |
| In-game sell (resource) | Floor-rounding dust (resource) → SINK → recirculated |
| In-game buy (equipment) | Tracker token AMM spread + rounding → SINK |
| In-game sell (equipment) | Tracker token AMM spread + rounding → SINK |
| On-chain AMM trade (direct trade) | AMM trading fees on vault-owned liquidity |
| Gold sinks | Crafting, training, travel, repair, etc. → SINK → 90% respawned as rewards, 10% operational |
| Item destruction (junk, durability) | Returns assets to RESERVE — prevents item inflation |

Every trade path contributes to economic health. Withdrawal to a private wallet does not reduce supply — the asset remains player-owned and in active circulation.

---

## Currency & Asset Types

| Asset | On-chain form | In-game form | Pricing mechanism |
|---|---|---|---|
| Gold (FCMGold) | XRPL issued currency | Integer amounts in FungibleGameState | Base currency — everything priced in gold |
| Resources (36 types) | XRPL issued currencies | Integer amounts in FungibleGameState | XRPL AMM pools (resource vs gold) |
| Common NFT items | XRPL NFTs (NFTokens) | Evennia objects in game world | Tracker token AMMs (see below) |
| Rare NFT items | XRPL NFTs (NFTokens) | Evennia objects in game world | Player-driven auction / external marketplace |
| Scrolls & Recipes | XRPL NFTs (NFTokens) | Consumable Evennia objects | Not shopkeeper-tradeable — earned only |

---

## AMM Trade Accounting

Every trade has two sides: the **player side** (integer amounts) and the **AMM side** (decimal amounts). The difference is "rounding dust" that flows to SINK — funding reward recirculation and operational costs.

### Buy Example — Player Buys 10 Wheat

1. The AMM formula says 10 wheat costs **10.15 gold** (constant product + fee)
2. We ceil-round: charge the player **11 gold** (integer)
3. The vault sends gold to the AMM and gets wheat back
4. The AMM actually takes **10.15 gold** from the vault (decimal)
5. The vault keeps **0.85 gold** (11 charged - 10.15 paid = profit)
6. Six DB operations record both sides: player debited 11 gold + credited 10 wheat, vault debited 10.15 gold to AMM + credited wheat from AMM

**Buy-side margin is always in gold.** Always >= 0 by ceil-rounding construction.

### Sell Example — Player Sells 10 Wheat

1. The AMM formula says 10 wheat is worth **9.85 gold** (constant product + fee)
2. We floor-round: pay the player **9 gold** (integer)
3. The vault sends wheat to the AMM and gets gold back
4. The AMM actually takes **9.85 wheat** from the vault (decimal)
5. The vault keeps **0.15 wheat** (10 taken from player - 9.85 sent to AMM = profit)
6. Six DB operations record both sides: player debited 10 wheat + credited 9 gold, vault debited 9.85 wheat to AMM + credited gold from AMM

**Sell-side margin is always in the resource.** Always >= 0 by floor-rounding construction.

### Dust Tracking

Both types of dust are tracked automatically on every AMM trade by moving the margin from RESERVE → SINK:

- **Gold dust** (buy-side and sell-side): RESERVE gold debited, SINK gold credited. Accumulates until the daily reallocation script drains SINK back to RESERVE.
- **Resource dust** (buy-side and sell-side): same RESERVE → SINK flow, per currency code.

### RESERVE Tracking

RESERVE balances track the actual decimal amounts exchanged with the AMM, so the DB stays in sync with on-chain reality. Player-facing amounts are always integers.

---

## Gold Sink Model (90/10 Respawn/Revenue)

All consumed gold flows to the **SINK** location in `FungibleGameState`. The daily `ReallocationServiceScript` drains SINK → RESERVE (100% for now; 10% gold burn to issuer deferred until vault signing). This enables the 90/10 model:

- **90% of sunk gold is respawned** — mob drops, loot, quest rewards
- **10% is game revenue** — hosting costs, LLM API fees, development

### All Consumption Flows (→ SINK)

- Crafting workshop fee
- Repair workshop fee
- Gem inset workshop fee
- Processing/conversion fee
- Skill/weapon training fee
- Recipe purchase fee
- Cemetery bind fee
- Purgatory early release fee
- Travel gateway gold cost
- Explore gateway gold cost
- Inn ale/stew purchases
- Trading post listing fee
- Junking gold/resources
- Eating food (bread consumption)
- Refueling lanterns (coal consumption)
- Quest tribute (collect quest completion)
- AMM gold dust (rounding margin on every buy and sell trade)
- AMM resource dust (rounding margin on resources)

### Cleanup Flows (→ RESERVE, not SINK)

- Corpse decay (gold/resources returned to reserve pool)
- Dungeon instance teardown
- Tutorial instance cleanup
- World rebuild (soft_rebuild)
- NFT deletion hooks (at_object_delete)

### Gold Flow Equation

```
gold_entering = mob_drops + quest_rewards + spawn_allocations
gold_leaving = gold_to_sink + exports
net_flow = gold_entering - gold_leaving
```

Target: slight net negative flow (mildly deflationary) — scarcity supports value.

---

## Market Structure — Three Tiers

### Tier 1: Resource Shopkeepers (Fungible AMM)

**What's sold:** Raw and processed fungible resources only (wheat, flour, bread, iron ore, ingots, wood, timber, herbs, etc.).

**Pricing:** Live XRPL AMM pool prices. Constant product formula (x * y = k) with AMM trading fee. Buy prices ceil-rounded up, sell prices floor-rounded down.

**Shop examples:** Farmer's market sells wheat/cotton. General store in town. Alchemist ingredient shop. Resource shops never sell equipment.

**Already implemented:** ShopkeeperNPC with list/quote/accept/buy/sell commands, AMMService integration, 6-operation atomic accounting.

### Tier 2: Equipment Shopkeepers (Tracker Token AMM)

**What's sold:** Common NFT equipment items — weapons, armor, jewellery, tools. Items that are functionally fungible (one Iron Sword at full durability = any other Iron Sword at full durability).

**Pricing:** Tracker token AMM pools (see [Tracker Tokens](#nft-item-market-making--tracker-tokens) below).

**Durability rule:** Shopkeepers only buy at full durability. "I don't buy damaged goods — repair it first." This means:
- Clean 1:1 tracker token mapping (no fractional accounting)
- Repair cost is a gold sink that factors into sell decisions
- Low-value items with high repair costs get junked instead → natural item drain → supports prices

**Shop examples:** Blacksmith sells/buys weapons and metal armor. Tailor sells/buys cloth armor. Jeweller sells/buys rings and amulets. Equipment shops never sell raw resources.

**Not yet implemented.** Requires tracker token infrastructure.

### Tier 3: Auction House (Player-Driven)

**What's sold:** Everything that doesn't fit Tier 1 or 2:
- Enchanted weapons with gem insets (bespoke names, unique effect combinations)
- Master and Grandmaster tier crafted items
- Scrolls and recipes (earned-only, never NPC-sold)
- Ultra-rare drops (population-gated items)
- Any item a player wants to sell to other players

**Pricing:** Player-set. No AMM involvement. Free market.

**Revenue:** Listing fee + transaction cut (gold sinks).

**Not yet implemented.** Design phase.

---

## NFT Item Market Making — Tracker Tokens

### The Problem

Common NFT items (iron swords, leather armor, mage robes) are functionally fungible but can't use on-chain AMMs directly because they're NFTs, not fungible tokens.

### The Solution

Issue "tracker tokens" — one XRPL issued currency per common NFT item type (e.g., `FCMTrkIronSword`, `FCMTrkMageRobe`). Set up real XRPL AMM pools: tracker token vs FCMGold. **Only the vault holds tracker tokens** — no external wallets can interfere.

### How It Works

- **Player sells an iron sword to shopkeeper** → vault sells 1 tracker token to AMM → gets gold → pays player (floor-rounded)
- **Player buys an iron sword from shopkeeper** → vault buys 1 tracker token from AMM → pays gold → gives player an item from RESERVE (ceil-rounded)
- Same AMMService pipeline, same rounding, same 6-operation accounting, same margin

### Why This Works

**Price discovery is automatic:**
- Players flood market with crafted swords → vault sells trackers → price drops → swords cheaper than crafting cost → players stop crafting, start buying → price recovers
- Swords become scarce → vault buys trackers → price rises → crafting becomes profitable → players craft more

**Closed-loop security:** Only the vault trades tracker tokens. No external wallets hold them. On-chain AMM used as a computational pricing engine, not a public marketplace. Manipulation is impossible.

### Setup Required

- Issue tracker tokens per common NFT item type via a second issuer wallet
- Seed AMM pools at recipe component cost (sum of resource AMM prices + crafting fee)
- Register as `CurrencyType` rows with `is_nft_tracker=True` flag (or similar)
- Extend shopkeeper to handle NFT buy/sell via tracker token AMM

---

## Spawn Algorithms

### 1. Fungible Resources — Consumption + AMM Price

For all resources traded via AMM (wheat, flour, ore, wood, etc.). Two inputs:

- **Consumption rate:** 24-hour rolling average of how much players actually consume. This is the baseline spawn rate — the system matches supply to real demand.
- **AMM price signal:** each resource has a target price band [floor, ceiling]. Closer to floor → reduce spawns. Closer to ceiling → increase spawns.

**Self-correcting loop:** If price drops too low, spawning dampens, supply contracts, price recovers. If price rises too high, spawning increases, supply grows, price drops. The AMM handles price discovery; consumption rate tracks actual demand; spawns control quantity entering the system.

### 2. NFT Items — Saturation-Based Drops

All non-AMM NFT items (scrolls, wands, recipes, rare items, rare ingredients) use a unified **saturation framework**. These items are discovery-only — never sold by shopkeepers, never in AMMs. Players find them as mob drops, quest rewards, or chest loot.

#### The Saturation Concept

Every droppable NFT item has a **saturation score** — how "saturated" is the game with this item? The definition of saturation differs by item category, but the loot selection logic is the same:

| Category | Saturation = | Measured against |
|---|---|---|
| Scrolls & recipes | (players who **know** it + unlearned copies in player hands) / active players with requisite mastery to learn it | Player knowledge + pipeline supply, denominator gated by eligibility |
| Rare items & ingredients | Count **in circulation** vs target ratio | Items in game world (transient) |
| Wands | TBD — charges are consumed, so may use circulation count like rare items | TBD |

**Knowledge items** (scrolls, recipes) saturate permanently — once learned, learned forever. No natural scarcity. A static drop rate would flood the game with scrolls nobody needs.

**Physical items** (rare weapons, rare ingredients) saturate transiently — they leave circulation via junk, destruction, or consumption — withdrawal to a private wallet does not remove them from circulation. The target is a ratio against active player count: as the player base grows, more rare items enter the game.

#### Saturation Snapshot (daily)

A daily script calculates saturation for all tracked items:

```
# Knowledge items (scrolls, recipes)
known_by = count of active players who have learned spell/recipe X
unlearned_copies = count of scroll/recipe X in player hands (CHARACTER + ACCOUNT in NFTGameState)
eligible_players = active players (7d) who have the requisite skill/school mastery to learn it
saturation = (known_by + unlearned_copies) / eligible_players

# Physical items (rare drops, ingredients)
saturation = current_in_circulation / (active_players_7d / rarity_divisor)
# saturation > 1.0 = oversaturated (too many in game), < 1.0 = undersaturated
```

Knowledge saturation counts both **learned knowledge** (permanent, from `db.spellbook`/`db.recipe_book`) and **unlearned copies in player hands** (scrolls/recipes sitting in inventory or bank). This prevents flooding the game with scrolls that are already piling up unused — a scroll in someone's bank is supply in the pipeline.

Updated daily — saturation moves slowly for both categories. One snapshot table covers everything.

#### Dynamic Loot Selection

Mobs don't drop specific items. Instead, loot tables define:
- **Does this mob drop a scroll/recipe/rare?** — flat % chance per kill, mob-specific
- **What tier?** — mob tier gates item tier (e.g., NOVICE-SKILLED scrolls from common mobs, EXPERT-MASTER from bosses)

When a drop triggers, the system:
1. Queries the latest saturation snapshot
2. Filters to items within the mob's tier range and drop category
3. **Hard eligibility check:** removes any item at or above its saturation threshold — only undersaturated items remain
4. **If no eligible items remain → drop is vetoed entirely.** Nothing drops. The mob's roll succeeded but the world doesn't need any of the items it could have dropped.
5. **Weights selection among eligible items** — lower saturation = higher weight
6. Picks one → that's the drop

#### Why This Works

- **Early game (day 1):** Everything at 0% saturation → all items drop equally → rapid initial distribution
- **Mid game:** Common spells reach 60-80% saturation → those drop rarely → undersaturated items dominate drops
- **Late game:** Most knowledge widely known → knowledge drops become very rare → finding an undersaturated scroll is an event
- **New players:** Even at high global saturation, a new player finding a "common" scroll is still valuable to them
- **Growing player base:** More active players = higher target circulation for rare items = more rare drops enter the game naturally
- **Shrinking player base:** Fewer active players = lower target = drops slow down = no glut of rare items on a dying server
- **Self-correcting:** No manual tuning needed. The system responds to actual game state.

#### Design Notes

- Saturation measured against **active players** (session in last 7 days), not all characters ever created
- The saturation curve shape (linear, exponential, etc.) is a per-item tuning knob — steeper curve = rarer drops at high saturation
- `rarity_divisor` for physical items is configurable per item type (e.g., 1 per 100 players, 1 per 500 players)
- Player-to-player trading is allowed (auction house, direct trade) — the system doesn't care how items spread, only that they have spread
- Non-craftable rare items that are consumed or destroyed reduce saturation → drop rate recovers automatically. Withdrawing to a private wallet does not reduce saturation — the item remains player-owned and in circulation.

---

## Supply-Side Levers

| Lever | Controls | Mechanism |
|---|---|---|
| AMM liquidity | Price stability | Add liquidity = stabilize. Remove = let market find price. Per-resource. "Playing the Fed." |
| Spawn rates | Quantity entering game | Node respawn timers, yield per node, mob drop rates. Adjusted by algorithm. |
| Gold sinks | Gold inflation/deflation | Crafting fees, training, travel, repair, purgatory, binding. Tracked via SINK location. |
| Tracker token pools | NFT item prices | Seed liquidity sets initial price. Market activity drives price from there. |

---

## Demand-Side Factors

| Factor | Drives | How to measure |
|---|---|---|
| Player-hours per day | Food chain demand (bread + precursors) | Session tracking |
| Active player count | Overall resource demand | Daily unique logins |
| Level/remort distribution | Tier-appropriate equipment demand | Character stats query |
| Resources in circulation | Price pressure via AMM | FungibleGameState aggregation |
| Items in circulation | Rare item drop gating | NFTGameState count per type |

---

## Anti-Manipulation Mechanisms

| Threat | Defence |
|---|---|
| **Large dumps** (crashing a resource price) | AMM slippage via constant product formula — large sells get progressively worse prices |
| **Hoarding** (cornering supply) | Self-limiting: hoarding drives AMM price up → spawn rate increases → new supply flows to non-hoarders. The hoarder pays real cost to maintain their position and must sell into an underpriced market to crash it, losing value. As the economy scales with more players and spawn locations, cornering a market becomes impractical |
| **External AMM manipulation** | Tracker tokens are closed-loop — only vault holds them, no external wallets can interfere |
| **Rounding exploitation** | Every trade pays rounding dust — there's no way to trade without paying the spread |
| **Bot farming** | Per-node cooldowns, diminishing returns, captcha-like interactions (future) |

**Worst case:** temporary price dislocation that corrects over hours/days. The AMM's constant product formula means large dislocations require exponentially more capital to sustain.

---

## Item Classification — What's Tradeable Where

| Item Category | AMM Tradeable? | Where Sold | Notes |
|---|---|---|---|
| Raw resources (wheat, ore, wood, etc.) | Yes — resource AMM | Resource shopkeepers | Tier 1 |
| Processed resources (flour, ingots, timber, etc.) | Yes — resource AMM | Resource shopkeepers | Tier 1 |
| Basic/Skilled equipment (weapons, armor) | Yes — tracker token AMM | Equipment shopkeepers | Tier 2, full durability only |
| Some Expert equipment | Yes — tracker token AMM | Equipment shopkeepers | Edge of Tier 2 |
| Master/GM tier equipment | **No** | Player-to-player trade only — no external market is made | Tier 3 — player-driven |
| Enchanted weapons (gem insets) | **No** | Player-to-player trade only — no external market is made | Bespoke names, unique combos |
| Deterministic enchanted wearables (rings, amulets) | **Yes** — own tracker token | Equipment shopkeepers | Standardised output = fungible |
| Scrolls (spell scrolls) | **No** | Earned only (mob drops, quests) | Never NPC-sold, gates progression |
| Recipes (crafting recipes) | **No (mostly)** | Earned from trainers or mob drops | Some lower-tier recipes trainer-sold, never shopkeeper-sold |
| Ultra-rare drops | **No** | Auction house / external XRPL market | Population-gated |

### The Tier Cutoff Principle

- **Basic, Skilled, some Expert items** → AMM economy (commodity market, high volume, liquid)
- **Master and Grandmaster items** → Auction economy (player-driven, scarcity-based, prestigious)

This creates a natural economic progression: new players enter a stable, liquid market. As they progress, the economy becomes increasingly player-driven and speculative. Master/GM crafters become known names — their reputation IS their brand.

---

## Enchanting Economics

### Wearables — Deterministic Enchanting

Enchanting wearables (rings, amulets, etc.) is **deterministic** — same inputs always produce the same output. A "Ruby Ring of Fire Resistance" is always the same as any other. This means:

- Enchanted wearables CAN get their own tracker token AMM pools
- Price naturally settles at: base item cost + material cost + enchanting fee + margin
- If materials get expensive, the enchanted item's AMM price rises to match

### Weapons — Gem Insets (Enchanter + Jeweller)

Two distinct crafting roles combine to produce enchanted weapons:

- **Enchanter** — enchants raw gems into enchanted gems. The outcome is probabilistic — the enchanter rolls on an effect table and the resulting gem has a known, inspectable effect. The risk (and speculation) is entirely at this step.
- **Jeweller** — insets an enchanted gem into a weapon. This is a deliberate choice: the player can inspect the gem's effects before committing. The inset consumes the gem and the fee.

Because there are potentially hundreds of weapon types and scores of possible gem enchants, every weapon+gem combination produces a functionally unique item with a bespoke name. These cannot be standardised into tracker token AMM pools — each one is one-of-a-kind and must be sold player-to-player.

### The Economic Loop

1. **Base weapon** — commodity-priced via tracker token AMM
2. **Raw gems** — commodity-priced via resource AMM (rare drop, so price reflects scarcity)
3. **Gem enchanting** — enchanter rolls effect; result is inspectable before inset decision
4. **Gem inset** — jeweller inserts enchanted gem into weapon; consumes gem + fee
5. **Enchanted weapon** — bespoke item sold player-to-player at player-determined price
6. **Every step funds the economy** — rounding dust, AMM fees, crafting fees, and inset fees flow to SINK for reward recirculation and operational costs

---

## Crafting Tier Progression & AMM Cutoff

| Mastery Tier | AMM Market? | Economic Character |
|---|---|---|
| Basic | Yes | Commodity — high volume, thin margins, accessible to all |
| Skilled | Yes | Commodity — same items in higher zones, wider variety |
| Expert | Some | Transition zone — common expert items are AMM, rare ones are auction |
| Master | **No** | Prestige — player-driven prices, reputation matters |
| Grandmaster | **No** | Ultra-prestige — name your price, limited supply |

### Crafter Economics by Tier

- **Basic/Skilled crafters:** Margin = AMM sell price - material cost - crafting fee. Thin margins, high volume. Viable with self-gathered materials.
- **Expert crafters:** Some items still liquid via AMM, others need auction buyers. Transition to reputation economy.
- **Master/GM crafters:** No AMM floor. Must find buyers. Reputation and relationships drive sales. A GM blacksmith who makes the best sword on the server names their price.

---

## Player Economic Roles

The economic system supports multiple viable playstyles beyond combat:

| Role | Gameplay | Profit Mechanism |
|---|---|---|
| **Gatherer** | Farm resources at harvesting nodes | Sell raw materials to AMM when prices are good |
| **Crafter** | Process raw materials into finished goods | Material cost → finished item value spread |
| **Trader/Arbitrageur** | Buy low, sell high, hold inventory | Exploit price swings between resources and over time |
| **Self-sufficient crafter** | Gather own materials + craft | Lower costs = wider margins, but more time investment |
| **Enchanter/Speculator** | Enchant weapons with gems | Risk vs reward — bad rolls lose money, good rolls sell high |
| **Merchant** | Buy from crafters, sell on auction house | Middleman margin on Master/GM items |

### Trader Gameplay Example

1. Wheat price crashes because 20 farmers are grinding → trader buys wheat cheap
2. Holds it until bakers need flour but farmers moved on to cotton → sells wheat at a premium
3. Or crafts bread when the flour-to-bread margin is good
4. Pure market-making gameplay with zero combat required

---

## Telemetry System (Implemented)

Hourly snapshots via `TelemetryService.take_snapshot()` → `TelemetryAggregatorScript` global script. Admin command: `economy`.

| Metric | Status | Storage |
|---|---|---|
| Active players (1h/24h/7d) | **Built** | `EconomySnapshot.unique_players_*` |
| Players online | **Built** | `EconomySnapshot.players_online` (from `PlayerSession`) |
| Player session tracking | **Built** | `PlayerSession` (puppet/unpuppet hooks) |
| Gold circulation/reserve | **Built** | `EconomySnapshot.gold_circulation/reserve` |
| Gold sinks per hour | **Built** | `EconomySnapshot.gold_sinks_1h` |
| Gold spawned per hour | **Built** | `EconomySnapshot.gold_spawned_1h` |
| Per-resource circulation | **Built** | `ResourceSnapshot.in_character/account/spawned/reserve/sink` |
| Per-resource velocity | **Built** | `ResourceSnapshot.produced/consumed/traded/exported/imported_1h` |
| AMM prices over time | **Built** | `ResourceSnapshot.amm_buy_price/sell_price` |
| AMM trade volume | **Built** | `EconomySnapshot.amm_trades_1h/amm_volume_gold_1h` |
| Import/export counts | **Built** | `EconomySnapshot.imports_1h/exports_1h` |
| Level/remort distribution | Phase 2 | Character stats query (not yet aggregated) |
| Per-NFT-type circulation | Phase 2 | NFTGameState count per type (not yet aggregated) |
| Per-NFT-type lifecycle rates | Phase 2 | Transfer log analysis (not yet aggregated) |
| NFT item saturation | **Built** | `SaturationSnapshot` (daily) — knowledge saturation (scrolls/recipes) + circulation saturation (rare items/ingredients) |

---

## Economic Invariants

These must always hold true. If any are violated, something is broken:

1. **Every trade has economic purpose:** every in-game trade generates rounding dust that flows to SINK (deflationary + reward recirculation). Every on-chain trade generates AMM fees on vault-owned liquidity. Withdrawal to a private wallet does not reduce supply — assets remain player-owned and in circulation.
2. **Ceil buys / floor sells:** player always pays more than AMM cost (buys) or receives less than AMM yield (sells). Margin >= 0 on every trade, always.
3. **Tracker tokens are closed-loop:** only the vault trades them. No external manipulation possible.
4. **RESERVE reconciliation:** `RESERVE + SPAWNED + ACCOUNT + CHARACTER + SINK = vault on-chain balance` — must always hold for every fungible currency. NFT items: `RESERVE + SPAWNED + ACCOUNT + CHARACTER + ONCHAIN = total minted NFTs`. AUCTION is a future potential feature (auction house not yet built) — direct player-to-player trade is the current mechanism. Once AUCTION is live, add it to the NFT equation: `RESERVE + SPAWNED + ACCOUNT + CHARACTER + ONCHAIN + AUCTION = total minted NFTs`.
5. **Consumption routing completeness:** every gold fee and resource consumption must use `return_gold_to_sink()` / `return_resource_to_sink()`. No untracked sinks. AMM dust is routed RESERVE → SINK automatically.
6. **Cleanup vs consumption distinction:** cleanup (corpse decay, dungeon teardown, world rebuild) uses `return_*_to_reserve()`. Consumption (fees, crafting, eating) uses `return_*_to_sink()`.
7. **Integer player amounts:** players only ever see and transact in whole numbers. Decimals exist only in RESERVE ↔ AMM accounting.

---

## Backlog: Reserve Capacity Monitoring

**Not yet built.** Need an early warning system that monitors consumption rate against remaining reserves for all asset types — FCMGold, game resources, PGold, item proxy tokens, and blank NFTs. If player growth outpaces supply, the game could run out of tokens to execute AMM swaps, honour exports, or spawn new items. The system should project time-to-depletion from hourly velocity data and alert before reserves hit critical levels. See [TELEMETRY.md](TELEMETRY.md) for the snapshot data this would consume.
