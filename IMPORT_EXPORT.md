# Import / Export — Chain Boundary

> Canonical design document for moving assets across the chain boundary — between a player's XRPL wallet and their account bank inside the game. For subscription billing, see [SUBSCRIPTIONS.md](SUBSCRIPTIONS.md). For the NFT mirror state machine and `at_post_move` lifecycle, see [INVENTORY_EQUIPMENT.md](INVENTORY_EQUIPMENT.md) § NFT Ownership Model. For the broader economy and AMM model, see [ECONOMY.md](ECONOMY.md). For the legal positions on screening and clawback, see `ops/COMPLIANCE_LEGAL.md` § 9.4 (private).

---

## Purpose

`import` and `export` are the only commands that move value between a player's XRPL wallet and the in-game economy. Everything else — bank deposits, AMM trades, crafting, drops, AMM pricing — operates entirely on game-side state. The chain boundary is therefore a small surface area but a high-stakes one, and it's where every legal, safety, and anti-abuse control gets enforced.

Both commands are OOC, account-level, and operate only on the player's `AccountBank`. To move imported assets onto a character, the player has to walk into a bank room and use `withdraw`. To export from a character, the player walks to a bank room, `deposit`s, then runs `export` from OOC mode.

---

## Table of Contents

- [Surface Area](#surface-area)
- [Gate Stack](#gate-stack)
- [Asymmetric Trial Gating](#asymmetric-trial-gating)
- [Import Pipeline](#import-pipeline)
- [Export Pipeline](#export-pipeline)
- [Mirror State Transitions](#mirror-state-transitions)
- [Settings](#settings)
- [Future Work — OFAC SDN Screening](#future-work--ofac-sdn-screening)
- [Files](#files)

---

## Surface Area

| Command | Direction | Asset types | Tx pattern |
|---|---|---|---|
| `import gold [amount\|all]` | Wallet → bank | Fungible (gold) | Player signs Payment to vault via Xaman (1 sig) |
| `import <resource> [amount\|all]` | Wallet → bank | Fungible (resources) | Player signs Payment to vault via Xaman (1 sig) |
| `import nft [N]` | Wallet → bank | NFT | Player creates sell offer via Xaman → vault accepts (1 player sig + 1 server-signed accept) |
| `export gold [amount\|all]` | Bank → wallet | Fungible (gold) | Vault sends Payment to player wallet (server-signed, no Xaman) |
| `export <resource> [amount\|all]` | Bank → wallet | Fungible (resources) | Vault sends Payment to player wallet (server-signed, no Xaman) |
| `export #<token_id>` | Bank → wallet | NFT | Vault creates sell offer → player accepts via Xaman (1 sig) |

**Why fungibles export server-signed but NFTs need a player signature:** A trust-line on the player's wallet is sufficient permission for the vault to push fungible balances at them — no per-transfer signature required. NFTs use the standard XRPL sell-offer flow because there's no equivalent push mechanism: somebody has to actively accept the offer, and for export that has to be the player.

All XRPL/Xaman calls run in worker threads (`twisted.internet.threads.deferToThread`) so the reactor stays responsive for other players during the 2-minute Xaman polling window.

---

## Gate Stack

Both `import` and `export` run a sequence of gates before any chain interaction. Gates run in this order, top to bottom — the first one that fails returns immediately.

| # | Gate | Source of truth | Failure message |
|---|---|---|---|
| 1 | **Universal kill switch** | `settings.XRPL_IMPORT_EXPORT_ENABLED` | "Import/export is currently disabled." |
| 2 | **Subscription / has_paid** | `subscriptions.utils.is_subscribed` and `has_paid` (asymmetric — see below) | "Your subscription has expired." or "[Import\|Export] is not available during the free trial." |
| 3 | **Bot block** | `settings.BOT_ACCOUNT_USERNAMES`, `settings.BOT_WALLET_ADDRESSES` | "Bot accounts cannot [import\|export] assets." |
| 4 | **Linked wallet** | `account.attributes.get("wallet_address")` | "No wallet linked to your account." |
| 5 | **Bank exists** | `account.db.bank` | "No account bank found." |

**Gates 4 and 5 are defensive — they should be unreachable in practice.** Every account is provisioned with a linked XRPL wallet and an `AccountBank` at creation time (`at_account_creation()`), so a player who reaches the `import`/`export` command should always satisfy both checks. The gates exist to fail loudly rather than crashing if account provisioning is ever incomplete (e.g. partial migration, manual DB editing, or an account created before wallet/bank provisioning was wired up). If either of these fires in production it's a sign of a real provisioning bug, not a player misconfiguration.

**Universal kill switch is absolute.** When `XRPL_IMPORT_EXPORT_ENABLED=False`, *no account* can use either command — paying players, bots, and moderator accounts alike are blocked. This is the alpha-phase default and is the master safety switch for outages, key rotations, vault maintenance, or sanctions incidents.

**Bot accounts** bypass the subscription helpers (`is_subscribed` and `has_paid` both return True for exempt accounts via `_is_exempt`), but `cmd_import.py` and `cmd_export.py` carry a *separate* per-command name/wallet check that explicitly blocks them. Bots can play the game without a subscription but cannot move chain assets in or out — keep that distinction in mind.

---

## Asymmetric Trial Gating

The subscription gate is the only place where `import` and `export` diverge.

| Command | Gates |
|---|---|
| `import` | `is_subscribed` AND `has_paid` |
| `export` | `has_paid` only |

**Why import requires both:** A player bringing chain assets *into* the game must be an actively-paying customer. There's no scenario where a lapsed-or-trial player needs to push more assets into the game — if anything, that's the wrong direction for a player who's stopped engaging.

**Why export requires `has_paid` only:** A paid player must never be trapped with their assets. If they decide to stop subscribing, they need to be able to recover everything they've earned from their account bank. Once `has_paid` is True (i.e. there's at least one `SubscriptionPayment` row for the account), export stays open forever — even after the subscription itself has lapsed.

**The `has_paid` gate alone is enough to close the free-trial recycling exploit.** A player who creates a brand-new account, plays the 48-hour trial, and tries to export their earnings is blocked because they have never recorded a payment. They can keep playing the game, but they cannot move tokens out. To unlock export they have to pay at least once. This is the minimum gate that closes "spam new accounts → play trial → export earnings → repeat" while still honouring the no-trap promise to anyone who's actually paid.

---

## Import Pipeline

### Fungible (gold or resource)

1. Gate stack passes
2. `_get_wallet_balances(wallet)` queries XRPL for the player's balance in the requested currency
3. Validate amount: positive, ≤ wallet balance, "all" resolves to the full balance
4. Y/N confirmation prompt to the player
5. `create_payment_payload()` builds a Xaman payload with a `MEMO_IMPORT` memo carrying `{type, currency, amount}`
6. Player opens the deeplink in Xaman, signs the Payment to `XRPL_VAULT_ADDRESS`
7. Game polls Xaman every 2 seconds (`MAX_POLL_ATTEMPTS = 60` → 2-minute timeout) via `deferToThread` + `delay()`
8. On signed result: `verify_fungible_payment()` confirms on-chain the destination, currency, amount, and issuer all match expectations
9. On verified: `bank.deposit_gold_from_chain(amount, tx_hash)` or `bank.deposit_resource_from_chain(resource_id, amount, tx_hash)` credits the bank and writes the mirror transition

### NFT

1. Gate stack passes
2. `_get_wallet_nfts(wallet)` queries XRPL for the player's NFTs
3. If no number specified, display the wallet NFT list with index numbers and stop
4. With a number, look up the chosen `nftoken_id` in `NFTGameState` to confirm it's a known game NFT with an assigned `item_type`
5. Y/N confirmation
6. `create_nft_sell_offer_payload()` builds a Xaman sell-offer payload (player → vault, price 0)
7. Player signs in Xaman; game polls until resolved
8. On signed: `_accept_nft_import()` runs in a worker thread — fetches the resulting transaction, extracts the offer ID from `meta`, then calls `accept_nft_sell_offer()` server-signed
9. On accepted: `BaseNFTItem.spawn_into(nftoken_id, bank, tx_hash=accept_tx_hash)` materialises the in-game item into the player's bank, the `at_post_move` hook fires the `ONCHAIN → ACCOUNT` mirror transition

### Failure modes

- **Xaman timeout** (2 minutes) — "Timed out waiting for Xaman signing."
- **Player rejects** — "Payment was rejected." / "NFT sell offer was rejected."
- **On-chain verification fails** — "On-chain verification failed." with the tx hash for admin follow-up
- **NFT not recognised** — "<name> is not a recognised game NFT." (the game has no `NFTGameState` row for this token)
- **NFT has no item type** — "<name> has no item type assigned — cannot import." (placeholder NFT, never minted as a game item)
- **Disconnect mid-flow** — `_connected(account)` checks every reactor callback; if the account has no active sessions the flow drops silently

All Xaman calls run in worker threads via `deferToThread` so a 2-minute polling cycle never blocks the reactor. The same `_msg(account, ...)` helper is used everywhere — it no-ops if the account has disconnected, so timed-out responses never crash on a dead session.

---

## Export Pipeline

### Fungible (gold or resource)

1. Gate stack passes
2. Validate amount: positive, ≤ bank balance, "all" resolves to the bank balance
3. `_check_trust_line(wallet, currency_code)` — without a trust line on the player's wallet for this currency, the vault cannot send. Failure surfaces a help message with the trust-line setup instructions
4. Y/N confirmation
5. `_send_payment(wallet, currency_code, amount, memos)` — vault server-signs and broadcasts a Payment to the player's wallet with a `MEMO_EXPORT` memo
6. On success: `bank.withdraw_gold_to_chain(amount, tx_hash)` or `bank.withdraw_resource_to_chain(resource_id, amount, tx_hash)` debits the bank and writes the mirror transition

No Xaman flow on the export side for fungibles — the trust line *is* the player's standing permission, so the vault pushes directly.

### NFT

1. Gate stack passes
2. Look up the NFT by token ID in the bank's contents — must be present and owned via the bank
3. Y/N confirmation
4. Vault server-signs an NFT sell offer (vault → player, price 0)
5. Game returns the offer ID and a Xaman accept-offer deeplink to the player
6. Player signs the accept in Xaman; game polls until resolved
7. On accepted: the NFT object is deleted from the bank, the `at_object_delete` hook fires the `ACCOUNT → ONCHAIN` mirror transition with the accept tx hash stashed on `obj.ndb.pending_tx_hash`

### Failure modes

- **No trust line** — vault can't push the asset; player is shown how to set up the trust line for the currency code in question
- **Insufficient bank balance** — "Your bank only has X gold." / "Your bank only has X <unit> of <name>."
- **Tx broadcast failure** — surfaces the error message and the partial tx hash if any (admin follow-up)
- **Player rejects the NFT accept** — "NFT accept was rejected." (NFT stays with the vault since the offer was vault-signed; vault later cleans up the offer)

---

## Mirror State Transitions

Every successful import/export call triggers a mirror state transition through Evennia's `at_post_move` / `at_object_delete` hooks. The chain-boundary transitions are a subset of the full state machine documented in [INVENTORY_EQUIPMENT.md](INVENTORY_EQUIPMENT.md) § NFT Ownership Model.

| Action | Source | Destination | Service call | Mirror flow |
|---|---|---|---|---|
| `import gold/resource` | wallet | bank | `bank.deposit_gold_from_chain` / `deposit_resource_from_chain` | n/a (fungible — handled by GoldService / FungibleService) |
| `import nft` | wallet | bank | `BaseNFTItem.spawn_into(token_id, bank, tx_hash=...)` | `ONCHAIN → ACCOUNT` |
| `export gold/resource` | bank | wallet | `bank.withdraw_gold_to_chain` / `withdraw_resource_to_chain` | n/a (fungible) |
| `export nft` | bank | wallet | `obj.delete()` (with `obj.ndb.pending_tx_hash` stashed) | `ACCOUNT → ONCHAIN` |

For NFTs the in-game object's lifecycle and the on-chain ownership are kept in lockstep by the hook dispatch — there is no separate "blockchain sync" step that the import/export commands have to call manually.

---

## Settings

All in `server/conf/settings.py`:

| Setting | Default | Purpose |
|---|---|---|
| `XRPL_IMPORT_EXPORT_ENABLED` | `False` (env var) | Universal kill switch — when off, both commands return "currently disabled" before any other check |
| `XRPL_VAULT_ADDRESS` | env var | Destination for fungible imports, source for fungible exports, and counterparty on all NFT sell offers |
| `XRPL_ISSUER_ADDRESS` | env var | The currency issuer expected on every imported fungible (verified by `verify_fungible_payment`) |
| `XRPL_GOLD_CURRENCY_CODE` | env var | Currency code used for in-game gold (e.g. `FCMGold` or its hex form) |
| `BOT_ACCOUNT_USERNAMES` | `[]` | Account names blocked from import/export |
| `BOT_WALLET_ADDRESSES` | `{}` | Wallet addresses blocked from import/export |

The kill switch defaults to `False` during alpha. When the alpha phase ends and player wallets are allowed to hold game tokens, this flag flips to `True` and the rest of the controls take effect.

---

## Future Work — OFAC SDN Screening

Sanctions screening will become a required gate **between the subscription/has_paid check and the bot block**, before any chain interaction. The full legal position lives in `ops/COMPLIANCE_LEGAL.md` § 9.4 (private); this section documents the design of the runtime gate and the supporting infrastructure.

> **Status: planned, not yet implemented.** The doc captures the intended behaviour so the next time anyone touches `cmd_export.py` they know what's coming and don't paint themselves into a corner.

### What gets screened

The entity will screen against the **OFAC Specially Designated Nationals (SDN) list**, which is publicly available from the US Treasury in CSV / XML / JSON. Two screening modes:

1. **Real-time screening on export.** Before any successful `export` call moves value off-chain, the player's wallet address and identity (name, email) are checked against the current SDN list. A match blocks the export immediately.
2. **Periodic batch screening of all active accounts.** A scheduled job re-screens every active player account and associated XRPL wallet against the latest SDN list. New SDN designations after a player has already exported are caught here.

Chain analysis (tracing wallet transaction histories or cluster associations) is **out of scope**. The entity does not impose more obligation on itself than good-faith published-list screening.

### What gets checked

- Player-provided XRPL wallet addresses checked against SDN-listed wallet addresses
- Player identity information (name, email) checked against SDN-listed names and aliases

### Enforcement on a screening match

If a screening match is identified:

1. **Immediate account suspension** — the player's account is suspended. They cannot log in, earn tokens, or interact with the economy.
2. **Export block** — any pending or future export from the flagged account is blocked.
3. **Token clawback** — if tokens were already exported to the flagged wallet (i.e. the match is identified by a periodic batch run after an earlier export), the entity exercises its XRPL clawback authority to recover all tokens from the player's trust lines. The XRPL issuer account is configured with `lsfAllowClawback` enabled, giving the entity the technical capability to execute this recovery.
4. **Internal record** — the match, the screening data, the actions taken, and the date are documented for the entity's own operational purposes.
5. **No notification to OFAC or other authority** — the entity is not a US person and has no filing obligation. If counsel later advises that a notification obligation exists under Paraguayan law or infrastructure-partner agreements, the entity will comply at that point.
6. **False-positive review** — SDN screening produces false positives (common names, partial matches). Before executing suspension or clawback, the entity applies reasonable judgement to assess whether the match is credible. A bare match on "John Smith" without corroborating information does not trigger enforcement. Both enforcement actions and false-positive dismissals are documented.

### Implementation notes (for the eventual build)

- **New gate position:** screening check goes between gate 2 (subscription/has_paid) and gate 3 (bot block) in the gate stack table above. It would also run from a scheduled job (separate code path) for batch re-screening.
- **List ingestion:** download the SDN list (CSV or JSON) on a daily schedule, normalise into an indexed lookup table, store in a dedicated `sanctions` Django app or as a JSON snapshot in the deploy bundle. Versioned by the publication date of the source.
- **Suspension state:** account-level flag (e.g. `account.db.sanctions_suspended`) checked by the gate. Suspension blocks login (`at_pre_login` hook) in addition to import/export, so the player can't accumulate more in-game value while flagged.
- **Clawback authority:** already in place — `lsfAllowClawback` is enabled on the issuer account. The runtime path needs a privileged admin command that walks the player's known trust lines and issues clawback transactions.
- **Audit log:** all screening events, both matches and clean checks, written to a sanctions audit table. Match enforcement actions written separately with the false-positive review reasoning.
- **Why not at import:** the legal exposure is on the *outflow* side — the entity facilitating value to a sanctioned party. Imports bring assets *in* and don't transfer value to the player. Gating import on SDN screening would be defensible but isn't required, and adds a UX cost (a flagged player can't even bring in a gift trade) without a corresponding compliance benefit. The decision is to gate export only.

### Why this lives in design, not just ops

The compliance positions belong in `ops/COMPLIANCE_LEGAL.md`. The runtime gate stack, suspension state model, and clawback command are *implementation* concerns and need to live alongside the code that enforces them. This section is the design contract between the legal positions and the eventual code.

---

## Files

| File | Purpose |
|---|---|
| `commands/account_cmds/cmd_import.py` | `import` command — Xaman-driven inbound flow for gold, resources, and NFTs |
| `commands/account_cmds/cmd_export.py` | `export` command — vault-signed outbound flow for fungibles, sell-offer flow for NFTs |
| `subscriptions/utils.py` | `is_subscribed`, `has_paid`, `_is_exempt` — gates 2 and 3 |
| `blockchain/xrpl/xrpl_tx.py` | `verify_fungible_payment`, `get_wallet_balances`, `get_wallet_nfts`, `accept_nft_sell_offer`, `_extract_offer_id`, `_check_trust_line`, server-signed payment helpers |
| `blockchain/xrpl/xaman.py` | `create_payment_payload`, `create_nft_sell_offer_payload`, `get_payload_status` — Xaman API client |
| `blockchain/xrpl/memo.py` | Memo builders (`MEMO_IMPORT`, `MEMO_EXPORT`, `MEMO_NFT_IMPORT`, `MEMO_NFT_EXPORT`) — provenance trail on every chain transaction |
| `typeclasses/items/base_nft_item.py` | `BaseNFTItem.spawn_into()`, `at_post_move`, `at_object_delete` — drives the mirror state transitions on import/export of NFTs |
| `typeclasses/mixins/fungible_inventory.py` | `deposit_gold_from_chain`, `withdraw_gold_to_chain`, `deposit_resource_from_chain`, `withdraw_resource_to_chain` — bank-side fungible service entry points |

---

## Tests

```bash
evennia test --settings settings tests.command_tests.test_cmd_import
evennia test --settings settings tests.command_tests.test_cmd_export
evennia test --settings settings tests.command_tests.test_subscription_gating
evennia test --settings settings tests.command_tests.test_subscription_utils
```

- **test_cmd_import / test_cmd_export** — kill-switch, wallet validation, balance validation, fungible flow happy path, NFT flow happy path
- **test_subscription_gating** — `import` blocked when expired or trial-only, `export` blocked while on trial but allowed after payment even when expired, all bypass when `SUBSCRIPTION_ENABLED=False`
- **test_subscription_utils** — `is_subscribed`, `has_paid`, `extend_subscription`, `grant_trial`, exempt-account behaviour, persistence-after-expiry semantics
