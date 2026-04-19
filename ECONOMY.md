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
- [Tier 2: Equipment Shopkeepers (Proxy Token AMM)](#tier-2-equipment-shopkeepers-proxy-token-amm)
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
| In-game buy (equipment) | Proxy token AMM spread + rounding → SINK |
| In-game sell (equipment) | Proxy token AMM spread + rounding → SINK |
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
| Common NFT items | XRPL NFTs (NFTokens) | Evennia objects in game world | Proxy token AMMs (see below) |
| Rare NFT items | XRPL NFTs (NFTokens) | Evennia objects in game world | Player-driven auction / external marketplace |
| Scrolls & Recipes | XRPL NFTs (NFTokens) | Consumable Evennia objects | Not shopkeeper-tradeable — earned only |

### NFT Token Pool Design

All XRPL NFTs are minted with **taxon 0** (uncategorized). NFTs are interchangeable containers — the on-chain token is just an opaque ID + URI. What an NFT *represents* changes constantly: when an item is destroyed the token's DB record is wiped and the same token is reused for another item when it is created. The game DB (`NFTGameState`) tracks what each token currently represents, not the on-chain metadata.

This means:
- No on-chain categorization (taxon is always 0)
- Token identity is decoupled from item identity
- The NFT pool is pre-minted and recycled indefinitely
- Item type, stats, and metadata are resolved server-side via the token's URI

**Why this works for game items but wouldn't for other domains:** Game NFTs exist to let players move items out of the game onto the blockchain, where they can trade them between wallets or hold them independently of the game. But the token is only meaningful insofar as it can be reimported into the game — the item's identity, utility, stats, and behavior exist entirely within the game server. This is fundamentally different from domains like real estate or identity, where an NFT must carry self-contained proof of what it represents because multiple independent systems need to verify ownership without a central authority. Game items have a single authoritative system (the game), so the on-chain token only needs to track *ownership* — not *identity*.

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

### Tier 2: Equipment Shopkeepers (Proxy Token AMM)

**What's sold:** Common NFT equipment items — weapons, armor, jewellery, tools. Items that are functionally fungible (one Iron Sword at full durability = any other Iron Sword at full durability).

**Pricing:** Proxy token AMM pools (see [Proxy Tokens](#nft-item-market-making--proxy-tokens) below).

**Durability rule:** Shopkeepers only buy at full durability. "I don't buy damaged goods — repair it first." This means:
- Clean 1:1 proxy token mapping (no fractional accounting)
- Repair cost is a gold sink that factors into sell decisions
- Low-value items with high repair costs get junked instead → natural item drain → supports prices

**Shop examples:** Blacksmith sells/buys weapons and metal armor. Tailor sells/buys cloth armor. Jeweller sells/buys rings and amulets. Equipment shops never sell raw resources.

**Implemented.** Proxy token infrastructure is live — see [Proxy Tokens](#nft-item-market-making--proxy-tokens) for the full pricing mechanism.

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

## NFT Item Market Making — Proxy Tokens

### The Problem

Common NFT items (iron swords, leather armor, mage robes) are functionally fungible but can't use on-chain AMMs directly because they're NFTs, not fungible tokens.

### The Solution

Issue "proxy tokens" — one XRPL issued currency per common NFT item type, prefixed `P` for proxy (e.g., `PTrainDagger`, `PBronzeSpear`, `PPotWatBull` for Watery Potion of the Bull). Set up real XRPL AMM pools: proxy token vs **PGold** (a proxy for FCMGold, pegged 1:1). Proxy tokens exist solely to enable AMM pricing formulas for in-game shops — they must never be accessible on-chain to anyone other than the game vault wallet.

**Two distinct AMM systems.** FCM's AMM usage splits into two categories with very different visibility:

- **Public AMM pools (FCMGold ↔ resources)** — these ARE publicly accessible on XRPL. Players can export FCMGold and resource tokens and trade them directly on the ledger. The in-game shop uses the same pool for pricing, but external arbitrage is expected and healthy.
- **Private AMM pools (PGold ↔ proxy tokens)** — these exist on XRPL but are **vault-only**. Proxy tokens are used purely as a computational pricing engine for in-game NFT shops. No player ever holds a proxy token.

**Why a separate PGold, not FCMGold?** If an item proxy pool paired `FCMGold` directly (e.g., `FCMGold ↔ PTrainDagger`), a player could export FCMGold from the game to their XRPL wallet, trade it on that public AMM, and acquire proxy tokens. The proxy tokens would then be out in the wild — breaking the closed-loop pricing system and letting players manipulate in-game shop prices via on-chain trades. Using `PGold` (which no player ever holds) keeps the entire proxy system inside the vault.

**Fungibility requires full durability.** NFTs of the same item type are only functionally fungible when all at 100% durability — an 80%-durability iron sword is not equivalent to a brand-new one and can't share the same proxy token price. To preserve the proxy token model, **in-game shopkeepers only trade in fully-repaired items**. Players bring damaged items to a repair NPC first, then sell to the shopkeeper. This ensures every item flowing through shop AMM pricing is interchangeable with every other item of the same type.

**Only the vault holds proxy tokens and PGold** — no external wallets can interfere.

### How It Works

- **Player sells an iron sword to shopkeeper** → vault sells 1 proxy token to AMM → gets gold → pays player (floor-rounded)
- **Player buys an iron sword from shopkeeper** → vault buys 1 proxy token from AMM → pays gold → gives player an item from RESERVE (ceil-rounded)
- Same AMMService pipeline, same rounding, same 6-operation accounting, same margin

### Why This Works

**Price discovery is automatic:**
- Players flood market with crafted swords → vault sells proxies → price drops → swords cheaper than crafting cost → players stop crafting, start buying → price recovers
- Swords become scarce → vault buys proxies → price rises → crafting becomes profitable → players craft more

**Closed-loop security:** Only the vault trades proxy tokens. No external wallets hold them. On-chain AMM used as a computational pricing engine, not a public marketplace. Manipulation is impossible.

### Setup Required

- Issue proxy tokens per common NFT item type via a second issuer wallet
- Seed AMM pools at recipe component cost (sum of resource AMM prices + crafting fee)
- Link each NFT item type to its proxy token via the `tracking_token` field on `NFTItemType` (e.g. `NFTItemType(name="Training Dagger", tracking_token="PTrainDagger")`)
- Extend shopkeeper to handle NFT buy/sell via proxy token AMM

---

## Spawn Algorithms

### 1. Fungible Resources — Consumption + AMM Price

For all resources traded via AMM (wheat, flour, ore, wood, etc.). Two inputs:

- **Consumption rate:** 24-hour rolling average of how much players actually consume. This is the baseline spawn rate — the system matches supply to real demand.
- **AMM price signal:** each resource has a target price band [floor, ceiling]. Closer to floor → reduce spawns. Closer to ceiling → increase spawns.

**Self-correcting loop:** If price drops too low, spawning dampens, supply contracts, price recovers. If price rises too high, spawning increases, supply grows, price drops. The AMM handles price discovery; consumption rate tracks actual demand; spawns control quantity entering the system.

### 2. NFT Items — Saturation-Based Drops

Two categories of NFT use a **saturation framework** for spawning — their drop rate is driven by how saturated the player base is with them, not by fixed drop tables:

1. **Knowledge items (implemented)** — spell scrolls and crafting recipes. The saturation algorithm counts how many eligible players already know a given spell or recipe and uses that to decide how many copies to inject into the world each cycle. Single-use: consuming a scroll/recipe permanently teaches the spell/recipe to the player.

2. **Rare NFT items (future)** — very rare world items will use a general saturation-based spawn algorithm. The framework is in place (snapshots track circulation counts per item type) but the `rarity_divisor` config that drives saturation-based spawning for generic items is not yet implemented.

Both categories are **discovery-only** — never sold by shopkeepers, never in AMMs. Players find them as mob drops, quest rewards, or chest loot.

**Note:** Wands are planned to move out of saturation-based spawning and into a **player-driven crafted market** — mages who know a spell and have the right ingredients will craft their own wands. This removes wands from the discovery-only loop and puts them into player hands via crafting.

#### The Saturation Concept

Every droppable NFT item has a **saturation score** — how "saturated" is the game with this item? The definition of saturation differs by item category, but the loot selection logic is the same:

| Category | Saturation = | Measured against |
|---|---|---|
| Scrolls & recipes | (players who **know** it + unlearned copies in player hands) / active players with requisite mastery to learn it | Player knowledge + pipeline supply, denominator gated by eligibility |
| Rare items & ingredients | Count **in circulation** vs target ratio | Items in game world (transient) |
| Wands | TBD — charges are consumed, so may use circulation count like rare items | TBD |

**Knowledge items** (scrolls, recipes) saturate permanently — once learned, learned forever. No natural scarcity. A static drop rate would flood the game with scrolls nobody needs.

**Physical items** (rare weapons, rare ingredients) saturate transiently — they leave circulation via junk, destruction, or consumption — withdrawal to a private wallet does not remove them from circulation. The target is a ratio against active player count: as the player base grows, more rare items enter the game.

#### Saturation Snapshot (hourly)

The `NFTSaturationService` runs hourly and writes a row per tracked item to `SaturationSnapshot`:

```
# Knowledge items (scrolls, recipes) — implemented
known_by = count of active players who have learned spell/recipe X
unlearned_copies = count of scroll/recipe X in player hands (CHARACTER + ACCOUNT in NFTGameState)
eligible_players = active players (7d) who have the requisite skill/school mastery to learn it
saturation = (known_by + unlearned_copies) / eligible_players

# Rare NFT items — framework in place, spawn algo still pending rarity_divisor config
in_circulation = current count of item type in CHARACTER/ACCOUNT/SPAWNED locations
saturation = 0.0  (placeholder until rarity_divisor is wired up)
```

Knowledge saturation counts both **learned knowledge** (permanent, from `db.spellbook`/`db.recipe_book`) and **unlearned copies in player hands** (scrolls/recipes sitting in inventory or bank). This prevents flooding the game with scrolls that are already piling up unused — a scroll in someone's bank is supply in the pipeline.

Updated hourly so a scroll spawned in hour N is counted as `unlearned_copies` in the hour N+1 snapshot, closing the gap naturally. This self-corrects even for tiny gaps (e.g. 1 new GM evocation player).

#### Deterministic Push Spawn (no random loot rolls)

**Compliance requirement:** FCM's loot spawning is **fully deterministic**. There are no `rand()`, no flat % drop chances, no dice rolls to decide whether a mob drops something. This is a deliberate design constraint driven by gambling law compliance in several jurisdictions — any chance-based loot system would classify parts of the game as gambling. Every scroll, recipe, and rare item that enters the world comes from a calculated budget based on current state, not from a probabilistic roll.

**Hourly push cycle, not per-kill pulls.** The `UnifiedSpawnScript` ticks every hour and runs calculators and distributors for every entry in `SPAWN_CONFIG`:

1. **Calculator** computes an integer budget for each item type.
   - **Knowledge (scrolls/recipes):** `budget = max(0, eligible_players - known_by - unlearned_copies)`. The budget is exactly the number of players who still need this scroll or recipe. If every eligible player already knows it or holds an unlearned copy, budget is 0 — nothing spawns.
   - **Resources and gold:** budget driven by AMM price targets and consumption rates (see resource/gold calculators).
   - **Rare NFTs:** framework in place using `RareNFTCalculator`, driven by a rarity target once the `rarity_divisor` config is finalised.

2. **Distributor** places the calculated amount onto targets tagged to receive that category (`spawn_scrolls`, `spawn_recipes`, `spawn_rare_nft`, etc.). Each target has a per-tier maximum (`spawn_scrolls_max`, etc.) to cap headroom. Items are placed on mobs, containers, or rooms up to the headroom limit. Any surplus budget that can't be placed (no targets with headroom) is dropped for the cycle.

3. **Quest debt** allows the quest system to pre-reserve part of a cycle's budget for quest rewards via `allocate_quest_reward()`. This lets quests guarantee specific rewards without breaking the deterministic budget model.

When a player kills a mob and finds a scroll, the scroll is there because the spawn system **already placed it** on the mob during a previous hourly cycle. The kill itself doesn't roll dice — it just reveals loot that was deterministically positioned.

**Why this works economically:** The gap-based formula means the system naturally distributes scrolls to meet demand. If 10 players unlock a new spell school, the next cycle spawns up to 10 scrolls for that spell across tagged mobs. Once absorbed into unlearned or learned inventory, the gap closes and spawning slows. No over-spawning, no under-spawning, no RNG tuning required.

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
| Proxy token pools | NFT item prices | Seed liquidity sets initial price. Market activity drives price from there. |

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
| **External AMM manipulation** | Proxy tokens are closed-loop — only vault holds them, no external wallets can interfere |
| **Rounding exploitation** | Every trade pays rounding dust — there's no way to trade without paying the spread |
| **Bot farming** | Per-node cooldowns, diminishing returns, captcha-like interactions (future) |

**Worst case:** temporary price dislocation that corrects over hours/days. The AMM's constant product formula means large dislocations require exponentially more capital to sustain.

---

## Item Classification — What's Tradeable Where

| Item Category | AMM Tradeable? | Where Sold | Notes |
|---|---|---|---|
| Raw resources (wheat, ore, wood, etc.) | Yes — resource AMM | Resource shopkeepers | Tier 1 |
| Processed resources (flour, ingots, timber, etc.) | Yes — resource AMM | Resource shopkeepers | Tier 1 |
| Basic/Skilled equipment (weapons, armor) | Yes — proxy token AMM | Equipment shopkeepers | Tier 2, full durability only |
| Watery/Weak potions (BASIC/SKILLED mastery) | Yes — proxy token AMM | Potion shopkeepers | Tier 2, per-quality-tier proxy tokens |
| Some Expert equipment | Yes — proxy token AMM | Equipment shopkeepers | Edge of Tier 2 |
| Standard/Potent/Ascendant potions (EXPERT+) | **No** | Player-to-player trade only | Crafted by skilled alchemists |
| Master/GM tier equipment | **No** | Player-to-player trade only — no external market is made | Tier 3 — player-driven |
| Enchanted weapons (gem insets) | **No** | Player-to-player trade only — no external market is made | Bespoke names, unique combos |
| Deterministic enchanted wearables (rings, amulets) | **Yes** — own proxy token | Equipment shopkeepers | Standardised output = fungible |
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

- Enchanted wearables CAN get their own proxy token AMM pools
- Price naturally settles at: base item cost + material cost + enchanting fee + margin
- If materials get expensive, the enchanted item's AMM price rises to match

### Weapons — Gem Insets (Enchanter + Jeweller)

> **Status: partially implemented — needs redesign work.** The original implementation used runtime `d100` rolls for gem enchanting, which is non-compliant with FCM's gambling law constraints (no random chance-based mechanics can decide the value a player receives for consideration paid). The enchanting mechanic is being redesigned around the pre-disclosure + slot consumption model described below. Gem tables, enchant recipes, and the inset command (`cmd_inset`) exist in code but the slot-based outcome queue and commit flow need to be built.

Two distinct crafting roles combine to produce enchanted weapons:

- **Enchanter** — consumes a raw gem to enchant it. The outcome is **pre-rolled and disclosed before the player commits** (see flow below). The player always knows what they are getting before paying. The resulting enchanted gem has a known, inspectable effect.
- **Jeweller** — insets an enchanted gem into a weapon. The inset is a deliberate choice — the player can inspect the gem's effects before committing. The inset consumes the gem and the fee.

Because there are potentially hundreds of weapon types and scores of possible gem enchants, every weapon+gem combination produces a functionally unique item with a bespoke name. These cannot be standardised into proxy token AMM pools — each one is one-of-a-kind and must be sold player-to-player.

### Compliance-Driven Design: Pre-Disclosure with Slot Consumption

**Why not just roll dice?** FCM does not use runtime randomness to determine the value a player receives for consideration paid. Runtime `rand()` or `d100` rolls at the point of purchase create an "uncertain future event" — which is one of the legal elements of gambling in several jurisdictions. To stay clearly on the non-gambling side of the line, every transaction must be deterministic from the player's perspective at the moment they pay.

**The model:** per `(gem_type, mastery_level)` combination, the system maintains a single "next available" enchantment slot. Each slot holds one pre-rolled outcome waiting to be claimed. Players query the slot to see what they would get, then either commit or walk away. The slot advances only when consumed — never on rejection, never on timeout, never on command spam.

#### Data model

```
EnchantmentSlot
├── table_key         "ruby_skilled", "sapphire_expert", etc.
├── slot_number       monotonically increasing integer, for display
├── current_outcome   the rolled result waiting to be claimed
└── rolled_at         timestamp of when this outcome was generated
```

One row per `(gem_type, mastery_level)` combination. At bootstrap, each row is seeded with `slot_number=1` and a freshly rolled outcome. Every slot has an outcome ready from day one — players never see an empty slot.

#### Query flow (`enchant <gem>`)

```
Player types: enchant ruby

System reads the ruby_skilled slot (pure read, no DB write):

    "The next ruby enchant [SKILLED] is enchant #312.
     It will have these effects:
       +1 perception bonus
       +1 stealth bonus
       Restriction: Thief class only

     If another player accepts this before you, it will become unavailable.
     Continue [Y]/N?"
```

**Nothing is modified.** The query is idempotent. Ten players can run `enchant ruby` back to back and all ten will see slot #312 until one of them commits.

#### Commit flow (player types Y)

The commit happens inside a database transaction using row-level locking:

```
BEGIN TRANSACTION
    SELECT EnchantmentSlot FOR UPDATE WHERE table_key = "ruby_skilled"
        ← lock acquired; concurrent commits block here

    IF slot.slot_number == remembered_slot_number:
        # ── Race won: slot is still what the player saw ──
        apply outcome to the gem
        consume gem + pay fee
        slot.slot_number += 1
        slot.current_outcome = roll_next_outcome()
        slot.rolled_at = now()
        COMMIT
        Display: "Your ruby is now enchanted with: ..."

    ELSE:
        # ── Race lost: another player consumed #312 first ──
        new_slot_number = slot.slot_number
        new_outcome = slot.current_outcome
        COMMIT (no changes written)
        Display: "Someone else claimed ruby enchant #312 just before you.

                  The next ruby enchant [SKILLED] is now #313.
                  Effects: ...
                  If another player accepts this before you, it will
                  become unavailable.
                  Continue [Y]/N?"
        (re-prompt with the new slot number)
```

`SELECT ... FOR UPDATE` (Django's `select_for_update()`) guarantees only one commit transaction can be "inside" a given slot at a time. The second transaction waits for the first to finish, then reads the updated state. The `slot_number` check acts as a tiebreaker so the race loser knows to re-prompt with the new slot.

#### Two rules, two purposes

The design has two rules that are often conflated but serve different goals:

1. **Pre-disclosure before commit** — *this is the compliance rule.* The player sees the outcome before paying, so there is no uncertain future event at the point of purchase. Legally, this is what removes the mechanic from the gambling definition.

2. **Slot persists until consumed** — *this is the anti-abuse rule.* Without it, players could spam `enchant ruby` and reject repeatedly, forcing the system to keep rolling until they saw an outcome they liked. Compliance would still hold (they'd know every outcome before committing), but the market tension around rare outcomes would evaporate. Locking the slot until consumption preserves the intended game design — players must burn through undesirable slots or wait for someone else to consume them.

#### Emergent market dynamics

Because the slot advances only on consumption, and the next slot's outcome is only generated at the moment of consumption, the system creates natural market tension:

- **Waiting strategy:** a player sees slot #312 is a mediocre outcome, walks away, hoping someone else consumes it so they can see #313.
- **Burning strategy:** a player really wants a rare outcome, so they enchant mediocre gems in sequence (accepting each one) to advance the counter and reveal what comes next.
- **Competitive claim:** when a rare outcome is sitting in the slot, any player with a gem can race to claim it — creating brief windows of intense demand for raw gems.

None of these are gambling behaviours — every transaction is fully disclosed before payment.

### The Economic Loop

1. **Base weapon** — commodity-priced via proxy token AMM
2. **Raw gems** — commodity-priced via resource AMM (rare drop, so price reflects scarcity)
3. **Gem enchanting** — player views the next slot's pre-disclosed outcome, decides to commit or walk away; if committed, the outcome is applied and the next slot rolls
4. **Gem inset** — jeweller inserts enchanted gem into weapon; consumes gem + fee
5. **Enchanted weapon** — bespoke item sold player-to-player at player-determined price
6. **Every step funds the economy** — rounding dust, crafting fees, and inset fees flow to SINK for reward recirculation and operational costs

---

## Crafting Tier Progression & AMM Cutoff

| Mastery Tier | AMM Market? | Economic Character |
|---|---|---|
| Basic | Yes | Commodity — high volume, thin margins, accessible to all |
| Skilled | Yes | Commodity — same items in higher zones, wider variety |
| Expert | Some | Transition zone — common expert items are AMM, rare ones are auction |
| Master | **No** | Prestige — player-driven prices, reputation matters |
| Grandmaster | **No** | Ultra-prestige — name your price, limited supply |

**Potions follow this model explicitly.** Each potion has 5 quality tiers (Watery/Weak/Standard/Potent/Ascendant) mapped 1:1 to mastery levels. Each tier is a separate `NFTItemType` row with its own proxy token. Currently only Watery (BASIC) and Weak (SKILLED) have proxy tokens — higher tiers are player-traded only. Adding proxy tokens to higher tiers later is a data-only change (no code changes needed). Shopkeepers list whichever tiers suit their area (e.g. beginner area stocks Watery only).

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
| **Enchanter/Speculator** | Enchant gems for weapon inset | No runtime rolls — each outcome is pre-disclosed before commit. Speculation lives in *slot position* (burn through mediocre slots to reach a rare one, or wait for others to consume them) rather than in dice-based chance |
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

1. **Every trade has economic purpose:** every in-game trade generates rounding dust that flows to SINK (deflationary + reward recirculation). Withdrawal to a private wallet does not reduce supply — assets remain player-owned and in circulation. Note: game-item AMM pools are created with **0% LP provider fees** by default, so on-chain trades against those pools do not generate revenue for the vault. If a third party adds liquidity and successfully votes to raise the fee, that could change — but this is not the default behaviour, and the game does not rely on AMM fees as a revenue source.
2. **Ceil buys / floor sells:** player always pays more than AMM cost (buys) or receives less than AMM yield (sells). Margin >= 0 on every trade, always.
3. **Proxy tokens are closed-loop:** only the vault trades them. No external manipulation possible.
4. **RESERVE reconciliation:** `RESERVE + SPAWNED + ACCOUNT + CHARACTER + SINK = vault on-chain balance` — must always hold for every fungible currency. NFT items: `RESERVE + SPAWNED + ACCOUNT + CHARACTER + ONCHAIN = total minted NFTs`. AUCTION is a future potential feature (auction house not yet built) — direct player-to-player trade is the current mechanism. Once AUCTION is live, add it to the NFT equation: `RESERVE + SPAWNED + ACCOUNT + CHARACTER + ONCHAIN + AUCTION = total minted NFTs`.
5. **Consumption routing completeness:** every gold fee and resource consumption must use `return_gold_to_sink()` / `return_resource_to_sink()`. No untracked sinks. AMM dust is routed RESERVE → SINK automatically.
6. **Cleanup vs consumption distinction:** cleanup (corpse decay, dungeon teardown, world rebuild) uses `return_*_to_reserve()`. Consumption (fees, crafting, eating) uses `return_*_to_sink()`.
7. **Integer player amounts:** players only ever see and transact in whole numbers. Decimals exist only in RESERVE ↔ AMM accounting.

---

## Backlog: Reserve Capacity Monitoring

**Not yet built.** Need an early warning system that monitors consumption rate against remaining reserves for all asset types — FCMGold, game resources, PGold, item proxy tokens, and blank NFTs. If player growth outpaces supply, the game could run out of tokens to execute AMM swaps, honour exports, or spawn new items. The system should project time-to-depletion from hourly velocity data and alert before reserves hit critical levels. See [TELEMETRY.md](TELEMETRY.md) for the snapshot data this would consume.
