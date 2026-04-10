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
| `XRPL_NETWORK_URL` | XRPL websocket endpoint | You | Default: `wss://s1.ripple.com:51233` (mainnet). Override via env var if needed |
| `XRPL_VAULT_WALLET_SEED` | Vault signer key A for server-signed transactions | You | Single-sig: vault master seed. Multisig: signer key A's seed. See [Vault Signing & Multisig](#vault-signing--multisig) |
| `XRPL_MULTISIG_ENABLED` | Enable multisig co-signing flow | You | `true` or `false`. When false, single-sig flow unchanged |
| `XRPL_COSIGNER_URL` | Co-signing service URL | You | Only when multisig enabled. e.g. `https://cosigner-xxxx.onrender.com` |
| `XRPL_COSIGNER_API_KEY` | Co-signer API authentication key | You | Production key: co-signs and submits to XRPL. Dev environments use the co-signer's `DEV_API_KEY` which runs the full pipeline but skips XRPL submission |
| `XRPL_ROOT_ADDRESS` | Dev/superuser wallet address | You | |
| `XAMAN_API_KEY` | Xaman wallet API key | You | |
| `XAMAN_API_SECRET` | Xaman wallet API secret | You | |
| `LLM_API_KEY` | OpenRouter API key for LLM NPCs | You | |
| `LLM_EMBEDDING_API_KEY` | OpenAI API key for embeddings | You | |

Locally, these secrets live in `server/conf/secret_settings.py` (not in Git). On Railway, they're environment variables — same values, different delivery mechanism. The `settings.py` file handles both: it tries to import `secret_settings.py` (local), and environment variables can override via Railway's injection.

**Staging and production should use different API keys and wallet addresses.** Complete isolation.

---

## Vault Signing & Multisig

### Current State (Single-Sig)

The game server holds `XRPL_VAULT_WALLET_SEED` — a single key with full signing authority over the vault wallet. This is the initial configuration before multisig is set up.

### Production Requirement (Mainnet)

For mainnet, no single compromised key should be able to move funds. Both the **vault** and **issuer** wallets use **XRPL native multisig** with a 2-of-3 signer configuration.

### Key Architecture

Each multisig wallet (vault, issuer) has 4 associated keys. **Key B and Key C are shared** across both wallets — compromise of either server yields half of both wallets regardless of whether the keys are shared or separate, so sharing reduces key management overhead without weakening security. Key A is unique per wallet since the vault and issuer have different operational contexts (game server vs manager/admin tool).

| Key | Vault | Issuer | Location | Purpose |
|-----|-------|--------|----------|---------|
| **Master key** | Vault master | Issuer master | Deep cold storage (offline, physical safe) | Emergency recovery only. Stays enabled but never touches a server. |
| **Signer key A** | Vault-specific | Issuer-specific | Game server / Manager tool (env var) | First signature. Operational tool builds transaction and signs with A. |
| **Signer key B** | Shared | Shared | Co-signing service (Render env var) | Second signature. Co-signer validates business rules, signs with B, submits to XRPL. |
| **Signer key C** | Shared | Shared | Cold storage (offline) | Recovery key. Used with A or B to rotate a compromised key. |

Signer keys A, B, and C do NOT need to be funded XRPL accounts — unfunded keypairs work as signers (only the master private key can sign, since Regular Key and key disable require a funded account). The main account (vault/issuer) needs sufficient XRP to cover its own base reserve plus the owner reserve for the SignerList object. Rotating a compromised key: generate a new keypair, update the `SignerListSet` using 2 of the remaining good keys, update the env var.

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
| `XRPL_COSIGNER_API_KEY` | Co-signer API key. Production uses the production key; dev/staging uses the `DEV_API_KEY` (no-op on-chain) |

`XRPL_VAULT_WALLET_SEED` stays as-is — when multisig is enabled it becomes key A's seed.

### Setup Tools

The `cosigner` repo includes setup scripts:
- `setup/generate_keys.py` — generate XRPL keypairs for signer keys A, B, C
- `setup/configure_signerlist.py` — submit `SignerListSet` on any XRPL account (set, verify, or remove)

### Migration Path (Single-Sig → Multisig)

