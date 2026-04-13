# World Objects — Fixtures, Interactions, and Mechanisms

> For exits and doors, see [EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md). For room types, see [ROOM_ARCHITECTURE.md](ROOM_ARCHITECTURE.md). For items (NFTs), see [INVENTORY_EQUIPMENT.md](INVENTORY_EQUIPMENT.md).

## Overview

World objects are non-NFT objects placed into rooms by zone builders. They provide environmental interactions — things players look at, open, climb, pull, search, read, and interact with. They are NOT blockchain-tracked.

There are **two base class lineages** for world objects, depending on whether the object is takeable:

- **`WorldFixture`** — immovable, `get:false` lock. The base for permanent environmental objects: signs, chests, climbables, switches, lit fixtures, library books. The bulk of this document is about `WorldFixture` and its mixins.
- **`WorldItem`** — takeable but non-NFT. The base for physical-but-non-blockchain items: keys, quest items, novelty props. `KeyItem` is the current concrete subclass.

Both lineages include `HeightAwareMixin` (vertical position + height-gated visibility) and `HiddenObjectMixin` (find-DC discovery).

---

## Base Classes

### WorldFixture (`typeclasses/world_objects/base_fixture.py`)

Foundation class for all permanent world objects.

**Provides:**
- `get:false` lock (immovable)
- `HeightAwareMixin` — `room_vertical_position`, `visible_min_height`/`visible_max_height` for height-gated visibility
- `HiddenObjectMixin` — `is_hidden`, `find_dc`, discoverable via `search`
- `InvisibleObjectMixin` — `is_invisible`, requires DETECT_INVIS to see
- `is_visible_to(character)` — combined visibility check

### WorldSign (`typeclasses/world_objects/sign.py`)

Read-only ASCII art sign. `sign_text`, `sign_style` (post/hanging/wall/stone). Display rendered via `return_appearance()`.

### WorldChest (`typeclasses/world_objects/chest.py`)

Closeable + lockable + smashable container with `FungibleInventoryMixin`. Starts closed. Contents gated on open state. Supports `loot_gold_max`, `spawn_scrolls_max`, `spawn_recipes_max` for unified spawn system integration.

### TrapChest (`typeclasses/world_objects/trap_chest.py`)

`WorldChest` + `TrapMixin`. Opening triggers the trap (whether or not it was detected via `search`). Smashing a trapped chest triggers the trap on all room occupants before breaking it open. Disarm via the `disarm` command (SUBTERFUGE skill check).

Builder sets `is_trapped`, `trap_damage_dice`, `trap_damage_type`, `trap_disarm_dc`, `trap_description` on the instance after creation.

### LitFixture (`typeclasses/world_objects/lit_fixture.py`)

Permanent light source — torch sconces, braziers, magical glow-crystals. The room's lighting system reads `is_lit` on any fixture present; a lit fixture satisfies the "has a light in the room" condition and suppresses the dark-room display. Unlike torches and lanterns (which are NFT items with consumable fuel), lit fixtures have no fuel management — they're permanent environmental lighting set by the builder.

### LibraryBook (`typeclasses/world_objects/library_book.py`)

Readable book that transports players to themed zones. `read <book>` shows flavour text and teleports the reader to the book's configured destination room. `recall` returns them to the library room they entered from. The entry point for the book-zone system (Hundred Acre Wood, etc.).

### KeyItem (`typeclasses/world_objects/key_item.py`)

Takeable consumable key. Subclass of `WorldItem`. Matched to a lock via `key_tag` — `LockableMixin.unlock()` consumes (deletes) the key on successful use. Use for any lock that should cost a physical key to open.

### XPButton (`typeclasses/world_objects/xp_button.py`) — dev tool only

Dev-only fixture that awards XP when pressed. Place it in a test room to level a character up fast so you can inspect how different classes and skills operate at different mastery tiers. **Not for live world placement** — do not ship any zone that exposes an XPButton to players.

---

## Interaction Mixins

Composable mixins that add specific interaction capabilities to any world object.

### ClimbableMixin (`typeclasses/mixins/climbable_mixin.py`)

Allows characters to change `room_vertical_position` without the FLY condition. Used for drainpipes, ladders, ropes, vines, trees.

| Attribute | Default | Purpose |
|-----------|---------|---------|
| `climbable_heights` | None | Set of supported heights, e.g. `{0, 1}` |
| `climb_dc` | 0 | DEX check DC. 0 = auto-succeed |
| `climb_up_msg` | "You climb upwards." | Success message going up |
| `climb_down_msg` | "You climb downwards." | Success message going down |
| `climb_fail_msg` | "You fail to get a grip..." | Failure message |

**Command:** `climb up/down <target>` (`commands/all_char_cmds/cmd_climb.py`)
**Concrete typeclass:** `ClimbableFixture(ClimbableMixin, WorldFixture)`
**Fall safety:** `BaseActor._check_fall()` checks for climbable fixtures before dealing fall damage — character slides down safely if a fixture supports their height.

**First instance:** Drainpipe in Back Alley, Millholm Rooftops entry point.

### SwitchMixin (`typeclasses/mixins/switch_mixin.py`)

