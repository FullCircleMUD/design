# Exit Architecture

> Technical design document for the exit system. For world building and zone design, see [WORLD.md](WORLD.md). For procedural dungeon design, see [PROCEDURAL_DUNGEONS.md](PROCEDURAL_DUNGEONS.md). For cartography and mapping, see [CARTOGRAPHY.md](CARTOGRAPHY.md).

## Design Philosophy

Exits are the connective tissue of the game world. A player's experience of moving through the world should be seamless and consistent — whether they're walking down a road, opening a locked door, diving underwater, or stepping into a procedural dungeon that didn't exist until they entered it.

All exits share a common foundation. Specialised behaviors (doors, dungeon entry, conditional routing) are composed from reusable building blocks — mixins and base classes — rather than built as parallel systems. A door that leads to a dungeon should behave like a door AND a dungeon entry, not one or the other.

## Class Hierarchy

```
DefaultExit (Evennia)
└── ExitBase
    └── ExitVerticalAware
        ├── ExitDoor (+ SmashableMixin, LockableMixin, CloseableMixin,
        │   │           InvisibleObjectMixin, HiddenObjectMixin)
        │   └── TrapDoor (+ TrapMixin — directional door trap)
        ├── TripwireExit (+ TrapMixin — directional passage trap)
        ├── DungeonExit (lazy room creation inside procedural dungeons)
        ├── DungeonPassageExit (dungeon → world exit, cleans up instance tags)
        ├── ConditionalRoutingExit (routes between rooms based on conditions)
        ├── ProceduralDungeonExit (+ ProceduralDungeonMixin — simple dungeon entry)
        ├── ConditionalDungeonExit (+ ProceduralDungeonMixin — quest-gated dungeon)
        ├── ExitTutorialStart (spawns a TutorialInstanceScript on traverse)
        └── ExitTutorialReturn (cleans up tutorial state, returns player to prior location)
```

### Composable Pieces

The exit system uses composition over inheritance for specialised behaviors:

```
ProceduralDungeonMixin          — provides enter_dungeon() capability (any exit)
ConditionalRoutingExit          — routes between rooms based on conditions
ProceduralDungeonExit           — ProceduralDungeonMixin + ExitVerticalAware
ConditionalDungeonExit          — ProceduralDungeonMixin + ConditionalRoutingExit
DungeonDoor                     — ProceduralDungeonMixin + ExitDoor (future)
```

See [PROCEDURAL_DUNGEONS.md](PROCEDURAL_DUNGEONS.md) for the full dungeon entry architecture.

## Base Classes

### ExitBase

**File:** `typeclasses/terrain/exits/exit_base.py`
**Inherits:** `DefaultExit`

The foundation for all game exits. Provides a single enhancement over Evennia's default: auto-generates a preview description from the destination room when no custom `db.desc` is set. This prevents the generic "This is an exit." from ever showing to players.

**When to use:** Never directly. Always use `ExitVerticalAware` or a more specific subclass.

### ExitVerticalAware

**File:** `typeclasses/terrain/exits/exit_vertical_aware.py`
**Inherits:** `ExitBase`

The standard exit for the game world. Provides two systems:

**Direction system:**
- `direction` AttributeProperty stores the compass direction (north, south, up, down, etc.)
- `set_direction()` sets the direction and auto-adds abbreviation aliases (n, s, u, d, etc.)
- `get_display_name()` formats as "direction: description" for the verbose `exits` command
- The room's `get_display_exits()` reads the `direction` attribute to show compact `[ Exits: n s e w ]`

**Vertical traversal checks:**
- Encumbrance: blocks flying/swimming while overloaded (falling, sinking)
- Height: blocks movement if character is too high for destination room
- Depth: blocks movement if character is too deep for destination room
- All checks run before `super().at_traverse()` — failed checks cancel movement silently

**Size gating:**
- `max_size` AttributeProperty — largest actor size that can pass. Defaults to `Size.GARGANTUAN.value` (unrestricted). Set to a smaller `Size.X.value` to restrict passage (e.g. `Size.TINY.value` for a mousehole, `Size.SMALL.value` for a halfling tunnel).
- Checked in `at_traverse()` after height gating: `size_value(actor.size) > size_value(self.max_size)` → "You are too large to fit through there."
- **Unlike height-gated exits, size-gated exits remain visible** in the exit list — the player should see the passage exists so they know a shrink spell is needed.
- Door helpers default `max_size` to `Size.MEDIUM.value` (standard human-sized door). Stable doors, castle gates, etc. override to `Size.LARGE.value` or higher.

