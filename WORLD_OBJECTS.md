# World Objects — Fixtures, Interactions, and Mechanisms

> For exits and doors, see [EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md). For room types, see [ROOM_ARCHITECTURE.md](ROOM_ARCHITECTURE.md). For items (NFTs), see [INVENTORY_EQUIPMENT.md](INVENTORY_EQUIPMENT.md).

## Overview

World objects are non-NFT, immovable objects placed into rooms by zone builders. They provide environmental interactions — things players look at, open, climb, pull, search, and interact with. They are NOT blockchain-tracked and cannot be picked up.

All world objects inherit from `WorldFixture` which provides `get:false` lock, hidden/invisible support, and height-aware positioning.

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
├── world_objects/
│   ├── base_fixture.py       ← WorldFixture (base class)
│   ├── sign.py               ← WorldSign (ASCII art signs)
│   ├── chest.py              ← WorldChest (container + gold/scroll spawn)
│   ├── climbable_fixture.py  ← ClimbableFixture
│   ├── switch_fixture.py     ← SwitchFixture
│   ├── lit_fixture.py        ← LitFixture (permanent light source)
│   └── jobs_board.py         ← JobsBoard
├── mixins/
│   ├── climbable_mixin.py    ← ClimbableMixin
│   ├── switch_mixin.py       ← SwitchMixin
│   ├── closeable.py          ← CloseableMixin
│   ├── lockable.py           ← LockableMixin
│   ├── smashable.py          ← SmashableMixin
│   ├── trap.py               ← TrapMixin
│   ├── container.py          ← ContainerMixin
│   ├── hidden_object.py      ← HiddenObjectMixin
│   ├── invisible_object.py   ← InvisibleObjectMixin
│   └── light_source.py       ← LightSourceMixin
commands/
├── all_char_cmds/
│   ├── cmd_climb.py          ← CmdClimb (climb up/down)
│   ├── cmd_switch.py         ← CmdSwitch (pull/push/turn/flip)
│   ├── cmd_search.py         ← search (find hidden + detect traps)
│   ├── cmd_open.py           ← open/close
│   ├── cmd_lock.py           ← lock/unlock
│   └── cmd_light.py          ← light/extinguish/refuel
```
