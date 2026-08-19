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
- Per-row shard partition enforced at the SQL layer via django-multitenant —
  every `ObjectDB` query carries `WHERE shard_id IN (current, '*')`, inserts
  are auto-stamped, and the router runs unscoped.
- `cross_shard_move` primitive — atomic DB writes, recursive
  inventory move, idmapper eviction, per-session ticket redirect.
- Ticket-based WebSocket auth + connection-level cross-shard redirect.
- Chargen wrapper that stamps new characters with the start-room's
  `shard_id`.
- Postgres-polled cross-shard message bus (`obj_msg`, `account_msg`,
  `send_cross_shard_message`).
- Admin commands: `@shard_check`, `cross_shard_dig`. Cross-shard movement
  rides the overridden `@tel` — the library replaces Evennia's teleport so
  a cross-shard destination is handled transparently.

**Shipped (in FCM, on top of the library):**

- Settings cascade (`settings_router.py`, `settings_shard0.py`,
  `settings_shard1.py`, `settings_common_shard_config.py`).
- `AccountBank` stamped `shard_id="*"` (global asset, loadable from any
  shard) — `at_post_login` and `ensure_bank` both set this.
- Startup restart helpers scoped per shard (router skips entirely,
  each shard scopes its character / mob query to its own partition):
  `_restart_purgatory_timers` and `_restart_mob_tickers`.
- `at_pre_puppet` / `at_post_puppet` retrofit `home` and
  `respawn_location` using pk-only lookups (`search_tag(...).values_list("pk")`
  plus direct writes to `db_home_id` / the `respawn_location`
  Attribute). The healing fallback chain (inn → Limbo for home,
  cemetery → respawn) is unchanged; the SQL projection avoids
  instantiating any row, so a global tag-join never depends on
  materialising a foreign-shard row the auto-filter would hide.
  `at_post_puppet`
  additionally scopes its searches to local shard (plus `"*"`) so
  the written pk is always a row this shard can later dereference.
  Defensive try/except wrappers around the eventual reads
  (`self.home` in the location fallback chain, `self.respawn_location`
  existence check) so a stored cross-shard reference from a previous
  shard's puppet pass doesn't raise out of the puppet flow — the
  refused load is treated as "not set" and the heal re-points to a
  local row.
- Per-shard `DEFAULT_HOME` in each shard's settings file
  (`settings_shard0.py`, `settings_shard1.py`). Evennia's
  `create_object` falls back to `settings.DEFAULT_HOME` when no
  `home=` kwarg is passed; resolving that dbref instantiates the
  target row, which on a sharded process must be local-shard (or
  global `"*"`) — the auto-filter hides anything else, so the lookup
  finds nothing and object creation fails. Each shard
  points `DEFAULT_HOME` at a room it owns: `#2` on shard0 (the
  migrate-created Limbo), and a `cross_shard_dig`-bootstrapped
  `Shard1-Limbo` on shard1. `settings_router.py` doesn't override —
  the router doesn't run `create_object` calls that need a default.
- Per-shard scaffold rooms in `fcm-world` (`shard0/scaffold/`,
  `shard1/scaffold/`) — every shard has its own Purgatory and
  `nft_recycle_bin`.
- Router-side `AUTO_PUPPET_ON_LOGIN = True` — the library's
  `shard_aware_at_post_login` redirect-to-last-shard branch activates
  on the flag, so a returning player whose `_last_puppet` is set with
  a usable shard_id auto-redirects to that shard instead of landing
  on the OOC character-select menu.

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
  both FKs reference local-shard rows after any cross-shard move
  (otherwise a later FK read from recall, rent, death/respawn, or any
  other non-IC code path that dereferences these fields resolves to
  nothing — the auto-filter hides the foreign-shard row, so the
  reference reads as missing rather than as an error).

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
[`interzone-travel.md`](interzone-travel.md)). These rooms are the only
places a character can cross between zones, and they are designed to
feel like a beat — a trailhead, a dock, a pass — where a brief
reconnect is unsurprising.

