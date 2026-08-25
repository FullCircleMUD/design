# database.md

> **THIS FILE covers database architecture** for FullCircleMUD — the five-database design, the SQLite/PostgreSQL toggle, how transactions work across them, where each alias lives, how migrations work, and developer workflow. Server provisioning and the deployment runbook live in the private ops repository (`ops/fcmud-ec2-staging-runsheet.md`). For technical implementation details and code patterns, see **src/game/CLAUDE.md**. For economic design, see **economy.md**. For the three embedding memory systems, see **combat-ai-memory.md** (combat), **lore-memory.md** (world knowledge), and **npc-mob-architecture.md** § Three Memory Systems (overview). For subscription payment system, see **subscriptions.md**.

---

## Table of Contents

- [Overview](#overview)
- [The Five Databases](#the-five-databases)
- [Why Two Database Backends](#why-two-database-backends)
- [How the Toggle Works](#how-the-toggle-works)
- [Transactions and Split Aliases](#transactions-and-split-aliases)
  - [Work that spans two databases](#work-that-spans-two-databases)
  - [Rolling back does not roll back Evennia's caches](#rolling-back-does-not-roll-back-evennias-caches)
  - [What the ledger changes](#what-the-ledger-changes)
  - [NFT moves and deletions](#nft-moves-and-deletions)
  - [Sweeps that run unattended](#sweeps-that-run-unattended)
  - [Pets](#pets)
- [What Developers Need to Know](#what-developers-need-to-know)
- [How Database Migrations Work](#how-database-migrations-work)
- [Common Scenarios](#common-scenarios)
- [pgvector for AI Memory](#pgvector-for-ai-memory)

---

## Overview

FullCircleMUD is built on Evennia (a Python/Django MUD framework). Like all Django applications, it uses a relational database to store game state — player accounts, characters, items, blockchain mirrors, NPC memories, and more.

The game uses **five separate databases** to keep concerns isolated. Locally, these are SQLite files (zero setup, instant). Deployed, they are PostgreSQL (robust, handles concurrent connections). Environment variables control both which backend is used and which alias lives on which instance — no code changes needed.

For how this fits into the deployment pipeline, see the deployment runbook in the private ops repository.

---

## The Five Databases

| Database | Local File | Purpose | Key Models |
|----------|-----------|---------|------------|
| **default** | `evennia.db3` | Evennia core — accounts, characters, rooms, scripts, attributes | AccountDB, ObjectDB, all typeclasses |
| **xrpl** | `xrpl.db3` | XRPL asset ownership and economy telemetry | CurrencyType, FungibleGameState, NFTGameState, transfer logs, snapshots |
| **ai_memory** | `ai_memory.db3` | NPC memory — conversation history with vector embeddings | NpcMemory |
| **subscriptions** | `subscriptions.db3` | Subscription plans and payment records | SubscriptionPlan, SubscriptionPayment |
| **archive** | `archive.db3` | Clone of Evennia's schema holding archived accounts and characters | ArchiveRecord, plus Evennia's own tables |

Each non-default alias has a **router** — a small Python class that tells Django "models with this app label go to this database." The routers work identically regardless of whether the underlying database is SQLite or PostgreSQL. They're purely about directing traffic, not about what kind of database is at the other end.

**Why separate databases?**

- **ai_memory** survives game database wipes. If we reset the game world during development, NPCs keep their memories. This is intentional — NPC personalities and relationships persist across resets.
- **xrpl** is the record of who owns what. On chain, an imported asset sits in the game's wallet; which character holds it inside the game exists only here. That makes this database the single source of truth for in-game ownership, and the one that must survive a world rebuild. See [Transactions and Split Aliases](#transactions-and-split-aliases) for what follows from that.
- **archive** cannot share `default`'s database — it is a clone of Evennia's schema, so the table names collide. `DATABASE_URL_ARCHIVE` is therefore not optional on a deployed box.
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

Each alias resolves its own connection, in this order:

| Order | Source | Meaning |
|---|---|---|
| 1 | `DATABASE_URL_<ALIAS>` | this alias has its own PostgreSQL instance |
| 2 | `DATABASE_URL` | share the default's PostgreSQL instance |
| 3 | SQLite file | local dev, one file per alias |

Which gives three deployment shapes:

- **Your laptop:** nothing set. Five SQLite files, no setup required.
- **Deployed, shared:** `DATABASE_URL` only. Every alias is one PostgreSQL database — except `archive`, which cannot share it.
- **Deployed, split:** `DATABASE_URL` plus one or more `DATABASE_URL_<ALIAS>` overrides. The named aliases move to their own instances; the rest stay with `default`. Staging runs this way, with all four overrides set.

`default` takes bare `DATABASE_URL` and has no `_DEFAULT` override — that variable is the contract every deploy already sets.

**Which alias lives where is a deployment decision, not a code one.** Moving one onto separate compute is one environment variable plus a dump/restore (or a fresh `migrate` on a new install). No application code names a database — see [What Developers Need to Know](#what-developers-need-to-know).

Resolution lives in `server/conf/db_config.py`, called from `server/conf/settings.py`, and uses `dj-database-url` to parse connection strings into the format Django expects. Behaviour is covered by `tests/server_tests/test_db_config.py`.

### Spell hosts and ports consistently

Two aliases count as the same database when engine, host, port and name all match. An omitted port is filled in from the engine's default before comparing, so `postgres://host/fcm` and `postgres://host:5432/fcm` match.

**Host spelling is not normalised.** Naming one alias `localhost` and another `127.0.0.1` reads as two different databases and will manufacture a router. Use the same host string across aliases.

A URL missing its host or database name raises `ImproperlyConfigured` at settings load rather than letting the server start against a misresolved alias.

### Routers are derived from the connections

A router is active for exactly those aliases that resolve to a physically different database from `default`. Nothing declares this — it falls out of the resolved connections:

| Deployment | Aliases differing from `default` | Routers active |
|---|---|---|
| Local SQLite | all four | all four |
| One shared PostgreSQL | none | none |
| `ai_memory` split off | `ai_memory` | `ai_memory` only |

**Why it has to work this way.** Locally each alias is a separate file (`evennia.db3`, `xrpl.db3`, `ai_memory.db3`, `subscriptions.db3`) and the routers direct each app's models to the right one; without them every table would land in `evennia.db3`. On a shared instance every alias *is* the same database, and a router there is actively harmful — its `allow_migrate()` would refuse to create the non-default tables while Django still recorded those migrations as applied, leaving a database that looks migrated and has none of the tables.

**The corollary:** an alias with an active router is not reached by a bare `migrate` and needs `migrate --database <alias>`.

---

## Transactions and Split Aliases

**A transaction covers one connection, not one process.** `transaction.atomic()` with no argument opens a transaction on `default`. When an alias is split, its router sends that app's queries out on a different connection — one the transaction does not cover. Writes there are not rolled back with it, and `select_for_update()` raises `TransactionManagementError`, because it would be issued in autocommit where the row lock would be released immediately.

So the transaction and the queries have to be on the same connection. **Derive the alias from the router** — ask it the same question the ORM asks itself:

```python
with transaction.atomic(using=router.db_for_write(FungibleGameState)):
    FungibleGameState.objects.select_for_update().get(...)
```

The querysets inside need nothing added: the router is already sending them to that alias, which is the point of deriving rather than naming. `db_for_write()` returns `None` when no router claims the model — the shared configuration — and `atomic(using=None)` means `default`, so the one line is right in every deployment shape.

Naming the literal works too, but only if *every* queryset in the block is pinned with `.using()` as well. `chain_sync.py` and `reallocation.py` do exactly that. Miss one and it goes to `default`: harmless where the alias is split, and a `TransactionManagementError` where it is not. Deriving has nothing to keep in sync, so nothing to forget.

A model with two homes is the exception. Evennia's own models live in both `default` and `archive`, so the router answers `default` and the archive code names its alias explicitly — `evennia_archive` pins every queryset with `.using(ARCHIVE_ALIAS)`.

### Work that spans two databases

No transaction can span two databases. PostgreSQL binds a connection to one database, and Django has no two-phase commit. Cross-boundary work uses nested blocks on the two connections:

```python
with transaction.atomic():                    # default
    ...game-side writes...
    with transaction.atomic(using="xrpl"):    # separate connection, separate transaction
        ...ownership writes...
```

Two rules make that safe:

- **The inner block goes last.** No writes on `default` after it — once it commits, nothing can undo it. Reads are fine.
- **The inner block is the ownership write.** The game-side object is derivable from the xrpl row; the reverse is not.

Which gives:

| Failure | `default` | `xrpl` |
|---|---|---|
| Inner block raises | rolled back | rolled back |
| Outer fails after the inner commits | rolled back | durable |
| Process dies between the two commits | rolled back | durable |

Rows two and three are the residual this design accepts: ownership recorded, the game world not yet updated. It is repairable, because the world can be re-derived from the xrpl row.

### Rolling back does not roll back Evennia's caches

A rollback restores the rows and nothing else. Evennia's `AttributeHandler` still holds the `Attribute` instances it wrote, so the object keeps reporting a balance the database no longer has, and nothing raises to say so. `reset_cache()` does not help — the re-fetch goes through the idmapper and gets the same instances back.

So every failure path following a transaction over Evennia state calls `discard_cached_attributes()` (`utils/attribute_cache.py`), which evicts the attributes from the idmapper first. Without it a failed transfer leaves the player looking at gold they do not have until the object is evicted or the server restarts.

### What the ledger changes

The XRPL ledger sits outside every transaction — no rollback reaches it. Two consequences shape the code:

- **A transaction cannot be held open across a swap.** The on-chain call runs first, outside the block, and only the recording goes inside it. `AMMService` splits into a swap half and a recording half for exactly this, so the mixin, the shopkeeper and the stew command can each place the seam correctly. Where the work is already split across a `deferToThread` boundary, the swap belongs to the worker thread and the transaction to the reactor callback.
- **A failure after the chain has moved needs a person, not a retry.** For deposits and withdrawals the chain has moved a player's own assets, so a failure there writes a `ReconciliationFailure` row — the exceptions list, read with the superuser `failures` command and closed out with `failures done <id> = <note>`. Only failures someone could act on are recorded; see `blockchain/xrpl/services/reconciliation.py`.

An AMM swap that happens without its recording is deliberately left alone. It moves assets the game already owns between its own vault and its own pool, so the only effect is a nudge to the next price. Nobody would unwind it by hand, and the player's balances are untouched on both sides — which is the pair that has to stay consistent.

### NFT moves and deletions

A move's ownership write already runs in the right place. Evennia calls `at_post_move()` as the last step of `move_to()`, and nothing writes after it returns, so `NFTMirrorMixin.move_to()` only has to wrap the call.

The rollback keys off the return value rather than an exception: `move_to()` never raises, because every step inside it is wrapped in Evennia's own `try/except`, which logs and returns `False`. A `False` return marks the transaction for rollback, which leaves the caller contract intact — a move that does not happen returns `False` rather than exploding. `False` also covers the ordinary refusals, where nothing was written and the rollback costs nothing.

Deletion needed an override for the opposite reason. Evennia calls `at_object_delete()` *first*, so the ownership write cannot live there — it has to happen once the object is destroyed, which is only reachable from `delete()`. That override holds the destruction and the write in one transaction, the write last. `at_object_delete()` keeps only what belongs before the deletion: an item's container cleanup.

Two seams make that work for both branches of the mixin:

- `_resolve_delete_disposition()` — where ownership is read from. An item uses its location chain; a pet is always standing in a room, so the chain would call every pet unowned and it answers from `owner_key` instead.
- `_mirror_on_delete()` — the write itself. Everything it needs is passed in, because the object no longer exists by the time it runs. `token_id` is an `AttributeProperty`: reading one off a deleted object raises, and then tries to write a default back to rows that are gone.

A rolled-back deletion restores the rows but not the Python instance — Django clears its pk and the idmapper has already evicted it. The failure path re-fetches by the pk captured beforehand and rebuilds the location's `contents_cache`, which is in-memory and untouched by the rollback. Whoever called `delete()` still holds the discarded instance and should not keep using it.

### Pets

A pet is an actor standing in a room, not an object in a character's inventory, which changes two things.

Ownership comes from `owner_key`, not from where the pet is — hence the `_resolve_delete_disposition()` override, and `_resolve_owner()` being disabled outright for pets.

And a pet changes hands without moving: `transfer_ownership()` updates `owner_key` and re-points who it follows, all in the same room. No move hook fires, so that method is the only record of the change and carries its own transaction. It returns `True` or `False` rather than raising, so a command can report the outcome. What the pet does next — following its new owner or waiting for them — is settled afterwards by reading `owner_key` back rather than assuming the write worked: a rolled-back transfer restores the pet to the state it was already in, so a pet told to wait beside its owner does not come back following them.

### Sweeps that run unattended

A command has a player waiting on it and can say what went wrong. Four paths destroy objects in batches with nobody watching, and each one has work queued behind the loop that matters more than any single item in it:

| Sweep | What sits behind the loop |
|---|---|
| `Corpse.despawn()` | Returning the corpse's gold and resources, then deleting the corpse — whose despawn timer has already fired and is never rescheduled |
| `RoomBase.at_object_delete()` | Deleting the room itself; a raise here leaves a stale duplicate behind on the next `wb_build` |
| `DungeonInstanceScript.collapse_instance()` | Reaching `state = "done"`; short of that the script keeps ticking and every room, exit and mob in the instance leaks |
| `TutorialInstanceScript.collapse_instance()` | Restoring the player's balances, granting the reward, returning them to the hub, releasing the instance |

So each loop guards **per item**, not per sweep, and always completes. Whatever survives stays alive and still owned, and Evennia's own `clear_contents()` relocates it when its container goes.

These log rather than record. An ownership write that fails has already written its own `ReconciliationFailure` row inside `NFTMirrorMixin.delete()`, so a second one at the sweep is a duplicate. A fungible return that fails leaves the amount parked at WORLD with the game and the mirror agreeing and no player out of pocket — noise on the exceptions list, not an entry for it. Same for a graduation reward that cannot be granted: the player is short a reward, not holding assets the ledger disagrees about.

### Proof of concept

Run 2026-08-24 on staging via `evennia shell`, against PostgreSQL with `xrpl` split:

| Check | Result |
|---|---|
| Bare `atomic()` + a routed queryset | raised, reproducing the production error |
| `atomic(using=X)` + `.using(X)` | `select_for_update` accepted |
| Inner block raises | both sides rolled back |
| Outer raises after the inner commits | `default` rolled back, `xrpl` durable |

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
evennia migrate --database archive
evennia start
```

**Migrate every alias before the first `evennia start`.** Evennia's `server.0002` is a data migration
that reads `ServerConfig.objects.all()` without pinning the alias, so migrating `archive` reads the
rows in `default` and tries to re-encode them. Before the first start those rows do not exist and the
migration is a no-op; after it, it fails with `UnpicklingError: pickle data was truncated`.

If you hit it on an already-running game, fake past it and continue:

```bash
evennia migrate server 0002_auto_20190128_2311 --database archive --fake
evennia migrate --database archive
```

Safe because the archive's own `server_serverconfig` is empty, so the data migration has nothing to
convert. The `AlterField` skipped alongside it is a no-op on SQLite; re-check before relying on this
on PostgreSQL.

SQLite database files appear in `server/`. They're in `.gitignore`. Each developer has their own local data.

### Running tests

```bash
evennia test --settings settings tests
```

**Tests run on SQLite.** That is the working assumption, not something the code enforces — nothing
pins it, so a suite launched from a shell that has loaded the Postgres environment would build its
test databases there instead. Local development is SQLite, so in practice that is what runs.

Staging is the only environment where the game runs on Postgres, and it is not somewhere to tie up
the databases for the two to three hours the full suite takes. Postgres on the current development
machine does not work against the same code that runs on staging; the difference has not been chased.

The cost of that is worth knowing: SQLite reports no `select_for_update` support, so Django skips both
the lock and its checks. A green suite says nothing about row locking or about a transaction being
opened on the wrong connection — staging is where that class of thing is confirmed.

Revisit when the infrastructure grows a dedicated test environment. Not before.

### Testing against PostgreSQL locally

Local Postgres is kept at the same version and configuration as the server, so a result on a laptop
means the same thing on the box. Which backend a shell uses depends on whether it has loaded the
environment file — a fresh shell is always SQLite.

Commands, the encrypted environment file, and the two traps worth knowing (setting `DATABASE_URL` also
disables `secret_settings.local`; the server is a daemon and does not follow a shell that changes mode)
are in `ops/DEVELOPMENT/DEV_SETUP.md` § Choosing a database backend.

---

## How Database Migrations Work

This is one of Django's best features, and it's worth understanding even if you're not a Django expert.

### What's a migration?

A migration is a Python file that describes a change to the database schema — "add this column," "create this table," "add this index." Django generates these automatically when you change a model.

### The developer workflow

```
1. Developer changes a model (e.g. adds a field to NpcMemory)
     ↓
2. Developer runs: evennia makemigrations ai_memory
     → Django compares models.py to existing migrations
     → Generates a new migration file (e.g. 0003_add_mood_field.py)
     ↓
3. Developer runs: evennia migrate --database ai_memory
     → Locally every alias is its own file, so its router is active
     → Migration applies to the local ai_memory.db3 file
     → The --database flag is required; the router blocks the bare call
     ↓
4. Developer commits the migration file to Git
     ↓
5. Code is merged and deployed
     ↓
6. deploy_migrate.py runs on the server, before evennia start
```

**The rule is the same in both places:** an alias with an active router needs its own `migrate --database <alias>`; everything sharing `default`'s database is covered by the bare `migrate`. Only the *number* of split aliases differs — four locally, however many overrides are set on a server.

### deploy_migrate.py

Run by hand on the server before the first `evennia start` against a new database:

```bash
python deploy_migrate.py
```

It calls Django directly rather than going through `evennia migrate`, because Evennia's launcher has its own database initialisation path that does not reliably pick up `DATABASE_URL` for the non-default aliases. It reads `DJANGO_SETTINGS_MODULE` from the environment — it never sees a `--settings` argument, so that variable is what selects the settings module.

What it does, in order:

1. **Prints the resolved connections** — engine, host and name for every alias, which aliases share a database, and which are split off. The startup banner is the fastest way to confirm a deploy is pointed where you think it is.
2. **Hard-gates the engine** — if `DATABASE_URL` is set but the resolved engine is not Postgres, it aborts rather than quietly migrating a throwaway SQLite file.
3. **Probes every distinct database** — a bad host in an override fails here, before any migration runs.
4. **Creates the `vector` extension** in each database that will hold embeddings. Extensions are per-database, so this is not necessarily `default`'s.
5. **Counts tables before and after.**
6. **Migrates** — one bare `migrate`, then `migrate --database <alias>` for each split alias.
7. **Fails if any database ends up with zero tables.** This is the guard that matters: a router can block table creation while Django still records the migrations as applied, leaving a database that looks migrated and holds nothing. The census catches it instead of the game finding out at runtime.

Any error aborts with exit code 1 rather than leaving you to start the server against a broken database.

**Migrations are incremental and idempotent.** Django only applies what is not already in `django_migrations`, so running it on every deploy is safe and existing data is preserved.

### pgvector

The extension must exist before any migration that creates a vector column, which is why step 4 precedes step 6. `ai_memory.0002_pgvector` is marked `atomic = False` because HNSW index creation blocks when run inside a transaction on PostgreSQL.

Vector searches also need `hnsw.iterative_scan` set, or a filtered search returns only the candidates that survive its `WHERE` clause — see [Vector search settings](#vector-search-settings) below.

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
5. Push, then run `deploy_migrate.py` on the server as part of the deploy

### "I changed a field on an existing model"

Same process. Django generates a migration that alters the column. Works on both SQLite and PostgreSQL.

### "I need to add seed data"

Use a data migration — a migration file that runs Python code to insert rows. The existing `0001_initial.py` in the xrpl app does this (it seeds 37 currency types and all NFT item types). Same migration runs on both backends.

### "Something broke on staging but works locally"

This is rare because the ORM abstracts backend differences. When it happens, it's usually:
- A raw SQL query that uses SQLite-specific syntax (avoid raw SQL)
- A timing/concurrency issue that only surfaces with real PostgreSQL (SQLite serializes all writes)
- An environment variable that's set locally but missing on the server

---

## pgvector for AI Memory

The NPC memory system uses **dual-backend embedding storage** that automatically adapts to the database engine:

| Backend | Field | Search Method | Index |
|---------|-------|---------------|-------|
| **SQLite** (local dev) | `embedding` (`BinaryField`, numpy binary blob) | Python loop with numpy cosine similarity — O(n) | None |
| **PostgreSQL** (deployed) | `embedding_vector` (`VectorField(1536)`, pgvector native) | Single SQL query with `<=>` cosine distance operator | HNSW (`m=16, ef_construction=64`) |

Both fields coexist on the `NpcMemory` model. Backend detection is automatic — `_is_postgres()` in `ai_memory/services.py` checks `settings.DATABASES["ai_memory"]["ENGINE"]`, which follows the existing `DATABASE_URL` toggle. No configuration, feature flags, or manual switching required.

### How it works

**Storage (`store_memory()`):** After generating an embedding via OpenAI `text-embedding-3-small` (1536 dimensions), the raw `list[float]` is written to `embedding_vector` on Postgres or converted to `np.float32.tobytes()` and written to `embedding` on SQLite.

**Search (`search_memories()`):** On Postgres, uses pgvector's `CosineDistance` ORM annotation — a single indexed SQL query replaces the Python loop. On SQLite, the existing numpy cosine similarity loop runs unchanged.

**Migration (`0002_pgvector.py`):** Conditionally enables the `vector` extension, adds the column, creates the HNSW index, and back-fills existing binary embeddings into the vector column. All conditional steps check `connection.vendor == "postgresql"` and are no-ops on SQLite.

### Vector search settings

A filtered vector search — `WHERE npc_id = X ORDER BY embedding <=> ...`, which is what `search_memories()` and the lore search both do — asks HNSW for `ef_search` candidates and applies the filter *afterwards*. Only the survivors come back. As the table grows and any one NPC's share of it shrinks, the query returns fewer rows than asked for while the relevant memories sit untouched in the table. Measured on 100k rows across 500 NPCs: **1 row returned of a requested 5**.

`hnsw.iterative_scan = relaxed_order` fixes it — the index keeps producing candidates until the filter yields enough. Requires pgvector 0.8.0 or later.

It is set in two places, deliberately:

| Where | Covers |
|---|---|
| `OPTIONS` on every Postgres connection, from `server/conf/db_config.py` | The application. Travels with the connection, so it follows a database to another cluster or to RDS, where there is no `postgresql.conf` to edit |
| `postgresql.conf` on the server | Manual `psql` sessions, so hand-run diagnostics behave the way the app does |

The connection setting is the one that matters. The server setting only stops a `psql` session quietly misleading you.

### Performance

The SQLite path works fine at a few thousand memories per NPC. The pgvector path with HNSW indexing scales to millions of vectors with sub-millisecond search — relevant as the memory system expands to three embedding tables:

| Table | Purpose | Design doc |
|---|---|---|
| `NpcMemory` | NPC interaction/dialogue memory | This doc (above) |
| `CombatMemory` | Combat encounter tactical history | [combat-ai-memory.md](combat-ai-memory.md) |
| `LoreMemory` | Embedded world knowledge | [lore-memory.md](lore-memory.md) |

All three share the same dual-backend infrastructure, HNSW indexing, and `_is_postgres()` branching. See [npc-mob-architecture.md](npc-mob-architecture.md) § Three Memory Systems for how they compose at prompt time.

### Future cleanup

Once confident in the pgvector path after production validation:
- Remove the `embedding` BinaryField and numpy conversion code
- Remove `_cosine_similarity()` helper
- Consider removing `numpy` from requirements if no other code uses it

---

## Summary

The database architecture is designed around one principle: **the environment decides the backend and the placement, not the code.**

- Develop locally with SQLite (zero setup)
- Deploy with PostgreSQL (zero code changes)
- `DATABASE_URL` chooses the backend; `DATABASE_URL_<ALIAS>` chooses where an individual alias lives
- Routers follow from the resolved connections rather than being declared
- Django handles schema differences, migrations, and SQL dialect translation
- Same migration files run on both backends
- Five logical databases, on one PostgreSQL instance or several, decided per deployment
- A transaction covers one connection — see [Transactions and Split Aliases](#transactions-and-split-aliases)
