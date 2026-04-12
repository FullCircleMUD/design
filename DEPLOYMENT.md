# DEPLOYMENT.md

> **THIS FILE covers the CI/CD pipeline, Railway deployment, environment configuration, and branching strategy** for FullCircleMUD. For database architecture (the three-database design, SQLite/PostgreSQL toggle, migration workflow), see **design/DATABASE.md**. For technical implementation details and code patterns, see **src/game/CLAUDE.md**. For economic design, see **design/ECONOMY.md**.

---

## Table of Contents

- [Overview](#overview)
- [Deployment Architecture](#deployment-architecture)
- [Branching Strategy](#branching-strategy)
- [How Code Gets Deployed](#how-code-gets-deployed)
- [Railway Configuration](#railway-configuration)
- [Setting Up a New Railway Project from Scratch](#setting-up-a-new-railway-project-from-scratch)
- [Environment Variables](#environment-variables)
- [Database & Migrations on Railway](#database--migrations-on-railway)
- [Networking & Domain Setup](#networking--domain-setup)
- [Vault Signing & Multisig](#vault-signing--multisig)
- [GitHub Actions CI](#github-actions-ci)
- [Common Scenarios](#common-scenarios)
- [NFT Metadata API (Separate Service)](#nft-metadata-api-separate-service)
- [Troubleshooting](#troubleshooting)

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

### Staging ↔ Production XRPL Strategy

Both staging and production point at the **same XRPL mainnet infrastructure** — same vault, same issuer, same AMM pools, same NFTs. The difference is how transactions are signed:

- **Production** uses the co-signer's **production API key** (`API_KEY`). Transactions are fully signed, combined, and submitted to the XRPL ledger. On-chain state changes are real.
- **Staging** uses the co-signer's **dev API key** (`DEV_API_KEY`). Transactions go through the full signing pipeline (build, autofill, sign with Key A, send to co-signer, validate business rules, sign with Key B, combine) but the final XRPL submission is **skipped**. The co-signer returns a mock success (`tx_hash`, `engine_result`) so the game server can complete its internal state transitions.

This means staging:
- **Reads real on-chain data** — AMM prices, reserve balances, NFT state are all live mainnet values
- **Exercises the full code path** — every line of transaction-building, signing, and co-signing code runs exactly as it would in production
- **Cannot modify on-chain state** — no gold moves, no NFTs transfer, no AMM trades execute on the ledger
- **Drifts from on-chain reality** over time — staging's internal mirror DB diverges because it records state changes that never happened on-chain

**Resynchronisation:** When staging drift needs to be corrected, run `sync_reserves` and `sync_nfts` on the staging server. These commands read the real XRPL state and reset the mirror DB to match, bringing staging back in line with production's on-chain reality.

```
┌─────────────────────────────────────────────────────────┐
│                    XRPL Mainnet                         │
│  (AMM pools, vault, issuer, NFTs — shared by both)     │
└──────────┬──────────────────────────┬───────────────────┘
           │ read                     │ read + write
           │                          │
┌──────────▼──────────┐    ┌──────────▼──────────────┐
│  Staging            │    │  Production              │
│  DEV_API_KEY        │    │  API_KEY                 │
│  mock submit        │    │  real submit             │
│  drift → resync     │    │  on-chain = in-game      │
└─────────────────────┘    └──────────────────────────┘
```

---

## Branching Strategy

```
main                    ← production deploys from here
 └── dev                ← development branch, merge to main when ready
      └── feature/*     ← your working branches
```

### Rules

- **`main` is production.** It should always be stable. Only merge from `dev` after testing.
- **`dev` is the working branch.** Develop here or on feature branches. Merge to `main` when ready to deploy.
- **Feature branches** are where development happens. Branch from `dev`, PR back into `dev`.

### Typical workflow

```
git checkout dev
git pull
git checkout -b feature/new-spell

# ... develop, test locally with SQLite ...

git push -u origin feature/new-spell
# Open PR: feature/new-spell → dev
# Merge when ready

# When ready for production:
git checkout main
git merge dev
git push origin main
# Railway auto-deploys production
git checkout dev
```

---

## How Code Gets Deployed

### The flow

1. **You develop locally** on a feature branch or `dev`. SQLite, fast iteration, no infrastructure needed.

2. **You merge to `main`.** Railway sees the new commit and automatically:
   - Builds the application (Nixpacks)
   - Starts the container
   - Runs `deploy_migrate.py` (database migrations)
   - Starts the Evennia game server

3. **Migrations are incremental.** Django only applies new migrations — existing data is preserved. The `deploy_migrate.py` script handles this automatically.

### What Railway does on every deploy

Railway runs the `startCommand` from `railway.toml`:

```toml
[build]
builder = "nixpacks"

[deploy]
startCommand = "python deploy_migrate.py && evennia start -l"
```

- **`deploy_migrate.py`** runs first — installs pgvector extension, then applies any pending database migrations. If there are none, it finishes in under a second and moves on. Django tracks which migrations have already been applied and only runs new ones.
- **`evennia start -l`** runs second — starts the Evennia game server. The `-l` flag keeps it in the foreground (Railway requires this — if the process backgrounds itself, Railway thinks it crashed).

### deploy_migrate.py

This script exists because Evennia's built-in `evennia migrate` command doesn't reliably pick up `DATABASE_URL` for all database aliases on Railway. The script:

1. Sets up Django directly (bypassing Evennia's launcher)
2. Installs the pgvector extension (required by ai_memory models) using autocommit to avoid transaction locks
3. Runs `manage.py migrate` with verbosity output
4. Catches and logs migration errors gracefully so the server can still start

---

## Railway Configuration

### railway.toml

This file lives in the game repo root and tells Railway how to build and run the application:

```toml
[build]
builder = "nixpacks"

[deploy]
startCommand = "python deploy_migrate.py && evennia start -l"
```

**Important:** Do NOT use `releaseCommand` — it runs in a separate context where environment variables may not be available. All migration logic is in `startCommand` via `deploy_migrate.py`.

---

## Setting Up a New Railway Project from Scratch

This is the complete procedure to deploy FullCircleMUD to a fresh Railway project with zero manual database intervention.

### Step 1: Create the Railway project

1. Create a new project on Railway
2. Connect it to the GitHub repository (`FullCircleMUD/game`)
3. Set the deploy branch to `main`

### Step 2: Add PostgreSQL

1. Add a **PostgreSQL** plugin to the project
2. Railway provisions a database and creates connection variables automatically
3. Ensure a **volume** is attached to the Postgres service with mount path `/var/lib/postgresql/data` (Railway should do this automatically, but verify)

### Step 3: Set environment variables on the GAME SERVICE

**CRITICAL:** All variables must be set on the **game service**, not as shared variables. Shared variable references (`${{Postgres.DATABASE_URL}}`) do not reliably resolve on Railway.

Set `DATABASE_URL` as a service variable on the game service with value `${{Postgres.DATABASE_URL}}`. Verify it resolves to an actual connection string (not empty). If it shows empty, paste the actual connection string from the Postgres service's Variables tab (`DATABASE_URL` — the private/internal one).

Set all other required variables (see [Environment Variables](#environment-variables) below).

### Step 4: Configure networking

1. **Game service** > **Settings** > **Networking**
2. Set port to **8080**
3. **Generate a Railway domain** (e.g. `game-production-xxxx.up.railway.app`) — this is needed for websocket routing
4. **Add custom domain** `fcmud.world` (or your domain)
5. Update DNS:
   - `CNAME @ → <railway-provided-target>.up.railway.app`
   - `TXT _railway-verify → <verification-string>`
6. Set `WEBSOCKET_CLIENT_URL` environment variable to `wss://<railway-domain>.up.railway.app/ws`

### Step 5: Deploy

Push to `main` or trigger a manual deploy. The deployment will:
1. Build the container
2. Install pgvector extension
3. Run all migrations from scratch (creates all 58 tables)
4. Create the superuser (from `EVENNIA_SUPERUSER_USERNAME`/`EVENNIA_SUPERUSER_PASSWORD` env vars)
5. Start the game server

No manual SQL required. No manual database intervention.

### Step 6: Verify

- Access `https://fcmud.world` — should show the Evennia web client
- Access `https://<railway-domain>.up.railway.app` — should also work
- Check Postgres Database tab — should show ~58 tables
- Connect to the game and verify the superuser account works

---

## Environment Variables

All variables are set on the **game service** (not shared variables).

### Required

| Variable | Purpose | Example |
|----------|---------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `${{Postgres.DATABASE_URL}}` or paste actual string |
| `SECRET_KEY` | Django cryptographic signing key | Random string, unique per environment |
| `EVENNIA_SUPERUSER_USERNAME` | Auto-creates superuser on first boot | `admin` |
| `EVENNIA_SUPERUSER_PASSWORD` | Superuser password | Strong password |
| `DEFAULT_ACCOUNT_PASSWORD` | Default password for new player accounts | |

### XRPL / Blockchain

| Variable | Purpose | Example |
|----------|---------|---------|
| `XRPL_NETWORK_URL` | XRPL websocket endpoint | `wss://xrplcluster.com` (mainnet) |
| `XRPL_ISSUER_ADDRESS` | Game token issuer wallet address | |
| `XRPL_VAULT_ADDRESS` | Vault wallet address | |
| `XRPL_VAULT_WALLET_SEED` | Vault signer key (single-sig: master seed, multisig: key A) | |
| `XRPL_IMPORT_EXPORT_ENABLED` | Enable player import/export | `true` or `false` |
| `SUPERUSER_XRPL_WALLET_ADDRESS` | Dev/superuser wallet address | |

### Multisig (when enabled)

| Variable | Purpose | Staging vs Production |
|----------|---------|----------------------|
| `XRPL_MULTISIG_ENABLED` | Enable multisig flow (`true`/`false`) | Same on both |
| `XRPL_COSIGNER_URL` | Co-signing service URL | Same on both (one co-signer serves both) |
| `XRPL_COSIGNER_API_KEY` | Co-signer API key | **Staging: `DEV_API_KEY`** (mock submit). **Production: `API_KEY`** (real submit). This is the only variable that differs between environments for XRPL operations. |

### Xaman Wallet

| Variable | Purpose |
|----------|---------|
| `XAMAN_API_KEY` | Xaman wallet API key |
| `XAMAN_API_SECRET` | Xaman wallet API secret |

### Subscriptions

| Variable | Purpose |
|----------|---------|
| `SUBSCRIPTION_ENABLED` | Enable subscription system (`true`/`false`) |
| `SUBSCRIPTION_CURRENCY_CODE` | Payment currency (default: `RLUSD`) |
| `SUBSCRIPTION_CURRENCY_ISSUER` | Issuer of subscription currency |

### LLM / AI

| Variable | Purpose |
|----------|---------|
| `LLM_API_KEY` | OpenRouter API key for NPC conversations |
| `LLM_EMBEDDING_API_KEY` | OpenAI API key for embeddings |

### Bot Testing

| Variable | Purpose | Format |
|----------|---------|--------|
| `BOT_LOGIN_ENABLED` | Enable bot login | `true` or `false` |
| `BOT_ACCOUNT_USERNAMES_JSON` | Bot usernames | `["billy","bianca"]` |
| `BOT_WALLET_ADDRESSES_JSON` | Bot wallet addresses | `{"billy":"rAddr1","bianca":"rAddr2"}` (valid JSON, no trailing commas) |
| `BOT_DEFAULT_PASSWORD` | Default bot password | |
| `BOT_PASSWORDS_JSON` | Per-bot passwords | `{"billy":"pw1","bianca":"pw2"}` |

### Networking

| Variable | Purpose | Example |
|----------|---------|---------|
| `WEBSOCKET_CLIENT_URL` | Websocket URL for web client | `wss://game-production-xxxx.up.railway.app/ws` |

### Auto-set by Railway (do NOT set manually)

| Variable | Purpose |
|----------|---------|
| `PORT` | Railway's dynamic port assignment — Evennia reads this automatically |

---

## Database & Migrations on Railway

### How it works

Locally, the game uses 4 separate SQLite databases (default, xrpl, ai_memory, subscriptions) with database routers directing queries to the correct file. On Railway, all 4 aliases point to the **same** PostgreSQL instance. The database routers are **automatically disabled** when `DATABASE_URL` is set, since they're unnecessary (and harmful) when all aliases share one database.

This means:
- A single `migrate` command creates all tables in one Postgres instance
- No need for `--database xrpl` or `--database ai_memory` flags on Railway
- Locally, `evennia migrate` + `evennia migrate --database xrpl` etc. still works as before

### deploy_migrate.py

The `deploy_migrate.py` script handles all migration logic on Railway:

```python
# 1. Install pgvector extension (autocommit to avoid transaction locks)
# 2. Run django migrate (single call, no routers, all apps)
# 3. Catch errors gracefully so server can still start
```

**Why not use `evennia migrate`?** Evennia's launcher has its own database initialization path that doesn't reliably pick up `DATABASE_URL` for custom database aliases. By using Django directly (`call_command("migrate")`), we ensure all migrations run against Postgres.

### pgvector

The `ai_memory` app uses pgvector for embedding-based semantic search. The pgvector extension must be installed before migrations that create vector columns. `deploy_migrate.py` handles this automatically with `CREATE EXTENSION IF NOT EXISTS vector`.

The `ai_memory.0002_pgvector` migration is marked `atomic = False` because HNSW index creation blocks when run inside a transaction on Railway's Postgres.

### Migration workflow for new features

1. Develop locally, create migrations with `evennia makemigrations`
2. Test locally with `evennia test --settings settings tests`
3. Merge to `dev`, then to `main`
4. Railway auto-deploys: `deploy_migrate.py` applies new migrations incrementally
5. Existing data is preserved — Django only runs migrations not yet in `django_migrations`

### Database persistence

Railway Postgres persists data between deploys via mounted volumes. The game service container is ephemeral (rebuilt on each deploy), but the Postgres service and its volume are persistent. Deploying new code does NOT wipe the database.

---

## Networking & Domain Setup

### Ports

Evennia runs multiple services on different ports inside the container:

| Port | Service | Purpose |
|------|---------|---------|
| 8080 | Webserver-proxy (Portal) | Main HTTP — web pages, REST API. **This is the port Railway must route to.** |
| 4002 | Websocket (Portal) | Web client websocket connection |
| 4005 | Webserver (Server) | Internal — Portal proxies to this |
| 4006 | AMP (internal) | Portal ↔ Server communication |

### Railway networking configuration

1. **Port:** Set to **8080** on the game service (Settings > Networking)
2. **Railway domain:** Generate one (e.g. `game-production-xxxx.up.railway.app`). This serves both HTTP and Websocket.
3. **Custom domain:** Add `fcmud.world` and configure DNS (CNAME + TXT verification)

### Websocket routing

The web client needs a websocket connection for real-time game communication. Railway routes websockets through the same domain as HTTP, but Evennia needs to know the websocket URL to tell the web client where to connect.

Set the `WEBSOCKET_CLIENT_URL` environment variable:
```
wss://<railway-generated-domain>.up.railway.app/ws
```

**Why use the Railway domain, not the custom domain?** Cloudflare (if used for DNS) may interfere with websocket connections depending on proxy settings. Using the Railway domain directly bypasses Cloudflare and provides a reliable websocket connection.

**Important:** The websocket URL must use `wss://` (not `ws://`) since Railway enforces HTTPS.

### DNS configuration (Cloudflare example)

| Type | Name | Value | Proxy |
|------|------|-------|-------|
| CNAME | `@` | `<railway-target>.up.railway.app` | Proxied (orange cloud) |
| TXT | `_railway-verify` | `railway-verify=<token>` | DNS only |

---

## Vault Signing & Multisig

### Current State (Single-Sig)

The game server holds `XRPL_VAULT_WALLET_SEED` — a single key with full signing authority over the vault wallet. This is the initial configuration before multisig is set up.

### Production Requirement (Mainnet)

For mainnet, no single compromised key should be able to move funds. The **vault** and **issuer** wallets use **XRPL native multisig** with a 2-of-2 signer configuration. The **operating** wallet does not use multisig — its keys are never exposed in a hot environment.

### Key Architecture

The vault and issuer wallets each have a 2-of-2 signer list (Key A + Key B). **Key B is shared** across both wallets — compromise of either server yields half of both wallets regardless of whether the keys are shared or separate, so sharing reduces key management overhead without weakening security. Key A is unique per wallet since the vault and issuer have different operational contexts (game server vs manager/admin tool). All three wallets (vault, issuer, operating) share a common **regular key (Key C)** for emergency recovery, with their master keys disabled.

| Key | Vault | Issuer | Operating | Location | Purpose |
|-----|-------|--------|-----------|----------|---------|
| **Master key** | Vault master | Issuer master | Operating master | Seed retained offline | Disabled on-chain. Seed retained as absolute last resort but cannot sign transactions unless re-enabled via the regular key. |
| **Signer key A** | Vault-specific | Issuer-specific | — | Game server / Manager tool (env var) | First multisig signature. Operational tool builds transaction and signs with A. |
| **Signer key B** | Shared | Shared | — | Co-signing service (Render env var) | Second multisig signature. Co-signer validates business rules, signs with B, submits to XRPL. |
| **Regular key (Key C)** | Shared | Shared | Shared | Encrypted and sharded (reconstituted when needed) | Emergency recovery. Can sign any transaction as a single signature — used to rotate the signer list, change account settings, or set a new regular key if Key C itself is suspected compromised. |

Signer keys A and B do NOT need to be funded XRPL accounts — unfunded keypairs work as signers. The main account (vault/issuer) needs sufficient XRP to cover its own base reserve plus the owner reserve for the SignerList object. The operating wallet has no signer list and is secured solely by Key C (regular key) with its master key disabled.

Master keys on all three wallets are **disabled** (`asfDisableMaster`). They were used once during initial setup (to submit `SignerListSet` and `SetRegularKey`), then disabled. The master seeds are retained offline as an absolute last resort, but they cannot sign transactions in their current disabled state. Re-enabling a master key would require a transaction signed by the regular key (Key C). The primary recovery mechanism is Key C — reconstituted from its encrypted shards when needed.

### On-Chain Configuration

Vault and issuer use `SignerListSet` with quorum 2. Operating has no signer list.

```
Vault:     key A (weight 1) + key B (weight 1), quorum 2
Issuer:    key A (weight 1) + key B (weight 1), quorum 2
Operating: no signer list — single-sig via regular key (Key C)
```

All three wallets have their **master key disabled** and a shared **regular key (Key C)** set via `SetRegularKey`.

Normal operations on vault/issuer use **A + B** (automatic, game server + co-signer). Recovery scenarios:
- Key A compromised → reconstitute Key C (regular key), use it to submit a new `SignerListSet` replacing Key A
- Key B compromised → reconstitute Key C (regular key), use it to submit a new `SignerListSet` replacing Key B
- Key C suspected compromised → use A + B multisig to immediately set a new regular key via `SetRegularKey`, re-shard the new key
- A + B both compromised → reconstitute Key C to replace the entire signer list

### Infrastructure Separation

```
Game Server (Railway)                Co-Signing Service (Render)
┌──────────────────┐                ┌──────────────────────────┐
│                  │                │                          │
│ Build tx         │  POST /cosign │ 1. Authenticate (API key)│
│ Autofill         │───────────────→│ 2. Look up wallet config │
│ Sign with key A  │               │ 3. Validate rules        │
│ (multisign=True) │               │ 4. Co-sign with key B    │
│                  │  {tx_hash,    │ 5. Combine signatures    │
│                  │   result}     │ 6. Submit to XRPL        │
│ Holds key A only │←──────────────│ Holds key B only         │
└──────────────────┘               └──────────────────────────┘
```

**Why separate providers:** Compromising Railway gives an attacker key A + the API key, but not key B. Compromising Render gives key B, but no ability to build valid game transactions. Both must be compromised simultaneously to steal funds.

### Co-Signing Service

The co-signing service (`FullCircleMUD/cosigner` repo) is a standalone FastAPI app deployed on **Render** (separate provider from the game server on Railway). It is **multi-wallet capable** — it can hold signing keys for any number of XRPL accounts with per-wallet business rules. Adding a new wallet = add a JSON config entry + set an env var.

**Endpoints:**
- `POST /cosign` — co-sign and submit a partially-signed XRPL transaction (requires `X-API-Key` header). Supports two API keys:
  - **Production key** (`API_KEY`) — full pipeline + XRPL submission
  - **Dev key** (`DEV_API_KEY`) — full pipeline (deserialise, validate, sign, combine) but skips XRPL submission, returning a mock success. Non-sensitive — if compromised, cannot authorise real transactions
- `GET /health` — health check (no auth)

**Per-wallet business rules** (configured in `wallets.json`):

| Rule | Description |
|------|-------------|
| `allowed_tx_types` | Transaction type must be in allow list (e.g. Payment, OfferCreate) |
| `blocked_tx_types` | Transaction type must NOT be in block list (e.g. AccountDelete, SignerListSet) |
| `require_issuer` | All issued currency amounts must reference the game's issuer address |
| `max_per_minute` | Rate limit per wallet |

Wallet seeds are stored in Render env vars, not in the config file. The JSON config (which maps addresses to rules and env var names) can safely live in the repo.

### Game Server Changes

When `XRPL_MULTISIG_ENABLED=true`, the 4 vault-signing functions in `xrpl_tx.py` and `xrpl_amm.py` use a `_cosign_and_submit()` helper instead of `submit_and_wait()`:
1. Set `account` to `settings.XRPL_VAULT_ADDRESS` (not `wallet.address` — since key A's seed derives a different address than the vault)
2. Autofill the transaction (sequence number, fee, last ledger)
3. Sign with key A (`sign(tx, wallet, multisign=True)`)
4. POST the serialised blob to the co-signer service via `httpx`
5. Co-signer returns `tx_hash`, `engine_result`, and full `meta` (AffectedNodes etc.) — the game server uses meta to extract NFT offer IDs

When `XRPL_MULTISIG_ENABLED=false` (default), existing single-sig flow is unchanged — `wallet.address` equals vault address because the seed IS the master key.

### Compromise Recovery Plan

See the full compromise recovery plan in the previous version of this document or in the security architecture documentation.

---

## GitHub Actions CI

A `.github/workflows/ci.yml` in the game repo runs automatically on every Pull Request and push to `main`:

- Runs the full test suite (`evennia test --settings settings tests`) using SQLite (fast, no infrastructure needed)
- Blocks merge if tests fail
- Optionally: linting, type checking

This is separate from Railway's deployment — GitHub Actions validates the code, Railway deploys it. A PR must pass CI before it can be merged, and merging is what triggers Railway to deploy.

---

## Common Scenarios

### "I want to deploy new code"

```
git checkout main
git merge dev
git push origin main
git checkout dev
```

Railway auto-deploys. Migrations run automatically. Existing data preserved.

### "I added a new model/field and need a migration"

1. Create migration locally: `evennia makemigrations`
2. Test locally: `evennia test --settings settings tests`
3. Commit the migration file
4. Merge to `main` and push
5. Railway auto-deploys and applies the new migration incrementally

### "I want to reset the database"

Delete the PostgreSQL plugin in Railway and re-add it. Fresh database. Then redeploy — `deploy_migrate.py` runs all migrations from scratch, recreating the schema. The superuser is auto-created from env vars.

### "I need to copy production data to staging"

Railway supports database snapshots and imports. Take a backup of production, restore to staging.

### "Railway deploy failed"

Check Railway's deploy logs. Common causes:
- Migration error (check for `Migration FAILED` in logs)
- Missing environment variable (check `DATABASE_URL` is set and resolved)
- Invalid JSON in bot env vars (no trailing commas in JSON)
- Python dependency issue
- pgvector extension not available (should be auto-installed by `deploy_migrate.py`)

Railway keeps the previous deployment running until the new one succeeds, so a failed deploy doesn't take down the live server.

### "I need to run a one-off command on Railway"

Railway provides a shell into the running service. Use it for things like:
- `evennia shell` (Django shell for data queries)
- Running management commands
- Debugging production state

### "The websocket isn't connecting"

1. Check `WEBSOCKET_CLIENT_URL` is set correctly: `wss://<railway-domain>.up.railway.app/ws`
2. Use the Railway-generated domain, not the custom domain (avoids Cloudflare interference)
3. Ensure the Railway domain has port 4002 exposed
4. Check browser console for websocket connection errors

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
│  (XRPL Mainnet)  │                    │  (XRPL Mainnet)  │
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

- `GET /{uri_id}` — XLS-24d compliant JSON metadata for XRPL marketplace resolution
- `GET /health` — health check (includes `environment` and `db_configured` fields)

---

## Troubleshooting

### Database wipes on deploy

**Symptom:** Every deploy applies all migrations from `0001_initial`, superuser gets recreated, all data lost.

**Cause:** `DATABASE_URL` is not set or not resolving on the game service. Without it, Django falls back to SQLite inside the ephemeral container, which is destroyed on each deploy.

**Fix:**
1. Check game service Variables tab — `DATABASE_URL` must show a resolved Postgres connection string, not empty
2. Set `DATABASE_URL` as a **service variable** directly on the game service (not a shared variable)
3. If `${{Postgres.DATABASE_URL}}` doesn't resolve, paste the actual connection string from the Postgres service

### Migrations say "No migrations to apply" but tables don't exist

**Symptom:** `evennia migrate --database xrpl` says nothing to apply, but xrpl tables don't exist in Postgres.

**Cause:** Database routers block table creation when all aliases point to the same Postgres. The migration is recorded but the DDL is silently skipped.

**Fix:** This is already fixed in the codebase — routers are disabled when `DATABASE_URL` is set. If you encounter this, ensure `settings.py` has the router conditional:
```python
if not _DATABASE_URL:
    DATABASE_ROUTERS = [...]
```

### pgvector migration hangs

**Symptom:** Deploy hangs on `Applying ai_memory.0002_pgvector...` for minutes.

**Cause:** HNSW index creation blocks inside a transaction. Or a previous failed deploy left a stuck `idle in transaction` process holding a lock.

**Fix:**
1. Check for stuck processes: `SELECT pid, state, query FROM pg_stat_activity WHERE state = 'idle in transaction';`
2. Kill them: `SELECT pg_terminate_backend(<pid>);`
3. Ensure `0002_pgvector` migration has `atomic = False`
4. Ensure `deploy_migrate.py` uses autocommit for `CREATE EXTENSION`

### Invalid JSON in bot env vars

**Symptom:** Deploy crash-loops with `json.decoder.JSONDecodeError`.

**Cause:** `BOT_WALLET_ADDRESSES_JSON` or `BOT_PASSWORDS_JSON` has trailing commas or single quotes.

**Fix:** Ensure valid JSON with double quotes and no trailing commas:
```json
{"billy":"rAddr1","bianca":"rAddr2"}
```

### 502 Bad Gateway / Application failed to respond

**Symptom:** Domain returns 502 or "Application failed to respond".

**Cause:** Railway isn't routing to the correct port, or the game isn't running.

**Fix:**
1. Set port to **8080** in game service Settings > Networking
2. Check deploy logs for startup errors
3. Verify the Railway domain works before debugging custom domain issues

---

## Summary

The deployment pipeline is fully automated and GitHub-driven:

1. **Develop** on `dev` branch (local, SQLite)
2. **Merge to `main`** → Railway auto-deploys to production (PostgreSQL)
3. **`deploy_migrate.py`** handles migrations, pgvector, and error recovery
4. **Database persists** between deploys via Railway Postgres volumes
5. **No manual SQL** required for normal deployments

Key files:
- `railway.toml` — build and start commands
- `deploy_migrate.py` — migration script for Railway
- `server/conf/settings.py` — database config, router conditional

The **NFT Metadata API** runs as a separate Railway service sharing the same PostgreSQL instance, ensuring XRPL marketplace metadata resolution stays available regardless of game server state.
