# SPAWN_BOSS_MOBS.md — Boss & Unique Mob Spawn System

> ⚠️ **Refactor pending.** This document describes the current per-boss hardcoded approach. The boss/unique mob system is slated for refactor — the design will likely move toward a more general conditional spawn system (see "Future: Conditional Spawn System" below). Treat the details below as a snapshot of current behaviour rather than the long-term design. Some sections (notably the RatKing dungeon collapse description) are also out of step with the current `persistent_until_empty` model used by `DungeonInstanceScript`; those will be cleaned up as part of the refactor.

> Design document for how boss mobs and unique named mobs are placed and respawn in the game world. For commodity mob spawning (wolves, kobolds, gnolls), see **SPAWN_COMMODITY_MOBS.md**.

---

## Two Spawn Systems — By Design

FCM uses two deliberately separate mob spawn systems:

| System | Handles | Mechanism |
|---|---|---|
| **ZoneSpawnScript** | Common, repeatable mobs | Population maintenance — target count, 15s tick, mobs deleted on death |
| **Boss Mob System** (this document) | Unique, named encounters | Persistent objects — placed once, delay-based respawn after death |

Mixing these would be a mistake. Wolves need to respawn reliably at scale. A boss dragon needs to feel singular, weighty, and rare. Different design goals demand different tools.

---

## What Makes a Boss Mob Different

A boss mob has `is_unique = True` (AttributeProperty on `CombatMob`). This single flag changes the entire lifecycle:

| Behaviour | Common Mob (`is_unique=False`) | Boss Mob (`is_unique=True`) |
|---|---|---|
| On death | Deleted from DB | Stays in DB (hidden, dead) |
| Respawn mechanism | ZoneSpawnScript creates fresh object | Legacy `_respawn()` via `delay()` |
| Respawn timer | Per-rule `respawn_seconds` in JSON | Per-mob `respawn_delay` AttributeProperty |
| Room placement | Spread across area_tag rooms | Fixed spawn room (hardcoded in `_respawn()`) |
| State persistence | None — fresh copy | State persists across death (override `_respawn()` to reset) |
| World builder placement | Via JSON spawn rule | Placed once in world build script |

---

## Implemented Boss Mobs

### KoboldChieftain

```
Location: Kobold Warren (millholm_mine district)
Stats:     L3, 28HP, 1d6+1 (STR 12), AC 13
Respawn:   10 minutes (600s)
```

**Behaviour:**
- Fixed position — overrides `ai_wander()` to scan for players but never leaves the room
- 20% dodge on each combat tick via `at_combat_tick()` (calls `execute_cmd("dodge")`)
- Rally cry (room-wide message) on first drop below 50% HP — one-time, tracked by `db.has_rallied`
- Retreats at 30% HP (cowers in place — no escape, just stops being aggressive)
- `_respawn()` resets `db.has_rallied = False` before calling `super()._respawn()`

**Design intent:** Solo challenge for level 3 players, or a pair of level 2s who've cleared the surrounding kobolds first.

---

### GnollWarlord

```
Location: Gnoll Camp (millholm_southern district)
Stats:     L6, 75HP, 2d6+3 (STR 16), AC 16
Respawn:   10 minutes (600s)
```

