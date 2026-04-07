# Telemetry & Snapshot System

> Technical design document for the telemetry aggregation pipeline — how economic data is collected, stored, and consumed across the game.

**Related documents:**
- [ECONOMY.md](ECONOMY.md) — Telemetry Requirements section lists all snapshot fields and their status
- [UNIFIED_ITEM_SPAWN_SYSTEM.md](UNIFIED_ITEM_SPAWN_SYSTEM.md) — Calculator + Distributor architecture for resources, gold, and knowledge NFTs
- [DATABASE.md](DATABASE.md) — Database architecture (xrpl DB hosts all snapshot tables)

## Architecture Overview

The telemetry system is a three-layer aggregation pipeline that captures raw economic data and transforms it into pre-computed summaries for spawn algorithms, admin monitoring, and the public markets dashboard.

```
Raw Data Sources (continuously updated)
  │
  ├── PlayerSession          (login/logout tracking)
  ├── FungibleGameState      (in-game balances by location)
  ├── FungibleTransferLog    (every gold/resource/proxy-token movement)
  └── NFTGameState           (per-NFT ownership and location)
  │
  ▼
Hourly Aggregation (TelemetryService.take_snapshot)
  ├── EconomySnapshot        (1 row per hour — global metrics)
  └── ResourceSnapshot       (1 row per currency per hour — circulation, velocity, AMM prices)
  │
  ▼
Daily Aggregation (NFTSaturationService.take_daily_snapshot)
  └── SaturationSnapshot     (1 row per tracked item per day — spell/recipe/item saturation)
```

## Scheduled Scripts — Hourly Pipeline

Three services run on a staggered hourly cycle, each offset by 60 seconds so downstream services always have fresh data from upstream:

| Script | Interval | Offset | Service Method | Purpose |
|---|---|---|---|---|
| TelemetryAggregatorScript | 3600s | +0s | `TelemetryService.take_snapshot()` | AMM prices, gold flows, circulation |
| NFTSaturationScript | 3660s | +60s | `NFTSaturationService.take_daily_snapshot()` | Knowledge gaps (eligible - known - unlearned) |
| UnifiedSpawnScript | 3720s | +120s | `SpawnService.run_hourly_cycle()` | Distribute resources, gold, and knowledge NFTs |

**Pipeline order:** Telemetry → Saturation → Spawn. The spawn calculator needs current AMM prices (from telemetry) and current knowledge gaps (from saturation). The 60-second stagger ensures each service has completed before the next reads its output.

**Why stagger, not chain?** Each service runs independently via Twisted's reactor timer. The timer resets from the *scheduled* time, not the completion time, so drift is effectively zero — nanoseconds per tick from floating point rounding. Over months of uptime the services cannot drift by even 1 second relative to each other, let alone the 60-second gap between them. Chaining would add coupling (telemetry failure blocks spawning) without meaningful benefit.

**Threading:** All three services run their work in background threads via `deferToThread` so the Twisted reactor stays responsive during DB queries and XRPL API calls.

All scripts are created once at server start via `_ensure_global_scripts()` in `at_server_startstop.py` and persist across restarts.

### Server Boot Recovery

On every server start:
- `TelemetryService.close_stale_sessions()` closes any `PlayerSession` rows with `ended_at=NULL` (crash recovery — sessions left open by unclean shutdown)
- Global scripts are re-ensured (created if missing, skipped if existing)

---

## Models

### PlayerSession

Tracks individual play sessions. Written in real-time on every puppet/unpuppet.

| Field | Type | Description |
|---|---|---|
| `account_id` | IntegerField | Evennia account ID (not unique per session) |
| `character_key` | CharField(80) | Character `db_key` at session start (snapshot, not live) |
| `started_at` | DateTimeField | auto_now_add — session creation |
| `ended_at` | DateTimeField (null) | Set on unpuppet; NULL = session still open |

**Indexes**: `started_at`, `(account_id, started_at)`

**Write path**: `FCMCharacter.at_post_puppet()` → `record_session_start()`, `FCMCharacter.at_post_unpuppet()` → `record_session_end()`

**Consumers**: EconomySnapshot (player counts), SaturationSnapshot (active player window), SpawnService (player-hours)

---

### EconomySnapshot

One row per hour. Global economy health metrics.

