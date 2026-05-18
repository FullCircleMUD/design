# Scaling Strategy — Initial Era

> **Status: partly shipped, partly designed.** The mechanism (cross-shard
> isolation, character handoff, ticket auth, cross-shard message bus, the
> router/shard role split) is implemented in the
> [`evennia-shards`](https://github.com/FullCircleMUD/evennia-shards)
> library and integrated into FCM via the `settings_router.py` /
> `settings_shard0.py` / `settings_shard1.py` cascade in
> `server/conf/`. FCM-specific shared-state work (per-shard SINK,
> RESERVE share allocation, channel bridging, cross-shard tells, the
> cross-shard `CrossShardExit` typeclass) is documented below but
> deferred until needed. Local-machine PoC on Windows is verified —
> three processes (router + shard0 + shard1) running from one game
> folder, world-builder and mob-spawner both shard-aware, cross-shard
> admin moves via the library's `CmdCrossShardMove` working
> end-to-end. Scope of this document is the **single-Postgres era**;
> beyond that is explicitly deferred (we will be at a revenue scale
> where dedicated infrastructure expertise is affordable).

---

## What's shipped vs. still on the roadmap

For a reader who just wants to know what they can rely on today vs.
what is still design work:

**Shipped (in the library, transparent to FCM):**

- Role-aware deployment (`monolith` / `router` / `shard`) — config-only
  selection per process via `SHARDS_ROLE` setting.
- `shard_id` column on `ObjectDB` with auto-stamp on save.
- Four read/write chokepoints (`from_db`, `pre_save`, `pre_delete`,
  `QuerySet.update`) that enforce the per-row shard partition.
- `cross_shard_character_move` primitive — atomic DB writes, recursive
  inventory move, idmapper eviction, per-session ticket redirect.
- Ticket-based WebSocket auth + connection-level cross-shard redirect.
- Chargen wrapper that stamps new characters with the start-room's
  `shard_id`.
- Postgres-polled cross-shard message bus (`obj_msg`, `account_msg`,
  `send_cross_shard_message`).
- Admin commands: `@shard_check`, `cross_shard_dig`, `cross_shard_move`.

**Shipped (in FCM, on top of the library):**

- Settings cascade (`settings_router.py`, `settings_shard0.py`,
  `settings_shard1.py`, `settings_common_shard_config.py`).
- `AccountBank` stamped `shard_id="*"` (global asset, loadable from any
  shard) — `at_post_login` and `ensure_bank` both set this.
- `_restart_purgatory_timers` scoped per shard (router skips entirely,
  each shard scopes its character query to its own partition).
- `at_pre_puppet` / `at_post_puppet` integrity-check `home` and
  `respawn_location` instead of healing with global `search_tag`.
- Per-shard scaffold rooms in `fcm-world` (`shard0/scaffold/`,
  `shard1/scaffold/`) — every shard has its own Purgatory and
  `nft_recycle_bin`.

**Roadmap (FCM-side, in this document):**

- Per-shard `FungibleGameState` SINK rows (shard_id added to unique key,
  hourly reallocation aggregates across shards).
- Hourly RESERVE share allocation by a coordinator (each shard spawns
  from its pre-allocated share, no hot-path RESERVE reads).
- Channel bridging across shards — currently each `Channel.msg()` is
  shard-local; cross-shard delivery requires using the library's
  message bus primitives (`account_msg`, `obj_msg`) under the hood.
- Cross-shard tells / who / where — sender's shard looks up target's
  current shard, RPC's the target shard with the message.
- `CrossShardExit` consumer typeclass — for normal player movement
  through zone gateways that cross shards. Currently only admin
  commands (`cross_shard_move`) invoke the primitive.
- Cross-zone arrival hook that updates `db_home` and
  `respawn_location` to the new shard's "trackside graveyard" room so
  the puppet integrity check holds after any cross-shard move.

---

## Goals & Non-Goals

**Goals**

- Provide a path from 1 → N Evennia Server processes ("shards")
  sharing one Postgres, without breaking gameplay.
- Use existing structural boundaries (zones, gateway rooms) so the
  sharding seam matches the world.
- Keep the single-shard deployment a special case of the multi-shard
  design (shard count = 1), so we don't carry two architectures.

**Non-goals**

- Multi-Postgres / multi-region / read-replica topology. Out of
  scope (and the library's
  [`single-Postgres-bound` principle](https://github.com/FullCircleMUD/evennia-shards/blob/main/CLAUDE.md#load-bearing-architectural-principles)
  reinforces this at the library level).
- Geographic distribution, edge presence, latency-driven sharding.
- Any redesign motivated by "what if we had millions of players." If
  we get there, we hire.
- Hot character migration mid-combat, mid-spell, or mid-script.
  Handoff is gated by safe state — the
  [library's `consumer-constraints.md`](https://github.com/FullCircleMUD/evennia-shards/blob/main/DESIGN/consumer-constraints.md)
  makes this an explicit invariant.

---

## The Sharding Seam

Zones are the natural boundary. The world already has
[`RoomGateway`](../src/game/typeclasses/terrain/rooms/) rooms at every
zone-to-zone transition (see
[`INTERZONE_TRAVEL.md`](INTERZONE_TRAVEL.md)). These rooms are the only
places a character can cross between zones, and they are designed to
feel like a beat — a trailhead, a dock, a pass — where a brief
reconnect is unsurprising.

Each shard owns a set of zones. A character is **resident on exactly
one shard at a time** — the shard whose zones contain her current
room. There is no point in the design where the same character is
live on two shards. The library enforces this at the row level via
the `shard_id` column and the four chokepoints.

### What lives where

| State | Lives on | Notes |
|---|---|---|
| `AccountDB` (login, characters list) | Router | Library: `SHARDS_ROLE=router` process owns account auth and the OOC menu |
| `AccountBank` (per-account asset row) | Global (`shard_id="*"`) | Any shard can load. Stamped by FCM's `Account.at_post_login` via `shard_writes_allowed_for` |
| `ObjectDB` for resident objects (characters, items, mobs, rooms in owned zones) | The owning shard | Library auto-stamps on save; chokepoints refuse cross-shard reads |
| `FungibleGameState` SINK rows | Per-shard rows; aggregated hourly | **TODO** — needs migration to add `shard_id` to unique key. See § Per-shard SINK |
| `FungibleGameState` RESERVE rows | Postgres; written once an hour by a coordinator | **TODO** — shards read their pre-allocated share, not RESERVE directly. See § RESERVE shares |
| `ChannelDB` | Postgres; messages bridged via library's message bus | **TODO** — each shard subscribes to all channel topics; cross-shard delivery rides `account_msg`/`obj_msg` |
| Mail | Postgres; pulled at post offices | Naturally shard-agnostic |
| Telemetry snapshots (`EconomySnapshot`, `SaturationSnapshot`, `ResourceSnapshot`) | Postgres; written by a single coordinator | Already hourly batch operations |

---

## The Cache Invariant

> **If an object is not resident on your shard, you do not cache it.**

This is the rule that makes Evennia's idmapper safe across shards.
Evennia's `SharedMemoryModel` (see
`evennia/utils/idmapper/models.py`) and `AttributeHandler._cache`
(see `evennia/typeclasses/attributes.py:488`) are both per-process
caches with no built-in cross-process invalidation. As long as no two
shards hold a live cached instance of the same row at the same time,
there is no drift.

**The library enforces this by construction.** Four chokepoints
(`from_db`, `pre_save`, `pre_delete`, `QuerySet.update`) refuse any
operation that would read or write a row stamped with a different
`shard_id`. A consumer (FCM, in our case) cannot accidentally cache
a foreign shard's row — the underlying `from_db` would raise
`ShardIsolationError` before the idmapper ever sees the data.

The `cross_shard_character_move` primitive in the library handles
eviction explicitly: source shard updates `shard_id`, evicts the
character (plus recursive inventory) from its idmapper and
`AttributeHandler` caches, then redirects the session to the target
shard via a ticket-authenticated WebSocket reconnect. See the
library's
[`handoff.py`](https://github.com/FullCircleMUD/evennia-shards/blob/main/src/evennia_shards/handoff.py)
for the implementation.

---

## The Shared-State Problem, Bounded

The genuinely shared state in FCM is small enough to enumerate and
address row-by-row. From an audit of
[`blockchain/xrpl/models.py`](../src/game/blockchain/xrpl/models.py):

### `AccountBank` — solved (global stamp)

`AccountBank` rows are account-attached (1:1 with the router-owned
account). They may be read/written from whichever shard the account
is currently puppeting on. FCM stamps newly-created banks with
`shard_id="*"` (global sentinel) via the library's
`shard_writes_allowed_for` bypass primitive — both at the canonical
creation site (`Account.at_post_login`) and the defence-in-depth site
(`ensure_bank` in `cmd_balance.py`). Production characters move
across shards freely without their bank tripping the chokepoint.

### `FungibleGameState` SINK rows — TODO, refactor required

Today, every gold-spending action (crafting fees, repair, training,
travel cost, junking, AMM rounding dust) writes to a single
`FungibleGameState` row per `(currency, vault, location_type=SINK)`.
Multiple shards writing this row would lost-update.

**Fix:** add `shard_id` to the unique key. Each shard writes its own
SINK row. The hourly `reallocate_sinks()` job sums across shards
before draining to RESERVE and resets per-shard rows to zero.
Telemetry queries that today read `WHERE location_type='SINK'` become
aggregating queries.

For `shard_count = 1` this is a no-op refactor — one shard_id, same
hot-path semantics.

### `FungibleGameState` RESERVE rows — TODO, allocate as shares

Today, the spawn algorithm reads RESERVE on every spawn decision to
know what's available. Multiple shards reading would be safe under
MVCC, but the row becomes a contention point on the writes that *do*
happen.

**Fix:** at the top of each hour, the coordinator reads RESERVE and
allocates a per-resource share to each shard, weighted by the prior
hour's per-shard consumption (already trackable via the per-shard
SINK rows). Each shard spawns from its local share and never reads
RESERVE on the hot path. Leftover share returns to RESERVE at the
next hourly reconciliation.

If a shard's share for a given resource runs out mid-hour, it spawns
lean for the remainder. Players naturally migrate or trade.
Cross-shard borrow can be added later if telemetry shows a real
problem; we do not build it up front.

### `EnchantmentSlot` — already correctly engineered

`EnchantmentSlot` rows
([`models.py:586`](../src/game/blockchain/xrpl/models.py#L586)) use
`select_for_update()` and race-loss is safe (no materials consumed on
a lost race). Postgres row-level locking handles cross-shard
contention correctly. **No changes needed.**

### Channels — TODO, bridge via the library's message bus

Evennia's `ChannelDB` is a database-backed channel registry, but
message broadcast is in-process: when shard A's `Channel.msg()`
fires, only sessions on shard A receive it. To bridge, each shard:

1. On local `Channel.msg()`, also publishes via
   `send_cross_shard_message` (library primitive) to every other
   shard, naming the local-channel-PK as the target.
2. The receiving shard's `MessageHandler` subclass routes the
   inbound `obj_msg` back into its local `Channel.msg()` chain
   (with a loop-break flag so we don't re-publish).
3. `ChannelDB` row reads themselves stay cached per-shard — channel
   config is effectively immutable at runtime, and any admin
   channel-config change can broadcast an invalidate over the same
   bus.

Earlier drafts of this document referenced Redis pub/sub; the library
landed on Postgres-polled `LoopingCall` instead (no Redis dependency,
same delivery guarantees, simpler ops). The cross-shard bus is
the library's, not FCM's; FCM only writes the *kind handlers* on
top of it.

### Tells, who, scry — TODO, RPC by current-shard lookup

Each character row's `shard_id` (library-maintained) is sufficient
to route. Sender's shard looks up the target's `shard_id` via a
`.values_list` query (no `from_db`, no chokepoint), then uses
`send_cross_shard_message` to deliver the message to the target
character on the target shard. The same pattern handles `who`,
`where`, scrying, and any other "find another player" command.

This is a tiny RPC surface — small enough that the library's existing
message bus is sufficient transport. We do not need a generalised
service mesh.

### Mail — already shard-agnostic

Mail items live in Postgres rows, delivered when a player visits a
post office. No bridging needed. The recipient's shard reads from
the DB at delivery time. This is incidentally already correct.

---

## The Handoff Protocol

The handoff is implemented in the library's
[`cross_shard_character_move`](https://github.com/FullCircleMUD/evennia-shards/blob/main/src/evennia_shards/handoff.py)
primitive — atomic DB writes, recursive inventory move, idmapper
eviction, per-session ticket-authenticated WebSocket redirect. The
[`ticket-auth-flow.md`](https://github.com/FullCircleMUD/evennia-shards/blob/main/DESIGN/ticket-auth-flow.md)
in the library documents the wire protocol.

FCM consumes the primitive in two places today:

1. **Admin command** — `cross_shard_move` (library-shipped admin
   command, auto-installed into `CharacterCmdSet` when running in a
   non-monolith role) lets a Developer-locked caller move themselves
   to any room PK on any configured shard. Used during world
   bootstrapping and live diagnosis.
2. **TODO: `CrossShardExit` typeclass** — for normal player movement.
   An exit on the source shard whose `at_traverse` checks safe state
   (not in combat, not casting, no in-flight delayed callbacks),
   updates `db_home` and `respawn_location` to the target shard's
   "trackside graveyard" room (preserving the shard-local invariant
   the puppet integrity check enforces), then calls
   `cross_shard_character_move`. This is the consumer-side gating
   layer the library leaves for FCM to write per its
   [`consumer-constraints.md`](https://github.com/FullCircleMUD/evennia-shards/blob/main/DESIGN/consumer-constraints.md).

Crash recovery: handled by the library. The `shard_id` write is the
linearisation point; if the source shard dies mid-handoff before
session redirect, the row already belongs to the target and a future
puppet attempt routes there.

---

## What This Looks Like at `shard_count = 1`

Every claim above degenerates cleanly:

- One SINK row per currency (just one `shard_id` value).
- The "hourly share allocation" allocates 100% to the one shard.
- The message bus has one publisher and no other subscribers — still
  works, low overhead.
- The router runs as a separate process but only ever redirects to
  one shard.
- The handoff protocol is never invoked because every gateway room
  is intra-shard.

The architecture lives at `shard_count = 1` today, with the library
mechanism in place but the FCM-side shared-state work
(SINK/RESERVE/channels/tells) still using monolith-shaped code paths
that happen to be correct under one shard. Adding `shard1` lights up
the FCM-side TODOs.

---

## What We Are Explicitly Not Solving

Deferred to the post-single-Postgres era:

- Multi-Postgres sharding or any distributed-database design.
- Read replicas or read/write splitting.
- Cross-region or multi-datacenter deployment.
- Live (non-reconnect) session migration across shards.
- Dynamic load-aware zone reassignment.
- Cross-shard combat, party mechanics across shards, follower trains
  across shards.

If the game grows to where any of these become necessary, that is a
milestone we can fund expertise to address. The single-Postgres era
should give us substantial runway — Postgres on a well-tuned box is
not the limit anyone realistically hits first in a MUD.

---

## Open Questions — current state

- ~~Where does the account-router live in the codebase?~~ —
  **Resolved.** It's a role in the
  [`evennia-shards`](https://github.com/FullCircleMUD/evennia-shards)
  library, selected via `SHARDS_ROLE=router` in the FCM
  `settings_router.py` config. No separate Django app needed; the
  router process owns `AccountDB` natively because every Evennia
  process owns it; the cross-shard partition is at the `ObjectDB`
  level.
- ~~Eviction API on `AttributeHandler` and the idmapper?~~ —
  **Resolved.** Library uses `flush_from_cache(force=True)` plus
  `refresh_from_db()` documented in
  [`shard-isolation.md`](https://github.com/FullCircleMUD/evennia-shards/blob/main/DESIGN/shard-isolation.md#cross-process-cache-staleness).
- ~~Redis vs NATS vs Postgres `LISTEN/NOTIFY` for the bus?~~ —
  **Resolved.** Library shipped Postgres-polled `LoopingCall`. No
  external bus dependency. See
  [`cross-shard-message-bus.md`](https://github.com/FullCircleMUD/evennia-shards/blob/main/DESIGN/cross-shard-message-bus.md).
- ~~How to represent `shard_id = 0` in the migration without
  rewriting every existing SINK row?~~ — **Resolved.** Not needed:
  the library uses string values (`"shard0"`, `"shard1"`, `"*"` for
  global). The FCM SINK migration (still TODO) just adds `shard_id`
  to the unique key with a default of the current single-shard value.

**Still open:**

- Cross-shard arrival hook design — how exactly does `db_home` and
  `respawn_location` get re-pointed when a character lands on a new
  shard via `CrossShardExit`? The current proposal is "trackside
  graveyard at every zone entrance, auto-updates home + cemetery"
  as a fallback (players are expected to re-set both at inns /
  cemeteries on the new shard). This is design-discussed but not
  yet captured as code or as a separate doc.
- Recall spell semantics across shards — recall should only succeed
  if `home` is on the current shard; otherwise fail with a clear
  message rather than auto-fallback (auto-fallback is an
  exploration-bypass exploit). Not yet implemented.
- Cross-shard `look <character>` / `who` / `where` UX — the
  message bus delivers the data but the player-facing rendering
  hasn't been designed yet.

---

## Local-machine PoC milestones

Verified on Windows during the shards-integration work:

- Three Evennia processes (router + shard0 + shard1) running from one
  `src/game/` folder, distinguished only by `--settings settings_<role>`.
  No view-gamedirs needed — Windows doesn't write `--pidfile` to twistd
  so the single-folder model works directly. (Unix needs symlinked
  view-gamedirs per the library's
  [`deployment-topology.md`](https://github.com/FullCircleMUD/evennia-shards/blob/main/DESIGN/deployment-topology.md#unix-linux-macos-wsl-view-gamedirs).)
- World-builder library coexists with shards: `wb_build` on a shard
  stamps new rooms with that shard's `shard_id` via the library's
  `pre_save` chokepoint, no special wiring.
- Mob-spawner library coexists with shards: `ms_load` ingests rules
  across shards; spawned mobs get auto-stamped to their spawning
  shard.
- Library's `cross_shard_dig` + `cross_shard_move` admin commands
  bootstrap a shard1 with no rooms (dig from shard0 stamping the row
  shard1, then walk over via the move primitive).
- `AccountBank` `shard_id="*"` stamping verified — accounts loaded
  across shards without tripping the chokepoint.
- `_restart_purgatory_timers` scoped per shard — each shard's
  startup reboots only its own characters' release timers.

The PoC validated that the architecture composes correctly with FCM's
existing libraries and runtime. What it didn't validate (because the
content isn't there yet) is the per-shard SINK/RESERVE/channel
mechanisms — those will be lit up when adding a second production
shard becomes a real requirement, not a spike.
