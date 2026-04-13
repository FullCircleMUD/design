# UNIFIED_ITEM_SPAWN_SYSTEM.md — Calculator + Distributor Architecture

> Unified design document for the spawn framework that handles resources, gold, and knowledge NFTs through shared infrastructure. For the economic model, see **ECONOMY.md**.

---

## Purpose

The game needs to spawn resources, gold, and knowledge items (spell scrolls, recipe scrolls) into the world via rooms, mobs, and containers. Each asset type has different economic constraints governing *how much* to spawn, but the mechanics of *where to place it* share common infrastructure: pool all eligible targets, allocate proportionally by headroom, drip-feed over the hour, and support quest reward deductions.

This document describes a two-system architecture that separates these concerns:

1. **Calculator** — determines the hourly budget for each spawnable item (the "how much")
2. **Distributor** — places that budget into the game world across eligible targets (the "where and when")

---

## Architecture Overview

```
┌───────────────────────────────────────────────────────────────┐
│                         Spawn Service                          │
│                                                                │
│  ┌───────────────────────┐    ┌─────────────────────────────┐  │
│  │      Calculators       │    │       Distributors           │  │
│  │                        │    │                              │  │
│  │  ResourceCalculator   ─┼───►│  FungibleDistributor         │  │
│  │  GoldCalculator       ─┼───►│  (resources + gold)          │  │
│  │                        │    │                              │  │
│  │  KnowledgeCalculator  ─┼───►│  NFTDistributor              │  │
│  │  RareNFTCalculator    ─┼───►│  (scrolls + rare items)      │  │
│  │                        │    │                              │  │
│  └───────────────────────┘    └─────────────────────────────┘  │
│                                          │                      │
│                       ┌─────────────────┼──────────┐           │
│                       ▼                 ▼          ▼           │
│                 Tagged Rooms    Tagged Mobs   Tagged            │
│                                              Containers        │
└───────────────────────────────────────────────────────────────┘
```

**Calculator** is stateless — reads telemetry and config, returns a number. Four calculator variants implement different economic formulas but expose the same interface.

**Distributor** is stateful — holds the mutable hourly budget per item, manages drip-feed scheduling, and exposes the budget to the quest reward system. Two distributor variants handle the different placement mechanics (fungible quantities vs NFT tokens). Both use the same tag-driven target pooling — all target types (rooms, mobs, containers) are treated as a single pool.

---

## Calculators

### Interface

Every calculator is constructed with the full spawn config and implements:

```python
calc = ResourceCalculator(config=SPAWN_CONFIG)

calc.calculate(item_type: str, type_key: str | int, **overrides) → int
```

| Parameter | Purpose |
|-----------|---------|
| `item_type` | Namespace — `"resource"`, `"gold"`, `"knowledge"`, or `"rare_nft"`. Prevents key collisions |
| `type_key` | Identifier within the namespace — resource_id (int), `"gold"`, or item_type_name (str) |
| `**overrides` | Optional per-call config overrides. Merged on top of the stored config for this call only. Used almost exclusively in tests and admin tuning commands — normal operation passes no overrides |

The config is loaded once at construction from `SPAWN_CONFIG`. Each call looks up the entry for `(item_type, type_key)` and applies any overrides on top. The return value is always an integer — the number of units to spawn this hour.

### ResourceCalculator

For raw gathering resources. Two-factor formula:

```
budget = consumption_rate × price_modifier
```

| Factor | Source | Behaviour |
|--------|--------|-----------|
| Consumption rate | 24h rolling average from `ResourceSnapshot.consumed_1h` | Baseline — how much players actually consume. Cold start falls back to `default_spawn_rate` |
| Price modifier | AMM buy price vs configured `[target_price_low, target_price_high]` band | Price too high → spawn more. Too low → spawn less. Within band → 1.0 |

Consumption captures demand. The AMM price captures market conditions. Together they form a self-correcting loop: price rises → spawn more → supply increases → price drops → spawn normalises. No manual tuning of supply targets required.

#### What Gets Spawned

Only **raw gathering resources** — things players harvest or loot directly:

| Category | Examples | Spawn Target |
|---|---|---|
| Agricultural | Wheat, Cotton | Rooms |
| Forestry | Wood | Rooms |
| Mining | Iron Ore, Copper Ore, Tin Ore, Silver Ore, etc. | Rooms |
| Alchemy herbs | Windroot, Bloodmoss, Arcane Dust, etc. | Rooms |
| Hunting | Hide | Mobs |

**Not spawned by this system:**
- Processed goods (Flour, Ingots, Timber, Cloth, Leather) — players produce these
- Gold — managed by GoldCalculator (see below)
- NFT items — managed by KnowledgeCalculator / RareNFTCalculator (see below)

#### Factor 1: Consumption Rate (Baseline)

**What it is:** 24-hour rolling average of how much of this resource players actually consumed.

**Why 24 hours?** Smooths across peak/off-peak hours and timezone differences. A server with European and American players has two daily demand spikes — a 1-hour window would generate wildly different spawn rates depending on when it runs. 24 hours captures the full daily cycle.

**Cold start:** On a new server with no consumption history, falls back to `default_spawn_rate` from the config file. This seeds the world with a reasonable amount until real data accumulates (usually within 24-48 hours of player activity).

**What counts as consumption?** Crafting (recipes consume resources as inputs), eating (bread is consumed), processing (ore → ingot consumes the ore), repair costs. NOT in-game trading between players — that's redistribution, not consumption.

#### Factor 2: Price Modifier

**What it is:** How does the current AMM price compare to the target price band?

```
target_band = [floor_price, ceiling_price]  # configured per resource

if price > ceiling:   modifier > 1.0   (price too high → spawn more)
if price < floor:     modifier < 1.0   (price too low → spawn less)
if price in band:     modifier = 1.0   (healthy → spawn at consumption rate)
```

**Two-segment linear interpolation:**
- Below floor: linearly reduce toward `modifier_min` (e.g. 0.25)
- Above ceiling: linearly increase toward `modifier_max` (e.g. 2.0)
- Within band: 1.0 flat