| Field | Type | Description |
|---|---|---|
| `hour` | DateTimeField (unique) | Bucket: `now.replace(minute=0, second=0, microsecond=0)` |
| **Player Activity** | | |
| `players_online` | IntegerField | Distinct accounts with open sessions at snapshot time |
| `unique_players_1h` | IntegerField | Distinct accounts active in past 1 hour |
| `unique_players_24h` | IntegerField | Distinct accounts active in past 24 hours |
| `unique_players_7d` | IntegerField | Distinct accounts active in past 7 days |
| **Gold Flows** | | |
| `gold_circulation` | Decimal | Total FCMGold in CHARACTER + ACCOUNT (liquid player gold) |
| `gold_reserve` | Decimal | Total FCMGold in RESERVE (unspawned vault gold) |
| `gold_sinks_1h` | Decimal | Total FCMGold in SINK (consumed, awaiting reallocation) |
| `gold_spawned_1h` | Decimal | Gold pickup transfers in past hour |
| **Trading Activity** | | |
| `amm_trades_1h` | IntegerField | Count of AMM buy/sell transfers in past hour |
| `amm_volume_gold_1h` | Decimal | Gold amount flowing through AMM (player-side only, not double-counted) |
| **Chain Activity** | | |
| `imports_1h` | IntegerField | `deposit_from_chain` transfers in past hour |
| `exports_1h` | IntegerField | `withdraw_to_chain` transfers in past hour |
| `created_at` | DateTimeField | auto_now_add |

**Idempotency**: `update_or_create(hour=bucket)` — re-running for the same hour overwrites with latest data.

**Consumers**: Admin `economy` command, SpawnService (players_online gate)

---

### ResourceSnapshot

One row per currency per hour. Covers all CurrencyType entries — game resources (FCMWheat, FCMIronOre, etc.), gold (FCMGold), and proxy tokens (PGold, PTrainDagger, PTrainSSword).

| Field | Type | Description |
|---|---|---|
| `hour` | DateTimeField | Same bucket as EconomySnapshot |
| `currency_code` | CharField(40) | CurrencyType code |
| **Unique constraint** | | `(hour, currency_code)` |
| **Circulation by Location** | | |
| `in_character` | Decimal | CHARACTER location balance |
| `in_account` | Decimal | ACCOUNT location balance |
| `in_spawned` | Decimal | SPAWNED location balance |
| `in_reserve` | Decimal | RESERVE location balance |
| `in_sink` | Decimal | SINK location balance |
| **Velocity (Past Hour)** | | |
| `produced_1h` | Decimal | craft_output + pickup transfers |
| `consumed_1h` | Decimal | craft_input + sink transfers |
| `traded_1h` | Decimal | amm_buy + amm_sell volume |
| `exported_1h` | Decimal | withdraw_to_chain transfers |
| `imported_1h` | Decimal | deposit_from_chain transfers |
| **AMM Pricing** | | |
| `amm_buy_price` | Decimal (null) | Gold cost to buy 1 unit. NULL if no pool. |
| `amm_sell_price` | Decimal (null) | Gold received from selling 1 unit. NULL if no pool. |
| **Spawn Metrics** | | |
| `spawn_budget` | Integer | Calculator budget for this hour (written by SpawnService) |
| `spawn_quest_debt` | Integer | Budget redirected to quest rewards |
| `spawn_placed` | Integer | Units actually placed on targets |
| `spawn_dropped` | Integer | Surplus dropped (no targets with headroom) |

**Spawn metrics** are written by the SpawnService at the end of each hourly cycle (+120s in the pipeline). The invariant `budget - quest_debt - placed - dropped = 0` holds for each row. Persistently high `spawn_dropped` for a resource signals that more tagged targets (mobs, containers, rooms) are needed.

**Reconciliation invariant**: `in_character + in_account + in_spawned + in_reserve + in_sink = total issued on-chain` (for any given currency_code).

**Three currency categories in this table**:

| Category | Examples | Has AMM prices? | Circulation tracking? |
|---|---|---|---|
| Game resources | FCMWheat, FCMIronOre | Yes (vs FCMGold pools) | Yes — full 5-location |
| Gold | FCMGold | No (it IS gold) | Yes — for reserve monitoring |
| Proxy tokens | PGold, PTrainDagger | Yes (vs PGold pools) | PGold: RESERVE only. Item proxies: RESERVE only (vault or AMM pool) |

**AMM Price Fetch**: The telemetry service calls `_fetch_amm_prices()` which makes two batch queries:
1. `get_multi_pool_prices(resource_codes)` — game resources priced against FCMGold
2. `get_multi_pool_prices(proxy_codes, gold_currency=PGold)` — proxy tokens priced against PGold

