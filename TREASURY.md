# Treasury & Subscription Processing

> **Scope:** Internal fiscal and monetary policy automation for FullCircleMUD. This document describes operational systems for managing game revenue and disciplining in-game currency issuance.
>
> **See also:** [SUBSCRIPTIONS.md](SUBSCRIPTIONS.md) for the subscription billing flow, [IMPORT_EXPORT.md](IMPORT_EXPORT.md) for the chain-boundary `import`/`export` commands and the planned OFAC SDN screening, [COMPLIANCE.md](COMPLIANCE.md) for the no-redemption model and language policy, and [ECONOMY.md](ECONOMY.md) for the in-game economy and SINK→RESERVE reallocation.
>
> **See also:** [SUBSCRIPTIONS.md](SUBSCRIPTIONS.md) for the subscription billing flow, [IMPORT_EXPORT.md](IMPORT_EXPORT.md) for the chain-boundary `import`/`export` commands and the planned OFAC SDN screening, [COMPLIANCE.md](COMPLIANCE.md) for the no-redemption model and language policy, and [ECONOMY.md](ECONOMY.md) for the in-game economy and SINK→RESERVE reallocation.

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

The three-wallet infrastructure exists on-chain today. The game code only references **two** of the three wallets (issuer and vault). The operating wallet is **intentionally kept off the game server** and managed manually — it is the operator's business banking layer, not part of the game runtime. This is by design, not a temporary state waiting for code to catch up.

| Wallet | Role | Holds | Game-server visibility |
|---|---|---|---|
| **Issuer** | Central financial authority — issues all FCM currencies, mints NFTs, receives subscription payments, holds treasury (RLUSD + T-Bills), issues gold on subscription events | Subscription revenue, RLUSD, T-Bills, game currency issuance authority | Yes (`XRPL_ISSUER_ADDRESS`) |
| **Vault** | Game operations — holds game asset inventory, executes AMM swaps, processes imports/exports | Game gold, resources, NFTs, AMM LP tokens | Yes (`XRPL_VAULT_ADDRESS`) |
| **Operating** | Business expenses — hosting, LLM API fees, development costs, fiat conversion | RLUSD moved from the issuer manually after subscription revenue has been accumulated | **No — and not planned to be.** Operated entirely outside the game server. |

### Why a Separate Operating Wallet?

The issuer is the game's central financial authority — it receives revenue, holds treasury assets, and controls currency issuance. That role is kept separate from day-to-day expense payments. The split:

- **Issuer** is the fiscal centre — revenue in, gold out, treasury management. Tightly controlled.
- **Operating** receives a periodic allocation from the issuer (the operator's accounting decision, not an automated 1-payment-1-split flow) and is used freely for business expenses without touching the issuer's treasury position.
- **Vault** is the game operations wallet — player-facing transactions only.
- Vault and issuer each have distinct multisig cosigner rules appropriate to their roles. Operating does not use multisig — it is secured by the shared regular key (Key C) with its master key disabled.

### Why the operating wallet is kept off the game server

The operating wallet handles real-world business expenses — hosting bills, LLM API fees, development costs, and conversion to fiat. None of those flows belong inside the game runtime:

- **Reduced attack surface.** Operational funds are isolated from any game-server compromise. A vulnerability in the game server cannot drain expense funds because the server has no signing authority over the operating wallet.
- **Manual operator judgement.** Allocation decisions, fiat off-ramp timing, and expense pacing are accounting decisions made by the operator, not automatable game-server behaviour. There is no mechanical rule that should be running them.
- **Clean separation of concerns.** The game runtime deals with player-facing chain operations (issuer mint, vault swaps, player imports/exports). Business banking is a different category and lives in a different operational tool (Xaman, manually).
- **No player-facing dependency.** Nothing in the game economy or player experience depends on the operating wallet. It can be moved, frozen, or restructured without any impact on the running game.

The operator periodically reviews issuer subscription revenue and moves an appropriate portion to operating manually. The 10% figure mentioned later in this doc is a guideline for that allocation, not an automated split rule.

---

## Subscription Processing

### Flow

```
Player pays $20 RLUSD ──→ Issuer Wallet (held in full)
                              │
                              ├──→ Treasury management — RLUSD / T-Bill allocation
                              │    (issuer-side automated rebalancing — see Treasury Management below)
                              │
                              └──→ Active player signal received
                                   ──→ Issuer issues GOLD_PER_SUBSCRIPTION ──→ Vault (RESERVE)
                                   (per-player economy provisioning — see Active Player Signal below)

Operator manually moves RLUSD from issuer to Operating Wallet
periodically, on a schedule the operator decides. The operating wallet
is not addressable from the game server. See Wallet Architecture above.
```

### Gold Provisioning — Per-Player Economy Allocation

Each active player's presence in the game economy requires a monthly gold allocation to be provisioned into the vault. This is a game design parameter — the amount of gold the economy needs per active player per month — and is **completely independent of subscription revenue**.

| Parameter | Value | Set By |
|---|---|---|
| `GOLD_PER_SUBSCRIPTION` | e.g. 5000 gold | Game policy (admin) |
| Review frequency | Periodic (e.g. monthly/quarterly) | Admin discretion |
| Adjustment criteria | In-game inflation rate, gold velocity, active supply, player count | Economic health indicators |

**Why decoupled from revenue?** If gold issuance were calculated from RLUSD value, it would create an implicit exchange rate — exactly what the compliance model prohibits. The gold amount should be a non-round number that bears no obvious arithmetic relationship to the subscription price or the retained RLUSD amount (e.g. 5000 gold, not 1800 — which would look suspiciously like $18.00 × 100). When this value is adjusted, it's a *game balance patch*, not a monetary policy change.

- The RLUSD is **revenue** (business income)
- The gold is **game economy provisioning** (content creation)
- They happen at the same time but one does not determine the other
- The admin can adjust `GOLD_PER_SUBSCRIPTION` independently of subscription price

### Active Player Signal — How Do We Know a Player Is Active?

Gold provisioning requires a reliable signal that a player can be counted as active for the coming month. The system needs to know *when* to provision each player's gold allocation and *how many* active players the economy is supporting.

**Options considered:**

1. **Login-based activity tracking** — Count a player as active if they log in at least N times in the preceding period (e.g. 5 logins in 30 days). This has the advantage of measuring actual engagement, but introduces complexity: when exactly do you provision the gold? At the Nth login? At month-end for qualifying players? It creates edge cases (player logs in 4 times, misses the threshold by one), requires a batch provisioning job, and could be gamed by trivial login-and-logout cycles. It also means gold provisioning happens at unpredictable times, complicating treasury monitoring.

2. **Subscription receipt** — When a player pays their monthly subscription, they have made a concrete commitment to play for the coming month. This is a clean, discrete event: one payment, one provisioning. No ambiguity about timing, no threshold edge cases, no gaming potential (it costs real money). The subscription receipt is already processed server-side, so gold provisioning piggybacks on existing infrastructure.

**Current approach:** Subscription receipt is used as the active player signal. It is the most reliable and least gameable indicator available, and it produces a clean 1:1 mapping between payment events and gold provisioning events. This may be revisited if the game introduces free-to-play tiers or alternative access models that decouple player activity from subscription payments.

### Server-Side Implementation

XRPL mainnet does not support on-chain Hooks. All processing is server-side:

1. **Transaction monitoring** — game server subscribes to the issuer wallet's transaction stream via XRPL WebSocket (`subscribe` command)
2. **Payment detection** — filter for incoming RLUSD Payment transactions
3. **Subscription verification** — match against pending subscription requests (existing `verify_fungible_payment()` pattern)
4. **Active player provisioning** — subscription receipt triggers the active player signal; issuer issues `GOLD_PER_SUBSCRIPTION` gold to vault via Payment
5. **Subscription activation** — existing `extend_subscription()` flow
6. **Memo tagging** — all transactions carry structured memos (see Memo System below)

The game server does **not** send RLUSD to the operating wallet — that flow is operator-managed manually outside the game runtime.

### Configuration

```python
# Subscription processing
SUBSCRIPTION_PAYMENT_DESTINATION = XRPL_ISSUER_ADDRESS    # issuer is the fiscal centre
GOLD_PER_SUBSCRIPTION = 5000            # gold provisioned per active player per month (game balance parameter)
```

The operating wallet has no game-server config because the game server has no signing authority over it. See Wallet Architecture above.

---

## Treasury Management

### Asset Allocation

The issuer wallet holds business revenue in two forms (after the operator has periodically moved a portion to the operating wallet manually — see Wallet Architecture):

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

The 10% burn is a **deflationary mechanism** — it reduces total gold supply, counteracting issuance from active player provisioning and other sources. It is NOT a revenue mechanism (burning gold doesn't generate RLUSD or any external value).

> **Relationship to player provisioning:** Over time, the system should tend toward equilibrium — gold provisioned for active players is offset by gold burned via sinks. The admin monitors this balance and adjusts `GOLD_PER_SUBSCRIPTION` if provisioning consistently outpaces sinks (inflationary) or vice versa (deflationary squeeze).

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
| `fcm/trust` | Trust line setup | `{"currency":"FCMWheat","limit":"1000000000"}` |
| `fcm/mint` | NFT minting | `{"id":42,"uri":"https://api.fcmud.world/nft/42"}` |
| `fcm/nft-transfer` | NFT transfer/offer | `{"direction":"issuer-to-vault","nftId":"000A...B3F4"}` |
| `fcm/amm-create` | AMM pool creation | `{"asset1":"FCMWheat","asset2":"FCMGold","amount1":"1000","amount2":"500"}` |
| `fcm/amm-deposit` | AMM liquidity deposit | `{"asset1":"FCMWheat","asset2":"FCMGold","amount1":"100","amount2":"50"}` |
| `fcm/amm-withdraw` | AMM liquidity withdrawal | `{"asset1":"FCMWheat","asset2":"FCMGold","lpTokens":"100"}` |
| `fcm/amm-vote` | AMM trading fee vote | `{"asset1":"FCMWheat","asset2":"FCMGold","fee":300}` |
| `fcm/transfer` | Token/XRP transfer between wallets | `{"asset":"XRP","amount":"10"}` |
| `fcm/config` | Account flags/settings/admin ops | `{"action":"set","flag":"DefaultRipple"}` |

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
| `fcm/subscription` | Active player gold provisioning | `{"goldAmount":5000,"policyRate":"GOLD_PER_SUBSCRIPTION"}` |
| `fcm/rebalance` | Treasury RLUSD/T-Bill rebalance | `{"direction":"buy-tbill","amount":"100","rlusdPctBefore":25,"rlusdPctAfter":20}` |
| `fcm/sink-burn` | Gold burned from SINK (10%) | `{"amount":"500","sinkTotal":"5000","burnPct":10}` |

> Issuer-to-operating transfers are operator-managed in Xaman, not generated by the game server. They get whatever memo the operator chooses to attach (or none) and are not part of the game-server memo audit trail.

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

### Phase 2: Operating Wallet (Already Deployed — No Game-Server Work)

The operating wallet exists on-chain today:
- Regular key signed by Key C (Key C is the shared regular key for the wallet)
- Master key disabled
- RLUSD trust line in place

There is **no game-server work** for this phase. The operating wallet is intentionally kept off the game runtime — see Wallet Architecture § "Why the operating wallet is kept off the game server". The operator handles allocation and fiat conversion manually in Xaman.

The only Phase 2 game-side task is adding the issuer trust line for the T-Bill issuer when one is selected, which is properly part of Phase 4 (Treasury Rebalancing).

### Phase 3: Subscription Processing

- Server-side transaction monitor for issuer wallet
- Gold provisioning trigger (issuer → vault) on subscription receipt
- `GOLD_PER_SUBSCRIPTION` configuration and admin adjustment interface

(No automated revenue split — the operator moves RLUSD from issuer to operating wallet manually as part of normal accounting.)

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
