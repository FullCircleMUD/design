# subscriptions.md

> **THIS FILE covers the subscription payment system** — how players pay for game access via XRPL, the subscription lifecycle, command gating for character entry, trial periods, and the payment flow. For the chain-boundary `import`/`export` commands and how they use the subscription helpers, see [import-export.md](import-export.md). For database architecture, see [database.md](database.md). For economic design, see [economy.md](economy.md). For technical implementation details, see `src/game/CLAUDE.md`.

---

## Purpose

Players need an active subscription to play FullCircleMUD. Payment is entirely in-game via XRPL Xaman wallet — no Stripe, no web portal. The subscription system is controlled by the `SUBSCRIPTION_ENABLED` setting — when disabled (default during pre-alpha), all players have free access.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Models](#models)
- [Subscription Lifecycle](#subscription-lifecycle)
- [Payment Flow](#payment-flow)
- [Command Gating](#command-gating)
- [OOC Menu Display](#ooc-menu-display)
- [Trial Period](#trial-period)
- [Configuration](#configuration)
- [Files](#files)
- [Tests](#tests)

---

## Overview

- Monthly subscription paid in RLUSD via XRPL (price stored in DB, adjustable without code changes)
- Staged pricing by development phase — current figures per phase live in `web/templates/website/costs.html` (Pre-Alpha / Alpha / Beta). The website is authoritative for pricing; this doc intentionally does not restate the numbers so they can't drift
- `SUBSCRIPTION_ENABLED=false` during pre-alpha — no payment required
- Payment destination: the issuer wallet (`XRPL_ISSUER_ADDRESS`)
- Subscription expiry stored on the Account typeclass as an `AttributeProperty`
- Payment records stored in separate `subscriptions` database (4th database)
- Replay protection via unique `tx_hash` constraint on payment records
- Game moderator and bot accounts bypass all subscription checks

---

## Architecture

```
subscriptions/              ← 4th Django app with own database
├── apps.py                 ← SubscriptionsConfig
├── db_router.py            ← Routes to "subscriptions" DB alias
├── models.py               ← SubscriptionPlan, SubscriptionPayment
├── utils.py                ← is_subscribed(), get_subscription_status(),
│                              extend_subscription(), grant_trial()
└── migrations/
    └── 0001_initial.py     ← Models + seed "monthly" plan

commands/account_cmds/
└── cmd_subscribe.py        ← OOC command (Xaman payment flow)
```

All subscription checks go through `subscriptions/utils.py` — no command or typeclass checks expiry directly.

---

## Models

### SubscriptionPlan

Registry of available subscription periods. Initially just "monthly".

| Field | Type | Notes |
|-------|------|-------|
| `key` | CharField(30), unique | "monthly" |
| `display_name` | CharField(50) | "Monthly" |
| `duration_days` | PositiveIntegerField | 30 |
| `price` | DecimalField(10,2) | 20.00 |
| `is_active` | BooleanField | Can disable plans |
| `sort_order` | PositiveIntegerField | Display ordering |

Prices are stored in the database, not settings. Adjustable via Django admin or direct DB update.

### SubscriptionPayment

Every payment transaction, for audit and replay protection. Also the durable record of a granted trial — see [One trial per wallet](#one-trial-per-wallet).

| Field | Type | Notes |
|-------|------|-------|
| `account_id` | IntegerField | Evennia AccountDB.id |
| `account_name` | CharField(80) | Account name at time of payment |
| `wallet_address` | CharField(50) | Paying wallet |
| `plan_key` | CharField(30) | "monthly" |
| `amount` | DecimalField(10,2) | Actual amount paid |
| `currency_code` | CharField(40) | "RLUSD" |
| `tx_hash` | CharField(64), unique | XRPL tx hash (replay protection) |
| `old_expiry` | DateTimeField, nullable | Expiry before this payment |
| `new_expiry` | DateTimeField | Expiry after this payment |
| `created_at` | DateTimeField, auto | Payment timestamp |

Indexes on `account_id`, `wallet_address`, and `created_at`.

---

## Subscription Lifecycle

1. **Account creation** → `grant_trial()` sets expiry to now + 48 hours (configurable) and records a zero-amount trial row against the wallet
2. **Active subscription** → player can use all commands normally
3. **Warning zone** (< 48h remaining) → OOC menu shows red warning with countdown
4. **Expiry** → immediate lockout of gated commands. No grace period.
5. **Renewal** → `subscribe` command extends from current expiry if still active, from now if expired

```
Created ──→ Trial (48h) ──→ Active ──→ Warning (<48h) ──→ Expired
                              ↑                              │
                              └───── subscribe (payment) ────┘
```

---

## Payment Flow

The `subscribe` command follows the same Xaman payment pattern as `import`:

1. Player types `subscribe` (OOC)
2. Show current status + available plans from DB
3. Player selects plan → confirmation prompt
4. Create Xaman Payment payload to `SUBSCRIPTION_PAYMENT_DESTINATION` for the plan price in the configured currency
5. Show deeplink → player signs in Xaman app
6. Poll every 2s via `deferToThread` + `delay()` (2-minute timeout)
7. On signed: verify on-chain via `verify_fungible_payment()`
8. On verified: `extend_subscription(account, plan.duration_days)`, create `SubscriptionPayment` record
9. Show confirmation with new expiry date, refresh OOC menu

---

## Command Gating

Two helpers in `subscriptions/utils.py` drive every subscription check in the codebase:

- **`is_subscribed(account)`** — True while the account's `subscription_expires_date` is in the future (also True if the subscription system is disabled, or the account is exempt).
- **`has_paid(account)`** — True if the wallet has ever recorded a *paid* `SubscriptionPayment` row. Trial rows are written to the same table and are excluded by `plan_key`. Exempt accounts return True.

`has_paid` keys on `wallet_address`, not `account_id`. A world rebuild re-issues every primary key, so an account-id lookup would report a player who has paid for a year as never having paid, and silently close export to them. See [account-recovery.md](account-recovery.md).

### Character entry gates (`is_subscribed`):

These commands are gated on an active subscription. They are the touchpoints for *playing* the game — entering a character, creating a new one, or deleting one — so an expired sub blocks them immediately.

| Command | File | Gate placement |
|---|---|---|
| `ic` | `cmd_override_ic.py` | Top of `func()`, before `super().func()` |
| `charcreate` | `cmd_override_charcreate.py` | Top of `func()`, before slot check |
| `chardelete` | `cmd_override_chardelete.py` | Top of `func()`, before arg check |

All show: "Your subscription has expired. Use subscribe to renew." Commands remain visible (not hidden by lock) so players understand what happened and how to fix it.

### Chain-boundary gates (handled by import-export.md)

The `import` and `export` commands use the same `is_subscribed` / `has_paid` helpers but with an asymmetric model — `import` requires both an active sub and a payment on record, while `export` requires only `has_paid` so paid players are never trapped with their assets. The full gate stack, the trial-recycling rationale, and the future OFAC SDN screening design live in [import-export.md](import-export.md).

### Allowed at all times:

`look` (OOC menu), `wallet`, `subscribe`, `quit`, `who`, `sessions`, and all other non-gated commands. Players must always be able to see their status and renew, regardless of subscription state.

---

## OOC Menu Display

The `{subscription}` placeholder in `ooc_appearance_template` shows status between sessions and characters sections.

| State | Display |
|-------|---------|
| Exempt account | Empty string (nothing shown) |
| Subscribed, > 48h remaining | `Subscribed until 09:13 UTC on 13 April 2026` |
| Subscribed, < 48h remaining | Red: `*** Your subscription expires in 27 hours and 56 minutes ***` |
| Expired | Red: `*** Your subscription has expired ***` + `Use subscribe to renew your subscription.` |

Built by `Account._build_subscription_line()` using `get_subscription_status()`.

---

## Trial Period

New accounts receive a free trial, granted from the tail of `Account.create()`. The trial sets `subscription_expires_date` to `now + SUBSCRIPTION_TRIAL_HOURS`.

- Default: 48 hours
- Configurable via `SUBSCRIPTION_TRIAL_HOURS` in settings
- Set to 0 to disable trials entirely
- No-op if the account already has an expiry set
- No message at creation time (no session yet) — the OOC menu shows status on first look

Exempt accounts don't receive trials (they're exempt from subscription checks entirely). The superuser guard sits at the call site: `grant_trial` does not check exemption itself.

### One trial per wallet

A trial is recorded as a zero-amount `SubscriptionPayment` row with `plan_key="trial"` and a synthetic `tx_hash` of `trial:<wallet>`. Because `tx_hash` is unique, a second trial for the same wallet is impossible at the database level, not merely refused by a check.

Two things follow:

- **The trial survives a world rebuild.** `subscription_expires_date` on the account is destroyed with the account; the payment row is not. Recovery can read the later of the two.
- **Trials cannot be farmed.** Deleting an account and signing in again with the same wallet gets no second trial.

This is why `grant_trial` runs from `create()` rather than `at_account_creation` — `wallet_address` is assigned *after* `create_account()` returns, so inside the hook there is no wallet to key on.

---

## Configuration

All subscription settings in `server/conf/settings.py`:

| Setting | Default | Purpose |
|---------|---------|---------|
| `SUBSCRIPTION_ENABLED` | `False` (env var) | Master toggle — when off, all players have free access |
| `SUBSCRIPTION_CURRENCY_CODE` | `"RLUSD"` | Currency for subscription payments |
| `SUBSCRIPTION_CURRENCY_ISSUER` | `""` (env var) | XRPL issuer address for the currency |
| `SUBSCRIPTION_PAYMENT_DESTINATION` | `XRPL_ISSUER_ADDRESS` | Wallet receiving payments |
| `SUBSCRIPTION_TRIAL_HOURS` | `48` | Free trial period for new accounts |
| `SUBSCRIPTION_BYPASS_SUPERUSER` | `True` | Moderator accounts skip subscription checks |

Currency code and issuer are environment-variable driven.

---

## Files

### Created

| File | Purpose |
|------|---------|
| `subscriptions/__init__.py` | Package init |
| `subscriptions/apps.py` | Django app config |
| `subscriptions/db_router.py` | Route to "subscriptions" DB |
| `subscriptions/models.py` | SubscriptionPlan, SubscriptionPayment |
| `subscriptions/utils.py` | Subscription check/extend/trial utilities |
| `subscriptions/migrations/0001_initial.py` | Models + seed monthly plan |
| `commands/account_cmds/cmd_subscribe.py` | Subscribe command (Xaman payment flow) |

### Modified

| File | Change |
|------|--------|
| `server/conf/settings.py` | INSTALLED_APPS, DATABASES, DATABASE_ROUTERS, subscription settings |
| `typeclasses/accounts/accounts.py` | `ooc_appearance_template` + `_build_subscription_line()` + `at_look()` subscription display + the trial grant at the tail of `create()` |
| `commands/account_cmds/cmdset_account_custom.py` | Register CmdSubscribe |
| `commands/account_cmds/cmd_override_ic.py` | Subscription gate (`is_subscribed`) |
| `commands/account_cmds/cmd_override_charcreate.py` | Subscription gate (`is_subscribed`) |
| `commands/account_cmds/cmd_override_chardelete.py` | Subscription gate (`is_subscribed`) |
| `commands/account_cmds/cmd_import.py` | `is_subscribed` + `has_paid` gates — see [import-export.md](import-export.md) |
| `commands/account_cmds/cmd_export.py` | `has_paid` gate only — see [import-export.md](import-export.md) |

---

## Tests

```bash
evennia test --settings settings tests.command_tests.test_subscription_utils
evennia test --settings settings tests.command_tests.test_subscription_gating
evennia test --settings settings tests.command_tests.test_cmd_subscribe
```

- **test_subscription_utils** — `is_subscribed`, `get_subscription_status`, `extend_subscription`, `grant_trial`, `has_had_trial`, `has_paid`, the trial payment row, one-trial-per-wallet across accounts, feature flag disabled behaviour
- **test_subscription_gating** — `ic`, `charcreate`, `chardelete` blocked when expired; allowed when subscribed or for exempt accounts; bypass when disabled. Also covers the `import` and `export` chain-boundary gates (asymmetric `is_subscribed` / `has_paid` model — see [import-export.md](import-export.md))
- **test_cmd_subscribe** — early-return paths (no wallet, exempt, status display, disabled message)