This is necessary because proxy token AMM pools use PGold (not FCMGold) as the pair currency — a closed-loop design that prevents external wallets from interfering with NFT pricing. See [ECONOMY.md § NFT Item Market Making — Tracker Tokens](ECONOMY.md#nft-item-market-making--tracker-tokens) for the full proxy token design rationale. The two result dicts are merged before writing to ResourceSnapshot.

**Consumers**: Markets web page (Resource (AMM) tab, NFT (AMM) tab), SpawnService (consumption baseline, price modifier), Admin `economy` command

---

### SaturationSnapshot

One row per tracked item per snapshot cycle. Controls knowledge NFT spawn budgets.

| Field | Type | Description |
|---|---|---|
| `day` | DateField | Date of snapshot |
| `item_key` | CharField(80) | Prefixed spell/recipe key (e.g. `scroll_magic_missile`, `recipe_bronze_dagger`) |
| `category` | CharField(10) | `SPELL`, `RECIPE`, or `ITEM` |
| **Unique constraint** | | `(day, item_key, category)` |
| **Knowledge Metrics** | | |
| `active_players_7d` | IntegerField | Distinct players active in past 7 days |
| `eligible_players` | IntegerField | Players who CAN learn this spell/recipe (mastery-filtered). 0 for items. |
| `known_by` | IntegerField | Players who know this spell/recipe |
| `unlearned_copies` | IntegerField | Scroll/recipe NFTs in player hands (CHARACTER + ACCOUNT) |
| **Circulation** | | |
| `in_circulation` | IntegerField | NFTs of this type in CHARACTER + ACCOUNT + SPAWNED + ONCHAIN + AUCTION |
| `saturation` | FloatField | Computed ratio — higher = more saturated, lower = more needed |
| **Spawn Metrics** | | |
| `spawn_budget` | IntegerField | Gap-based budget for this cycle (written by SpawnService) |
| `spawn_quest_debt` | IntegerField | Budget redirected to quest rewards |
| `spawn_placed` | IntegerField | Scrolls/recipes actually placed on targets |
| `spawn_dropped` | IntegerField | Surplus dropped (no targets with headroom) |

**Spawn metrics** are written by the SpawnService at the end of each hourly cycle (+120s in the pipeline). For knowledge items, `spawn_budget` equals the gap (`eligible - known_by - unlearned_copies`).

**Saturation formula**:
- **Spells/Recipes**: `(known_by + unlearned_copies) / eligible_players` (0.0 if no eligible)
- **Items**: Placeholder 0.0 — awaits rarity_divisor config per item type

**Data sources**: PlayerSession (7d active), ObjectDB characters (db.spellbook, db.recipe_book), NFTGameState (location counts), SPELL_REGISTRY, RECIPES dict

**Consumers**: SpawnService (KnowledgeCalculator reads gap = eligible - known_by - unlearned_copies)

**Key format**: Item keys use the `scroll_` / `recipe_` prefix to match spawn config type_keys (e.g. `scroll_magic_missile`, `recipe_bronze_dagger`). This ensures the KnowledgeCalculator can look up saturation data directly by its config key without prefix stripping.

---

## Data Flow Diagram

### Writes (What Feeds Each Model)

```
                    ┌──────────────────────────┐
                    │    Game Events (live)     │
                    └──────────┬───────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                 ▼
        PlayerSession    FungibleGameState   NFTGameState
        (puppet/unpuppet)  (every trade,     (every spawn,
                            craft, fee)       pickup, delete)
              │                │                 │
              │    ┌───────────┤                 │
              │    │           │                 │
              ▼    ▼           ▼                 │
         ┌─────────────────────────┐             │
         │  TelemetryService       │             │
         │  (hourly)               │             │
         │                         │             │
         │  Reads: PlayerSession,  │             │
         │    FungibleGameState,   │             │
         │    FungibleTransferLog, │             │
         │    CurrencyType,        │             │
         │    XRPL AMM pools       │             │
         │                         │             │
         │  Writes:                │             │
         │    EconomySnapshot      │             │
         │    ResourceSnapshot     │             │
         └─────────────────────────┘             │
                                                 │
         ┌─────────────────────────┐             │
         │  NFTSaturationService   │◄────────────┘
         │  (hourly, +60s offset)  │
         │                         │
         │  Reads: PlayerSession,  │
         │    ObjectDB characters, │
         │    NFTGameState,        │
         │    SPELL_REGISTRY,      │
         │    RECIPES              │
         │                         │
         │  Writes:                │
         │    SaturationSnapshot   │
         └─────────────────────────┘
```

### Reads (What Consumes Each Model)

| Consumer | Reads | Purpose | Details |
|---|---|---|---|
| SpawnService (ResourceCalculator) | EconomySnapshot, ResourceSnapshot, FungibleGameState, PlayerSession | Two-factor resource spawn algorithm | See [UNIFIED_ITEM_SPAWN_SYSTEM.md](UNIFIED_ITEM_SPAWN_SYSTEM.md) |
| Markets web page | ResourceSnapshot (latest hour), CurrencyType, NFTItemType | Public economic dashboard | `/markets/` route |
| Admin `economy` command | EconomySnapshot (latest), ResourceSnapshot (latest) | Superuser monitoring | `economy [resource]` |
| SpawnService (KnowledgeCalculator) | SaturationSnapshot (latest) | Gap-based knowledge item budgets (eligible - known - unlearned) | See [UNIFIED_ITEM_SPAWN_SYSTEM.md](UNIFIED_ITEM_SPAWN_SYSTEM.md) |

---

## AMM Price Collection

The telemetry service collects AMM prices from two distinct pool ecosystems:

### Resource Pools (FCMResource / FCMGold)

Standard game resources trade against FCMGold in XRPL AMM pools. These are player-visible markets — the prices shown on resource shopkeepers.

```
_fetch_amm_prices():
  resource_codes = CurrencyType codes where resource_id IS NOT NULL
  get_multi_pool_prices(resource_codes)  → defaults to FCMGold pair
```

### Proxy Token Pools (PToken / PGold)

NFT equipment proxy tokens trade against PGold in isolated AMM pools. These are vault-internal — players never see PGold, they pay/receive FCMGold through the NFTAMMService.

```
_fetch_amm_prices():
  proxy_codes = CurrencyType codes from NFTItemType.tracking_token (non-NULL)
  get_multi_pool_prices(proxy_codes, gold_currency=PGold)
```

Both result sets are merged into a single dict. ResourceSnapshot stores whichever price was found — the snapshot row doesn't know or care whether the price came from an FCMGold pool or a PGold pool.

**Graceful degradation**: If XRPL is unavailable, `_fetch_amm_prices()` returns `{}` and all prices are stored as NULL. The telemetry system continues to collect circulation and velocity data. The spawn algorithm falls back to `price_modifier = 1.0` when price is NULL.

---

## Proxy Token Snapshot Behaviour

With proxy tokens (PGold, PTrainDagger, PTrainSSword) registered as CurrencyType entries, they automatically get ResourceSnapshot rows during the hourly telemetry run. Here's what each proxy token's snapshot looks like:

### PGold

| Field | Expected Value | Notes |
|---|---|---|
| `in_reserve` | Non-zero | Vault's PGold balance — goes up on item sells, down on item buys |
| `in_sink` | Small amounts | Rounding dust from trades |
| `in_character` | Always 0 | Players never hold PGold |
| `in_account` | Always 0 | Players never hold PGold |
| `in_spawned` | Always 0 | PGold is never spawned in the world |
| `amm_buy_price` | NULL | PGold has no AMM pool of its own |
| `amm_sell_price` | NULL | PGold has no AMM pool of its own |
| `traded_1h` | Non-zero | nft_amm_buy + nft_amm_sell transfer volume |

### PTrainDagger / PTrainSSword (Item Proxy Tokens)

| Field | Expected Value | Notes |
|---|---|---|
| `in_character` | NFT count | NFTGameState rows with this item_type in CHARACTER location |
| `in_account` | NFT count | NFTGameState rows with this item_type in ACCOUNT location |
| `in_spawned` | NFT count | NFTGameState rows with this item_type in SPAWNED location |
| `in_reserve` | NFT count | NFTGameState rows with this item_type in RESERVE location (blank tokens excluded — only assigned items) |
| `in_sink` | Always 0 | NFTs don't have a SINK location |
| `amm_buy_price` | Non-zero | Price from PTrainDagger/PGold AMM pool |
| `amm_sell_price` | Non-zero | Price from PTrainDagger/PGold AMM pool |
| `traded_1h` | Non-zero | nft_amm_buy + nft_amm_sell + nft_amm_swap transfer volume |

**Circulation source**: For proxy token currencies, the telemetry service populates circulation fields from `NFTGameState` (not `FungibleGameState`). A single grouped query — `_nft_circulation_by_tracking_token()` — counts all NFTs with `item_type IS NOT NULL` grouped by `item_type.tracking_token` and `location`. This is efficient because NFTGameState rows with `item_type = NULL` are blank reserve tokens (not in circulation).

The AMM prices on proxy token snapshots represent what a player would pay/receive in gold for that equipment item, since PGold = FCMGold at 1:1.

---

## Markets Web Page

The markets dashboard at `/markets/` renders data from the latest hourly snapshot across three tabs:

### Resource (AMM) Tab

Shows all game resources (CurrencyType entries with `resource_id IS NOT NULL`). Displays buy price, sell price, spread, circulation (CHARACTER + ACCOUNT + SPAWNED), and 1-hour trade volume.

### NFT (AMM) Tab

Shows NFT item types with proxy token AMM pricing (NFTItemType entries with `tracking_token IS NOT NULL`). Dynamically detects tradeable items — as new proxy tokens are created and assigned to item types, they automatically appear. Displays buy price, sell price, spread, and NFT circulation count from the hourly snapshot.

### Non-AMM Tab

Shows NFT item types without proxy tokens — items that exist in the game but aren't AMM-priced (Master/GM crafted items, enchanted weapons, rare drops). Excludes recipes and scrolls. No pricing data, only circulation counts from a live `NFTGameState` query (not the hourly snapshot — these items have no ResourceSnapshot rows). Only items with at least one copy in circulation are shown. This gives visibility into the broader item economy beyond what's AMM-tradeable.

---

## Velocity Categories

Transfer types are grouped into velocity categories for snapshot aggregation:

| Category | Transfer Types | Meaning |
|---|---|---|
| Produced | `craft_output`, `pickup` | Resources entering player hands |
| Consumed | `craft_input`, `sink` | Resources leaving player hands permanently |
| Traded | `amm_buy`, `amm_sell` | Resource-side AMM volume |
| Traded (NFT) | `nft_amm_buy`, `nft_amm_sell`, `nft_amm_swap` | Proxy token AMM volume |
| Exported | `withdraw_to_chain` | Sent to external wallets |
| Imported | `deposit_from_chain` | Received from external wallets |

---

## Spawn System Telemetry (Future)

The unified spawn system (`SpawnService`) currently tracks per-item metrics in memory via `BudgetState` counters and Python logging:

| Counter | Description |
|---|---|
| `spawned_this_hour` | Units actually placed on targets this hour |
| `dropped_this_hour` | Surplus that couldn't be placed (all targets at cap) |
| `quest_debt` | Pending deduction from quest rewards |
| `surplus_bank` | Carried from previous tick (targets unavailable) |

**Not yet in a snapshot model.** These counters reset each hourly cycle and are only visible in server logs. A future `SpawnSnapshot` model (one row per item per hour) would enable:
- Trending surplus over days/weeks to identify capacity bottlenecks ("hide surplus is consistently 40% — need more wolves")
- Quest debt impact analysis (how much budget is suppressed by quest rewards)
- Per-calculator budget vs actual placed comparison
- Admin dashboard visibility into spawn health

**Deferred until production data is needed.** During early dev, Python logging is sufficient. See [UNIFIED_ITEM_SPAWN_SYSTEM.md](UNIFIED_ITEM_SPAWN_SYSTEM.md) § Surplus Banking for the design intent.

---

## Design Principles

1. **Pre-computed summaries over live queries** — Snapshot data is computed once per hour and read many times. The markets page, spawn algorithm, and admin tools all read from snapshots rather than re-aggregating raw data on every request.

2. **Idempotent aggregation** — `update_or_create(hour=bucket)` means re-running the same hour overwrites safely. No duplicate rows, no stale data after recovery.

3. **Graceful degradation** — XRPL unavailability doesn't break telemetry. Prices store as NULL, everything else continues. The spawn algorithm falls back to `price_modifier = 1.0`.

4. **Zero-balance deletion** — FungibleGameState deletes rows at zero balance. All `SUM` aggregates coalesce NULL → `Decimal(0)` to handle this transparently.

5. **Rolling windows, not calendar boundaries** — Player counts use "sessions overlapping the past N hours" rather than fixed hour/day boundaries. This gives smoother, more accurate activity metrics.

6. **Unified table, multiple categories** — ResourceSnapshot stores game resources, gold, and proxy tokens in the same table. Consumers filter by category (resource_id, tracking_token, is_gold) as needed. One aggregation pass captures everything.