1. **Generate keypairs** — 3 keys per wallet (A, B, C) using `generate_keys.py`
2. **Configure SignerList** — `configure_signerlist.py` with vault's master key
3. **Deploy co-signer** to Render with key B's seed
4. **Enable multisig** on game server — set `XRPL_MULTISIG_ENABLED=true` + cosigner URL/key
5. **Verify** — confirm shopkeeper trades, exports (if enabled) land on-chain with 2 signatures
6. **Store master key** in cold storage — it stays enabled on-chain but never online

### Compromise Recovery Plan

The multisig architecture with clawback creates a layered defence. Here's the incident response plan for each compromise scenario.

#### Scenario 1: Game Server Compromised (Key A Exposed)

**Blast radius:** Attacker has Key A + cosigner API key. They can submit transactions that the cosigner will co-sign — but ONLY transaction types allowed by the cosigner's business rules (Payment, OfferCreate, etc.). They CANNOT change the SignerList, delete the account, or modify account settings.

**Immediate response (minutes):**
1. **Shut down the co-signing service** on Render (suspend or scale to 0). This instantly kills the attacker's ability to get co-signatures — Key A alone cannot meet quorum.
2. **Revoke the Railway deployment** — stop the compromised game server.

**Recovery (hours):**
3. **Rotate the cosigner API key** on Render (new random secret).
4. **Generate a new Key A** keypair for the affected wallet(s).
5. **Update the SignerList on-chain** using Key B + Key C (both uncompromised). This replaces the old Key A with the new one.
6. **Clawback any unauthorized token transfers** using the issuer's master key (from cold storage). Clawback pulls tokens back from any recipient address.
7. **Deploy clean game server** on Railway with the new Key A seed and new API key.
8. **Restart the co-signing service** on Render.

#### Scenario 2: Co-Signing Service Compromised (Key B Exposed)

**Blast radius:** Attacker has Key B only. Useless without Key A — they cannot build valid game transactions or meet quorum alone.

**Immediate response:**
1. **Shut down the co-signing service** on Render.

**Recovery:**
2. **Generate a new Key B** keypair.
3. **Update the SignerList on-chain** using Key A + Key C.
5. **Update Render env var** with new Key B seed.
6. **Restart the co-signing service.**

#### Scenario 3: Both Server and Co-Signer Compromised

**Blast radius:** Attacker has Key A + Key B — they can sign any transaction type the cosigner rules allow. However, they still cannot SignerListSet, AccountDelete, or AccountSet (blocked in cosigner rules).

**Immediate response (seconds matter):**
1. **Shut down both services immediately** (Render + Railway).
2. **Retrieve master key from cold storage.**
3. **Submit a new SignerListSet** using the master key — replace all 3 signer keys with freshly generated ones.
4. **Clawback any unauthorized transfers** using the issuer's clawback capability.
5. **Audit on-chain transactions** from the compromise window.
6. **Deploy fresh services** with new keys and new API secret.

#### Scenario 4: Key Loss (Not Compromise)

| Keys Lost | Recovery Path |
|-----------|--------------|
| Key A only | Use B + C to update SignerList with new A |
| Key B only | Use A + C to update SignerList with new B |
| Key C only | Use A + B to update SignerList with new C |
| A + B | Master key from cold storage |
| A + C | Master key from cold storage |
| B + C | Master key from cold storage |
| Master + any 2 | **Unrecoverable** — this is why the master key is stored in a physical safe |

#### Why Clawback Is Essential

The issuer wallet has `asfAllowClawback` enabled (set before any trustlines were created — this is irreversible and cannot be enabled retroactively). This means:

- **Any FCM-issued token** (gold, resources, proxy tokens) can be pulled back from any holder address by the issuer.
- Even if an attacker manages to transfer tokens to external wallets during a compromise window, the issuer can reclaim them.
- Clawback is the last line of defence — it makes token theft recoverable even in the worst case.
- XRP itself cannot be clawed back (native asset), but game tokens are all issuer-controlled.

#### Prevention: What the Cosigner Rules Block

Even with both Key A and Key B, the cosigner's `blocked_tx_types` prevent:

| Blocked Transaction | Why |
|---|---|
| `SignerListSet` | Cannot change who can sign — attacker can't lock out the operator |
| `AccountDelete` | Cannot destroy the wallet |
| `SetRegularKey` | Cannot add a backdoor signing key |
| `AccountSet` | Cannot change wallet flags (e.g. disable clawback... though that's already irreversible) |

These rules are enforced server-side in the cosigner before co-signing. An attacker with Key A cannot bypass them without also compromising the cosigner's code on Render.

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
