# Legal & Regulatory Compliance

## Overview

FullCircleMUD issues gold as an XRPL token. Players earn gold through gameplay. Gold, resources, and items exist as tokens on the XRPL, tradeable on the ledger's protocol-native decentralised exchange (DEX) and automated market maker (AMM) infrastructure.

**FullCircleMUD does not offer gold redemption.** There is no mechanism by which a player can exchange gold for fiat currency, stablecoins, or any other asset through FullCircleMUD. Players who wish to trade gold do so on the XRPL DEX/AMM — public, protocol-native infrastructure that FullCircleMUD did not build, does not operate, and cannot control.

The game entity participates in AMM pools as part of its game economy management — seeding liquidity for NPC shop pricing, managing gold supply to prevent inflation, and maintaining healthy game balance. This is the same function every MMO operator performs with their virtual economy; the difference is that FullCircleMUD's economy is implemented transparently on a public blockchain.

The compliance strategy rests on two pillars:

1. **No Financial Product Offered.** FullCircleMUD does not offer redemption, does not promise backing, does not advertise a peg, and does not market gold as having real-world monetary value. There is no financial product to regulate.

2. **Blockchain as Database.** The XRPL is FullCircleMUD's database layer. Issuing tokens on XRPL is the implementation mechanism for the game's item and currency systems. The entity's participation in AMM pools is game economy infrastructure — the equivalent of stocking NPC vendor inventories and setting shop prices — not financial market-making.

Compliance obligations are triggered by the enabling of token export on XRPL mainnet. Pre-alpha and alpha deployments on XRPL mainnet with token export disabled (closed-loop model) are not subject to the obligations described herein.

> **Authoritative document:** The full compliance strategy with legal analysis, risk mitigation, internal economy management policy, and language policy is maintained in `ops/COMPLIANCE_LEGAL.md`. This design document is a summary for developers working on the game.

---

## Nature of Gold Token

### Classification

In-game gold is characterised as a **digital collectible** within the meaning of the joint SEC-CFTC interpretive release of March 17, 2026 (Release Nos. 33-11412 and 34-105020). That release explicitly identifies "in-game items" as a digital collectible category and confirms digital collectibles are not securities under federal law.

### What Gold Is

- An in-game currency used for NPC transactions, player-to-player trade, and progression
- A token on the XRPL, giving players true ownership of their in-game currency
- Tradeable on the XRPL's protocol-native DEX/AMM by any holder
- One of many game tokens (alongside resource tokens, item tokens, etc.)

### What Gold Is Not

- Not redeemable through FullCircleMUD for any fiat currency, stablecoin, or other asset
- Not advertised as backed, pegged, collateralised, or having guaranteed value
- Not a payment instrument, stablecoin, e-money token, or security
- Not an investment product

### XRPL DEX and AMM

Trading of gold on the XRPL DEX or AMM is user-initiated activity against protocol-native infrastructure. FullCircleMUD is not a party to such trades and does not custody assets in connection with them.

### NPC Shop Transactions

When a player buys or sells through an in-game NPC shopkeeper, the game reads pricing from the relevant AMM pool and executes the corresponding trade on-chain through a game-operated vault wallet. This is game infrastructure — the on-chain equivalent of a traditional MMO updating a player's inventory at a vendor. The game transacts on its own account as principal, not as agent transmitting value on behalf of players. See `ops/COMPLIANCE_LEGAL.md` §7.4 for the full FinCEN analysis.

---

## Game Economy Management

The entity's AMM activity falls into two distinct categories:

**Resource/gold pools (creator and LP):** The entity creates and provides liquidity to resource/gold AMM pools. These are game infrastructure — NPC shops read pricing from them. This is equivalent to stocking vendor inventories in a traditional MMO.

**Gold/RLUSD or gold/XRP pools (customer only):** The entity does **not** create or provide liquidity to any pool that prices gold against a fiat-adjacent asset or XRP. If such pools exist, they were created by third parties on the public XRPL. The entity may trade against them as a customer for operational purposes (converting income, acquiring gold for economy management, closing pricing discrepancies across pools).

This distinction is critical: the entity builds and maintains game economy infrastructure (resource/gold), but is merely a customer on any market that gives gold a real-world price.

The entity holds company funds like any business. Some are held in RLUSD and XRP on the XRPL because the game economy operates on-chain and the entity needs on-chain assets to operate. This is an operational requirement, not a financial product.

See `ops/COMPLIANCE_LEGAL.md` Part B for internal economy management policy.

---

## Website — No Variant Architecture Required

Under the no-redemption model, all visitors see the same website content worldwide. There are no jurisdictional restrictions and no geo-variant page serving required.

