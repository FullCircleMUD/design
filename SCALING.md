# Scaling Strategy — Initial Era

> **Status: Draft strategy.** Nothing in this document is implemented. Scope is bounded to the **single-Postgres era** — from one Evennia Server today through however many shards we can run against a single (vertically scaled) Postgres instance. If we ever exhaust that, we will be at a revenue scale where dedicated infrastructure expertise is affordable, and the design beyond that point is explicitly deferred.

---

## Goals & Non-Goals

**Goals**

- Provide a path from 1 → N Evennia Server processes ("shards") sharing one Postgres, without breaking gameplay.
- Use existing structural boundaries (zones, gateway rooms) so the sharding seam matches the world.
- Keep the single-shard deployment a special case of the multi-shard design (shard count = 1), so we don't carry two architectures.
- Define a small, concrete PoC: two shards, one shared Postgres, one zone handoff working end-to-end.

**Non-goals**

- Multi-Postgres / multi-region / read-replica topology. Out of scope.
- Geographic distribution, edge presence, latency-driven sharding. Out of scope.
- Any redesign motivated by "what if we had millions of players." If we get there, we hire.
- Hot character migration mid-combat, mid-spell, or mid-script. Handoff is gated by safe state.

---

## The Sharding Seam

Zones are the natural boundary. The world already has [`RoomGateway`](../src/game/typeclasses/terrain/rooms/) rooms at every zone-to-zone transition (see `design/INTERZONE_TRAVEL.md`). These rooms are the only places a character can cross between zones, and they are designed to feel like a beat — a trailhead, a dock, a pass — where a brief reconnect is unsurprising.

Each shard owns a set of zones. A character is **resident on exactly one shard at a time** — the shard whose zones contain her current room. There is no point in the design where the same character is live on two shards.

### What lives where

| State | Lives on | Notes |
|---|---|---|
| `AccountDB` (login, characters list, account bank) | Account-router (separate process) | One shard does not own accounts |
| `ObjectDB` for resident objects (characters, items, mobs, rooms in owned zones) | The owning shard | Idmapper-cached on that shard only |
| `FungibleGameState` SINK rows | Per-shard rows; aggregated hourly | See **Per-shard SINK** below |
| `FungibleGameState` RESERVE rows | Postgres; written once an hour by a coordinator | Shards read their pre-allocated share, not RESERVE directly |
| `ChannelDB` | Postgres; messages bridged via Redis pub/sub | Each shard subscribes to all channel topics |
| Mail | Postgres; pulled at post offices | Naturally shard-agnostic |
| Telemetry snapshots (`EconomySnapshot`, `SaturationSnapshot`, `ResourceSnapshot`) | Postgres; written by a single coordinator | Already hourly batch operations |

---

## The Cache Invariant

> **If an object is not resident on your shard, you do not cache it.**

This is the rule that makes Evennia's idmapper safe across shards. Evennia's `SharedMemoryModel` (see `evennia/utils/idmapper/models.py`) and `AttributeHandler._cache` (see `evennia/typeclasses/attributes.py:488`) are both per-process caches with no built-in cross-process invalidation. As long as no two shards hold a live cached instance of the same row at the same time, there is no drift.

The handoff protocol enforces this invariant: a character is evicted from the source shard's idmapper *before* the target shard loads it.

The account-router service holds `AccountDB` rows but explicitly **does not** load `ObjectDB` rows for characters (it only knows "character X is currently resident on shard Y" — a small mapping table). Shards never load `AccountDB` rows.

---

## The Shared-State Problem, Bounded

The genuinely shared state in FCM is small enough to enumerate and address row-by-row. From an audit of `src/game/blockchain/xrpl/models.py`:

### `FungibleGameState` SINK rows — write-hot, refactor required

Today, every gold-spending action (crafting fees, repair, training, travel cost, junking, AMM rounding dust) writes to a single `FungibleGameState` row per `(currency, vault, location_type=SINK)`. Multiple shards writing this row would lost-update.

**Fix:** add `shard_id` to the unique key. Each shard writes its own SINK row. The hourly `reallocate_sinks()` job sums across shards before draining to RESERVE and resets per-shard rows to zero. Telemetry queries that today read `WHERE location_type='SINK'` become aggregating queries.

For `shard_count = 1` this is a no-op refactor — one shard_id, same hot-path semantics.

### `FungibleGameState` RESERVE rows — read-hot, allocate as shares

Today, the spawn algorithm reads RESERVE on every spawn decision to know what's available. Multiple shards reading would be safe under MVCC, but the row becomes a contention point on the writes that *do* happen.

**Fix:** at the top of each hour, the coordinator reads RESERVE and allocates a per-resource share to each shard, weighted by the prior hour's per-shard consumption (already trackable via the per-shard SINK rows). Each shard spawns from its local share and never reads RESERVE on the hot path. Leftover share returns to RESERVE at the next hourly reconciliation.

If a shard's share for a given resource runs out mid-hour, it spawns lean for the remainder. Players naturally migrate or trade. Cross-shard borrow can be added later if telemetry shows a real problem; we do not build it up front.

### `EnchantmentSlot` — already correctly engineered

