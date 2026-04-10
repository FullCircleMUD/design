# SUBSCRIPTIONS.md

> **THIS FILE covers the subscription payment system** — how players pay for game access via XRPL, the subscription lifecycle, command gating, trial periods, and the payment flow. For database architecture, see **design/DATABASE.md**. For economic design, see **design/ECONOMY.md**. For technical implementation details, see **src/game/CLAUDE.md**.

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
- Staged pricing by development phase:
  - **Pre-Alpha:** Free — `SUBSCRIPTION_ENABLED=false`, no payment required
  - **Alpha:** 2 RLUSD/month — covers hosting and LLM costs
  - **Beta / Full Release:** 20 RLUSD/month — full subscription
- Payment destination: the issuer wallet (`XRPL_ISSUER_ADDRESS`)
- Subscription expiry stored on the Account typeclass as an `AttributeProperty`
- Payment records stored in separate `subscriptions` database (4th database)
- Replay protection via unique `tx_hash` constraint on payment records
- Superuser and bot accounts bypass all subscription checks

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

Every payment transaction, for audit and replay protection.

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

1. **Account creation** → `grant_trial()` sets expiry to now + 48 hours (configurable)
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

### Blocked when expired:

| Command | File | Gate placement |
|---------|------|----------------|
| `ic` | `cmd_override_ic.py` | Top of `func()`, before `super().func()` |
| `charcreate` | `cmd_override_charcreate.py` | Top of `func()`, before slot check |
| `chardelete` | `cmd_override_chardelete.py` | Top of `func()`, before arg check |
| `import` | `cmd_import.py` | After kill-switch check, before bot check |

All show: "Your subscription has expired. Use subscribe to renew."

Commands remain visible (not hidden by lock) so players understand what happened and how to fix it.

### Allowed when expired:

`look` (OOC menu), `export`, `wallet`, `subscribe`, `quit`, `who`, `sessions`, and all other non-gated commands.

**Design rationale:** Players must always be able to see their status, manage their wallet, export assets, and renew. Blocking `export` would trap player assets.

---

## OOC Menu Display

The `{subscription}` placeholder in `ooc_appearance_template` shows status between sessions and characters sections.

| State | Display |
|-------|---------|
| Exempt (superuser/bot) | Empty string (nothing shown) |
| Subscribed, > 48h remaining | `Subscribed until 09:13 UTC on 13 April 2026` |
| Subscribed, < 48h remaining | Red: `*** Your subscription expires in 27 hours and 56 minutes ***` |
| Expired | Red: `*** Your subscription has expired ***` + `Use subscribe to renew your subscription.` |

Built by `Account._build_subscription_line()` using `get_subscription_status()`.

---

## Trial Period

New accounts receive a free trial on creation (in `at_account_creation()`, after bank creation). The trial sets `subscription_expires_date` to `now + SUBSCRIPTION_TRIAL_HOURS`.

- Default: 48 hours
- Configurable via `SUBSCRIPTION_TRIAL_HOURS` in settings
- Set to 0 to disable trials entirely
- No-op if account already has an expiry set
- No message at creation time (no session yet) — the OOC menu shows status on first look

Superuser accounts don't receive trials (they're exempt from subscription checks entirely).

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
| `SUBSCRIPTION_BYPASS_SUPERUSER` | `True` | Superusers skip subscription checks |

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
| `typeclasses/accounts/accounts.py` | `ooc_appearance_template` + `_build_subscription_line()` + `at_look()` subscription display + `at_account_creation()` trial grant |
| `commands/account_cmds/cmdset_account_custom.py` | Register CmdSubscribe |
| `commands/account_cmds/cmd_override_ic.py` | Subscription gate |
| `commands/account_cmds/cmd_override_charcreate.py` | Subscription gate |
| `commands/account_cmds/cmd_override_chardelete.py` | Subscription gate |
| `commands/account_cmds/cmd_import.py` | Subscription gate |

---

## Tests

```bash
evennia test --settings settings tests.command_tests.test_subscription_utils
evennia test --settings settings tests.command_tests.test_subscription_gating
evennia test --settings settings tests.command_tests.test_cmd_subscribe
```

- **test_subscription_utils** (22 tests) — is_subscribed, get_subscription_status, extend_subscription, grant_trial, feature flag disabled behaviour
- **test_subscription_gating** (13 tests) — ic, charcreate, chardelete, import blocked when expired, allowed when subscribed/superuser, bypass when disabled
- **test_cmd_subscribe** (5 tests) — early-return paths (no wallet, exempt, status display, disabled message)