**Behaviour:**
- Fixed position — never wanders
- 20% dodge on each combat tick
- Inherits Rampage from `Gnoll` — `at_kill()` fires an instant free `execute_attack()` on the next living player. Party wipe risk when the warlord kills someone mid-fight.
- `aggro_hp_threshold = 0.0` — never retreats, fights to the death
- Override `ai_retreating()` immediately switches back to wander state (can't be forced into retreat)

**Design intent:** The hardest mob in Millholm. Requires a coordinated party of level 5-6. The Rampage ability punishes sloppy play — if the warlord kills anyone, the party's HP totals suddenly matter.

---

### RatKing (Dungeon Boss)

```
Location: Rat Cellar dungeon instance (Harvest Moon Inn)
Stats:     L2, 15HP, 1d4, AC 11
Respawn:   None (respawn_delay=0, dungeon mob)
```

**Behaviour:**
- Does not wander (dungeon mob — single room)
- Longer initial attack delay than regular cellar rats (lets smaller rats engage first)
- On death: fires `boss_killed` quest event for all characters in the dungeon instance
- Triggers `instance.on_boss_defeated()` — starts the post-boss collapse timer (60 second linger before dungeon tears down)

**Design intent:** Tutorial combat boss. The quest dungeon manages its entire lifecycle — the RatKing is created when the instance spawns and is never respawned. Each new dungeon instance gets a fresh RatKing.

---

## Respawn Mechanism

When a boss mob dies, `CombatMob.die()` checks `is_unique`:

```python
def die(self, cause, killer=None):
    # ... death sequence (corpse creation, XP, etc.)
    if self.is_unique:
        # Don't delete — schedule respawn
        delay(self.respawn_delay, self._respawn)
    else:
        self.delete()  # ZoneSpawnScript handles repopulation
```

`_respawn()` on `CombatMob`:
1. Restores HP to max
2. Re-enables AI (resets state to "wander")
3. Moves object back to its spawn room (if dislodged)
4. Clears any lingering combat state

Subclasses override `_respawn()` to reset boss-specific state (e.g. `KoboldChieftain._respawn()` resets `db.has_rallied = False`).

**The boss object persists through death.** While dead (between `die()` and `_respawn()`), the mob exists in the DB but is inactive — no AI ticking, not visible in rooms.

---

## Fixed Position Design

Boss mobs override `ai_wander()` to never leave their room:

```python
def ai_wander(self):
    """Scan for players but never leave the room."""
    if not self.location:
        return
    if self.scripts.get("combat_handler"):
        return
    players = self.ai.get_targets_in_room()
    if players:
        self._schedule_attack(random.choice(players))
```

**Why fixed position?** A wandering boss creates a terrible player experience:
- Players prepare for the encounter, the boss isn't there
- Boss wanders into low-level zones and wipes new players
- The boss's lair loses its identity — the encounter has no sense of place

Fixed bosses feel like the game respects the player's time. If you're level 6 and you go to the Gnoll Camp, the Warlord will be there.

---

## World Builder Placement

Boss mobs are placed explicitly in world build scripts, not via JSON spawn rules:

```python
# In spawn_millholm_mobs.py
def spawn_gnoll_warlord():
    warlord = create_object(GnollWarlord, key="gnoll warlord", location=gnoll_camp_room)
    warlord.tags.add("gnoll_warlord", category="spawn_zone")  # for tracking
```

**Idempotency:** Rebuild scripts check for existing unique mobs by `spawn_zone` tag before creating. A soft rebuild won't duplicate the Warlord if it already exists in the DB.

---

## Future: Conditional Spawn System

The current implementation is per-mob coded behaviour. A future generalised boss spawn system would add:

**Spawn conditions (planned, not yet built):**
- **Time gating** — certain bosses only appear at night, during winter, during a storm
- **Spawn chance** — instead of guaranteed respawn, a percentage chance per respawn window. If it doesn't trigger, check again next window.
- **Population prerequisite** — boss only spawns when enough regular mobs are present in its zone (the pack needs to be intact for the chieftain to appear)
- **One-at-a-time enforcement** — explicit "is this boss alive anywhere?" check before spawning. Important when boss items are at stake.
- **Global events** — world state flags can enable/disable boss spawns (seasonal, quest-driven)

**Why not built yet:** The current two bosses work fine with simple hardcoded behaviour. The overhead of a general conditional system isn't justified until there are enough bosses to benefit from it.

---

## System Interactions

| System | How It Relates |
|---|---|
| **ZoneSpawnScript** | Sibling system for commodity mobs — explicitly does not handle unique mobs |
| **AI Handler** | Bosses use the same state machine (wander/retreating) but override states for fixed-position behaviour |
| **CombatHandler** | Same per-combatant Script architecture as all mobs |
| **DungeonInstanceScript** | Dungeon bosses (RatKing) are tied to instance lifecycle — created with instance, no respawn |
| **Quest System** | Boss deaths fire quest events (`boss_killed` on RatKing) — quest system checks this via `FCMQuestHandler` |
| **SPAWN_COMMODITY_MOBS.md** | Companion document for the other spawn system |
| **Unified Spawn System** | Boss drops use pre-placement via `spawn_nfts` / `spawn_scrolls` tags — see **UNIFIED_ITEM_SPAWN_SYSTEM.md** |

---

## Design Notes

**Corpse on death:** All mobs (unique or not) drop a `Corpse` object on death. The corpse contains the mob's loot, despawns after `corpse_despawn_delay` (default 300 seconds). The boss mob *object* persists but is inactive; the corpse is separate.

**`respawn_delay` is a floor, not a guarantee.** The `delay()` fires after the configured seconds, but Evennia processes it on the next reactor tick. Heavy load may add a small delay. For exact timing, don't rely on boss respawn — it's a "roughly 10 minutes" guarantee, not a precise schedule.

**Dungeon bosses are not world bosses.** RatKing and future dungeon bosses have `respawn_delay = 0` because their lifecycle is owned by `DungeonInstanceScript`. The dungeon creates them fresh on instance spawn. No global respawn timer applies.