Each shard owns a set of zones. A character is **resident on exactly
one shard at a time** — the shard whose zones contain her current
room. There is no point in the design where the same character is
live on two shards. The library enforces this at the row level via
the `shard_id` column and django-multitenant's auto-filter.

### What lives where

| State | Lives on | Notes |
|---|---|---|
| `AccountDB` (login, characters list) | Router | Library: `SHARDS_ROLE=router` process owns account auth and the OOC menu |
| `AccountBank` (per-account asset row) | Global (`shard_id="*"`) | Any shard can load. Stamped `"*"` by FCM's `Account.at_post_login`; the router creates it unscoped, so this is a first stamp, not a re-stamp |
| `ObjectDB` for resident objects (characters, items, mobs, rooms in owned zones) | The owning shard | Library auto-stamps on insert; the auto-filter hides cross-shard rows from every query |
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

**The library enforces this by construction.** Tenancy is applied at the
SQL layer via [django-multitenant](https://github.com/citusdata/django-multitenant):
every query through `ObjectDB.objects` carries
`WHERE shard_id IN (<this shard>, '*')` automatically, and inserts are
auto-stamped with the current shard. A consumer (FCM, in our case) cannot
accidentally cache a foreign shard's row, because the row never comes back
across the wire — the boundary is in the database, not in Python, so
`from_db` is never called on it and the idmapper never sees it.

**The failure mode is silence, not an error.** A query that would have
matched a foreign-shard row simply returns fewer rows. Nothing raises.
That is safe for the cache invariant but unforgiving of a mistaken
assumption: code that expects a global view and doesn't have one gets a
plausible-looking subset rather than a loud failure. Anything that
genuinely needs to see across shards must say so explicitly — the router
runs unscoped, and `shard_context(None)` opens a scoped escape hatch for
handoff, chargen and admin tooling.

The `cross_shard_move` primitive in the library handles
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
`shard_id="*"` (the global sentinel), which the multitenant auto-filter
admits from every shard's scope — so a character moving between shards
keeps its bank visible throughout.

No bypass primitive is involved. The bank is created on the router,
which runs unscoped, so the insert auto-stamp is skipped and the row
lands `shard_id=NULL`; assigning `"*"` and saving is then a legitimate
first stamp, since multitenant's immutability check only refuses
*re*-stamping an already-tagged row. The stamp is skipped entirely in
monolith, where the column does not exist.

### Global scripts — mechanism and classification settled

`ScriptDB` is not tenant-scoped, so a persistent global script is one
row visible to every process, and each process attaches its own
`LoopingCall` to it. A global script therefore ticks **once per
process**, not once per cluster. The mechanism, and the test for
whether a given script is safe under it, is the single source of truth
in [evennia-shards: global scripts run one instance per
process](../libraries/evennia-shards/docs/shard-settings.md#global-scripts-run-one-instance-per-process).

**How FCM declares it.** Each global script declares the deployment
roles it runs on — `shard`, `router`, `monolith` — plus optional tags,
in `server/conf/at_server_startstop.py`. At boot a process starts only
the scripts naming its own role. `monolith` is a first-class role
rather than shorthand for "shard plus router", so a script needed only
in a sharded deployment can say exactly that.

> **That file is authoritative for which scripts run where.** Read it
> rather than trusting a list here. Every script is classified; there is
> no unclassified backlog.

**The classification each script needs.** A script is only safe to run
once per process if it holds no persistent state (counters on `ndb`,
never `db`) and acts solely on process-local data — the canonical shape
being walking `SESSION_HANDLER` and touching only the puppets connected
to this process. Before declaring roles for a script, answer:

1. Does it hold persistent script state, or only `ndb`?
2. Does it query the world? On a shard it silently sees that shard's
   rows only; on the router — which runs **unscoped** — it sees every
   shard's rows at once.
3. Must its side effect happen exactly once cluster-wide? Item
   spawning, telemetry aggregation and gold reallocation are the
   candidates: N spawn runs or N reallocations per interval would be a
   real economic fault.

The failure mode is quiet. A shard-scoped query returns a subset rather
than erroring, so a script starting with no errors in the log is *not*
evidence that it is correct.

**Exactly-once comes from the role table, not from a gate.** The three
candidates — item spawning, telemetry aggregation and gold reallocation
— all run on `router` only, and the library mandates a single router. No
election, no nominated shard, no consumer-side lock. Where a
cluster-wide side effect must happen once, the answer is to name one
role rather than to coordinate between several.

**The role table alone does not keep a script off other processes.** It
gates what each process *creates*. Evennia's
`update_scripts_after_server_start()` walk then iterates every active
`ScriptDB` row, knows nothing about roles, and attaches a `LoopingCall`
to any row still carrying a pause marker — so **the first process to
boot picks up every marked script in the cluster**, not just its own.
Processes booting later find the markers consumed and quietly get only
what they claim for themselves.

Observed live before the fix: with the router starting first, it ran its
own five *and* the six `GAME_ROLES` world scripts, unscoped, on every
boot. Booting a shard first would instead have handed it the router-only
scripts, `reallocation_service` among them — the one whose own comment
notes that a second process running it concurrently mints currency that
was never spent.

**How it is closed.** Both halves declare where they belong, as data,
and the shards library keeps them there —
[script-confinement.md](../libraries/evennia-shards/docs/script-confinement.md):

- **Global scripts** — `start_scripts()` stamps `owning_roles` from the
  roles column as it creates each one. Same table, now enforced past
  creation.
- **Per-shard scripts** — one row per rule-set file, a set that grows
  with content, so no static table can name them. `evennia-mob-spawner`'s
  Deployer stamps `owning_shard` at `ms_load` time instead. Without it
  the router's ticks produced mobs stamped `shard_id=NULL`.

The two are mutually exclusive per script, and a script declaring
neither is left unconfined — which is correct for anything genuinely
belonging everywhere, such as the cosigner keepalive.

**Not every cross-process concern is a script.** The cross-shard message
bus is a Twisted `LoopingCall` started from `at_server_start()`, not a
`ScriptDB` row — deliberately, so it cannot sit wedged in "stopped"
while the game looks healthy. Work that is a direct response to an
inbound message therefore needs no script at all: item placement on a
shard is driven by the bus handler, which is why only the router runs a
spawn script.

### Game time — solved, derived from wall clock

Evennia's `gametime()` has two modes, chosen by `TIME_IGNORE_DOWNTIMES`:

```python
if IGNORE_DOWNTIMES:
    gtime = epoch + (time.time() - server_epoch()) * TIMEFACTOR   # wall clock
else:
    gtime = epoch + (runtime() - GAME_TIME_OFFSET) * TIMEFACTOR   # accumulated uptime
```

**The uptime branch is unsafe with more than one process.** `runtime()`
reads a module global that each Server process keeps in its own memory,
loads once from `ServerConfig["runtime"]` at its first maintenance tick,
and writes back every 60 seconds. Evennia assumes a single Server. Run
four and they overwrite one another every minute, last write wins, and
none of them re-reads afterwards so none notices. The stored value
flaps, and a reloading process can read a total *lower* than the one it
held — moving game time backwards.

Backwards time is the failure that matters. An unexpected nightfall
reads as another morning; a return to summer reads as impossible.
Cross-shard travel is narrated as taking days, which absorbs a phase
mismatch on arrival, but nothing absorbs a season going the wrong way.

**The wall-clock branch bypasses the accumulator entirely.** Every term
is a settings constant, a database constant written once at creation
(`server_epoch`, from `initial_setup.py`), or the OS clock. All
processes therefore agree with no coordination, no writer, no round
trip, no cold-start case, and nothing to persist per tick.

FCM uses wall clock. Three consumers — `season_service`,
`day_night_service` and `durability_decay_service` — are correct and
mutually consistent as a result, untouched: their derivation chains were
always sound, only their input was broken.

**What it costs.** Game time advances while the server is down. Players
cannot observe this — game time passes while they are logged off either
way, and a restart disconnects them, so they are inside the gap
regardless. The only systems that would notice are any holding a
game-time deadline, which a forward jump would expire in bulk.

**What reversing it would take.** Turning the setting off is not a
one-line change under sharding; it reintroduces the multi-writer race.
The accumulator would first have to be made single-writer:

- Router publishes an anchor, `ServerConfig["shard_clock"] =
  (runtime(), time.time())`, on its own timer. The timestamp is required
  because `ServerConfig` has no modified column, so a reader cannot
  otherwise tell how stale a value is.
- Shards are barred from writing `runtime`, via a guard on
  `ServerConfigManager.conf` — the narrowest available seam, since the
  write itself is one line inside `server_maintenance`, which also
  flushes the idmapper and recycles connections.
- Shards override `runtime()` to extrapolate from the anchor, which
  makes read latency irrelevant: a value read 59 seconds late yields the
  same answer. A staleness cap makes them stop extrapolating once the
  router stops writing — the correct definition of downtime, since
  nobody can log in without the router.

That is roughly a module, two Evennia patches, a tuning constant and an
entry in the library's integration-risks register to re-diff on every
Evennia upgrade. Weigh it against what downtime actually costs before
building it.

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
to route. The sender's shard looks up the target's `shard_id`, then uses
`send_cross_shard_message` to deliver the message to the target
character on the target shard. The same pattern handles `who`,
`where`, scrying, and any other "find another player" command.

**The lookup must escape the auto-filter.** By definition the target is
on another shard, so a default-scoped query cannot see the row —
including a `.values_list` projection, which is filtered at the SQL
`WHERE` level like any other query. The lookup runs inside
`shard_context(None)`, the same narrow escape the library uses for
`cross_shard_move`'s destination validation and for
`shard_aware_global_search`. Keep the escape to the single projection
query: resolve the `shard_id`, then leave the block before doing
anything else.

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
[`cross_shard_move`](https://github.com/FullCircleMUD/evennia-shards/blob/main/src/evennia_shards/handoff.py)
primitive — atomic DB writes, recursive inventory move, idmapper
eviction, per-session ticket-authenticated WebSocket redirect. The
[`ticket-auth-flow.md`](https://github.com/FullCircleMUD/evennia-shards/blob/main/DESIGN/ticket-auth-flow.md)
in the library documents the wire protocol.

**The primitive is the interface.** The library deliberately ships no
cross-shard movement *command* of its own — a consumer drives
`cross_shard_move()` from wherever its game says movement happens: an
exit, a portal, a ship, a teleport pad, a login-time placement rule. It
returns what actually happened (`objects_moved`, `sessions_redirected`,
`failures`), so a caller can report or recover.

In-game today, cross-shard movement rides the overridden **`@tel`**: the
library replaces Evennia's teleport command so that a destination on
another shard is handled transparently, and a same-shard destination
falls through to vanilla behaviour untouched.

> **Future work.** The paragraph below describes the finished state. No
> `contrib/` exists in `evennia-shards` today. FCM does not need it at
> `shard_count = 1` — `@tel` covers every case that arises before a second
> live shard. Remove this note once the module lands.

**Walkable exits between shards ship as a library `contrib` module** — a
`CrossShardExit` typeclass, opt-in rather than core. Traversal is a game
concept, so the library must not own it; but doors between shards are the
anticipated common case, so an implementation is provided to use, adapt or
read. See [library-standards.md](library-standards.md#contrib--conditional)
for why that split exists.

Whatever drives the primitive, the consumer-side gating is FCM's to
write, per the library's
[`consumer-constraints.md`](https://github.com/FullCircleMUD/evennia-shards/blob/main/docs/consumer-constraints.md):
check safe state (not in combat, not casting, no in-flight delayed
callbacks), and update `db_home` and `respawn_location` to a room on the
destination shard so both FKs stay dereferenceable there — otherwise a
later recall / rent / respawn read reaches for a row this shard cannot
see.

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

