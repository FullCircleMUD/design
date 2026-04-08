# Treasury & Subscription Processing

> **Scope:** Internal fiscal and monetary policy automation for FullCircleMUD. This document describes operational systems for managing game revenue and disciplining in-game currency issuance.

---

## What This System IS

- An internal mechanism to prevent over-issuance of game gold
- Automated cash management for business operations
- Self-imposed fiscal discipline — constraining the operator from inflating away player value
- Operational treasury management (liquidity allocation between cash and short-term instruments)

## What This System IS NOT

- **Not a peg, backing, or collateralisation of game gold.** Gold has no redemption right, no fixed exchange rate to any external asset, and no guaranteed value. See `design/COMPLIANCE.md` for the full no-redemption model.
- **Not a promise to players.** No aspect of this system creates any obligation, expectation, or undertaking regarding the value of game gold or any other in-game asset.
- **Not a reserve system.** The RLUSD held in treasury is business revenue, not a reserve backing game currency. The operator retains full discretion over treasury allocation.
- **Not a stablecoin mechanism.** Game gold floats freely based on in-game supply and demand. This system governs *issuance discipline*, not price stability.

> **Language rule (from COMPLIANCE.md):** Never describe this system using "backed", "pegged", "redeemable", "collateralised", "guaranteed value", or "reserve" in any player-facing or public context. Internally, use "treasury" for business funds and "RESERVE" only for the in-game location enum (existing convention).

---

## Wallet Architecture

### Current (Two Wallets)

| Wallet | Role |
|---|---|
| **Issuer** | Issues all FCM currencies, mints NFTs, receives subscription payments |
| **Vault** | Holds game asset inventory, executes AMM swaps, processes imports/exports |

### Proposed (Three Wallets)

| Wallet | Role | Holds |
|---|---|---|
| **Issuer** | Central financial authority — issues all FCM currencies, mints NFTs, receives subscription payments, holds treasury (RLUSD + T-Bills), issues gold on subscription events | Subscription revenue, RLUSD, T-Bills, game currency issuance authority |
| **Vault** | Game operations — holds game asset inventory, executes AMM swaps, processes imports/exports | Game gold, resources, NFTs, AMM LP tokens |
| **Operating** (NEW) | Business expenses — hosting, LLM API fees, development costs | RLUSD allocated from subscription splits; funds may be moved to exchange for fiat conversion |

### Why a Separate Operating Wallet?

The issuer is the game's central financial authority — it receives revenue, holds treasury assets, and controls currency issuance. That role should not be muddied with day-to-day expense payments. Separating the operating wallet:

- **Issuer** remains the fiscal centre — revenue in, gold out, treasury management. Tightly controlled.
- **Operating** receives its allocation (10% of subscription revenue) and is used freely for business expenses without touching the issuer's treasury position.
- **Vault** remains the game operations wallet — player-facing transactions only.
- Each wallet has distinct multisig cosigner rules appropriate to its role.

> **No migration needed** — subscription payments already go to the issuer. The issuer sends the operating split to the new operating wallet.

---

## Subscription Processing

### Flow

```
Player pays $20 RLUSD ──→ Issuer Wallet
                              │
                              ├──→ 10% ($2 RLUSD) ──→ Operating Wallet
                              │    (business expenses — hosting, LLM, dev)
                              │
                              ├──→ 90% ($18 RLUSD) ──→ Retained in issuer
                              │    (treasury management — RLUSD / T-Bill allocation)
                              │
                              └──→ Issuer issues POLICY_GOLD_AMOUNT ──→ Vault (RESERVE)
                                   (game economy provisioning — decoupled from revenue)
```

### Gold Issuance — Decoupled from Revenue

The gold issuance amount is **not derived from the RLUSD value**. It is a game design parameter:

| Parameter | Value | Set By |
|---|---|---|
| `GOLD_PER_SUBSCRIPTION` | e.g. 1000 gold | Game policy (admin) |
| Review frequency | Periodic (e.g. monthly/quarterly) | Admin discretion |
| Adjustment criteria | In-game inflation rate, gold velocity, active supply, player count | Economic health indicators |

**Why decoupled?** If gold issuance were calculated from RLUSD value, it would create an implicit exchange rate — exactly what the compliance model prohibits. Instead:

- The RLUSD is **revenue** (business income)
- The gold is **game economy provisioning** (content creation)
- They happen at the same time but one does not determine the other
- The admin can adjust `GOLD_PER_SUBSCRIPTION` independently of subscription price

### Server-Side Implementation

XRPL mainnet does not support on-chain Hooks. All processing is server-side:

1. **Transaction monitoring** — game server subscribes to the issuer wallet's transaction stream via XRPL WebSocket (`subscribe` command)
2. **Payment detection** — filter for incoming RLUSD Payment transactions
3. **Subscription verification** — match against pending subscription requests (existing `verify_fungible_payment()` pattern)
4. **Operating split** — issuer sends 10% RLUSD to operating wallet
5. **Gold provisioning** — issuer issues `GOLD_PER_SUBSCRIPTION` gold to vault via Payment
6. **Subscription activation** — existing `extend_subscription()` flow
7. **Memo tagging** — all transactions carry structured memos (see Memo System below)

### Configuration

```python
# Subscription processing
SUBSCRIPTION_PAYMENT_DESTINATION = XRPL_ISSUER_ADDRESS    # issuer is the fiscal centre
SUBSCRIPTION_OPS_SPLIT_PCT = 10          # % of subscription RLUSD to operating wallet
GOLD_PER_SUBSCRIPTION = 1000            # gold issued per subscription (game policy)

# Operating wallet (business expenses — hosting, LLM, dev, fiat conversion)
XRPL_OPERATING_ADDRESS = "<operating_address>"
XRPL_OPERATING_WALLET_SEED = os.environ.get("XRPL_OPERATING_WALLET_SEED", "")
```

---

## Treasury Management

### Asset Allocation

The issuer wallet holds business revenue (after the operating split) in two forms:

| Asset | Purpose | Target Allocation |
|---|---|---|
| **RLUSD** | Liquid cash for near-term operating expenses | 20% |
| **Tokenized T-Bills** | Short-term yield on idle cash (e.g. OpenEden TBILL on XRPL) | 80% |

> **Fiscal discipline mechanism:** The issuer's treasury position (RLUSD + T-Bills) constrains gold issuance — the operator only issues new gold when subscription revenue flows in, preventing arbitrary currency creation. This is purely internal issuance discipline: a self-imposed rule to prevent the operator from debasing the in-game currency through unchecked supply expansion.
>
> **This has no bearing on gold's market price.** If anyone were to create a secondary market in game gold (e.g. a gold/RLUSD pair on the XRPL DEX), gold would trade at a freely floating price determined entirely by market supply and demand — completely unlinked to the issuer's treasury position, the subscription price, or any internal policy parameter. The treasury position does not back, peg, support, or stabilise the price of gold. It exists solely to discipline the operator's issuance behaviour.

### Rebalancing

| Parameter | Value |
|---|---|
| Target RLUSD allocation | 20% |
| Lower band | 17% |
| Upper band | 23% |
| Trigger | After each subscription payment received |
| Action | Buy or sell T-Bills to return to 20% target |

**Rebalancing logic (pseudocode):**

```
after each subscription payment:
    rlusd_balance = get_issuer_rlusd_balance()
    tbill_balance = get_issuer_tbill_balance()
    total = rlusd_balance + tbill_balance  # (T-Bill valued at face)
    rlusd_pct = rlusd_balance / total

    if rlusd_pct < 0.17:
        sell T-Bills for RLUSD to reach 20%
    elif rlusd_pct > 0.23:
        buy T-Bills with RLUSD to reach 20%
```

### T-Bill Integration

