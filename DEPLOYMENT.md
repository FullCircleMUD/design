# DEPLOYMENT.md

> **THIS FILE covers the CI/CD pipeline, Railway deployment, environment configuration, and branching strategy** for FullCircleMUD. For database architecture (the three-database design, SQLite/PostgreSQL toggle, migration workflow), see **design/DATABASE.md**. For technical implementation details and code patterns, see **src/game/CLAUDE.md**. For economic design, see **design/ECONOMY.md**.

---

## Table of Contents

- [Overview](#overview)
- [Deployment Architecture](#deployment-architecture)
- [Branching Strategy](#branching-strategy)
- [How Code Gets Deployed](#how-code-gets-deployed)
- [Railway Configuration](#railway-configuration)
- [Environment Variables](#environment-variables)
- [Vault Signing & Multisig](#vault-signing--multisig)
- [GitHub Actions CI](#github-actions-ci)
- [Common Scenarios](#common-scenarios)

---

## Overview

FullCircleMUD deploys to **Railway**, a cloud platform that connects directly to GitHub. When code is pushed to a specific branch, Railway automatically builds and deploys it. Each environment (staging, production) is completely isolated — its own server, its own PostgreSQL database, its own environment variables.

The deployment pipeline is designed around three principles:
1. **Environment decides configuration, not code** — the same codebase runs everywhere; environment variables control behavior (see [DATABASE.md](DATABASE.md) for how this works with database backends)
2. **Code flows one direction** — feature branches → staging → production, always via Pull Requests
3. **Migrations run automatically** — Django handles database schema changes on every deploy with zero manual intervention

---

## Deployment Architecture

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Your Laptop │     │  Railway Staging  │     │ Railway Production│
│              │     │                   │     │                   │
│  SQLite      │     │  PostgreSQL       │     │  PostgreSQL       │
│  (automatic) │     │  (automatic)      │     │  (automatic)      │
│              │     │                   │     │                   │
│  Feature     │────→│  staging branch   │────→│  main branch      │
│  branches    │ PR  │  auto-deploys     │ PR  │  auto-deploys     │
└──────────────┘     └──────────────────┘     └──────────────────┘
```

**Railway** hosts the game server and its PostgreSQL database. It connects directly to GitHub — when you merge a Pull Request into a deploy branch, Railway automatically builds and deploys the new code.

Each Railway **environment** (staging, production) is completely isolated:
- Its own PostgreSQL instance with its own data
- Its own set of environment variables (API keys, wallet seeds, network URLs)
- Its own deployment tied to a specific Git branch
- Its own URL/domain

---

## Branching Strategy

```
main                    ← production deploys from here
 └── staging            ← staging deploys from here
      └── feature/*     ← your working branches
```

### Rules

- **Never push directly to `main` or `staging`.** Always use Pull Requests.
- **`main` is production.** It should always be stable. Only merge from `staging` after testing.
- **`staging` is the proving ground.** Merge feature branches here. Test with real PostgreSQL. Break things safely.
- **Feature branches** are where development happens. Branch from `staging`, PR back into `staging`.

### Typical workflow

```
git checkout staging
git pull
git checkout -b feature/new-spell

# ... develop, test locally with SQLite ...

git push -u origin feature/new-spell
# Open PR: feature/new-spell → staging
# GitHub Actions runs tests
# Merge when green

# Railway auto-deploys staging
# Test on staging server

# When ready for production:
# Open PR: staging → main
# Merge
# Railway auto-deploys production
```

---

## How Code Gets Deployed

### The flow

1. **You develop locally** on a feature branch. SQLite, fast iteration, no infrastructure needed.

2. **You open a Pull Request into `staging`.** GitHub Actions runs the test suite automatically. If tests fail, the PR is blocked.

3. **PR merges into `staging`.** Railway sees the new commit on the `staging` branch and automatically:
   - Builds the application
   - Runs the **release command** (database migrations)
   - Restarts the game server with the new code

4. **You test on staging.** Real PostgreSQL, real Evennia server. Verify everything works as expected.

5. **You open a Pull Request from `staging` into `main`.** Same review process.

6. **PR merges into `main`.** Railway deploys to production using the same build → migrate → start sequence.

### What Railway does on every deploy

Railway runs two commands in sequence, configured in `railway.toml`:

```toml
[deploy]
releaseCommand = "evennia migrate && evennia migrate --database xrpl && evennia migrate --database ai_memory"
startCommand = "evennia start -l"
```

- **Release command** runs first — applies any pending database migrations. If there are none, it finishes in under a second and moves on. Django tracks which migrations have already been applied and only runs new ones. This is safe to run on every deploy. For details on how migrations work, see [DATABASE.md § How Database Migrations Work](DATABASE.md#how-database-migrations-work).
- **Start command** runs second — starts the Evennia game server. The `-l` flag keeps it in the foreground (Railway requires this — if the process backgrounds itself, Railway thinks it crashed).

---

## Railway Configuration

### Setting up a new environment

1. **Create a Railway project** and connect it to the GitHub repository (`src/game/`)
2. **Add a PostgreSQL plugin** — Railway provisions a database and automatically injects `DATABASE_URL` into the environment
3. **Set the deploy branch** — `staging` for the staging environment, `main` for production
4. **Add other environment variables** — secrets that the game needs (see below)
5. **Deploy** — Railway builds, runs migrations, starts the server

### railway.toml

This file lives in the repo root and tells Railway how to build and run the application:

```toml
[build]
builder = "nixpacks"

[deploy]
releaseCommand = "evennia migrate && evennia migrate --database xrpl && evennia migrate --database ai_memory"
startCommand = "evennia start -l"
```

---

## Environment Variables

Railway lets you set environment variables per environment. The game needs:

| Variable | Purpose | Set by | Notes |
|----------|---------|--------|-------|
| `DATABASE_URL` | PostgreSQL connection string | Railway (automatic) | Auto-set when you add the PostgreSQL plugin. Controls the SQLite/PostgreSQL toggle — see [DATABASE.md § How the Toggle Works](DATABASE.md#how-the-toggle-works) |
| `SECRET_KEY` | Django secret key for cryptographic signing | You | Generate a random string per environment |
| `XRPL_NETWORK_URL` | XRPL websocket endpoint | You | Testnet: `wss://s.altnet.rippletest.net:51233` (default if not set). Mainnet: `wss://s1.ripple.com:51233`. Only production needs this set — staging and local dev use the testnet default |
| `XRPL_VAULT_WALLET_SEED` | Vault signer key for server-signed transactions | You | Staging: testnet seed. Production: will become `XRPL_VAULT_SIGNER_A_SEED` when multisig is implemented — see [Vault Signing & Multisig](#vault-signing--multisig) |
| `XRPL_ROOT_ADDRESS` | Dev/superuser wallet address | You | |
| `XAMAN_API_KEY` | Xaman wallet API key | You | |
| `XAMAN_API_SECRET` | Xaman wallet API secret | You | |
| `LLM_API_KEY` | OpenRouter API key for LLM NPCs | You | |
| `LLM_EMBEDDING_API_KEY` | OpenAI API key for embeddings | You | |

Locally, these secrets live in `server/conf/secret_settings.py` (not in Git). On Railway, they're environment variables — same values, different delivery mechanism. The `settings.py` file handles both: it tries to import `secret_settings.py` (local), and environment variables can override via Railway's injection.

**Staging and production should use different API keys and wallet addresses.** Staging points at XRPL testnet. Production points at mainnet (when ready). Complete isolation.

---

## Vault Signing & Multisig

### Current state (testnet / staging)

The game server holds `XRPL_VAULT_WALLET_SEED` — a single key with full signing authority over the vault wallet. This is acceptable for testnet where the tokens have no real value.

### Production requirement (mainnet)

For mainnet, the vault wallet must use **XRPL native multisig**. No single compromised key should be able to move funds.

**Architecture:**

```
Game Server (Railway)                Co-signing Service (separate infra)
┌─────────────────────┐             ┌──────────────────────────┐
│                     │             │                          │
│  Player exports     │  POST /sign │  1. Validate request     │
│  gold/NFT           │────────────→│  2. Check business rules │
│                     │             │  3. Co-sign transaction  │
│  Server builds tx   │             │  4. Submit to XRPL       │
│  Signs with key A   │←────────────│  5. Return tx result     │
│                     │   response  │                          │
│  Holds key A only   │             │  Holds key B only        │
└─────────────────────┘             └──────────────────────────┘
```

**How XRPL multisig works:**
- A wallet has a **SignerList** — a set of authorized signing keys with weights
- Each transaction requires signatures meeting a **quorum** (weight threshold)
- Example: key A (weight 1) + key B (weight 1), quorum 2 = both must sign

**The co-signing service** is a small, separate API (not part of the game server) that:
1. Receives a partially-signed transaction from the game server
2. Validates against business rules (expected transaction type, known destination wallet, amount within limits, reasonable request rate)
3. Co-signs with key B and submits to XRPL
4. Returns the result

**Why separate infrastructure:** The whole point is that compromising one system isn't enough. Game server compromised = attacker has key A but can't submit without key B. Co-signer compromised = attacker has key B but can't build valid game transactions without key A.

**Tiered approval (future):**

| Tier | Rule | Approval |
|------|------|----------|
| Auto | Small exports (< 100 gold) | Co-signer approves immediately |
| Delayed | Medium exports (100-1000 gold) | 5-minute delay before co-signing |
| Manual | Large exports (> 1000 gold), bulk, unusual | Queued for manual approval |

**The same signing keys work on both testnet and mainnet** — they're just cryptographic keypairs. The `XRPL_NETWORK_URL` environment variable determines which network receives the signed transaction. This means the full multisig flow can be tested on staging (testnet) before going live.

**Migration path:**
1. Set up the vault wallet's SignerList on-ledger (key A + key B, quorum 2)
2. Deploy the co-signing service with key B on separate infrastructure
3. Update game server's `xrpl_tx.py` to partial-sign and forward to co-signer
4. Rename env var from `XRPL_VAULT_WALLET_SEED` to `XRPL_VAULT_SIGNER_A_SEED`

---

## GitHub Actions CI

A `.github/workflows/ci.yml` in the game repo runs automatically on every Pull Request and push to `staging` or `main`:

- Runs the full test suite (`evennia test --settings settings tests`) using SQLite (fast, no infrastructure needed)
- Blocks merge if tests fail
- Optionally: linting, type checking

This is separate from Railway's deployment — GitHub Actions validates the code, Railway deploys it. A PR must pass CI before it can be merged, and merging is what triggers Railway to deploy.

---

## Common Scenarios

### "I want to reset the staging database"

Delete the PostgreSQL plugin in Railway and re-add it. Fresh database. Then redeploy — the release command runs all migrations from scratch, recreating the schema and seed data.

### "I need to copy production data to staging"

Railway supports database snapshots and imports. Take a backup of production, restore to staging. This gives you realistic data for testing without risking production.

### "Railway deploy failed"

Check Railway's deploy logs. Common causes:
- Migration error (rare — usually a malformed migration file)
- Missing environment variable
- Python dependency issue
- The release command has a typo

Railway keeps the previous deployment running until the new one succeeds, so a failed deploy doesn't take down the live server.

### "I need to run a one-off command on Railway"

Railway provides a shell into the running service. Use it for things like:
- `evennia shell` (Django shell for data queries)
- Running management commands
- Debugging production state

---

## Testnet Reinit (After XRPL Testnet Wipe)

XRPL testnet is periodically wiped by Ripple (~every 90 days). All on-chain state is destroyed: accounts, trust lines, token balances, NFTs, AMM pools. The game DB is unaffected — it's the source of truth. With import/export disabled during alpha, the ledger is purely a settlement layer and player wallets are just identity credentials (Xaman signing).

The `testnet_reinit` management command rebuilds the entire XRPL testnet state from the game DB.

### Usage

```bash
cd EvenniaFCM/FullCircleMUD
source ../venv/bin/activate
evennia testnet_reinit --settings settings --issuer-seed sEdXXX
```

Flags: `--dry-run`, `--skip-amm`, `--skip-nfts`, `--force` (skip confirmation).

### What It Does (8 Phases)

1. **Pre-flight** — verify IS_TESTNET, validate seeds match settings addresses, check wipe state
2. **Fund wallets** — issuer, vault, FakeRLUSD issuer via testnet faucet HTTP API
3. **Configure issuer** — DefaultRipple, AllowTrustLineClawback, set vault as NFT minter
4. **Trust lines** — vault→issuer for all CurrencyType rows; issuer→FakeRLUSD issuer for subscription currency
5. **Issue fungible supply** — for each currency, SUM(FungibleGameState.balance) → Payment from issuer to vault
6. **Create AMM pools** — resource pools (FCMResource/FCMGold) using last ResourceSnapshot prices, proxy token pools (PGold/proxy), all at **0% LP fee**
7. **Mint & reassign NFTs** — re-mint all NFTGameState rows with same uri_id, **5% royalty**, transfer to vault, update nftoken_id in game DB
8. **Cleanup** — mark orphaned XRPLTransactionLog entries

### What Players Experience

- **During reinit:** Game offline (stop server before running, start after)
- **After reinit:** Everything works as before. All progress intact.
- **Subscribed players:** On next `subscribe`, prompted to re-sign FakeRLUSD trust line (existing flow handles this — the script cannot sign on behalf of players)

### Design Decisions

- **Issuer seed as argument** — not in Django settings (only the address is). Passed via `--issuer-seed` to keep out of persistent config.
- **0% AMM LP fee** — standardised on reinit regardless of pre-wipe fee
- **5% NFT royalty** — standardised on reinit regardless of pre-wipe transfer fee
- **AMM price recovery** — resource pools use last `ResourceSnapshot.amm_buy_price` to preserve market prices across wipes
- **Idempotent phases** — trust lines and token issuance are safe to re-run; AMM creation catches "already exists" errors; NFT minting tracks progress via nftoken_id updates

### File

`src/game/blockchain/xrpl/management/commands/testnet_reinit.py`

---

## NFT Metadata API (Separate Service)

### Why a Separate Service?

The game server (Evennia) serves NFT metadata at `GET /nft/<id>` for XRPL marketplace resolution (xrp.cafe, onXRP, etc.). The URI baked into each minted NFToken points to this endpoint. The problem: if the game server is down for maintenance, crashed, or restarting, every NFT in the game becomes unresolvable on-chain. Marketplaces show broken metadata, wallets can't display item details, and listings fail.

The solution: a **standalone API service** (`nft_api` repo) that connects directly to the same PostgreSQL database and serves NFT metadata independently of the game server lifecycle.

### Architecture

A single API instance serves both staging and production by routing on the request hostname. Two custom domains point to the same Railway service — the `Host` header determines which database connection pool is used.

```
┌──────────────────┐                    ┌──────────────────┐
│  Staging         │                    │  Production      │
│  PostgreSQL      │                    │  PostgreSQL      │
│  (XRPL Testnet)  │                    │  (XRPL Mainnet)  │
└────┬─────────────┘                    └─────────────┬────┘
     │                                                │
     ▼                                                ▼
┌──────────────────────────────────────────────────────────┐
│                     NFT API (FastAPI)                      │
│                                                            │
│  api.dev.fcmud.world  ──→  staging DB pool (read-only)     │
│  api.fcmud.world      ──→  production DB pool (read-only)  │
└──────────────────────────────────────────────────────────┘
```

- **One Railway service, two domains, two database pools**
- Both connections are **read-only** — only queries `xrpl_nftgamestate` and `xrpl_nftitemtype`
- If a database isn't configured (e.g. production before beta launch), requests to that domain return 503
- Game servers (staging and production) run as separate Railway services with full R/W access to their respective databases

### Technical Details

| | Game Server | NFT API |
|---|---|---|
| **Repo** | `FullCircleMUD/game` | `FullCircleMUD/nft_api` |
| **Framework** | Django (Evennia) | FastAPI + uvicorn |
| **DB access** | Read/write, all tables | Read-only, 2 tables |
| **Dependencies** | Evennia, Django, XRPL SDK, etc. | FastAPI, psycopg2, python-dotenv |
| **Domains** | game.fcmud.world / dev.fcmud.world | api.fcmud.world / api.dev.fcmud.world |
| **Startup time** | ~30s (full game engine) | ~2s (lightweight API) |

### Environment Variables (Railway)

| Variable | When | Description |
|----------|------|-------------|
| `DATABASE_URL_DEV` | From alpha | Staging PostgreSQL connection string |
| `DATABASE_URL_PROD` | From beta | Production PostgreSQL connection string |
| `NFT_IMAGE_BASE_URL` | Yes | Railway Bucket public URL (see below) |

### NFT Item Images

Item images (one PNG per `prototype_key`) are stored in a **Railway Bucket** — Railway's native S3-compatible object storage. The API constructs image URIs by convention: `NFT_IMAGE_BASE_URL + prototype_key + ".png"`. No image paths are stored in the database.

The bucket is public-read so XRPL marketplaces and wallets can resolve images directly without authentication.

### Endpoints

- `GET /nft/{uri_id}` — XLS-24d compliant JSON metadata for XRPL marketplace resolution
- `GET /health` — health check (includes `environment` and `db_configured` fields)

### XLS-24d Compliance

The metadata format follows the XRPL XLS-24d standard that marketplaces expect when resolving a token's URI. Includes item name, description, category, prototype, image URI, and any per-instance metadata (durability, custom names, etc.).

---

## Summary

The deployment pipeline is fully automated and GitHub-driven:

1. **Develop** on feature branches (local, SQLite)
2. **PR into staging** → GitHub Actions tests → Railway auto-deploys to staging (PostgreSQL)
3. **Test on staging** with real infrastructure
4. **PR into main** → Railway auto-deploys to production (PostgreSQL)

The **NFT Metadata API** runs as a separate Railway service sharing the same PostgreSQL instance, ensuring XRPL marketplace metadata resolution stays available regardless of game server state.

Every deploy runs migrations automatically. Environment variables control all environment-specific configuration. No manual steps between merge and live deployment.