**When to use:** Most world exits. Created via `connect_bidirectional_exit()` helper.

### ExitDoor

**File:** `typeclasses/terrain/exits/exit_door.py`
**Inherits:** `SmashableMixin, LockableMixin, CloseableMixin, InvisibleObjectMixin, HiddenObjectMixin, ExitVerticalAware`

A door that can be opened, closed, locked, unlocked, hidden, made invisible, and smashed. Doors are always created in pairs (one per side) linked via `link_door_pair()`.

**MRO rule:** SmashableMixin must come first, LockableMixin before CloseableMixin, so the `can_open()` lock check fires via the super() chain.

**Key behaviors:**
- Closed doors are hidden from the `[ Exits: ]` auto-exit line
- Closed doors are visible via the `exits` command with state indicators
- Locked doors block both opening and traversal
- Hidden doors (`is_hidden=True`) are invisible until discovered via `search`
- Invisible doors require `DETECT_INVIS` condition to see
- Reciprocal pairing syncs state changes between both sides
- `door_name` attribute controls what `open door` matches — use distinct names when multiple doors exist in one room (e.g. "door" vs "stone door")
- **Auto-close:** all doors auto-close after 5 minutes by default (`auto_close_seconds = 300`). Set to `0` to disable, or any other value for custom timing (e.g. `30` for a spring-loaded door). The timer starts when the door is opened and is cancelled if the door is manually closed. The auto-close fires `at_close()` so reciprocal pairing syncs the other side automatically.

**When to use:** Any door connection. Created via `connect_bidirectional_door_exit()` helper.

## Conditional Routing

**File:** `typeclasses/terrain/exits/conditional_routing_exit.py`
**Inherits:** `ExitVerticalAware`

An exit that routes between two destinations based on player state. The exit is always visible; the destination changes. Inherits the full direction system and vertical checks from `ExitVerticalAware`.

**Attributes:**
- `condition_type` — `"quest_active"`, `"quest_complete"`, or `"has_tag"`
- `condition_key` — the quest key, tag name, etc.
- `alternate_destination_id` — room ID for the alternate destination

