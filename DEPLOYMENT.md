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
| `XRPL_VAULT_WALLET_SEED` | Vault signer key A for server-signed transactions | You | Single-sig: vault master seed. Multisig: signer key A's seed. See [Vault Signing & Multisig](#vault-signing--multisig) |
| `XRPL_MULTISIG_ENABLED` | Enable multisig co-signing flow | You | `true` or `false`. When false, single-sig flow unchanged |
| `XRPL_COSIGNER_URL` | Co-signing service URL | You | Only when multisig enabled. e.g. `https://cosigner-xxxx.onrender.com` |
| `XRPL_COSIGNER_API_KEY` | Co-signer API authentication key | You | Shared secret between game server and co-signer |
| `XRPL_ROOT_ADDRESS` | Dev/superuser wallet address | You | |
| `XAMAN_API_KEY` | Xaman wallet API key | You | |
| `XAMAN_API_SECRET` | Xaman wallet API secret | You | |
| `LLM_API_KEY` | OpenRouter API key for LLM NPCs | You | |
| `LLM_EMBEDDING_API_KEY` | OpenAI API key for embeddings | You | |

Locally, these secrets live in `server/conf/secret_settings.py` (not in Git). On Railway, they're environment variables — same values, different delivery mechanism. The `settings.py` file handles both: it tries to import `secret_settings.py` (local), and environment variables can override via Railway's injection.

**Staging and production should use different API keys and wallet addresses.** Staging points at XRPL testnet. Production points at mainnet (when ready). Complete isolation.

---

## Vault Signing & Multisig

### Current State (Testnet / Staging)

The game server holds `XRPL_VAULT_WALLET_SEED` — a single key with full signing authority over the vault wallet. This is acceptable for testnet where the tokens have no real value.

### Production Requirement (Mainnet)

For mainnet, no single compromised key should be able to move funds. Both the **vault** and **issuer** wallets use **XRPL native multisig** with a 2-of-3 signer configuration.

### Key Architecture

Each multisig wallet (vault, issuer) has 4 associated keys:

| Key | Location | Purpose |
|-----|----------|---------|
| **Master key** | Deep cold storage (offline, physical safe) | Emergency recovery only. Stays enabled but never touches a server. Last resort if 2+ signer keys are lost. |
| **Signer key A** | Game server (Railway env var) | First signature. Game server builds transaction and signs with A. |
| **Signer key B** | Co-signing service (Render env var) | Second signature. Co-signer validates business rules, signs with B, submits to XRPL. |
| **Signer key C** | Cold storage (offline) | Recovery key. Used with A or B to rotate a compromised key. |

Signer keys A, B, and C are unfunded XRPL addresses — they don't need to exist on-chain or hold any XRP. They're purely signing identities. This means rotating a compromised key costs nothing: generate a new keypair, update the `SignerListSet` using 2 of the remaining good keys, update the env var.

The vault/issuer master keys are used once during initial setup (to submit `SignerListSet`), then locked away in cold storage permanently. They remain enabled on-chain as a disaster recovery fallback — if 2+ signer keys are lost, the master key can still authorise transactions. The risk of physical theft from a safe is far lower than the risk of catastrophic key loss.

### On-Chain Configuration

Both wallets use `SignerListSet` with quorum 2:

```
Vault:  key A (weight 1) + key B (weight 1) + key C (weight 1), quorum 2
Issuer: key A (weight 1) + key B (weight 1) + key C (weight 1), quorum 2
```

Normal operations use **A + B** (automatic, game server + co-signer). Any 2-of-3 can sign, so recovery scenarios are:
- Key A compromised → rotate using B + C
- Key B compromised → rotate using A + C
- Key A lost → use B + C until new A is provisioned
- Catastrophic (2+ keys lost) → master key from cold storage

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
- `POST /cosign` — co-sign and submit a partially-signed XRPL transaction (requires `X-API-Key` header)
- `GET /health` — health check (no auth)

**Per-wallet business rules** (configured in `wallets.json`):

| Rule | Description |
|------|-------------|
| `allowed_tx_types` | Transaction type must be in allow list (e.g. Payment, OfferCreate) |
| `blocked_tx_types` | Transaction type must NOT be in block list (e.g. AccountDelete, SignerListSet) |
| `require_issuer` | All issued currency amounts must reference the game's issuer address |
| `max_per_minute` | Rate limit per wallet |

Wallet seeds are stored in Render env vars, not in the config file. The JSON config (which maps addresses to rules and env var names) can safely live in the repo.

**Tiered approval (future):**

| Tier | Rule | Approval |
|------|------|----------|
| Auto | Small exports (< 100 gold) | Co-signer approves immediately |
| Delayed | Medium exports (100-1000 gold) | 5-minute delay before co-signing |
| Manual | Large exports (> 1000 gold), bulk, unusual | Queued for manual approval |

### Game Server Changes

When `XRPL_MULTISIG_ENABLED=true`, the 4 vault-signing functions in `xrpl_tx.py` and `xrpl_amm.py` use a `_cosign_and_submit()` helper instead of `submit_and_wait()`:
1. Set `account` to `settings.XRPL_VAULT_ADDRESS` (not `wallet.address` — since key A's seed derives a different address than the vault)
2. Autofill the transaction (sequence number, fee, last ledger)
3. Sign with key A (`sign(tx, wallet, multisign=True)`)
4. POST the serialised blob to the co-signer service via `httpx`
5. Co-signer returns `tx_hash`, `engine_result`, and full `meta` (AffectedNodes etc.) — the game server uses meta to extract NFT offer IDs

When `XRPL_MULTISIG_ENABLED=false` (default), existing single-sig flow is unchanged — `wallet.address` equals vault address because the seed IS the master key.

**New env vars on game server (Railway):**

| Variable | Purpose |
|----------|---------|
| `XRPL_MULTISIG_ENABLED` | Feature flag (`true`/`false`) |
| `XRPL_COSIGNER_URL` | Co-signer service URL (e.g. `https://cosigner-xxxx.onrender.com`) |
| `XRPL_COSIGNER_API_KEY` | Shared secret for co-signer authentication |

`XRPL_VAULT_WALLET_SEED` stays as-is — when multisig is enabled it becomes key A's seed.

### Setup Tools

The `cosigner` repo includes setup scripts:
- `setup/generate_keys.py` — generate XRPL keypairs for signer keys A, B, C
- `setup/configure_signerlist.py` — submit `SignerListSet` on any XRPL account (set, verify, or remove)

### Migration Path (Testnet → Mainnet)

1. **Generate keypairs** — 3 keys per wallet (A, B, C) using `generate_keys.py`
2. **Configure SignerList** on testnet — `configure_signerlist.py` with vault's master key
3. **Deploy co-signer** to Render with key B's seed
4. **Enable multisig** on game server — set `XRPL_MULTISIG_ENABLED=true` + cosigner URL/key
5. **Test on testnet** — verify shopkeeper trades, exports (if enabled) land on-chain with 2 signatures
6. **Repeat for mainnet** — same keys work on both networks (only `XRPL_NETWORK_URL` changes)
7. **Store master key** in cold storage — it stays enabled on-chain but never online

**The same signing keys work on both testnet and mainnet** — they're just cryptographic keypairs. The `XRPL_NETWORK_URL` environment variable determines which network receives the signed transaction.

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
