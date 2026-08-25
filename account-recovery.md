# account-recovery.md

> **THIS FILE covers how players survive a world rebuild** — what is archived and when, how a wallet
> sign-in restores an account and its characters, and how items and balances come back from the XRPL
> mirror rather than the archive. For the archive library itself see
> [evennia-archive's docs](../libraries/evennia-archive/docs/INDEX.md). For the database layout and
> the `archive` alias see [database.md](database.md). For subscription state across a rebuild see
> [subscriptions.md](subscriptions.md).

---

## Purpose

FCM rebuilds its world from YAML whenever content changes or data goes stale. Evennia identity is the
`ObjectDB` primary key, so a rebuild re-issues every key and a player's account and characters are
gone with it.

Recovery makes those two things compatible: the world can be rebuilt at will, and a returning player
signs in with the same wallet and finds their account, characters, progression, items and balances
intact. That is what makes invited playtesting viable — a rebuild costs a tester nothing.

---

## The two sources

Nothing about a player is reconstructed from one place. Two databases survive a rebuild and each owns
a different half.

| | Holds | Written |
|---|---|---|
| **archive** (`archive.db3`) | Account and character rows — levels, skills, progression, attributes, tags, nicks | At seams: creation, session end, a level, training, remort |
| **xrpl** (`xrpl.db3`) | Ownership — which wallet and character holds which items, gold and resources | Transactionally, as the player plays |

The split matters because the mirror is **always the more recent of the two**. The archive is a
snapshot taken at intervals; the mirror is written as things happen. So on recovery the archive
supplies who the character *is*, and the mirror supplies what they *have*. Balances taken from the
archive would be as of the last seam; taken from the mirror they are as of the moment the world went
down.

**The mirror is read and never written during recovery.** An item being rebuilt has not been crafted,
deposited or spawned — it is ownership that already exists, regaining a game object. Writing would
re-book an arrival that never happened.

---

## Identity

Two attributes carry identity across the rebuild. Both are stored unpickled (`strattr`), so archive
lookups are plain string equality rather than a pickled blob comparison.

| Attribute | On | Purpose |
|---|---|---|
| `archive_id` | Account, Character | Minted once at creation by `ArchivableMixin`; matches a live row to its archived copy |
| `account_wallet` | Character | Copy of the owning account's wallet — the only thing tying a restored character to an account |

`account_wallet` exists because `restore()` drops every foreign key. A restored character has no
`db_account`, and the account's `_playable_characters` is a list of dbrefs into the database that was
rebuilt. The stamped wallet is what crosses the gap.

`Account.at_account_creation` overrides Evennia's hook without calling `super()`, so it mints its
identity explicitly with `at_archive_init()`. Removing that line breaks nothing at runtime and leaves
every account unarchivable.

---

## When things are archived

The archive is written at seams rather than continuously. Each was chosen because it captures
something that cannot be recovered from anywhere else.

**Account** — three points, all going through `Account.archive_now()`:

| Seam | Why |
|---|---|
| Tail of `Account.create()` | Not `at_account_creation`: `wallet_address` is assigned *after* `create_account()` returns, so the hook would archive an account the wallet search can never find |
| `at_disconnect` | The freshness seam. Covers quit, dropped client, link-dead sweep, and going IC — all end the router session |
| After a subscription payment | The one change worth its own call; waiting for logout could cost a paying player their subscription |

`archive_now()` **does nothing on a shard.** Account state only changes while a player is OOC, which
means on the router; a shard would write a copy of what the router already holds, once per IC/OOC
handoff.

**Character** — seven points, all through `FCMCharacter.archive_now()`: chargen, `at_post_unpuppet`,
an XP level, a guild level, general/class/weapon training, and remort. That helper refuses on the
**router** rather than on a shard — the inverse of the account's guard, because characters are only
played on shards. Chargen is the exception and passes `allow_router=True`, since `create_character`
is an account-side flow.

**Not archived:** reload and shutdown. Neither disconnects sessions, so `at_disconnect` does not fire
there. The gap is an account changed while OOC on the router with no session end before a restart —
narrow, and everything in it either self-heals or is recoverable. Adding it would mean overriding
`at_server_reload` and `at_server_shutdown`, and running those archives *synchronously*, since a
deferred write dispatched during teardown may never run.

---

## What is not archived, and why

| | Recovered from |
|---|---|
| Items (NFTs) | The XRPL mirror — `NFTItemType` carries the typeclass and prototype, `metadata` carries per-item state |
| Gold and resources | The XRPL mirror |
| The account bank | Recreated empty by `at_post_login`, then refilled from the mirror |
| Rooms, mobs, world content | Rebuilt from YAML |

Archiving items would duplicate a record that already survives, and would risk the two disagreeing.

**Accepted losses**, all inside the same narrow window — logged in, state changed, hard restart before
any session end:

- `quest_completion_counts` — at most one quest's worth of a starter-quest cap
- ToS acceptance — re-prompted at next login by the existing check in `at_post_login`
- Staff permissions — granted through Evennia's `perm`, which FCM does not own; re-granted by hand

---

## The sign-in flow

Order is load-bearing. Creating an account before checking the archive takes the username and mints a
fresh `archive_id`, leaving the archived account unrestorable.

```
wallet verified
   ↓
live account for this wallet?  ── yes ─→ log in
   ↓ no
archived account for this wallet?  ── error ─→ REFUSE, ask them to retry
   ↓ no                                        (never create — see below)
   ↓                    ── yes ─→ restore account
create a new account                ↓
                                 rebuild the bank    (items + balances from the mirror)
                                    ↓
                                 restore characters  (archive), then their items and balances (mirror)
                                    ↓
                                 log in
```

**A failed archive lookup refuses rather than falling through.** A momentary fault would otherwise
create an account that permanently orphans the player's real one. A retry costs them nothing by
comparison.

**A failed bank or character recovery does not.** By that point the account is live and usable, the
mirror is untouched, and blocking sign-in would cost them more than the wait. They are told their
ownership records are intact and the sequence continues.

Where a restored character *goes* needed no new code: `at_pre_puppet` already repairs a character
whose world was rebuilt underneath it, falling back home → Harvest Moon Inn → Limbo, and location →
dungeon entrance → last rent → home.

---

## Names are reserved by the archive

After a rebuild the live database holds no accounts or characters, so every name reads as free until
its owner signs in. Both creation paths therefore check the archive as well as the live database:

- Character names — chargen's uniqueness check
- Account usernames — the wallet sign-in creation path

Refusing a newcomer costs them a retry. Letting the name go would cost the returning player their
name, and renaming a character on restore would detach it from its gold and items, which are keyed on
`character_key` — the character's **name**, not a dbref.

`restore()` still renames a colliding *account* username (`rowan` → `rowan1`, recorded under
`archive_renamed_from`). These checks make that unreachable in the wallet flow; it remains as the
library's backstop for accounts created by paths that never ran the check.

**`chardelete` deletes the archived copy.** Delete means gone: the name is released, the character
cap holds, and the character does not return at the next rebuild. Its existing guards — refusing while
the character holds NFTs, gold or resources — are what make the name safe to reissue, because a
deletable character has no ownership rows carrying its name. A failed archive delete writes a
`ReconciliationFailure` row, since the copy surviving needs a person.

**`@name` refuses characters and accounts** for the same reason. Renaming a character would silently
orphan every ownership row keyed to the old name.

---

## Shard stamping

Recovery runs on the router, unscoped, inside a worker thread with no tenant context — so an insert
lands `shard_id=NULL` and the shards guard refuses it. Items are therefore created inside an explicit
shard context:

- Banked items → `"*"`, matching the bank, reachable from every shard
- A character's items → that character's own shard

The same rule now applies in ordinary play. `NFTMirrorMixin` moves an item's stamp when it changes
hands: banking makes it `"*"`, taking it out gives it the holder's shard. Without that an item kept
its creation shard for life — invisible with one populated shard, and a silent disappearance with two
(Fred banks on shard0, Lancelot withdraws on shard1, and the row still says shard0 while shard1
filters `shard_id IN ('shard1', '*')`).

The stamp is changed with `qs.update` rather than `save()`, because the shards library flags an
assignment to the tenant column and the next `save()` raises. That is the library's own technique —
see `evennia_shards.handoff.cross_shard_move`. Instances are then evicted from the idmapper and the
contents caches at **both** ends rebuilt: `contents_cache` holds instances rather than keys, so
flushing alone would leave the destination serving an evicted object while a query by pk built a
second one for the same row.

---

## Known gaps

- **Room references are not translated.** `respawn_location`, `prelogout_location` and similar hold
  pickled dbrefs that are not remapped on restore. They resolve correctly only because a rebuild from
  the same YAML re-issues the same keys in the same order. Change the world and they point somewhere
  else, silently. Giving rooms a stable identity from the YAML is the fix.
- **A worn item comes back unworn.** Wearslot maps hold dbrefs to objects that no longer exist.
- **The archive cannot hold two accounts with the same username.** It inherits Evennia's unique
  constraint, so a second account with a name already in the archive fails to archive — logged, not
  fatal. `restore()` renames on the way out but `archive()` does not on the way in.
- **Every rebuild that recreates `root` leaves an orphaned pair behind**, since the new superuser is a
  new object with a new identity. Harmless, accumulates.
- **`_restamp_pks` has no unit test.** It needs the `shard_id` column, which the monolith test
  settings lack, and the suite does not boot under `settings_shard0`.

---

## Where the code lives

| File | What |
|---|---|
| `typeclasses/accounts/accounts.py` | `archive_now()`, `at_disconnect`, the mint in `at_account_creation`, the archive call in `create()` |
| `typeclasses/actors/character.py` | `archive_now()`, `account_wallet`, the seams, the archive delete in `at_object_delete` |
| `commands/unloggedin_cmds/cmd_override_unconnected_connect.py` | The sign-in flow, account/character/bank recovery |
| `typeclasses/mixins/nft_mirror.py` | `spawn_into` (rebuild from a mirror row), the `recovering` flag, shard stamp following |
| `server/main_menu/chargen/chargen_menu.py` | Wallet stamp, chargen archive, archive name check |
| `commands/all_char_cmds/cmd_override_name.py` | `@name` refusing characters and accounts |