**Routing logic:**
- Condition met → traverse to `destination` (the exit's normal destination)
- Condition NOT met → traverse to `alternate_destination_id`
- No condition configured → always primary destination
- No alternate set and condition not met → "The way is blocked."

Both paths use `super().at_traverse()` so vertical checks, encumbrance, and height restrictions all apply regardless of which destination is chosen.

**Condition types:**

| Type | Checks | True when |
|---|---|---|
| `quest_active` | `quests.has(key) and not quests.is_completed(key)` | Quest accepted but not finished |
| `quest_complete` | `quests.is_completed(key)` | Quest finished |
| `has_tag` | `tags.get(key, category="player_flag")` | Player has the flag tag |

This class has zero dungeon knowledge. It's a pure routing mechanism usable for any condition-based exit (quest gates, alignment locks, level restrictions, etc.).

## Dungeon Exits

### ProceduralDungeonMixin

**File:** `typeclasses/mixins/procedural_dungeon.py`

A mixin that adds dungeon instance creation capability to any exit. Provides `enter_dungeon(traversing_object)` as a utility method — does NOT override `at_traverse`. The host class decides when to call it.

**Attributes:**
- `dungeon_template_id` — template to instantiate
- `dungeon_destination_room_id` — for passage dungeons, the destination world room

**Key method — `enter_dungeon(traversing_object)`:**
1. Template lookup + validation
2. Dungeon tag check (prevent stacking)
3. Character collection (group/solo/shared mode)
4. Instance resolution (find existing shared or create new)
5. Start dungeon or join existing shared instance
6. Entry announcement
7. Returns True/False

### ProceduralDungeonExit

**File:** `typeclasses/terrain/exits/procedural_dungeon_exit.py`
**Inherits:** `ProceduralDungeonMixin, ExitVerticalAware`

Simple dungeon entry — always enters a procedural dungeon on traversal. `at_traverse` calls `self.enter_dungeon()` directly. Does not call `super()` since dungeon entry handles its own movement.

**Use for:** Unconditional dungeon entries (deep woods passages, cave of trials).

### ConditionalDungeonExit

**File:** `typeclasses/terrain/exits/conditional_dungeon_exit.py`
**Inherits:** `ProceduralDungeonMixin, ConditionalRoutingExit`

Quest-gated dungeon entry with fallback room. Condition met → `enter_dungeon()`. Condition not met → normal traversal to alternate destination (with full vertical checks).

**Use for:** Quest-gated dungeons where non-quest players should reach a different room (e.g. rat cellar: quest active → dungeon instance, no quest → empty cellar).

## Tutorial Hub Exits

Two specialised exits drive the Tutorial Hub. Both inherit from `ExitVerticalAware`.

### ExitTutorialStart

**File:** `typeclasses/terrain/exits/exit_tutorial_hub.py`

Per-chunk tutorial entry exit. Each instance carries a `tutorial_num` attribute (1, 2, or 3) selecting which tutorial chunk to launch. On traverse:

1. Validates `tutorial_num` is in (1, 2, 3)
2. Refuses entry if the player already has a `tutorial_character` tag
3. Creates a fresh `TutorialInstanceScript`, wires it to the hub, and calls `start_tutorial(player, chunk_num)`

The script is responsible for moving the player into the tutorial — `at_traverse` does not call `super()`.

### ExitTutorialReturn

**File:** `typeclasses/terrain/exits/exit_tutorial_hub.py`

South exit from the Tutorial Hub. Returns the player to their pre-tutorial location, or The Harvest Moon if they're a brand-new character. On traverse:

1. Resolves the destination via `db.pre_tutorial_location_id` → fallback to "The Harvest Moon" → last-resort Limbo
2. Cleans up any lingering `tutorial_character` tags by collapsing the matching `TutorialInstanceScript` and removing the tag
3. Deletes any inventory items flagged with `db.tutorial_item = True`
4. Teleports the player and clears `pre_tutorial_location_id`

This pattern (cleanup-on-leave rather than cleanup-on-enter) means a crash or disconnect inside the tutorial leaves no orphan state in the player's persistent inventory.

### DungeonExit

**File:** `typeclasses/terrain/exits/dungeon_exit.py`
**Inherits:** `ExitVerticalAware`

Exit inside a procedural dungeon with lazy room creation. Points to its own location as a placeholder; on first traversal, asks the `DungeonInstanceScript` to generate the real destination room. Forward exits are gated by the `not_clear` tag (encounter clearance). Return exits always allow passage.

### DungeonPassageExit

**File:** `typeclasses/terrain/exits/dungeon_passage_exit.py`
**Inherits:** `ExitVerticalAware`

Exit at dungeon boundaries (back to the world). Removes the character's dungeon instance tag on traverse, and explicitly untags followers (since follower cascade bypasses `at_traverse`).

## Visibility Filtering

Exits participate in the game's visibility system at multiple levels:

| Check | Where | What it filters |
|---|---|---|
| `is_visible_to(looker)` | `get_display_exits()` | Auto-exit line `[ Exits: ]` |
| `is_visible_to(looker)` | `CmdExits.func()` | Verbose `exits` command |
| `is_visible_to(looker)` | `find_exit_target()` | `open`, `close`, `lock`, `unlock`, `picklock` |
| `is_visible_to(looker)` | `CmdLook.func()` | `look <direction>` |
| `is_open` | `get_display_exits()` | Closed doors hidden from auto-exits |

Hidden objects that fail visibility checks produce the message "You don't see 'X' here" (no period) — a subtle hint for observant players that something is there to find. Objects that truly don't exist produce "You don't see 'X' here." (with period).

## Direction Lookup

The `look <direction>` command intercepts direction abbreviations (n/s/e/w/ne/nw/se/sw/u/d and full names) before they hit Evennia's generic object search. This prevents single-letter abbreviations from matching room names, character names, or other objects via substring.

Room names are also excluded from `look <target>` matches — players who want to see the room just type `look`.

If a direction has no visible exit: "You see nothing special in that direction."

## Builder Helpers

**File:** `utils/exit_helpers.py`

**CRITICAL: All exits MUST be created through a helper function.** Never use bare `create_object()` for exits. If an exit type isn't covered by an existing helper, create a new helper first, then use it. This ensures consistency across all zones and makes auditing/refactoring possible.

All bidirectional helpers use the `OPPOSITES` dict to auto-derive the reverse direction.

### Bidirectional Exits (create exits in BOTH directions)

| Helper | Exit Type | Purpose |
|---|---|---|
| `connect_bidirectional_exit(room_a, room_b, direction)` | ExitVerticalAware | Standard passage. Optional `desc_ab`/`desc_ba` for custom exit names. `max_size` defaults to GARGANTUAN (unrestricted). Returns `(exit_ab, exit_ba)`. |
| `connect_bidirectional_door_exit(room_a, room_b, direction, key, ...)` | ExitDoor | Door pair with reciprocal open/close/lock sync via `link_door_pair()`. Accepts per-side open/closed descriptions, lock settings, `auto_close_seconds` (default 300). `max_size` defaults to MEDIUM (standard door). Returns `(door_ab, door_ba)`. |
| `connect_bidirectional_trapped_door_exit(room_a, room_b, direction, ...)` | TrapDoor + ExitDoor | Door pair with a trap on ONE side (`trap_side="ab"` or `"ba"`). Trap is directional — only fires when opened from the trapped side. Accepts all door params plus trap config and `auto_close_seconds` (default 300). `max_size` defaults to MEDIUM. Returns `(door_ab, door_ba)`. |
| `connect_bidirectional_tripwire_exit(room_a, room_b, direction, ...)` | TripwireExit + ExitVerticalAware | Passage pair with a tripwire on ONE side. Trap fires on traverse if undetected; safe step-over if detected. Accepts trap config. `max_size` defaults to GARGANTUAN. Returns `(exit_ab, exit_ba)`. |

### One-Way Exits (create a SINGLE exit)

| Helper | Exit Type | Purpose |
|---|---|---|
| `connect_oneway_loopback_exit(room, direction, key=None, destination=None)` | ExitVerticalAware | Boundary illusion — exit loops back to the same room by default, or to `destination` if supplied. `max_size` defaults to GARGANTUAN. Returns the exit object. |

## Mixins Used by Exits

> For the full shared mixin table (including world object mixins like SwitchMixin, ClimbableMixin, etc.), see [WORLD_OBJECTS.md](WORLD_OBJECTS.md).

Exit-specific mixins:

| Mixin | Provides | Used by |
|---|---|---|
| CloseableMixin | `is_open`, `auto_close_seconds`, `open()`, `close()`, `can_open()` | ExitDoor |
| LockableMixin | `is_locked`, `relock_seconds`, `unlock()`, `picklock()`, `lock()` | ExitDoor |
| HiddenObjectMixin | `is_hidden`, `find_dc`, `discovered_by`, `discover()` | ExitDoor |
| InvisibleObjectMixin | `is_invisible`, `is_invis_visible_to()` | ExitDoor |
| SmashableMixin | `smash_hp`, `take_smash_damage()`, `at_smash_break()` | ExitDoor |
| TrapMixin | `is_trapped`, `trigger_trap()`, `disarm_trap()` | TrapDoor, TripwireExit |
| ProceduralDungeonMixin | `enter_dungeon()` | ProceduralDungeonExit, ConditionalDungeonExit |

## Key Implementation Rules

1. **All exits MUST be created through helper functions** in `utils/exit_helpers.py`. Never use bare `create_object()` for exits. If a new exit type isn't covered by an existing helper, create the helper first, then use it.
2. **All world exits should use ExitVerticalAware or a subclass** — never bare `DefaultExit`. The direction system and vertical checks are standard.
3. **Doors must be created in pairs** via `connect_bidirectional_door_exit()`. Unpaired doors will desync.
4. **Use `set_direction()`, not manual alias adds** — it handles both the attribute and aliases atomically.
5. **Distinct `door_name` values** when multiple doors exist in one room — prevents `open door` from matching the wrong one.
6. **at_traverse super() chain** — each exit class checks its own conditions then calls `super().at_traverse()`. Breaking the chain skips downstream checks (e.g. skipping ExitVerticalAware's height checks). Dungeon entry exits (`ProceduralDungeonExit`) intentionally skip super since they handle movement internally.
7. **Follower cascade bypasses exit at_traverse** — when a leader moves, followers are moved by `at_post_move` hooks, not by re-traversing the exit. Any cleanup that must happen for followers (e.g. dungeon tag removal) must be done explicitly in the leader's exit traverse.
8. **ProceduralDungeonMixin is a capability, not an exit type** — it provides `enter_dungeon()` but never overrides `at_traverse`. The host class decides when to call it. This allows dungeon entry to compose with any exit behavior.

## Test Coverage

The dungeon and exit systems have comprehensive test coverage across:
- Template registry and lookup
- Instance lifecycle (creation, collapse, tick-based expiry)
- Lazy room creation via DungeonExit
- Exit budget and branching constraints
- Boss room generation and encounter gating
- ProceduralDungeonExit (entry, group mode, follower rejection, already-in-dungeon)
- ConditionalRoutingExit (quest_active, quest_complete, has_tag, no condition, no alternate)
- ConditionalDungeonExit (quest-gated dungeon + fallback routing)
- Passage dungeons (return exits, terminal exits, tag cleanup)
- Shared instance mode (joining, persistence after empty)
- Follower teleport guard