`EnchantmentSlot` rows ([`models.py:586`](../src/game/blockchain/xrpl/models.py#L586)) use `select_for_update()` and race-loss is safe (no materials consumed on a lost race). Postgres row-level locking handles cross-shard contention correctly. **No changes needed.**

### Channels — bridge via Redis pub/sub

Evennia's `ChannelDB` is a database-backed channel registry, but message broadcast is in-process: when shard A's `Channel.msg()` fires, only sessions on shard A receive it. To bridge, each shard:

1. Subscribes to a Redis topic per channel on startup.
2. On local broadcast, also publishes to the topic.
3. On receiving a published message from another shard, delivers it to local subscribers without re-publishing (loop break via shard-id stamp).

`ChannelDB` row reads themselves stay cached per-shard — channel config is effectively immutable at runtime, and any admin channel-config change can broadcast an invalidate over the same bus.

### Tells, who, scry — RPC by current-shard lookup

Each character row gains a `current_shard` attribute (or sits in the account-router's character→shard mapping table). Sender's shard looks up the target's current shard, RPCs the target shard with the message. The same pattern handles `who`, `where`, scrying, and any other "find another player" command.

This is a tiny RPC surface — small enough that a single Redis-backed message channel per shard is sufficient transport. We do not need a generalised service mesh.

### Mail — already shard-agnostic

Mail items live in Postgres rows, delivered when a player visits a post office. No bridging needed. The recipient's shard reads from the DB at delivery time. This is incidentally already correct.

---

## The Handoff Protocol

When a character `walk`s through a `RoomGateway` and the destination room is on a different shard:

1. **Quiesce on source shard.** The character must be in safe state — not in combat, not casting, no in-flight `utils.delay()` callbacks tied to her object. Gateway rooms are designed as safe rooms in `INTERZONE_TRAVEL.md`, so this is naturally satisfied; if the character is somehow not safe, the handoff is refused with a normal "you cannot leave while X" message.
2. **Persist any non-DB state.** `ndb` is by definition non-persisted; we either explicitly drain it (preferred — most `ndb` keys are session-local UI state that re-derives) or formally serialize a small allowlist of keys at the gateway. Scripts attached to the character are unscheduled from the source shard's `TickerHandler`.
3. **Update the router.** Account-router updates `character.current_shard = shard_B`. This is the linearisation point of the handoff — after this write, target shard owns the character.
4. **Evict from source idmapper.** Source shard removes the character (and her carried inventory rows, and any followers) from its idmapper and `AttributeHandler` caches.
5. **Reconnect session.** The player's connection drops to the gateway and re-establishes against shard B's Portal. The player sees a brief "you cross the trailhead..." beat. **This is the cheapest mechanism that works** — modifying Evennia's Portal to multiplex across Server processes is out of scope for the initial era.
6. **Load on target shard.** Shard B loads the character via normal idmapper mechanisms, re-attaches scripts to its own `TickerHandler`, and places her in the destination room.

Crash recovery: if the source shard dies between steps 3 and 4, the router's `current_shard` write is authoritative. The dead shard's idmapper is gone with its process; on restart, it will not resurrect the character because she now belongs to shard B.

---

## What This Looks Like at `shard_count = 1`

Every claim above degenerates cleanly:

- One SINK row per currency (just `shard_id = 0`).
- The "hourly share allocation" allocates 100% to the one shard.
- The Redis bus has one publisher and one subscriber — still works, low overhead.
- The account-router runs as a separate process but only ever points at one shard.
- The handoff protocol is never invoked because every gateway room is intra-shard.

This is the point: we should be able to deploy the **multi-shard architecture from day one**, running it with `shard_count = 1`, and only flip the count up when we need to. There is no flag-day rewrite.

---

## Proof-of-Concept Scope

The PoC is the smallest configuration that exercises every novel piece:

- **Two Evennia shards**, one shared Postgres, one Redis instance, one account-router process.
- **A static zone-to-shard mapping** in config (no dynamic rebalancing). One zone on shard A, one on shard B, connected by exactly one gateway pair.
- **The cache invariant enforced**: shard A loads no rows owned by shard B.
- **One handoff path tested end-to-end**: walk through the gateway, see the brief reconnect, arrive on the other shard with inventory intact.
- **Per-shard SINK refactor**: the migration applied, both shards writing their own SINK rows, hourly reallocation aggregating correctly.
- **Channel bridging**: `say` and a public channel both work across the two shards.
- **Tell across shards**: `tell <player on B>` from shard A delivers correctly.

Out of PoC: cross-shard combat, cross-shard pets following, dynamic zone reassignment, automated rebalancing.

---

## What We Are Explicitly Not Solving

These are deferred to the post-single-Postgres era:

- Multi-Postgres sharding or any distributed-database design.
- Read replicas or read/write splitting.
- Cross-region or multi-datacenter deployment.
- Live (non-reconnect) session migration across shards.
- Dynamic load-aware zone reassignment.
- Cross-shard combat, party mechanics across shards, follower trains across shards.

If the game grows to where any of these become necessary, that is a milestone we can fund expertise to address. The single-Postgres era should give us substantial runway — Postgres on a well-tuned box is not the limit anyone realistically hits first in a MUD.

---

## Open Questions

- Where does the account-router live in the codebase? Likely a new top-level Django app that owns `AccountDB` and a `CharacterShardAssignment` model. Needs design.
- What's the exact eviction API on `AttributeHandler` and the idmapper, and is it safe to call mid-flight? Needs spike.
- Redis vs NATS vs Postgres `LISTEN/NOTIFY` for the bus. `LISTEN/NOTIFY` is tempting because it adds zero new infrastructure, but its delivery guarantees are weaker than Redis pub/sub. Worth a small comparison before committing.
- How do we represent `shard_id = 0` in the migration without rewriting every existing SINK row? Likely a default-zero column on the existing rows + a unique constraint change.