**No AMM pool:** If a resource has no AMM pool (shouldn't happen in steady state), modifier defaults to 1.0.

**Self-correcting loop:** Price rises → spawn more → supply increases → price drops → spawn normalises. Price drops → spawn less → supply contracts → players have to buy from others → price recovers.

#### Why Not a Supply Modifier?

An earlier design included a third factor comparing circulating supply per player-hour against a manually configured target. This was intended as an anti-hoarding defence. It was removed because:

- **Consumption rate already captures demand.** If players are consuming more, Factor 1 increases spawning. If less, it decreases.
- **The AMM price already captures market conditions.** Hoarding drives prices up → Factor 2 increases spawning → new supply flows to non-hoarders.
- **Hoarding is self-limiting.** The hoarder pays real cost to maintain their position. The market corrects as new supply enters. As the economy scales, cornering a market becomes impractical.
- **The manual target required guesswork.** `target_supply_per_ph` was a designer-configured constant — a guess at what "healthy" looks like. Two telemetry-driven factors are better than two plus a manual dial.

### GoldCalculator

Gold has a unique constraint: the game maintains a finite gold supply in the RESERVE pool. Gold cannot be spawned beyond what exists in RESERVE — the system enforces scarcity at the infrastructure level. This fundamentally changes the calculation.

```
budget = consumption_rate × buffer × reserve_throttle
```

| Factor | Source | Behaviour |
|--------|--------|-----------|
| Consumption rate | 24h rolling average of gold sunk (from `EconomySnapshot.gold_sinks_1h`) | Baseline — how much gold players are spending per hour |
| Buffer | Configured multiplier (e.g. 1.15) | Spawns slightly more than consumed so players accumulate wealth |
| RESERVE throttle | RESERVE pool balance vs projected burn rate | Prevents draining the spawn pool; linear ramp-down as runway shrinks |

**No price modifier.** Gold is the unit of account — there is no "price of gold in gold."

**No supply modifier.** Gold hoarding is fine — that's player wealth held in their wallets. Unlike resources, gold sitting in wallets doesn't create inflationary pressure on gold itself.

#### The Reserve Throttle

The critical safety valve. Prevents the game from spawning more gold than exists in the RESERVE pool:

```
daily_burn = hourly_budget × 24
runway_days = vault_reserve / daily_burn

if runway_days >= min_runway:   throttle = 1.0    (healthy)
if runway_days < min_runway:    throttle = runway_days / min_runway   (0.0 → 1.0 linear)
```

`min_runway` is configured (e.g. 7 days). If RESERVE has 5 days of gold left at current rates, throttle = 5/7 ≈ 0.71. Spawning slows gradually rather than hitting a cliff.

As gold flows back through sinks and the entity adds gold to the economy via AMM operations, the throttle lifts back toward 1.0.

#### The Buffer and Net Flow

The buffer determines how much players collectively earn beyond what they spend. With buffer = 1.15, players earn 15% more than they consume. However, the 90/10 sink reallocation means only 90% of sunk gold returns to RESERVE — 10% is operational revenue. The steady-state net flow:

```
gold_spawned = consumption × buffer                   (e.g. 1.15 × consumption)
gold_recycled = consumption × 0.9                     (90% of sunk gold back to RESERVE)
net_reserve_drain = gold_spawned - gold_recycled       (0.25 × consumption)
```

This drain is replenished by gold flowing back through sinks and reallocation. If inflow stops, the RESERVE throttle gradually reduces spawning to sustainable levels.

#### Vault Reserve Source

The RESERVE balance comes from `FungibleGameState` where `currency_code = "FCMGold"` and `location = "RESERVE"`. This is the pool of gold available for spawning into the game world.

### KnowledgeCalculator

For spell scrolls and crafting recipe scrolls. These are permanent knowledge — once learned, they never leave the game. The gap-based model spawns exactly the number of copies needed to fill the gap between eligible players and those who already have the knowledge.

```
budget = max(0, eligible_players - known_by - unlearned_copies)
```

| Component | Source | Definition |
|-----------|--------|-----------|
| `eligible_players` | Hourly `SaturationSnapshot` | Players with the requisite skill/school mastery to learn this scroll/recipe. Mastery-filtered — a GM evocation scroll only counts GM evokers |
| `known_by` | Hourly `SaturationSnapshot` | Players who have already learned this spell/recipe (permanent knowledge) |
| `unlearned_copies` | Hourly `SaturationSnapshot` | Scroll/recipe NFTs in CHARACTER or ACCOUNT locations (in player hands but not yet consumed) |

**Per-item budgets, not per-category.** Each individual scroll type (Magic Missile, Fireball, etc.) gets its own budget from the calculator. The distributor places each type independently onto tagged mobs.

#### Why Gap-Based, Not Rate-Based

The previous design used `budget = base_drop_rate × (1 - saturation)` where `base_drop_rate` was a fixed number per tier (e.g. 3/hour for basic). This caused over-spawning when the gap was small:

- 1 new GM evocation player → `3 × (1.0 - 0%) = 3` scrolls/hour × 24 hours = 72 scrolls for 1 player
- Even `1/hour` with a daily snapshot meant 24 copies for a gap of 1

The gap-based formula spawns exactly what's needed. 1 player needs a scroll → budget = 1. 50 players need it → budget = 50. Zero waste, zero flooding.

#### The Problem: Permanent Knowledge = Permanent Saturation

With a weapon, it can be exported, junked, lost to decay. Supply fluctuates. But knowledge, once learned, never leaves the game. If 80% of active players have already learned "Magic Missile", more Magic Missile scrolls have almost zero value. They'll sit in banks forever.

The gap-based model handles this naturally:
- **Early game:** All gaps are large (nobody knows anything) → high budgets → scrolls appear on mobs rapidly
- **Mid-game:** Common spells mostly known → small gaps → few scrolls spawned → loot stays relevant
- **Late-game:** Most knowledge widely known → near-zero gaps → finding a scroll is notable
- **Fully saturated (gap = 0):** Spawning stops entirely. New eligible players create a gap and spawning resumes

#### Saturation Formula

Still tracked for monitoring/display purposes:

```
saturation = (known_by + unlearned_copies) / eligible_players
```

But the spawn budget uses the raw gap, not the percentage.

#### Hourly Saturation Snapshot

The saturation service runs **hourly** (60 seconds after the telemetry snapshot) so the spawn system sees up-to-date knowledge gaps. A scroll spawned in hour 1 is counted as an `unlearned_copy` in the hour 2 snapshot, closing the gap and preventing further spawning.

The `NFTSaturationScript` calculates for every tracked scroll and recipe:

1. Determine `eligible_players` — characters with the requisite skill/school mastery
2. Count `known_by` from `db.spellbook` / `db.recipe_book` across active characters
3. Count `unlearned_copies` from `NFTGameState` in CHARACTER + ACCOUNT locations
4. Write `SaturationSnapshot` row with gap components and saturation ratio

**Why hourly?** With a daily snapshot, a gap of 1 would trigger 24 spawns (one per hourly spawn tick) before the snapshot refreshed. Hourly snapshots ensure the gap closes within one cycle. The compute cost is modest — ~100 DB writes per cycle with no external API calls, lighter than the telemetry service.

#### Self-Correcting Feedback Loop

1. New player gains GM evocation → gap = 1 for GM evocation scrolls
2. Saturation snapshot (hour N) records gap = 1
3. Spawn cycle (hour N, +120s) spawns 1 scroll onto a tagged mob
4. Saturation snapshot (hour N+1) counts the spawned scroll as unlearned → gap = 0
5. Spawn cycle (hour N+1) → budget = 0. No more scrolls spawned.

The system converges in exactly 1 cycle for any gap size.

**Cold start:** On day 1, saturation is ~0% for any item with eligible players — those items get their full base rate. Scrolls enter the world quickly for skills players have actually trained. As knowledge spreads, the system naturally throttles. Items with zero eligible players get zero budget — no scrolls spawn for skills nobody has learned yet.

#### Spell Scrolls vs Recipe Scrolls

The saturation calculation is identical. The difference is in distribution:

| | Spell Scrolls | Recipe Scrolls |
|---|---|---|
| **Who uses them** | Mages (and clerics via Holy scrolls) | Any class with the relevant crafting skill |
| **Mastery-gated** | Yes — scrolls are tiered BASIC through GM | Yes — recipe mastery requirements |
| **Mob sources** | Common mobs (low-tier scrolls), dungeon bosses (high-tier) | Trainers sell BASIC recipes; mobs drop SKILLED+ |
| **Trainer availability** | Trainers sell some BASIC spells directly | Trainers sell BASIC recipes; higher tiers must be found |

**Design intent:** Basic spells and basic recipes should be accessible (trainer-sold + common drops). Skilled and above should require earning — either finding the scroll/recipe or training to higher mastery.

#### Enchanting Exception

Enchanting recipes are **never dropped** as scrolls — they are auto-granted when the enchanter reaches the required mastery tier. This reflects the flavour that enchanting knowledge is passed directly from master to apprentice, not written down in recoverable form.

#### Knowledge Config Generation

Knowledge entries in `SPAWN_CONFIG` are generated dynamically at `SpawnService` init via `populate_knowledge_config()`. This function reads the spell registry (`SPELL_REGISTRY`) and recipe registry (`RECIPES`) and creates one config entry per registered scroll/recipe with `base_drop_rate` and `tier` derived from the item's mastery requirement. Mastery-to-tier mapping: `_MASTERY_TO_TIER` converts mastery integers to tier strings (basic/skilled/expert/master/gm).

### RareNFTCalculator

For rare and legendary non-craftable items — population-gated boss drops, ultra-rares, unique items. A placeholder implementation exists that returns a fixed `spawn_rate` from config (default 1).

```python
class RareNFTCalculator(BaseCalculator):
    def calculate(self, item_type, type_key, **overrides):
        cfg = self.get_item_config(item_type, type_key, **overrides)
        return max(0, int(cfg.get("spawn_rate", 1)))
```

#### Rare Item Saturation (Future)

When fully implemented, rare items will use population-scaled saturation:

```
saturation = current_in_circulation / (active_players_7d / rarity_divisor)
```

| Component | Definition |
|---|---|
| `current_in_circulation` | Count of this item type in CHARACTER + ACCOUNT + SPAWNED locations (`NFTGameState`) |
| `active_players_7d` | Distinct characters with sessions in the last 7 days |
| `rarity_divisor` | Configured per item — "how many active players per one copy of this item" |

**Examples:**
- `rarity_divisor = 100`: "1 copy per 100 active players" → on a 400-player server, target = 4 copies
- `rarity_divisor = 500`: "1 copy per 500 active players" → ultra-rare, most servers will never have more than 1-2

**Saturation > 1.0** = oversaturated (too many copies for current population → drop rate falls to near zero)
**Saturation < 1.0** = undersaturated (not enough copies → drop rate increases)

#### Population-Gated Ultra-Rares

Some items are intentionally limited to a strict ratio — e.g., "at most 1 in existence per 500 active players." On a small server, this may never drop at all. On a large server, it might exist in a handful of copies — each one known and traded.

**Design principles:**
- Ultra-rares should be *known* by the community ("who has the Blade of the Fallen King?")
- Their value is partly social — reputation, proof of achievement
- They should not drop when someone already has one unless population has grown enough to justify another copy
- A new copy can only drop if the item is consumed/destroyed OR the player population grows enough to justify another copy — withdrawing to a private wallet does not free up the slot

**Why population-scaled targets?** A static "max 2 copies ever" is broken on a growing server (everybody wants it, almost nobody has it) and broken on a shrinking server (2 copies for 10 active players is over-saturated). Population-scaling means the item economy naturally scales with the game's health.

Detailed design for specific rare item rules will be developed when these items are ready for implementation.

---

## Distributors

Both distributors share the same tag-driven target pooling, proportional allocation, alternating direction, drip-feed scheduling, and mutable budget state described in the sections below. They differ only in placement mechanics:

| | FungibleDistributor | NFTDistributor |
|---|---|---|
| **Places** | Resources and gold (quantities) | NFT items (individual tokens) |
| **Placement call** | `receive_resource_from_reserve()` / `receive_gold_from_reserve()` (harvest rooms: increment `resource_count`) | `assign_to_blank_token()` + `spawn_into()` |
| **Budget unit** | Integer count of fungible units | Integer count of individual items |
| **Target tags** | `spawn_resources`, `spawn_gold` | `spawn_scrolls`, `spawn_recipes`, `spawn_nfts` |
| **Headroom** | `spawn_<cat>_max[key] - current` (harvest rooms: `spawn_resources_max[key] - resource_count`) | `spawn_<cat>_max[key] - current` (at-or-below tier filtering for scrolls/recipes) |

NFTDistributor has three subclasses: **ScrollDistributor** (tag `spawn_scrolls`, category `scrolls`), **RecipeDistributor** (tag `spawn_recipes`, category `recipes`), and **RareNFTDistributor** (tag `spawn_nfts`, category `nfts`). ScrollDistributor and RecipeDistributor use at-or-below tier filtering; RareNFTDistributor uses exact-match on `spawn_nfts_max` keys.

---

## Shared Distribution Mechanics

### Tag-Driven Target Pooling

The distributor does not distinguish between rooms, mobs, and containers. All targets for a given category are discovered via a single tag and treated as a single pool:

```
Query: everything tagged spawn_resources
Filter: targets where spawn_resources_max contains resource_id 8 (hide)
Result: Wolf (headroom 1), Wolf (headroom 1), Harvest Room (headroom 15), Chest (headroom 10)
Total headroom: 27
Budget for this tick: 20
```

**Where an item spawns is controlled entirely by which targets have the tag and what their `spawn_<category>_max` attribute allows.** If wheat should only appear in rooms, only rooms get the `spawn_resources` tag with wheat in their `spawn_resources_max` dict. No configured shares, no channel split math — tags and capacity attributes are the single source of truth.

### Proportional Allocation by Headroom

Each tick distributes its budget across the target pool proportionally by headroom (capacity minus current held):

```
share = budget × (target_headroom / total_headroom)
```

Allocated amounts are floored to integers, **except** where the proportional share is less than 1 — in that case it is rounded up to 1 (minimum allocation). The remainder after rounding is distributed one-at-a-time starting from the end of the sorted list that has priority this tick (see alternating direction below). Allocation stops when the budget is exhausted.

### Alternating Direction

Proportional allocation with integer rounding creates a systematic bias: targets at the start of the sort order absorb rounding remainders, targets at the end may get slightly less than their share. With many low-cap targets (mobs with headroom 1-2), rounding up compounds — potentially consuming budget before high-cap targets (treasure chests) get their share.

The solution: **alternate the sort direction every tick.**

```
Odd ticks  (high→low):  high-headroom targets (chests, rich veins) get priority
Even ticks (low→high):  low-headroom targets (mobs, starter rooms) get priority
```

Over two ticks, both ends of the spectrum get fair treatment:

- **High→low tick:** The boss's treasure chest fills up. Some rabbits miss out.
- **Low→high tick:** Every rabbit gets 1. The chest gets whatever's left.
- **Net effect over the hour:** Both early-game players (killing rabbits) and late-game players (looting treasure hoards) find loot consistently.

When budget is plentiful relative to total headroom, the direction doesn't matter — everyone gets their proportional share regardless. The alternation only matters when budget is tight and rounding effects compound.

### Per-Tick Algorithm

At each tick:

1. **Calculate effective budget:** `tick_amount + surplus_bank - quest_debt`
2. **Query all tagged targets** (late-bound — targets may have died, been looted, or filled since scheduling)
3. **Filter by headroom:** only targets where `current < max`
4. **Sort by headroom** (direction alternates each tick)
5. **Allocate proportionally:** `floor(budget × headroom / total_headroom)` per target
6. **Distribute remainder:** 1 each to targets in sort order until exhausted
7. **Place items** on each target (via distributor-specific placement — see below)
8. **Bank surplus** if budget exceeds total headroom
9. **Final tick:** log and drop any remaining surplus

### Late-Bound Target Selection

All targets are queried at tick time, not at scheduling time. Mobs die and respawn. Containers get looted and emptied. Rooms get harvested. Pre-allocating to specific targets would result in resources vanishing when those targets change state between ticks. Late-binding handles all volatility naturally.

### Surplus Banking

When the budget exceeds total headroom across all targets, the surplus is banked for the next tick. This happens when:
- All targets are near capacity (the world is well-stocked)
- A player is actively farming and consuming faster than the drip-feed (surplus accumulates, then gets placed once they move on)

Surplus is logged for telemetry. On the final tick of the hour, remaining surplus is dropped and logged. Persistent surplus signals that more targets are needed (more mobs, more containers) or per-target caps should be increased.

### Placement

**FungibleDistributor** calls `FungibleInventoryMixin` methods on whatever the target is — the mixin handles the rest:

```python
target.receive_resource_from_reserve(resource_id, amount)   # resources
target.receive_gold_from_reserve(amount)                     # gold
```

These methods sync with the blockchain mirror DB (`FungibleGameState` RESERVE → SPAWNED transition). The distributor doesn't know or care whether the target is a room, a mob, or a chest.

**Harvest room exception:** Harvest rooms don't use `FungibleInventoryMixin` — they store resources in their own `db.resource_count` attribute (see § Headroom Calculation). The distributor increments `resource_count` directly instead of calling `receive_resource_from_reserve()`. The blockchain state transition is the same (RESERVE → SPAWNED), just applied via the room's own method.

**NFTDistributor** follows a two-step process for each item placed:

```python
token_id = BaseNFTItem.assign_to_blank_token(item_type_name)   # allocate blank token, assign type
BaseNFTItem.spawn_into(token_id, target_location)               # create game object, place it
```

1. **Allocate** — picks the next blank pre-minted XRPL NFToken from RESERVE, assigns it to the item type (e.g. "scroll_magic_missile"). Concurrency-safe via `select_for_update()`.
2. **Instantiate** — creates the Evennia game object from the item type's prototype (attributes, stats, etc.) and moves it into the target's contents (mob inventory, chest, room).

No on-chain transaction at spawn time — the blank tokens were pre-minted in bulk. This keeps placement fast with no blockchain latency.

**No loot tag needed.** Items placed by NFTDistributor are real `BaseNFTItem` instances. Humanoid mobs can also carry equipment — weapons and armour they wield or wear for combat stats — but those are `MobItem` subclasses (`MobDagger`, `MobLongsword`, etc.), a separate class hierarchy that never enters the NFT supply. On death, `CombatMob._create_corpse()` uses `isinstance(obj, MobItem)` as the discriminator: MobItem instances are deleted, everything else (real NFT loot, fungibles) transfers to the corpse. The class split means there is no need for a per-item `loot` tag — the item's own type tells you whether it's player loot or mob combat gear. See [NPC_MOB_ARCHITECTURE.md](NPC_MOB_ARCHITECTURE.md) § Loot on Death and [INVENTORY_EQUIPMENT.md](INVENTORY_EQUIPMENT.md) § MobItem Hierarchy for the full class split.

**NFT type name resolution:** The distributor resolves a config `type_key` (e.g. `"scroll_magic_missile"`) to the `NFTItemType.name` needed by `assign_to_blank_token()` via `_resolve_nft_item_type_name()`. This looks up the `prototype_key` from the config entry, then finds the matching `NFTItemType` record.

### Headroom Calculation — `get_current_count()`

The distributor needs a uniform way to answer "how much of X does this target currently hold?" for headroom calculation (`spawn_<cat>_max[key] - current`). A single method handles all categories with internal branching:

```python
def get_current_count(target, category, key):
    if category == "resources":
        if hasattr(target.db, 'resource_count'):
            # Harvest room — single resource, built-in count
            return target.db.resource_count or 0
        # Mob/container — fungible inventory
        return target.db.resources.get(key, 0)

    elif category == "gold":
        return target.db.gold or 0

    elif category in ("scrolls", "recipes", "nfts"):
        # Count matching NFTs in contents + equipped
        return count_nfts(target, category, key)
```

**The harvest room exception.** Resource harvesting rooms store their count in `db.resource_count` (a single int), not in `db.resources` (the `FungibleInventoryMixin` dict). This is intentional — harvest rooms model resources embedded in the environment (ore in the wall, wheat in the field), not items sitting on the ground. Players must use the harvest command (`mine`, `chop`, `harvest`, etc.) to extract them. If harvest rooms used `FungibleInventoryMixin`, players could pick resources up directly, bypassing the harvest mechanic. Keeping the room's own `resource_count` preserves this gameplay while the distributor handles it with a simple branch.

**NFT current count** is derived by inspecting the target's `contents` (Evennia object contents) plus any equipped items. An equipped weapon still consumes the slot — Jupiter wielding his Lightning Bolt means the `spawn_nfts_max` slot is full. Equipment is handled by a hook on the mob or item (`at_object_receive()` or `at_post_move()`), not by the distributor — the distributor's job ends at placement.

### NFT Per-Tier Capacity and At-Or-Below Filtering

Scrolls and recipes use **per-tier capacity** with an **at-or-below** rule. Each tier slot accepts items of that tier or any lower tier:

```
Tier hierarchy: basic < skilled < expert < master < gm
```

A `gm: 1` slot is the **most flexible** — it can hold any scroll from basic to gm. A `basic: 1` slot is the **most restrictive** — it only accepts basic scrolls.

```python
# Kobold Chieftain — 1 slot for basic scrolls only
spawn_scrolls_max = {"basic": 1, "skilled": 0, "expert": 0, "master": 0, "gm": 0}

# Dragon Boss — 1 slot for anything up to expert, 1 slot for anything up to gm
spawn_scrolls_max = {"basic": 0, "skilled": 0, "expert": 1, "master": 0, "gm": 1}
```

The Dragon Boss can hold 2 scrolls total. The expert slot might hold a basic, skilled, or expert scroll. The gm slot can hold anything. If a master scroll needs placing, only the gm slot accepts it.

**Placement order — highest tier first.** The distributor places gm scrolls before basic scrolls. This ensures high-tier scrolls land in the only slots that accept them (gm slots), rather than those slots being consumed by lower-tier scrolls that could have fit elsewhere.

**Within a tier — saturation-priority.** When multiple scroll types compete for the same slot, the most undersaturated type is placed first:

```
Processing scrolls for Dragon Boss (expert: 1 open, gm: 1 open):
1. GM scrolls first — Vampiric Touch (12% saturation, budget 1) → placed in gm slot
2. Expert scrolls next — none budgeted this tick, skip
3. Skilled scrolls — Shield (5% saturation, budget 2) → placed in expert slot (skilled ≤ expert)
4. Basic scrolls — Magic Missile (40% saturation, budget 1) → both slots full, try next target
```

**Recipes follow the same model.** `spawn_recipes_max` uses the same tier hierarchy and at-or-below rule.

**Rare NFTs use exact match.** `spawn_nfts_max` keys are `typeclass.prototype` strings, not tiers. No at-or-below filtering — either the target accepts that specific item or it doesn't:

```python
# Jupiter Boss — only mob that can hold Jupiter's Lightning
spawn_nfts_max = {"UniqueWeapon.jupiters_lightning": 1}
```

#### Mob Knowledge Capacity

Scroll and recipe tier capacity is set explicitly per mob subclass via `spawn_scrolls_max` / `spawn_recipes_max` AttributeProperties. Builders choose exactly which tiers and how many slots each mob gets — there is no auto-derivation from mob level.

Example from `Kobold` (L2): `spawn_scrolls_max = {"basic": 1}` — one basic-tier scroll slot.
Example from `GnollWarlord` (L6): `spawn_scrolls_max = {"basic": 1, "skilled": 1, "expert": 2}` — four total slots across three tiers.

---

## Drip-Feed Scheduling

The hourly budget is spread across evenly spaced ticks within the hour:

```
ticks = min(budget, MAX_TICKS_PER_HOUR)     # max 12 ticks
interval = 3600 / ticks                      # min 5 min apart
per_tick = budget // ticks                   # base amount per tick
remainder = budget % ticks                   # distributed to first N ticks
```

Each tick fires a `delay()` callback that runs the per-tick algorithm described above.

**Why drip-feed?** Players active throughout the hour always find something. A single hourly dump creates "rush hours" where early arrivals get everything.

---

## Mutable Budget State

The budget state for each spawnable item is accessible to both the drip-feed ticks and the quest reward system:

```python
budget_state = {
    "item_key": "wheat",          # resource_id, "gold", or item_type_name
    "total": 45,                  # total hourly budget from calculator
    "remaining": 32,              # what hasn't been scheduled yet
    "quest_debt": 0,              # deducted from upcoming ticks
    "surplus_bank": 0,            # carried from previous tick (target unavailable)
    "tick_direction": True,       # alternates each tick (True = high→low)
    "spawned_this_hour": 13,      # telemetry: how much actually placed
    "dropped_this_hour": 0,       # telemetry: surplus that couldn't be placed
}
```

This state lives on the spawn service/script instance — not in `delay()` closures — so the quest system can reach it.

---

## Quest Reward Integration

Quests that reward items should not inflate the economy — they redirect items from the spawn budget instead of creating new ones.

### The Simple Case

Quest rewards a small amount (e.g. 5 gold). The next drip tick has 8 gold budgeted:

```
effective_tick = tick_amount - min(tick_amount, quest_debt)
quest_debt -= (tick_amount - effective_tick)

# 8 budgeted, 5 debt → effective = 3, debt = 0
```

The tick distributes 3 gold to targets. The quest's 5 gold was absorbed in one tick.

### The Multi-Tick Case

Quest rewards a large amount (e.g. 30 gold). Each tick has 8 budgeted:

```
Tick 1:  8 budgeted, 30 debt → effective = 0, debt = 22
Tick 2:  8 budgeted, 22 debt → effective = 0, debt = 14
Tick 3:  8 budgeted, 14 debt → effective = 0, debt = 6
Tick 4:  8 budgeted,  6 debt → effective = 2, debt = 0
```

The large quest reward suppresses spawning for ~3.5 ticks. If debt exceeds the remaining hourly budget, it rolls into the next hour's budget.

### The allocate_quest_reward() Method

```
allocate_quest_reward(category, key, amount) → bool
```

1. Adds `amount` to `quest_debt` on the item's budget state
2. Delivers the reward to the player immediately (via `receive_gold_from_reserve()`, `receive_resource_from_reserve()`, or NFT allocation)
3. Returns True if the debt can be absorbed within a reasonable window (e.g. 2 hours of budget), False if the amount is too large (caller decides whether to proceed)

Quest completions register debt automatically via `FCMQuest.complete()`, which calls `_register_quest_debt()` after awarding gold and bread rewards. This uses `get_spawn_service()` to reach the running `SpawnService` singleton — graceful no-op if the service isn't running yet.

### Processed Good Rewards

A quest that rewards bread (a processed good) can't deduct from a bread spawn budget — bread isn't spawned. The quest designer manually specifies the upstream deductions when configuring the quest:

```python
# Quest rewards 5 bread. Bread requires 2 wheat each, so deduct 10 wheat.
allocate_quest_reward("resource", 1, 10)   # 10 wheat (resource_id 1)
```

No automatic recipe lookup — the quest designer knows the inputs and writes the deduction explicitly. This avoids a layer of dynamic recipe-chain resolution that adds complexity without real benefit.

---

## Always-On Spawning

The spawn service runs every hour regardless of whether players are online.

All targets have hard caps (max per room, max per mob, max per container). Spawning into an empty server fills targets to caps — no runaway accumulation. Skipping hours when nobody is online creates a deficit that makes returning players find empty rooms and wait for the pipeline to catch up. Always-on means players return to a stocked world.

---

## Configuration

### Per-Item Spawn Config

Each spawnable item has a config entry. The config only contains calculator parameters — distribution is controlled entirely by tags on targets:

```python
SPAWN_CONFIG = {
    # Resources (keyed by resource_id)
    ("resource", 1): {  # Wheat
        "calculator": "resource",
        "default_spawn_rate": 10,
        "target_price_low": 8,
        "target_price_high": 15,
        "modifier_min": 0.25,
        "modifier_max": 2.0,
    },

    ("resource", 8): {  # Hide
        "calculator": "resource",
        "default_spawn_rate": 8,
        "target_price_low": 10,
        "target_price_high": 20,
        "modifier_min": 0.25,
        "modifier_max": 2.0,
    },

    # Gold
    ("gold", "gold"): {
        "calculator": "gold",
        "default_spawn_rate": 50,
        "buffer": 1.15,
        "min_runway_days": 7,
    },

    # Knowledge NFTs — populated dynamically by populate_knowledge_config()
    # from SPELL_REGISTRY and RECIPES at SpawnService init time.
    # Example entries (auto-generated):
    # ("knowledge", "scroll_magic_missile"): {
    #     "calculator": "knowledge",
    #     "base_drop_rate": 2,
    #     "tier": "basic",
    #     "prototype_key": "spell_scroll_magic_missile",
    # },

    # Rare NFTs (keyed by typeclass.prototype)
    # ("rare_nft", "UniqueWeapon.jupiters_lightning"): {
    #     "calculator": "rare_nft",
    #     "spawn_rate": 1,
    # },
}
```

No channel shares. No max-per-target-type. All distribution behaviour comes from the tags and caps on the targets themselves.

### Per-Target Configuration

Targets declare what they can hold via **tags** and **attributes**. The system uses exactly **5 tags** — one per spawn category — paired with a matching `spawn_<category>_max` attribute. The tag makes the target discoverable; the attribute defines what it accepts and how much.

**Unified tagging standard:**

| Tag | Attribute | Format |
|---|---|---|
| `spawn_resources` | `spawn_resources_max` | `{resource_id: max, ...}` |
| `spawn_gold` | `spawn_gold_max` | `int` |
| `spawn_scrolls` | `spawn_scrolls_max` | `{tier: max, ...}` |
| `spawn_recipes` | `spawn_recipes_max` | `{tier: max, ...}` |
| `spawn_nfts` | `spawn_nfts_max` | `{typeclass.prototype: max, ...}` |

Every tag follows `spawn_<category>`. Every attribute follows `spawn_<category>_max`. No exceptions.

**Examples:**
```python
# Wolf mob — carries 1 hide
# Tags: spawn_resources
spawn_resources_max = {8: 1}

# Kobold Chieftain — carries gold + can drop a basic scroll
# Tags: spawn_gold, spawn_scrolls
spawn_gold_max = 12
spawn_scrolls_max = {"basic": 1, "skilled": 0, "expert": 0, "master": 0, "gm": 0}

# Treasure chest — carries wheat + gold + can contain a basic recipe
# Tags: spawn_resources, spawn_gold, spawn_recipes
spawn_resources_max = {1: 10}
spawn_gold_max = 20
spawn_recipes_max = {"basic": 1, "skilled": 0, "expert": 0, "master": 0, "gm": 0}

# Jupiter Boss — carries gold, high-tier scrolls, and a unique rare drop
# Tags: spawn_gold, spawn_scrolls, spawn_nfts
spawn_gold_max = 15
spawn_scrolls_max = {"basic": 0, "skilled": 0, "expert": 1, "master": 0, "gm": 1}
spawn_nfts_max = {"UniqueWeapon.jupiters_lightning": 1}

# Harvest room — carries wheat only
# Tags: spawn_resources
spawn_resources_max = {1: 20}
```

**5 tags scale forever.** Adding a new resource type, scroll tier, or rare item never creates a new tag — it's a new key in an existing dict attribute. Hundreds of item types, zero tag growth.

**The dicts shape the distribution.** A mob with `spawn_gold_max = 2` and a chest with `spawn_gold_max = 20` — proportional allocation by headroom naturally concentrates gold into the chest. No configured shares needed.

**At-or-below filtering for scrolls/recipes.** A `gm` slot accepts any scroll; a `basic` slot only accepts basic. See § NFT Per-Tier Capacity and At-Or-Below Filtering.

### Tag Registration

Tags and max attributes are registered in `CombatMob.at_object_creation()` based on the mob's AttributeProperty values:
- `loot_resources` dict → `spawn_resources` tag + `spawn_resources_max`
- `loot_gold_max > 0` → `spawn_gold` tag + `spawn_gold_max`
- `scroll_loot_slots > 0` → `spawn_scrolls` tag + `spawn_scrolls_max` (via `_build_tier_max()`)
- `recipe_loot_slots > 0` → `spawn_recipes` tag + `spawn_recipes_max` (via `_build_tier_max()`)

`ZoneSpawnScript._spawn_mob()` re-syncs all tags after spawning to handle spawn rule attribute overrides.

`WorldChest.at_object_creation()` registers `spawn_gold` tag + `spawn_gold_max` when `loot_gold_max > 0`.

---

## NFT Items in the Unified System

### Commodity NFTs — Not Spawned

Commodity NFT items (iron swords, leather armor) are **player-crafted**, not system-spawned. Their supply enters the economy through player crafting activity. These items are not part of the spawn system.

#### Tracker Token AMM (Future)

Common crafted items are functionally identical — one iron sword at full durability equals any other. But they're NFTs, so they can't participate in a standard AMM directly. The solution: issue one XRPL issued currency per common item type as a "tracker token" (`FCMTrkIronSword`, etc.) and set up an on-chain AMM pool for each tracker token vs FCMGold. Only the vault holds these tokens — they're purely a computational pricing mechanism.

**When a player sells an iron sword to a shopkeeper:**
1. Shopkeeper accepts the item (moves to RESERVE in game)
2. Vault sells 1 tracker token to the AMM → receives gold
3. Player paid floor-rounded gold amount

**When a player buys an iron sword from a shopkeeper:**
1. Vault buys 1 tracker token from the AMM → spends gold
2. Item pulled from RESERVE → moved to player
3. Player charged ceil-rounded gold amount

**Tier cutoff:** Basic and Skilled items use Tracker Token AMM. Some Expert items (functionally standard only). Master/GM items are Auction House only. Enchanted weapons are never tracker-token eligible.

**Status:** Not yet implemented. See `ops/ECONOMY_BACKLOG.md` for the full task list.

### Knowledge NFTs — KnowledgeCalculator + NFTDistributor

Spell scrolls and recipe scrolls are fully integrated into the unified system:

- **KnowledgeCalculator** produces an hourly budget per scroll/recipe type based on saturation
- **ScrollDistributor** / **RecipeDistributor** (NFTDistributor subclasses) place those scrolls onto tagged targets via the standard drip-feed
- Targets are discovered via `spawn_scrolls` / `spawn_recipes` tags; `spawn_scrolls_max` / `spawn_recipes_max` dicts define per-tier capacity with at-or-below filtering
- Within a tier, the most undersaturated scroll type is placed first (saturation-priority)
- When a mob dies, placed NFT scrolls transfer to the corpse — players loot them normally. No loot tag is needed: the corpse filter uses `isinstance(obj, MobItem)` to delete mob combat gear while transferring everything else (real NFTs, fungibles) to the corpse

This replaces the previous event-driven design where drop chance was rolled at mob death. Instead, scrolls are pre-placed on living mobs by the distributor. The statistical outcome is identical — undersaturated scrolls appear on more mobs — but with one fewer system to maintain.

**Saturation snapshot** is calculated hourly by `NFTSaturationScript` (60s after telemetry). The KnowledgeCalculator reads the latest snapshot to determine per-item gap-based budgets.

### Rare / Legendary NFTs — RareNFTCalculator + NFTDistributor

Rare and legendary non-craftable items use `RareNFTDistributor` (an NFTDistributor subclass) and differ in two ways:

1. **Exact-match capacity** instead of at-or-below tiers — `spawn_nfts_max = {"UniqueWeapon.jupiters_lightning": 1}` on the Jupiter boss only. The dict key is the specific item, not a tier.
2. **RareNFTCalculator** provides the budget — currently a fixed `spawn_rate` from config, with population-gated rules planned for the future.

The distributor queries the `spawn_nfts` tag, finds all targets with that tag, checks each target's `spawn_nfts_max` for the specific item key, and places if headroom exists. A boss with `spawn_nfts_max = {"UniqueWeapon.jupiters_lightning": 1}` that already holds one has 0 headroom — no second copy can be placed.

### Enchanted Items

Enchanted weapons (gem insets by jewellers) are not a drop — they're **player-crafted**. The loot drop system handles:
- Enchanted gems themselves (rare drops → jeweller uses to inset into weapons)
- Raw components for high-tier crafting

The enchanting *process* consumes gems, the gems may have been drops, the finished weapon is fully player-crafted and sold via Auction House.

---

## What This Replaces

| Current System | Unified Replacement |
|----------------|-------------------|
| `ResourceSpawnService._process_resource()` calculation | `ResourceCalculator.calculate()` |
| `ResourceSpawnService._process_resource()` room distribution | `FungibleDistributor` — unified target pool |
| `ResourceSpawnService.schedule_mob_drip_feed()` | `FungibleDistributor` — unified target pool |
| `ResourceSpawnService._apply_mob_drip()` | `FungibleDistributor._apply_tick()` |
| `RESOURCE_SPAWN_CONFIG` | Unified `SPAWN_CONFIG` |
| `MOB_RESOURCE_SPAWN_CONFIG` | Removed — tags on targets control distribution |
| No gold spawn system | `GoldCalculator` + `FungibleDistributor` |
| Event-driven knowledge item drops (designed, not built) | `KnowledgeCalculator` + `NFTDistributor` |
| Future rare NFT drops | `RareNFTCalculator` + `RareNFTDistributor` |

---

## What This Does NOT Replace

- **Commodity NFT supply** — player crafting. Not system-spawned. (Future: Tracker Token AMM for pricing.)
- **Gold sink/reallocation** — the 90/10 SINK → RESERVE cycle via `ReallocationServiceScript` is unchanged. The gold calculator just reads the resulting RESERVE balance.
- **Saturation snapshot** — `NFTSaturationScript` runs hourly (60s after telemetry, 60s before spawn — see [TELEMETRY.md](TELEMETRY.md) § Scheduled Scripts — Hourly Pipeline) to feed the `KnowledgeCalculator`. The unified system consumes this data, not replaces it.

---

## System Interactions

| System | How It Relates |
|--------|---------------|
| **AMM Pools** | ResourceCalculator reads AMM prices for price modifier |
| **Telemetry** | Calculators read `ResourceSnapshot`, `EconomySnapshot` for consumption baselines |
| **Player Sessions** | GoldCalculator uses player activity data; KnowledgeCalculator uses eligible player count (players with requisite skill/mastery) for saturation denominator |
| **FungibleGameState** | GoldCalculator reads RESERVE balance |
| **NFTSaturationScript** | Hourly saturation snapshot consumed by KnowledgeCalculator |
| **SaturationSnapshot** | Per-item saturation data read by KnowledgeCalculator at each hourly cycle |
| **BaseNFTItem** | NFTDistributor calls `assign_to_blank_token()` + `spawn_into()` for NFT placement |
| **SpellbookMixin / RecipeBookMixin** | Source of `known_by` data for saturation calculation |
| **FungibleInventoryMixin** | FungibleDistributor calls `receive_resource_from_reserve()` / `receive_gold_from_reserve()` on targets |
| **Quest System** | `FCMQuest.complete()` registers quest debt via `_register_quest_debt()` → `allocate_quest_reward()` deducts from budget state |
| **RoomHarvesting** | Target for resources (tagged rooms with capacity) |
| **CombatMob** | Target for resources/gold/NFTs (tagged mobs with capacity); death transfers all loot to corpse |
| **WorldChest** | Target for resources/gold (tagged containers with capacity) |
| **ZoneSpawnScript** | Maintains mob populations; syncs target tags on spawned mobs |
| **Corpse** | Mob death transfers resources/gold/NFTs to lootable corpse; unclaimed corpses return to RESERVE |
| **ReallocationServiceScript** | Daily SINK → RESERVE cycle; determines gold RESERVE available to GoldCalculator |
| **SPELL_REGISTRY / RECIPES** | Source data for `populate_knowledge_config()` — auto-generates knowledge entries in SPAWN_CONFIG |

---

## Implementation Status

All 9 phases are complete:

| Phase | Description | Status |
|---|---|---|
| 1 | Calculator + Distributor infrastructure (all calculators, distributors, budget state, headroom, service orchestrator) | Complete |
| 2 | Migrate resource spawning to unified tags + `UnifiedSpawnScript` | Complete |
| 3 | Disconnect old `ResourceSpawnService`, `RESOURCE_SPAWN_CONFIG`, `MOB_RESOURCE_SPAWN_CONFIG` | Complete |
| 4 | Knowledge item spawning — scrolls/recipes pre-placed on mobs via `ScrollDistributor` / `RecipeDistributor` | Complete |
| 5 | Disconnect old knowledge infrastructure — doc update only (no code to remove) | Complete |
| 6 | Quest debt methods — `allocate_quest_reward()`, `add_quest_debt()`, `get_spawn_service()` | Complete (built in Phase 1) |
| 7 | Retrofit quest givers — `FCMQuest.complete()` registers quest debt for gold and bread rewards | Complete |
| 8 | Gold spawning — `spawn_gold` tags on mobs and `WorldChest` | Complete |
| 9 | Rare NFT spawning POC — `RareNFTCalculator` placeholder + end-to-end test | Complete |

Comprehensive test coverage across `tests/spawn_tests/` and `tests/quest_tests/` covers calculators, distributors, budget state, headroom, service orchestration, scroll/recipe placement, gold spawning, rare NFT placeholder, and quest debt integration.

### Deferred (Not In This Plan)

- **Full RareNFTCalculator** — population-gated rules, designed when rare items are ready
- **Tracker Token AMM** — common equipment pricing mechanism (separate from spawn system)
- **NFT quest rewards** — future quests awarding NFTs will need quest debt registration
- **Telemetry dashboards** — spawn telemetry, surplus tracking
- **Auction House** — Tier 3 market for Master+ and enchanted items

---

## Tuning Philosophy

- **Default config is a starting point, not gospel.** Watch telemetry. If wheat is chronically oversupplied, raise the target price band. If players are constantly running out, do the reverse.
- **Don't tune too aggressively.** The system corrects over 24-hour cycles. A spike in one direction resolves itself.
- **The AMM is the backstop.** Players can always buy resources from the AMM shopkeeper. The spawn system determines whether that's cheap or expensive, not whether resources are available at all.
- **Cold start is the dangerous period.** New server, no consumption history, default spawn rates. Watch closely for the first 48 hours and adjust defaults if the economy looks broken.
- **Watch surplus logs.** If surplus is consistently banked and dropped, the mob population or per-target caps need increasing. The spawn system can only place items on targets that exist and have headroom.

---

## Design Properties

- **Separation of concerns:** "how much" (calculator) is completely independent of "where" (distributor). Change the economic formula without touching distribution. Add new target types without touching economic logic.
- **Pluggable calculators:** New asset types get a new calculator class with the same interface. The distributor doesn't care where the number came from.
- **Tag-driven distribution:** No configured channel shares. Tags on targets are the single source of truth for where items can appear. Add a `spawn_gold` tag to a mob type and it joins the gold pool automatically.
- **Proportional by headroom:** Targets with more capacity receive proportionally more. A treasure chest (cap 20) naturally receives more than a rabbit (cap 1). No explicit weighting needed.
- **Alternating direction:** Proportional allocation with integer rounding can systematically favour one end of the capacity spectrum. Alternating sort direction each tick ensures both high-cap targets (treasure hoards) and low-cap targets (starter mobs) get fair treatment.
- **Quest-aware budgets:** The mutable budget state means quests don't inflate the economy — they redirect already-budgeted supply. Large rewards spread their deduction across multiple ticks naturally.
- **Gold is safe:** The RESERVE throttle ensures gold spawning can never outpace the available supply pool. The system degrades gracefully as the RESERVE pool depletes.
- **Always-on:** Hard caps on all targets prevent runaway accumulation. Players always return to a stocked world.
- **Late-bound targets:** Mobs die, containers get looted, rooms fill up. Querying at tick time rather than scheduling time handles all volatility naturally.
- **Observable:** Surplus banking and dropping is logged. Persistent surplus signals tuning needs (more mobs, higher caps, more containers).
- **Knowledge adapts automatically:** As players learn spells and recipes, saturation rises, budgets drop, fewer scrolls enter the world. New content starts at 0% saturation and rapidly distributes. No manual drop rate tuning needed.
- **Truly unified:** Four calculator variants and two distributor variants share the same scheduling, budgeting, and target pooling infrastructure. Adding a new asset type means writing a new calculator — the rest is reused.

---

## Implementation Reference

| Component | File |
|---|---|
| Spawn config | `blockchain/xrpl/services/spawn/config.py` — `SPAWN_CONFIG`, `populate_knowledge_config()` |
| Base calculator | `blockchain/xrpl/services/spawn/calculators/base.py` — `BaseCalculator` |
| Resource calculator | `blockchain/xrpl/services/spawn/calculators/resource.py` — `ResourceCalculator` |
| Gold calculator | `blockchain/xrpl/services/spawn/calculators/gold.py` — `GoldCalculator` |
| Knowledge calculator | `blockchain/xrpl/services/spawn/calculators/knowledge.py` — `KnowledgeCalculator` |
| Rare NFT calculator | `blockchain/xrpl/services/spawn/calculators/rare_nft.py` — `RareNFTCalculator` |
| Base distributor | `blockchain/xrpl/services/spawn/distributors/base.py` — `BaseDistributor` |
| Fungible distributor | `blockchain/xrpl/services/spawn/distributors/fungible.py` — `FungibleDistributor` |
| NFT distributors | `blockchain/xrpl/services/spawn/distributors/nft.py` — `ScrollDistributor`, `RecipeDistributor`, `RareNFTDistributor` |
| Budget state | `blockchain/xrpl/services/spawn/budget.py` — `BudgetState` |
| Headroom utils | `blockchain/xrpl/services/spawn/headroom.py` — `get_current_count()`, `count_nfts()` |
| Service orchestrator | `blockchain/xrpl/services/spawn/service.py` — `SpawnService`, `get_spawn_service()` |
| Hourly script | `typeclasses/scripts/unified_spawn_service.py` — `UnifiedSpawnScript` |
| Saturation script | `typeclasses/scripts/nft_saturation_service.py` — `NFTSaturationScript` |
| Saturation service | `blockchain/xrpl/services/nft_saturation.py` — `NFTSaturationService.take_daily_snapshot()` (still so named, but runs hourly) |
| Mob base (tags) | `typeclasses/actors/mob.py` — `CombatMob`, `_build_tier_max()` |
| Zone spawn (tag sync) | `typeclasses/scripts/zone_spawn_script.py` — `ZoneSpawnScript._spawn_mob()` |
| Quest debt hook | `world/quests/base_quest.py` — `FCMQuest._register_quest_debt()` |
| Chest (gold tags) | `typeclasses/world_objects/chest.py` — `WorldChest` |
| Tests | `tests/spawn_tests/` and `tests/quest_tests/` |