The geo-detection middleware and infrastructure is retained in `web/middleware/geo.py` for potential future use if jurisdiction-specific restrictions become necessary, but `_RESTRICTED_PATHS` is currently empty and no template branches on `geo_variant`.

### Geo-Detection Re-Activation Triggers

Geo-blocking is re-enabled if legal counsel advises that a specific jurisdiction's regulatory framework captures gold tokens and the entity cannot or chooses not to comply with that framework. The most likely trigger is MiCA enforcement determining that non-redeemable game tokens traded near stable values on a public DEX fall within its scope, or a specific country issuing guidance that captures the token under existing financial regulation. The infrastructure is already built — re-enabling is a configuration change (`_RESTRICTED_PATHS` and template branching), not a development task. The decision to geo-block a jurisdiction is made by the operator on counsel's advice, not automatically.

See `design/WEBSITE.md` for current page inventory and status.

---

## Terms of Service — Required Clauses

The Terms of Service cover the following areas. Specific wording to be finalised by legal counsel.

- **Nature of In-Game Assets** — game items, not financial instruments; no guaranteed value; no redemption
- **No Redemption** — no mechanism exists to exchange gold for any currency through FullCircleMUD
- **Trading on External Protocols** — XRPL DEX/AMM is protocol-native, not operated by FullCircleMUD; player trades at own risk
- **Game Economy Management** — entity manages economy at its discretion; game operations, not financial services
- **Issuer Clawback Authority** — entity retains clawback for game integrity (exploits, fraud, ToS violations); does not negate ownership; consistent with standard XRPL issuer practice
- **Alpha and Pre-Export Phases** — closed system during alpha; no export; no entitlement to monetary value

See `design/WEBSITE.md` § Terms of Service for page status.

---

## Data Retention

### Login Geo-Logging

Login geo-logging (IP hash + country code) is gated by `settings.LOG_PLAYER_GEO_DATA` (currently `False`). When disabled, no IP hashes or country codes are recorded. The infrastructure is retained and can be re-enabled if jurisdictional tracking becomes necessary.

The `geo_country` field used here is populated from Cloudflare's `cf-ipcountry` header on the WebSocket handshake. For the transport-layer reasons FCM is WebSocket-only and what telnet/SSH would require to maintain the same compliance posture, see [CONNECTION_TRANSPORT.md](CONNECTION_TRANSPORT.md).

### Game Activity Logs

Standard game activity logs (session times, economic activity, transfer logs) are maintained for game operations, fraud prevention, and dispute resolution. These are operational records, not compliance audit trails.

---

## Gambling Law — Deterministic Distribution + Pre-Disclosure

FullCircleMUD is designed to sit clearly outside the legal definition of gambling in every jurisdiction where the game is available. The compliance strategy addresses the "chance" element of the gambling test directly — at no point does a player pay consideration for an outcome that is uncertain at the time of payment.

### Loot Distribution (spawning)

All item, currency, and knowledge distribution is **algorithmically deterministic**. No dice rolls, no `rand()` calls, no flat % drop chances. The spawn system runs hourly and pushes exact quantities of resources, gold, scrolls, recipes, and rare NFTs onto targets in the world based on calculated budgets from live game state. When a player kills a mob and finds a scroll, the scroll is there because the system placed it — not because of a per-kill roll.