Generic toggle mechanism — levers, buttons, valves, anything that turns on/off to trigger an effect. The mixin provides the mechanism; the effect is always custom via hooks.

| Attribute | Default | Purpose |
|-----------|---------|---------|
| `is_activated` | False | Current toggle state |
| `switch_verb` | "pull" | Action verb ("pull", "push", "turn", "flip") |
| `switch_name` | "switch" | Display name ("lever", "button", "valve") |
| `can_deactivate` | True | False = one-way switch, can't toggle back |
| `activate_msg` | "You {verb} the {name}." | First-person activation message |
| `deactivate_msg` | "You {verb} the {name} back." | First-person deactivation message |

**Methods:**
- `activate(caller)` → sets `is_activated = True`, calls `at_activate(caller)` hook
- `deactivate(caller)` → sets `is_activated = False`, calls `at_deactivate(caller)` hook
- `at_activate(caller)` — **override this** to define the effect (disarm trap, open door, etc.)
- `at_deactivate(caller)` — **override this** to define the reverse effect

**Command:** `pull/push/turn/flip <target>` (`commands/all_char_cmds/cmd_switch.py`)
**Concrete typeclass:** `SwitchFixture(SwitchMixin, WorldFixture)`

**Usage example (zone builder):**
```python
class TrapLever(SwitchMixin, WorldFixture):
    switch_verb = AttributeProperty("pull")
    switch_name = AttributeProperty("lever")

    def at_activate(self, caller):
        for obj in self.location.contents:
            if hasattr(obj, "trap_armed"):
                obj.trap_armed = False
                caller.msg("You hear a click as the trap mechanism disengages.")
```

### Height-Gated Visibility (`HeightAwareMixin`)

Any object with `HeightAwareMixin` can optionally set `visible_min_height` / `visible_max_height` to restrict which characters can see it based on their `room_vertical_position`. None = no restriction (default).

- `RoomBase.get_display_things()` filters by `is_height_visible_to(looker)`
- `RoomBase.get_display_characters()` filters by `is_height_visible_to(looker)`

**Example:** Sunken wreck at depth -2 with `visible_max_height = -1` — only visible when diving.

---

## Shared Mixin Summary

| Mixin | Provides | Used by |
|---|---|---|
| CloseableMixin | `is_open`, `open()`, `close()` | WorldChest, ExitDoor |
| LockableMixin | `is_locked`, `unlock()`, `picklock()` | WorldChest, ExitDoor |
| SmashableMixin | `smash_hp`, `take_smash_damage()` | WorldChest, ExitDoor |
| TrapMixin | `is_trapped`, `trigger_trap()`, `disarm_trap()` | TrapChest, TrapDoor, TripwireExit, PressurePlateRoom |
| SwitchMixin | `is_activated`, `activate()`, `deactivate()` | SwitchFixture (levers, buttons) |
| ClimbableMixin | `climbable_heights`, `climb_dc` | ClimbableFixture (drainpipes, ladders) |
| HiddenObjectMixin | `is_hidden`, `find_dc` | WorldFixture, ExitDoor |
| InvisibleObjectMixin | `is_invisible` | WorldFixture, ExitDoor |
| ContainerMixin | capacity, put/get | WorldChest, ContainerNFTItem |
| LightSourceMixin | `is_lit`, `fuel_remaining` | TorchNFTItem, LanternNFTItem |

---

## Trap System

See `typeclasses/mixins/trap.py` and subclasses. Traps can be placed on:
- **Doors** (TrapDoor) — triggers on open
- **Chests** (TrapChest) — triggers on open
- **Exits** (TripwireExit) — triggers on traverse if undetected
- **Rooms** (PressurePlateRoom) — triggers on attempted leave

**Builder helper:** `connect_trapped_door()` in `utils/exit_helpers.py` — creates a bidirectional door pair with a trap on one side.

**Detection:** Passive (`_check_traps_on_entry` in `at_post_move`) and active (`search` command).
**Disarming:** `disarm` command (SUBTERFUGE skill check) or SwitchMixin levers (custom wiring).

---

## Files

```
typeclasses/
├── world_objects/              ← concrete fixture and world item classes
│   ├── base_fixture.py           (WorldFixture — the untakeable base)
│   ├── base_world_item.py        (WorldItem — the takeable base)
│   └── ...                       (one file per fixture subclass described above)
├── mixins/                     ← composable interaction mixins
│   ├── climbable_mixin.py
│   ├── switch_mixin.py
│   ├── closeable.py
│   ├── lockable.py
│   ├── smashable.py
│   ├── trap.py
│   ├── container.py
│   ├── hidden_object.py
│   ├── invisible_object.py
│   └── light_source.py

commands/all_char_cmds/         ← player-facing interaction commands
├── cmd_climb.py                  (climb up/down)
├── cmd_switch.py                 (pull/push/turn/flip)
├── cmd_search.py                 (find hidden + detect traps)
├── cmd_open.py                   (open/close)
├── cmd_lock.py                   (lock/unlock)
└── cmd_light.py                  (light/extinguish/refuel)
```

The Base Classes section above is the source of truth for which concrete subclasses exist. Directory listings are intentionally sparse so new fixture types can be added without touching this doc.