Tokenized T-Bills on XRPL are issued currencies from a third-party issuer. The treasury wallet needs:

- Trust line to the T-Bill issuer
- DEX or AMM access for RLUSD ↔ T-Bill swaps
- Price feed or face-value assumption for allocation calculations

> **Decision needed:** Which tokenized T-Bill issuer to use on XRPL mainnet? This depends on what's available and liquid at launch time. The system should be designed to swap the T-Bill issuer without code changes (configuration-driven).

---

## Gold Sink Integration

The existing economy has a SINK → RESERVE reallocation cycle (see `design/ECONOMY.md`):

- Gold consumed by game activity (crafting fees, repair, training, etc.) flows to SINK
- `ReallocationServiceScript` drains SINK → RESERVE daily
- Currently 100% goes back to RESERVE; a 10% operational burn is deferred

### Connecting Sinks to Treasury

When the deferred 10% burn is activated:

```
SINK (daily reallocation)
  ├──→ 90% ──→ RESERVE (respawned as rewards)
  └──→ 10% ──→ Burned (issuer reclaims — reduces total issued supply)
```

The 10% burn is a **deflationary mechanism** — it reduces total gold supply, counteracting issuance from subscriptions and other sources. It is NOT a revenue mechanism (burning gold doesn't generate RLUSD or any external value).

> **Relationship to subscription issuance:** Over time, the system should tend toward equilibrium — gold issued via subscriptions is offset by gold burned via sinks. The admin monitors this balance and adjusts `GOLD_PER_SUBSCRIPTION` if issuance consistently outpaces sinks (inflationary) or vice versa (deflationary squeeze).

---

## Transaction Memo System

All XRPL transactions on game wallets carry structured memos for audit trail and accounting.

### Memo Format

Each transaction includes one Memo with:

| Field | Encoding | Value |
|---|---|---|
| `MemoType` | hex-encoded UTF-8 | Operation category (e.g. `fcm/subscription`) |
| `MemoData` | hex-encoded UTF-8 | JSON payload with operation details |
| `MemoFormat` | hex-encoded UTF-8 | `application/json` |

### Memo Categories

#### Setup Operations (Manager Tool — One-Time)

| MemoType | Trigger | MemoData Example |
|---|---|---|
| `fcm/issue` | Currency issuance | `{"currency":"FCMWheat","amount":"1000000"}` |
| `fcm/trust` | Trust line setup | `{"currency":"FCMWheat"}` |
| `fcm/mint` | NFT minting | `{"startId":1,"count":50}` |
| `fcm/nft-transfer` | NFT batch transfer | `{"direction":"issuer-to-vault"}` |
| `fcm/amm-create` | AMM pool creation | `{"asset1":"FCMWheat","asset2":"FCMGold"}` |
| `fcm/amm-deposit` | AMM liquidity deposit | `{"asset1":"FCMWheat","amount":"500"}` |
| `fcm/config` | Account flags/settings | `{"flag":"DefaultRipple","action":"set"}` |

#### Runtime Operations (Game Server — Ongoing)

| MemoType | Trigger | MemoData Example |
|---|---|---|
| `fcm/export` | Player exports gold/resource | `{"type":"resource","currency":"FCMWheat","amount":"50"}` |
| `fcm/import` | Player imports gold/resource | `{"type":"gold","amount":"100"}` |
| `fcm/nft-export` | Player exports NFT item | `{"nftId":"000..."}` |
| `fcm/nft-import` | Player imports NFT item | `{"nftId":"000..."}` |
| `fcm/swap` | AMM swap (shopkeeper trade) | `{"sell":"FCMWheat","buy":"FCMGold","sellAmt":"10","buyAmt":"15"}` |

#### Subscription & Treasury Operations

| MemoType | Trigger | MemoData Example |
|---|---|---|
| `fcm/subscription` | Gold provisioned from subscription | `{"goldAmount":1000,"policyRate":"GOLD_PER_SUBSCRIPTION"}` |
| `fcm/ops-split` | Operating split from subscription | `{"subscriptionTx":"<hash>","splitPct":10,"amount":"2.00"}` |
| `fcm/rebalance` | Treasury RLUSD/T-Bill rebalance | `{"direction":"buy-tbill","amount":"100","rlusdPctBefore":25,"rlusdPctAfter":20}` |
| `fcm/sink-burn` | Gold burned from SINK (10%) | `{"amount":"500","sinkTotal":"5000","burnPct":10}` |

### Memo Design Principles

- **No player identifiers on-chain.** Memos do not contain player IDs, character names, or wallet addresses beyond what's already in the transaction itself. Player correlation is done server-side via `SubscriptionPayment` and `XRPLTransactionLog` records.
- **Memos are for the operator's audit trail**, not for public consumption. They help with accounting, debugging, and compliance reporting.
- **Xaman-signed transactions** (player imports, subscription payments) — the game server builds the Xaman payload including the memo before sending to the player for signing. The player signs the complete transaction including the memo.

---

## Implementation Phases

### Phase 1: Memo System (Foundation)

Add memo support to all existing transaction paths:
- Manager tool: `submitXrplTx()` helper accepts optional memo
- Game server: `xrpl_tx.py` functions accept optional memo parameter
- Xaman payloads: include memos in payment/offer templates

### Phase 2: Operating Wallet

- Create third XRPL wallet for operating expenses
- Configure multisig with appropriate cosigner rules
- Trust line for RLUSD
- Issuer trust lines for T-Bill issuer (when selected)

### Phase 3: Subscription Processing

- Server-side transaction monitor for issuer wallet
- Revenue split logic (10% ops / 90% retained)
- Gold provisioning trigger (issuer → vault)
- `GOLD_PER_SUBSCRIPTION` configuration and admin adjustment interface

### Phase 4: Treasury Rebalancing

- T-Bill issuer selection and trust line setup
- Balance monitoring after each subscription
- Auto-rebalance within tolerance bands
- Rebalance transaction with memo audit trail

### Phase 5: Sink Burn Activation

- Activate the deferred 10% gold burn in `ReallocationServiceScript`
- Burn transactions carry `fcm/sink-burn` memos
- Dashboard reporting: issuance vs burn rate over time

---

## Monitoring & Reporting

### Internal Dashboard Metrics

| Metric | Source | Purpose |
|---|---|---|
| Issuer RLUSD balance | XRPL query | Cash position |
| Issuer T-Bill balance | XRPL query | Yield allocation |
| RLUSD allocation % | Calculated | Rebalancing trigger |
| Gold issued (period) | Memo audit trail | Issuance monitoring |
| Gold burned (period) | Memo audit trail | Deflation monitoring |
| Net gold flow | Issued - burned | Inflation/deflation indicator |
| Active subscriptions | SubscriptionPayment DB | Revenue forecasting |
| SINK balance | FungibleGameState | Pending reallocation |

### Transparency (Optional, Operator Discretion)

The operator may choose to publish selected metrics (e.g. total gold supply, burn rate) as part of a transparency commitment. This is a **voluntary disclosure**, not an obligation. Any published metrics should be clearly framed as informational, not as a promise or guarantee.

---

## Open Questions

1. **T-Bill issuer:** Which tokenized T-Bill is available and liquid on XRPL mainnet? Needs research at launch time.
2. **Gold provisioning amount:** Initial value for `GOLD_PER_SUBSCRIPTION`? Needs calibration against expected player economy.
3. **Sink burn timing:** Activate at mainnet launch, or defer until economy stabilizes?
4. **Cosigner rules per wallet:** What business rules should the cosigning service enforce for issuer (fiscal centre) vs vault (game ops) vs operating (expenses)?
5. **Rebalancing frequency:** After every subscription, or batched (e.g. daily)?
6. **Operating wallet fiat off-ramp:** Which exchange/process for converting RLUSD to fiat for bill payments?