See [ECONOMY.md § Deterministic Push Spawn](ECONOMY.md#deterministic-push-spawn-no-random-loot-rolls) for the mechanical detail and `ops/COMPLIANCE_LEGAL.md` §9.5.3 for the legal framing.

### Variable-Outcome Mechanics (enchanting)

Certain crafting systems produce outcomes that vary between sessions — the most obvious example being **gem enchanting**, where a raw gem is transformed into an enchanted gem with one of many possible effect combinations. Pure determinism would remove the discovery aspect entirely and collapse these into recipe lookups. FullCircleMUD's solution is **pre-disclosure with informed consent**:

1. For each variable-outcome mechanic, a single "next available" slot holds one pre-generated outcome waiting to be claimed.
2. When a player queries the system (e.g. `enchant ruby`), the full details of the waiting outcome are disclosed — effects, restrictions, costs, everything.
3. The player is asked to confirm. Declining does nothing — no payment, no slot change. Confirming applies the outcome, consumes the fee, and generates the next slot.
4. If another player claims the slot between a player's query and their confirmation, the system detects the race (via row-level database lock), re-prompts the second player with the new slot's disclosed outcome, and never charges them for something they did not consent to.

**The compliance test:** at the moment the player commits to paying, do they know exactly what they will receive? With the pre-disclosure model, the answer is always yes. The "uncertain future event" element of the gambling test is absent.

**Internal use of RNG.** The system may use random number generation *internally* to generate the next slot's outcome after a previous slot is consumed. This is not part of any player's transaction — the player never pays for a roll, they pay for a disclosed outcome. The RNG runs in a separate system operation, and the result is not delivered to any player until it has been disclosed in full and explicitly consented to.

**The design commitment** is therefore not "no random number generation anywhere in the codebase." It is the stricter, compliance-relevant commitment:

> **No player will ever pay consideration for an outcome that is not fully disclosed to them before payment.**

Every mechanic where a player spends gold, tokens, items, resources, or time-gated actions must either have a fully deterministic outcome (computable from public game state) or must use the pre-disclosure-with-consent model.

### Design Guardrails

The following must **never** be introduced without a gambling law review by legal counsel:

- Purchasable loot boxes or mystery containers
- Premium currency that can be spent on undisclosed outcomes
- Pay-per-roll or pay-to-re-roll mechanics for crafting, enchanting, or any other system
- Gacha-style mechanics (paying for a draw from a pool without disclosure of the specific item being purchased)
- Any mechanic where a player spends something of value for an outcome not fully disclosed before payment
- Any modification to the pre-disclosure flow for enchanting or similar variable-outcome systems that would reveal the outcome *after* payment rather than *before*

See `ops/COMPLIANCE_LEGAL.md` §9.5 for the full legal analysis, compliance position, and design guardrails.

---

## Transparency Reporting

FullCircleMUD publishes periodic game economy reports to a public GitHub repository. These reports cover gold circulation, token trading activity, player metrics, and operating income/expenses.

These are **game economy health reports**, not financial disclosures. They do not discuss backing, collateralisation, or any concept implying gold is a financial instrument.

See `ops/COMPLIANCE_LEGAL.md` §12 for report framing and the repository disclaimer.

---

## Development Phase Strategy

| Phase | Infrastructure | Token Export | Compliance Obligations |
|---|---|---|---|
| Pre-Alpha | XRPL mainnet, export locked | Disabled — closed loop | None |
| Alpha | XRPL mainnet, export locked | Disabled — closed loop | Minimal — see `ops/COMPLIANCE_LEGAL.md` §10.3 |
| Pre-Mainnet | Transition period | Disabled | Review compliance framework with counsel; prepare website copy; establish entity structure |
| Mainnet Launch | XRPL mainnet, full economy | Enabled | All obligations active |

All development phases deploy on XRPL mainnet with token export disabled (closed-loop model). All tokens are held in the game's vault wallet — no player holds tokens outside the game, no one can interact with AMM pools externally, and the game economy is functionally identical to a traditional MMO's internal economy. The issuer wallet has `lsfAllowClawback` enabled, so any tokens that appear outside the vault are provably illegitimate and can be clawed back.

See `ops/COMPLIANCE_LEGAL.md` §10.3 for the full closed-loop analysis.

---

## Regulatory Review Schedule

| Review Type | Frequency / Trigger |
|---|---|
| Token classification review | Annually, or upon material regulatory development |
| GENIUS Act implementation review | January 2027 (effective date / implementing rules) |
| MiCA review | Annually, or upon new guidance on game tokens |
| Entity AMM activity review | Annually |
| Terms of Service review | Annually |
| Full legal review | Annually |

---

## Legal Process — Pre-Mainnet Blockers

The following must be completed before mainnet launch:

- [ ] Legal counsel engaged with expertise in cryptocurrency regulation, gaming law, and financial services
- [ ] Gold token classification confirmed under no-redemption model
- [ ] Entity AMM activity assessed for registration/licensing obligations
- [ ] GENIUS Act exposure assessed
- [ ] Operating entity structure confirmed (jurisdiction, licensing)
- [ ] Terms of Service finalised and signed off by counsel
- [ ] Compliance framework published to GitHub transparency repository
- [ ] Confirm US player tax reporting position (expected: no 1099 obligation for non-US entity)

---

## Language Policy

See `ops/COMPLIANCE_LEGAL.md` §18 for the full prohibited and acceptable language lists. Key rules:

**Never use (anywhere, public or internal):** "backed", "pegged", "redeemable", "reserve" (in public-facing contexts), "collateralised", "guaranteed value", "earn real money", "cash out", "the gold is real", "fully backed", "investment/returns/yield" (to players).

**Acceptable:** "own your items", "true ownership", "transparent economy", "player-driven economy", "trade with other players", "blockchain-native", "items persist on the XRP Ledger".
