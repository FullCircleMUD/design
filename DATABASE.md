# DATABASE.md

> **THIS FILE covers database architecture** for FullCircleMUD — the four-database design, the SQLite/PostgreSQL toggle, how migrations work, and developer workflow. Deployment pipeline, CI/CD, Railway configuration, and branching strategy live in the private ops repository (`ops/DEPLOYMENT.md`). For technical implementation details and code patterns, see **src/game/CLAUDE.md**. For economic design, see **design/ECONOMY.md**. For the three embedding memory systems, see **design/COMBAT_AI_MEMORY.md** (combat), **design/LORE_MEMORY.md** (world knowledge), and **design/NPC_MOB_ARCHITECTURE.md** § Three Memory Systems (overview). For subscription payment system, see **design/SUBSCRIPTIONS.md**.

---

## Table of Contents

- [Overview](#overview)
- [The Four Databases](#the-four-databases)
- [Why Two Database Backends](#why-two-database-backends)
- [How the Toggle Works](#how-the-toggle-works)
- [What Developers Need to Know](#what-developers-need-to-know)
- [How Database Migrations Work](#how-database-migrations-work)
- [Common Scenarios](#common-scenarios)
- [Future: pgvector for AI Memory](#future-pgvector-for-ai-memory)

---

## Overview

FullCircleMUD is built on Evennia (a Python/Django MUD framework). Like all Django applications, it uses a relational database to store game state — player accounts, characters, items, blockchain mirrors, NPC memories, and more.

The game uses **four separate databases** to keep concerns isolated. Locally, these are SQLite files (zero setup, instant). In production on Railway, they're PostgreSQL (robust, handles concurrent connections). A single environment variable controls which backend is used — no code changes needed.

For how this fits into the deployment pipeline, see the deployment runbook in the private ops repository.

---

## The Four Databases

| Database | Local File | Purpose | Key Models |
|----------|-----------|---------|------------|
| **default** | `evennia.db3` | Evennia core — accounts, characters, rooms, scripts, attributes | AccountDB, ObjectDB, all typeclasses |
| **xrpl** | `xrpl.db3` | Blockchain mirror — XRPL asset tracking, economy telemetry | CurrencyType, FungibleGameState, NFTGameState, transfer logs, snapshots |
| **ai_memory** | `ai_memory.db3` | NPC memory — conversation history with vector embeddings | NpcMemory |
| **subscriptions** | `subscriptions.db3` | Subscription plans and payment records | SubscriptionPlan, SubscriptionPayment |

Each database has its own **router** — a small Python class that tells Django "models with this app label go to this database." The routers work identically regardless of whether the underlying database is SQLite or PostgreSQL. They're purely about directing traffic, not about what kind of database is at the other end.

**Why separate databases?**

- **ai_memory** survives game database wipes. If we reset the game world during development, NPCs keep their memories. This is intentional — NPC personalities and relationships persist across resets.
- **xrpl** isolates blockchain state from game state. The XRPL database is the game's mirror of on-chain reality. Keeping it separate makes reconciliation, backup, and debugging cleaner.
- **subscriptions** isolates payment records from game state. Subscription payments are financial records that must never be lost in a game database reset. The `tx_hash` unique constraint provides replay protection for on-chain payment verification.
- **default** is Evennia's standard database. It holds everything the framework expects — accounts, objects, scripts, channels, configuration.

---

## Why Two Database Backends

**SQLite** is a file-based database built into Python. No installation, no server process, no configuration. You clone the repo, install Python dependencies, and the database just works. It's perfect for development and testing.

**PostgreSQL** is a server-based database. It runs as a separate process, handles concurrent connections properly, supports advanced features like full-text search and (with extensions) vector similarity search. It's what you want for a live game server.

SQLite uses file-level locking — only one process can write at a time. The game server itself mostly funnels writes through a single event loop (Twisted's reactor), so in-game operations rarely collide. The real limitation is **multi-process access**: a separate management API, a payment gateway webhook receiver, a background analytics job, or a blockchain sync service would each need their own database connection. With SQLite, these external processes would compete for the file lock and hit "database is locked" errors. PostgreSQL handles hundreds of concurrent connections from different processes natively — which keeps the door open for the ecosystem of services a production game needs around it.

**The solution:** use SQLite locally (zero friction), PostgreSQL in production (robust). Same code, same models, same queries. Django's ORM (Object-Relational Mapper) translates Python code into the correct SQL dialect automatically.

---

## How the Toggle Works

A single environment variable called `DATABASE_URL` controls everything.

```
┌─────────────────────────────────────────────────┐
│                  settings.py                     │
│                                                  │
│   if DATABASE_URL is set:                        │
│       → use PostgreSQL (parse the URL)           │
│   else:                                          │
│       → use SQLite files (default, zero config)  │
│                                                  │
└─────────────────────────────────────────────────┘
```

- **Your laptop:** `DATABASE_URL` is not set. Django uses SQLite. No setup required.
- **Railway staging:** Railway injects `DATABASE_URL` pointing to a staging PostgreSQL instance. Django uses PostgreSQL. You didn't change any code.
- **Railway production:** Railway injects a different `DATABASE_URL` pointing to the production PostgreSQL instance. Same code, different database.

The toggle lives in `server/conf/settings.py`. It uses `dj-database-url`, a standard Django library that parses a PostgreSQL connection string into the format Django expects.

**In PostgreSQL mode, all four database aliases (default, xrpl, ai_memory, subscriptions) point to the same physical PostgreSQL database.** Table prefixes (`xrpl_*`, `ai_memory_*`, `subscriptions_*`) keep them separate within the single database. This is simpler and cheaper than running four separate PostgreSQL instances.

### Routers are mode-dependent

The database routers in `blockchain/xrpl/db_router.py`, `ai_memory/db_router.py`, and `subscriptions/db_router.py` are **only active in SQLite (local dev) mode**. In `settings.py`:

```python
if not _DATABASE_URL:
    DATABASE_ROUTERS = [
        "blockchain.xrpl.db_router.XRPLRouter",
        "ai_memory.db_router.AiMemoryRouter",
        "subscriptions.db_router.SubscriptionsRouter",
    ]
```

**Why?** Locally, each alias is a **separate SQLite file** (`evennia.db3`, `xrpl.db3`, `ai_memory.db3`, `subscriptions.db3`). The routers direct each app's models to the right file. Without them, all tables would land in `evennia.db3`.

**On Railway**, all aliases point to the **same** Postgres instance. Enabling routers here is actively harmful — their `allow_migrate()` method would block table creation for non-default apps during a single `migrate` call, leaving the `xrpl`, `ai_memory`, and `subscriptions` tables uncreated while still recording the migrations as applied.

For how Railway injects `DATABASE_URL` and manages environments, see the deployment runbook in the private ops repository.

---

## What Developers Need to Know

### Local development (SQLite)

Nothing changes from what you're used to:

```bash
# From FCM/src/game/ with venv activated
evennia migrate
evennia migrate --database xrpl
evennia migrate --database ai_memory
evennia migrate --database subscriptions
evennia start
```

SQLite database files appear in `server/`. They're in `.gitignore`. Each developer has their own local data.

### Running tests

Tests always use SQLite regardless of `DATABASE_URL`. Django creates temporary test databases that are destroyed after the test run.

```bash
evennia test --settings settings tests
```

### If you want to test with PostgreSQL locally (optional)

This is rarely needed — the ORM ensures queries behave the same on both backends. But if you want to verify PostgreSQL-specific behavior:

1. Install Docker Desktop
2. Start a PostgreSQL container:
   ```bash
   docker compose up -d
   ```
3. Set the environment variable:
   ```bash
   export DATABASE_URL=postgres://postgres:fcm@localhost:5432/fcm
   ```
4. Run migrations and start as normal

This is entirely optional. Day-to-day development works fine on SQLite.

---

## How Database Migrations Work

This is one of Django's best features, and it's worth understanding even if you're not a Django expert.

### What's a migration?

A migration is a Python file that describes a change to the database schema — "add this column," "create this table," "add this index." Django generates these automatically when you change a model.

### The developer workflow

Local development and Railway deployment migrate differently because of the SQLite vs Postgres mode difference.

```
1. Developer changes a model (e.g. adds a field to NpcMemory)
     ↓
2. Developer runs: evennia makemigrations ai_memory
     → Django compares models.py to existing migrations
     → Generates a new migration file (e.g. 0003_add_mood_field.py)
     ↓
3. Developer runs: evennia migrate --database ai_memory
     → Routers are active (SQLite mode)
     → Migration applies to the local ai_memory.db3 file
     → Must use --database flag because routers block cross-alias migrations
     ↓
4. Developer commits the migration file to Git
     ↓
5. Code gets merged and deployed to Railway
     ↓
6. Railway runs deploy_migrate.py:
     → Routers are DISABLED (DATABASE_URL is set)
     → Single `migrate` call (no --database flag) applies ALL pending
       migrations from every app to the shared Postgres instance
     → Database schema now matches the code
```

**Key distinction:**
- **Local (SQLite, routers active):** Use `evennia migrate --database <alias>` for each custom database. The routers require this.
- **Railway (Postgres, routers disabled):** A single `migrate` call handles everything. This is what `deploy_migrate.py` does automatically.

For how Railway runs migrations automatically on deploy, see the deployment runbook in the private ops repository.

### Why this is safe

- **Migrations are idempotent.** Running `migrate` when there's nothing new to apply does nothing. It checks which migrations have already been applied (Django tracks this in a `django_migrations` table) and only runs new ones. This is why the release command can run migrations on every deploy without risk.

- **Migrations are backend-agnostic.** The same migration file works on SQLite and PostgreSQL. Django translates the abstract operations ("add a text field") into the correct SQL dialect for whichever database is connected.

- **Migrations are ordered.** Each migration knows which migration came before it (via a `dependencies` list). Django applies them in the correct sequence, even if multiple developers created migrations in parallel (it handles merge conflicts in the dependency graph).

### What could go wrong?

The main risk is **destructive migrations** — dropping a column or table that still has data you need. Django will warn you during `makemigrations` if a migration would delete data. In practice:

- Adding fields, tables, and indexes is always safe
- Renaming fields needs care (Django asks if you're renaming vs. deleting and recreating)
- Removing fields should be done in two steps: first deploy code that stops using the field, then deploy the migration that removes it

---

## Common Scenarios

### "I added a new model"

1. Create the model in `models.py`
2. Run `evennia makemigrations <app_label>`
3. Run `evennia migrate --database <alias>`
4. Commit the migration file
5. Push — Railway runs the migration automatically on deploy

### "I changed a field on an existing model"

Same process. Django generates a migration that alters the column. Works on both SQLite and PostgreSQL.

### "I need to add seed data"

Use a data migration — a migration file that runs Python code to insert rows. The existing `0001_initial.py` in the xrpl app does this (it seeds 37 currency types and all NFT item types). Same migration runs on both backends.

### "Something broke on staging but works locally"

This is rare because the ORM abstracts backend differences. When it happens, it's usually:
- A raw SQL query that uses SQLite-specific syntax (avoid raw SQL)
- A timing/concurrency issue that only surfaces with real PostgreSQL (SQLite serializes all writes)
- An environment variable that's set locally but missing on Railway

---

## pgvector for AI Memory

The NPC memory system uses **dual-backend embedding storage** that automatically adapts to the database engine:

| Backend | Field | Search Method | Index |
|---------|-------|---------------|-------|
| **SQLite** (local dev) | `embedding` (`BinaryField`, numpy binary blob) | Python loop with numpy cosine similarity — O(n) | None |
| **PostgreSQL** (Railway) | `embedding_vector` (`VectorField(1536)`, pgvector native) | Single SQL query with `<=>` cosine distance operator | HNSW (`m=16, ef_construction=64`) |

Both fields coexist on the `NpcMemory` model. Backend detection is automatic — `_is_postgres()` in `ai_memory/services.py` checks `settings.DATABASES["ai_memory"]["ENGINE"]`, which follows the existing `DATABASE_URL` toggle. No configuration, feature flags, or manual switching required.

### How it works

**Storage (`store_memory()`):** After generating an embedding via OpenAI `text-embedding-3-small` (1536 dimensions), the raw `list[float]` is written to `embedding_vector` on Postgres or converted to `np.float32.tobytes()` and written to `embedding` on SQLite.

**Search (`search_memories()`):** On Postgres, uses pgvector's `CosineDistance` ORM annotation — a single indexed SQL query replaces the Python loop. On SQLite, the existing numpy cosine similarity loop runs unchanged.

**Migration (`0002_pgvector.py`):** Conditionally enables the `vector` extension, adds the column, creates the HNSW index, and back-fills existing binary embeddings into the vector column. All conditional steps check `connection.vendor == "postgresql"` and are no-ops on SQLite.

### Performance

The SQLite path works fine at a few thousand memories per NPC. The pgvector path with HNSW indexing scales to millions of vectors with sub-millisecond search — relevant as the memory system expands to three embedding tables:

| Table | Purpose | Design doc |
|---|---|---|
| `NpcMemory` | NPC interaction/dialogue memory | This doc (above) |
| `CombatMemory` | Combat encounter tactical history | [COMBAT_AI_MEMORY.md](COMBAT_AI_MEMORY.md) |
| `LoreMemory` | Embedded world knowledge | [LORE_MEMORY.md](LORE_MEMORY.md) |

All three share the same dual-backend infrastructure, HNSW indexing, and `_is_postgres()` branching. See [NPC_MOB_ARCHITECTURE.md](NPC_MOB_ARCHITECTURE.md) § Three Memory Systems for how they compose at prompt time.

### Future cleanup

Once confident in the pgvector path after production validation:
- Remove the `embedding` BinaryField and numpy conversion code
- Remove `_cosine_similarity()` helper
- Consider removing `numpy` from requirements if no other code uses it

---

## Summary

The database architecture is designed around one principle: **the environment decides the backend, not the code.**

- Develop locally with SQLite (zero setup)
- Deploy to Railway with PostgreSQL (zero code changes)
- A single environment variable (`DATABASE_URL`) controls the switch
- Django handles schema differences, migrations, and SQL dialect translation
- Same migration files run on both backends
- Four logical databases, isolated by routers, sharing one physical PostgreSQL instance in production
