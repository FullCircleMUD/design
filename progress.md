# Progress

A running log of high-level milestones as FullCircleMUD moves from design into build. Each entry is a
brief note pointing to whatever artefact (test result, design doc, live run) is the evidence for that
milestone. New entries go at the top.

This is **not a changelog** — `git log` already holds what changed, and duplicating it here just
creates a second source of truth that drifts. It is **not a roadmap** either; that lives in
`ops/DEVELOPMENT/0_LONG_TERM`, `1_BACKLOG` and `2_IN_PROGRESS`.

What belongs here is what neither of those can tell you: **that something was validated, under what
conditions, and what remains unproven.** Environmental caveats, hard-won findings, and "we ran it and
it worked" — the context a commit message has no room for.

It also keeps status out of the design documents. A design doc in [docs/](INDEX.md) describes how the
game *works*, in present tense, with a short removable note where the code hasn't caught up — see
[doco-structure.md](doco-structure.md#conventions-for-docs-documents). The same convention runs in
each library's own `docs/progress.md`.

## Milestones

### 2026-08-06 — Sharded FCM validated end-to-end on macOS

The real game ran sharded on macOS for the first time: router + `shard0` + `shard1`, three daemonized
processes against one SQLite database, from the view gamedirs.

**macOS needs a non-Apple SQLite.** Without it `evennia start` hangs at `Server starting  ...`
forever — no exception, no timeout, nothing in any log. Apple's `libsqlite3` runs
`sqlite3_initialize()` through libdispatch, which cannot survive `fork()`, and `twistd` daemonizes
with a double fork. Any SQLite call in the child then blocks permanently, and Evennia always has a
connection open before the fork. Cleared by swapping in `sqlean.py`, gated on `darwin`. Windows,
Linux, `--nodaemon` and Postgres are all unaffected — which is why it went unnoticed until the move
from a Windows box to a Mac. Reasoning in the library's
[`deployment-topology.md` § macOS](../libraries/evennia-shards/docs/deployment-topology.md).

**Validated in one live session:** login on the router → `ic` to `shard0` → `cross_shard_dig shard1`
→ `@tel` across (`failures=0`) → `ooc` back to the router. World content built onto both shards with
`wb_build` while gameplay continued, role-scoped services ticking on the shard, and the LLM NPC layer
answering in character from a built room.

**Not proven:** the mob spawner (expected working, unexercised), and the per-shard SINK / RESERVE /
channel mechanisms, which only light up with a second *live* shard.

### 2026-05-21 — Shards architecture composes with FCM, on Windows

FCM wired as an `evennia-shards` consumer and run as a three-process proof of concept on Windows,
where `twistd` writes no `--pidfile` and all roles start from one folder.

The point of the exercise was compatibility rather than features: that `evennia-world-builder` and
`evennia-mob-spawner` both keep working under sharding, that new rows land on the right shard, and
that a character can be moved between shards with the session following. All held.

Left unproven at the time: anything needing a second *live* shard in production, and any Unix
deployment — the macOS gap above went undiscovered for months as a result. See
[scaling.md](scaling.md) for the design.
