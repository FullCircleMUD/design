# Inventory & Equipment System

## Overview

The inventory system is a multi-layered architecture with three distinct storage mechanisms unified under a single carrying capacity model. All weight and stat tracking uses nuclear recalculate patterns — full rebuilds from scratch on every change — to eliminate drift.

## Storage Model

### Physical Items (NFTs)

Items exist in Evennia's standard `contents` hierarchy. An item's `location` determines where it is:

- `item.location = room` — item is in the world
- `item.location = character` — item is in character inventory
- `item.location = container` — item is inside a container

No separate inventory data structure exists. The game uses Evennia's native `contents` property.

### Equipment (Wearslots)

Equipped items remain in `contents`. A separate `db.wearslots` dict on the character holds references to equipped items:

```python
wearslots = {
    "HEAD":              <Iron Helm>,    # occupied
    "NECK":              None,           # empty
    "BODY":              <Leather Armor>,
    "LEFT_RING_FINGER":  <Ring of Fire>,
    ...
}
```

**Key design principle:** equipping/unequipping does NOT move items between locations. The item stays in `contents` — only the wearslot reference changes. This means weight does not change on equip/unequip (the character is still carrying the item either way).

### Fungible Inventory (Gold & Resources)

Gold and resources are not objects — they are quantities stored as Evennia attributes:

```python
character.db.gold = 500           # integer gold count
character.db.resources = {4: 10}  # {resource_id: quantity}
```

Managed by `FungibleInventoryMixin` (`typeclasses/mixins/fungible_inventory.py`). Mixed into `FCMCharacter`, `RoomBase`, and `AccountBank`. Must call `self.at_fungible_init()` from `at_object_creation()`.

### FungibleInventoryMixin API

| Method group | Methods |
|---|---|
| Queries | `get_gold()`, `has_gold()`, `get_resource()`, `has_resource()`, `get_all_resources()` |
| In-game transfers | `transfer_gold_to(target, amount)`, `transfer_resource_to(target, resource_id, amount)` |
| From reserve | `receive_gold_from_reserve(amount)`, `receive_resource_from_reserve(resource_id, amount)` |
| To reserve | `return_gold_to_reserve(amount)`, `return_resource_to_reserve(resource_id, amount)` |
| To sink | `return_gold_to_sink(amount)`, `return_resource_to_sink(resource_id, amount)` |
| Chain boundary | `deposit_gold_from_chain(amount, tx_hash)`, `withdraw_gold_to_chain(amount, tx_hash)` |
| Chain boundary | `deposit_resource_from_chain(resource_id, amount, tx_hash)`, `withdraw_resource_to_chain(resource_id, amount, tx_hash)` |

**Reserve vs sink:** Returning to the *reserve* puts gold/resources back into the spawnable pool — the next loot drop or treasury draw can re-emit them. Returning to the *sink* permanently removes them from circulation, used by economy drains (travel/sail food costs, training fees, repair costs). Use the sink for any cost that should not feed back into the spawn budget.

Classification helpers (used internally):
- `_classify_fungible(obj)` → `"CHARACTER"` / `"ACCOUNT"` / `"WORLD"`
- `_get_wallet()` — CHARACTER: `account.wallet_address`, ACCOUNT: `self.wallet_address`, WORLD: vault address
- `_get_character_key()` — CHARACTER: `self.key`, others: `None`

`transfer_gold_to(None)` raises `ValueError` pointing to `return_gold_to_reserve()`. WORLD→WORLD transfers raise `ValueError` (unsupported).

## Equipment System

### Wearslot Architecture

`BaseWearslotsMixin` provides the core equip/unequip logic. Subclasses define available slots:

| Mixin | Slots | Used By |
|-------|-------|---------|
| `HumanoidWearslotsMixin` | HEAD, FACE, LEFT_EAR, RIGHT_EAR, NECK, CLOAK, BODY, LEFT_ARM, RIGHT_ARM, HANDS, LEFT_WRIST, RIGHT_WRIST, LEFT_RING_FINGER, RIGHT_RING_FINGER, WAIST, LEGS, FEET, WIELD, HOLD | FCMCharacter |
| `DogWearslotsMixin` | DOG_NECK, DOG_BODY | Proof of concept |

Items declare which slot(s) they fit via `wearslot` AttributeProperty (single enum value or list). Creature-type restriction is implicit — a human has no DOG_NECK slot, so dog collars can't be equipped. The wearslot enums in `enums/wearslot.py` are the single source of truth for slot names.

**Defined enums (in `enums/wearslot.py`):**
- `HumanoidWearSlot` — used by `HumanoidWearslotsMixin`
- `DogWearSlot` — used by `DogWearslotsMixin`
- `MuleWearSlot` (PANNIER, BRIDLE, HORSE_SHOES) — *enum defined, mixin not yet implemented*
- `HorseWearSlot` (SADDLE_BAG, SADDLE, BRIDLE, HORSE_SHOES) — *enum defined, mixin not yet implemented*

The Mule and Horse enums are placeholder slot definitions ahead of the corresponding wearslots mixins, intended for the mounts feature.

### BaseWearslotsMixin API

| Method | Returns | Purpose |
|---|---|---|
| `at_wearslots_init()` | — | Initialize `self.db.wearslots = {}` (call from `at_object_creation()`) |
| `wear(item)` | `(bool, str)` | Equip item, calls `item.at_wear(self)` after |
| `remove(item)` | `(bool, str)` | Unequip item, calls `item.at_remove(self)` after |
| `is_worn(item)` | `bool` | Is item in any wearslot? |
| `get_all_worn()` | `list` | All currently worn items |
| `get_slot(slot_name)` | item or None | What's in a specific slot |
| `get_carried()` | `list` | Contents minus worn items (for inventory display) |
| `equipment_cmd_output(header)` | `str` | Formatted equipment display |

### Wear Effects (Data-Driven)

Items declare effects in their prototype's `wear_effects` list:

```python
wear_effects = [
    {"type": "stat_bonus", "stat": "armor_class", "value": 2},
    {"type": "condition", "condition": "darkvision"},
    {"type": "damage_resistance", "damage_type": "fire", "value": 25},
    {"type": "hit_bonus", "weapon_type": "longsword", "value": 1},
    {"type": "damage_bonus", "weapon_type": "longsword", "value": 1},
]
```

On equip (`at_wear`): condition effects are applied incrementally (ref-counted), then `_recalculate_stats()` rebuilds all numeric stats from scratch.

On unequip (`at_remove`): condition effects are removed (ref-count decremented), then `_recalculate_stats()` rebuilds.

#### Supported Effect Types

| Type | Format | Example |
|---|---|---|
| `stat_bonus` | `{"type": "stat_bonus", "stat": "<name>", "value": <int>}` | `{"type": "stat_bonus", "stat": "armor_class", "value": 1}` |
| `damage_resistance` | `{"type": "damage_resistance", "damage_type": "<type>", "value": <int>}` | `{"type": "damage_resistance", "damage_type": "piercing", "value": 50}` |
| `condition` | `{"type": "condition", "condition": "<name>"}` | `{"type": "condition", "condition": "darkvision"}` |
| `condition` (compound) | `{"type": "condition", "condition": "<name>", "effects": [...]}` | `{"type": "condition", "condition": "hasted", "effects": [{"type": "stat_bonus", "stat": "attacks_per_round", "value": 1}]}` |
| `hit_bonus` | `{"type": "hit_bonus", "weapon_type": "<WeaponType.value>", "value": <int>}` | `{"type": "hit_bonus", "weapon_type": "unarmed", "value": 1}` |
| `damage_bonus` | `{"type": "damage_bonus", "weapon_type": "<WeaponType.value>", "value": <int>}` | `{"type": "damage_bonus", "weapon_type": "unarmed", "value": 1}` |

**Supported stats for `stat_bonus`:** Any AttributeProperty on FCMCharacter — `strength`, `dexterity`, `constitution`, `intelligence`, `wisdom`, `charisma`, `armor_class`, `crit_threshold`, `initiative_bonus`, `attacks_per_round`, `stealth_bonus`, `move_max`, etc. Tier 2 stats store equipment/spell bonuses only — ability score modifiers are computed at check time via Tier 3 @properties.

**Weapon-type-specific bonuses (`hit_bonus`/`damage_bonus`):** Stored in `hit_bonuses` and `damage_bonuses` dicts on FCMCharacter, keyed by `WeaponType.value` string (e.g. `"unarmed"`, `"long_sword"`, `"dagger"`). Multiple sources stack additively. Rebuilt from scratch during `_recalculate_stats()`. At combat time, look up `character.hit_bonuses.get(weapon_type_value, 0)`.

**Damage types for `damage_resistance`:** `slashing`, `piercing`, `bludgeoning`, `fire`, `cold`, `lightning`, `acid`, `poison`, `magic`, `force` (see `enums/damage_type.py`). Values are integer percentages — `50` means 50% resistance. Capped to [-75, 75] on read via `get_resistance()`.

#### Stacking Rules

Standalone `stat_bonus` effects stack additively (two +1 STR rings = +2 STR — both counted during recalculate). Compound `condition` effects with nested `"effects"` only apply companion bonuses once per condition during recalculate — wearing a second haste item increments the condition ref count but the companion stat bonus is only accumulated once (tracked via `_accumulated_companions` set during recalculate).

**Prototypes only apply at spawn time** — editing a prototype affects new spawns only. Existing items keep their original values. This is intentional for an NFT game (legacy items).

### Equip Flow

1. Item must be in character `contents`
2. Item must not already be worn
3. Item must declare a `wearslot`
4. A matching slot must be empty
5. Item usage restrictions checked via `ItemRestrictionMixin.can_use()` (see below)
6. Creature-type check via `can_wear()` — slot must exist on this creature's mixin
7. `db.wearslots[slot] = item` — reference set
8. `item.at_wear(character)` — apply condition effects, trigger nuclear stat recalculate

### ItemRestrictionMixin (`typeclasses/mixins/item_restriction.py`)

Data-driven item usage restrictions mixed into `BaseNFTItem`. Default is unrestricted — items only become restricted when a prototype sets restriction fields.

| Field | Logic | Description |
|---|---|---|
| `required_classes` | OR | Character has ANY listed class → pass |
| `excluded_classes` | AND-NOT | Character has ANY listed class → fail (vetoes) |
| `min_class_levels` | ALL | Each `{class: level}` must be met |
| `required_races` | OR | Character's race in list → pass |
| `excluded_races` | AND-NOT | Character's race in list → fail |
| `min_alignment_score` | ≥ | `character.alignment_score >= value` (e.g. 300 = Good+ only) |
| `max_alignment_score` | ≤ | `character.alignment_score <= value` (e.g. -300 = Evil+ only) |
| `min_total_level` | ≥ | `character.total_level >= value` |
| `min_remorts` | ≥ | `character.num_remorts >= value` |
| `min_attributes` | ALL | Each `{ability: score}` must be met |
| `min_mastery` | ALL | Each `{skill: level}` must be met |

- `can_use(character)` → `(bool, str)` — checks all restrictions, short-circuits on first failure
- `is_restricted` property — `True` if any field is non-default
- Hooked into `wear()` validation chain — `can_use()` runs before `can_wear()`

## Durability

Items wear out over time and use. Durability is a single integer on each item (`durability` / `max_durability`) with a repairable flag (`repairable`). When `durability` reaches zero, the item is broken — equipment stops applying its effects and weapons no longer deal bonus damage (precise behaviour depends on the subclass hook).

### Sources of wear

| Source | Trigger | Loss |
|---|---|---|
| **Combat hit** | Weapon strikes a target | -1 on the weapon, -1 on the target's body armour |
| **Parry** | Parry succeeds | -1 on both the attacker's weapon and the defender's parrying weapon |
| **Crit downgrade** | Helmet's CRIT_IMMUNE reduces an incoming crit to a normal hit | -1 on the helmet |
| **Passive decay** | Global daily tick (`DurabilityDecayService`, 1 game day = 3600 real seconds) | -1 on every equipped item, for each IC/puppeted character |

Combat wear and passive decay **stack** — an item in heavy use wears out faster than its baseline material tier suggests.

### Material Durability Tiers

The passive decay service gives each material a baseline lifetime in **game days** of continuous wear (ignoring combat). Material tier is the dominant lever for balancing gear longevity vs economy turnover.

| Material | Baseline game days |
|---|---|
| Cloth | 720 |
| Silk / Leather / Wood | 1,440 |
| Hardwood / Wyvern leather | 2,880 |
| Bronze / Copper / Silver | 3,600 |
| Iron | 5,400 |
| Steel | 7,200 |
| Mithral / Adamantine | 9,000 |

At `TIME_FACTOR=24`, 1 game day = 1 real hour of play. A cloth item worn continuously while online lasts roughly 720 real hours of play before passive decay alone exhausts it; a mithral item lasts ~9,000 hours. Combat pulls these numbers down.

### DurabilityDecayService (`typeclasses/scripts/durability_decay_service.py`)

Global persistent script. Ticks every 3,600 real seconds (1 game day). Each tick:

1. Iterate every IC (puppeted, online) character.
2. For each, call `reduce_durability(1)` on every equipped item via `get_all_worn()`.
3. Stagger per-character processing via `delay()` so a full server doesn't block the reactor on a single tick.

Key design choices:

- **No offline catch-up** — items only decay while the character is being played. Logging out preserves gear.
- **Puppeted only** — unpuppeted characters on the login screen are skipped.
- **Per-character independent** — one character's decay tick does not affect another's.

`get_game_day_number()` is a module-level free function returning the absolute game-day count, usable by any system that needs "what game day is it" without caring which phase of the cycle.

### Repair

`repair_to_full()` on `DurabilityMixin` restores durability in full. Surfaced via the `repair` command at `RoomCrafting` rooms — see `CRAFTING_SYSTEM.md § Repair Flow`. The `repairable` flag gates this: disposable or degradation-locked items can be marked non-repairable.

## Carrying Capacity

### Three Weight Components

| Component | Attribute | Recalculate Trigger | Pattern |
|-----------|-----------|---------------------|---------|
| Item weight | `items_weight` | `at_object_receive` / `at_object_leave` | Nuclear |
| Fungible weight | `current_weight_fungibles` | `_at_balance_changed()` | Nuclear |
| **Total** | `current_weight_carried` (property) | Sum of above | Derived |

### Effective Capacity

```
effective_capacity = max_carrying_capacity_kg + get_attribute_bonus(strength) * 5
```

Ability modifier is NEVER cached — always computed at check time.

### Container Weight Propagation

Containers (backpacks) track their own `current_contents_weight`. When `transfer_weight=True`, changes to container contents trigger a nuclear recalculate on the carrier character.

| Container Type | `transfer_weight` | Carrier Impact |
|---------------|-------------------|----------------|
| Backpack | True | Carrier weight includes backpack contents |
| Panniers (mount) | False | Only container's own weight counts |

## Nuclear Recalculate Systems

### Why Nuclear Over Incremental

The codebase uses two nuclear recalculate systems that rebuild data from scratch rather than patching incrementally. The rationale:

1. **Eliminates drift.** Incremental tracking breaks when hooks don't fire. Example: `item.delete()` does not fire `at_object_leave()` on the parent — the old incremental weight system left ghost weight on characters after tutorial cleanup deleted items. Nuclear recalculate is immune: the next event triggers a full rebuild from actual state.

2. **Handles multi-source effects.** Two rings both granting +1 STR — removing one must not break the other. Rebuilding from scratch naturally handles this without tracking which source contributed what.

3. **Simpler invariants.** One truth source (rebuild from current state) instead of maintaining correctness across hundreds of increment/decrement sites.

4. **Acceptable cost.** Characters carry ~20 items max. Iterating 20 items is negligible. The fungible weight calculation queries a cached resource type table. Both complete in <1ms.

### _recalculate_stats() — Combat Stats

**File:** `typeclasses/actors/base_actor.py`

Triggered by: equip, unequip, buff apply, buff expire, `/recalc` command.

Rebuilds from scratch:
- Ability scores (STR, DEX, CON, INT, WIS, CHA)
- HP/mana/move pool maximums
- Armor class, initiative, attacks per round
- Hit bonuses and damage bonuses (per weapon type)
- Damage resistances
- Stealth and perception bonuses

Sources iterated (in order):
1. Racial effects
2. Worn equipment `wear_effects`
3. Active named effects (spells, potions, combat buffs)

**Conditions are NOT rebuilt** — they use ref-counting (add_condition/remove_condition) and are managed incrementally. Only numeric stats are nuclear.

### _recalculate_item_weight() — Carrying Weight

**File:** `typeclasses/mixins/carrying_capacity.py`

Triggered by: `at_object_receive()`, `at_object_leave()`, container `_propagate_weight_to_carrier()`.

Rebuilds `items_weight` by iterating all objects in `contents`:
- Adds each object's `.weight`
- For containers with `transfer_weight=True`, also adds `current_contents_weight`

Uses `exclude` parameter for `at_object_leave` because Evennia fires the hook before the item is actually removed from `contents`.

### _at_balance_changed() — Fungible Weight

**File:** `typeclasses/mixins/carrying_capacity.py` (override of `FungibleInventoryMixin`)

Triggered by: `_add_gold()`, `_remove_gold()`, `_add_resource()`, `_remove_resource()`.

Calls `get_total_fungible_weight()` to rebuild fungible weight from scratch (gold quantity * weight per unit + each resource quantity * weight per unit).

## NFT Ownership Model

Each item has dual representation:
- **Evennia object** — in-game item with attributes, location, behavior
- **NFTMirror record** — blockchain DB row tracking ownership state

### Ownership States

| State | Meaning |
|-------|---------|
| RESERVE | Unallocated in vault pool |
| SPAWNED | In game world, no player owner |
| CHARACTER | In player inventory |
| ACCOUNT | In player bank |
| ONCHAIN | On player's external wallet |

Transitions are dispatched automatically by `BaseNFTItem.at_post_move()` based on source/destination. Same-wallet moves (inventory to own backpack) are no-ops.

### at_post_move Dispatch (source → destination)

**Creation (source is None — item entering the game world):**

| Destination | Service call | Flow |
|---|---|---|
| `WORLD` (room) | `NFTService.spawn()` | RESERVE → SPAWNED |
| `CHARACTER` | `NFTService.craft_output()` | RESERVE → CHARACTER |
| `ACCOUNT` (bank) | `NFTService.deposit_from_chain()` | ONCHAIN → ACCOUNT |

**Movement (source is not None):**

| Source → Destination | Service call | Flow |
|---|---|---|
| `WORLD` → `CHARACTER` | `NFTService.pickup()` | SPAWNED → CHARACTER |
| `CHARACTER` → `WORLD` | `NFTService.drop()` | CHARACTER → SPAWNED |
| `CHARACTER` → `CHARACTER` | `NFTService.transfer()` | CHARACTER → CHARACTER |
| `CHARACTER` → `ACCOUNT` | `NFTService.bank()` | CHARACTER → ACCOUNT |
| `ACCOUNT` → `CHARACTER` | `NFTService.unbank()` | ACCOUNT → CHARACTER |
| `WORLD` → `WORLD` | *(no-op)* | stays SPAWNED |

**tx_hash for import:** Pass via `move_to(bank, tx_hash="0x...")` — Evennia forwards kwargs to `at_post_move`.

### at_object_delete Dispatch (by current location)

| Location | Service call | Flow |
|---|---|---|
| `WORLD` (room) | `NFTService.despawn()` | SPAWNED → RESERVE |
| `CHARACTER` | `NFTService.craft_input()` | CHARACTER → RESERVE |
| `ACCOUNT` (bank) | `NFTService.withdraw_to_chain()` | ACCOUNT → ONCHAIN |

**tx_hash for export:** Stash on `obj.ndb.pending_tx_hash` before calling `obj.delete()` — ndb is still alive when the hook fires.

### Factory Methods (spawn/despawn lifecycle)

**Spawning** (blank token pool → game object):
1. `BaseNFTItem.assign_to_blank_token(item_type_name)` — picks lowest RESERVE blank token, assigns item_type + default_metadata via `NFTService.assign_item_type()` with `select_for_update()` for concurrency safety. Returns `(token_id, chain_id, contract_address)`.
2. `BaseNFTItem.spawn_into(token_id, location, chain_id, contract_address)` — creates Evennia object from NFTMirror data using `spawn()` for prototype application, applies metadata as attributes, calls `move_to(location)` to trigger hooks.

**Despawning** (game object → reserve):
- `obj.delete()` triggers `at_object_delete` → service dispatch → `NFTService._reset_token_identity(nft)` wipes `item_type=None`, `metadata={}` on all RESERVE return paths (despawn, craft_input, account_to_reserve).

### Mob Item Spawning (non-NFT path)

`MobItem.spawn_mob_item(prototype_key, location)` — creates a mob-only item from the same prototype data used by NFT items. Reads the `mob_typeclass` field from the prototype dict instead of `typeclass`. No token_id, no NFTService, no blockchain. Mob items are ephemeral — deleted on mob death, never enter the player economy.

**Prototype dual-typeclass pattern:** Each weapon prototype defines both paths:
```python
IRON_LONGSWORD = {
    "prototype_key": "iron_longsword",
    "typeclass": "typeclasses.items.weapons.longsword_nft_item.LongswordNFTItem",
    "mob_typeclass": "typeclasses.items.mob_items.mob_weapons.MobLongsword",
    "key": "Iron Longsword",
    "base_damage": "d8",
    "material": "iron",
    ...
}
```

The stat fields (base_damage, material, damage_type, speed, weight) are shared — single source of truth. NFT-only fields (max_durability, excluded_classes) are stripped by `spawn_mob_item()`. Evennia's `spawn()` ignores unknown fields, so `mob_typeclass` is inert for the existing NFT spawn path.

**Callers:**
| Caller | Method | Purpose |
|---|---|---|
| Unified spawn service | `BaseNFTItem.spawn_into()` | Place NFT loot on mobs/rooms |
| Crafting / Import | `BaseNFTItem.spawn_into()` | Create NFT items |
| Mob creation / spawn scripts | `MobItem.spawn_mob_item()` | Equip mob with gear |

### Static/Class Methods

- `BaseNFTItem._classify(obj)` → `"CHARACTER"` / `"ACCOUNT"` / `"WORLD"` / `None`
- `BaseNFTItem.get_nft_mirror(token_id, chain_id, contract_address)` → delegates to `NFTService.get_nft()` (read-only mirror lookup — use this instead of importing NFTService directly)

### Extracted Composition Mixins

Equip and weapon combat mechanics are extracted into standalone mixins so they can be shared between NFT items (player economy) and MobItems (mob-only equipment, deleted on death). This gives mobs identical combat behaviour to players without consuming NFT tokens.

| Mixin | File | Provides |
|---|---|---|
| **WearableMixin** | `typeclasses/mixins/wearable_mixin.py` | `wearslot`, `wear_effects`, `at_wear()`/`at_remove()` with nuclear recalculate |
| **WeaponMechanicsMixin** | `typeclasses/items/weapons/weapon_mechanics_mixin.py` | All weapon AttributeProperties (`base_damage`, `material`, `damage_type`, `speed`, `two_handed`, `is_finesse`, `can_dual_wield`), `get_damage_roll()`, `get_wielder_mastery()`, 14 combat hooks, 10 mastery-scaled query methods |
| **Weapon identity mixins** | Same file as their NFT counterpart (e.g. `DaggerMixin` in `dagger_nft_item.py`) | Mastery tables and method overrides that define what makes a dagger a dagger (crit scaling, extra attacks, etc.) |

**MRO enables overrides without inheritance between mixins.** A weapon identity mixin (e.g. `DaggerMixin`) overrides methods defined on `WeaponMechanicsMixin` without inheriting from it. When composed — `MobDagger(DaggerMixin, MobWeapon)` — Python's MRO is: `MobDagger → DaggerMixin → MobWeapon → WeaponMechanicsMixin → ...`. So `get_extra_attacks()` resolves to DaggerMixin's override, while `get_wielder_mastery()` (which DaggerMixin calls but doesn't define) falls through to WeaponMechanicsMixin. Any hook the identity mixin doesn't override (e.g. `at_hit`) falls through to the base default.

### NFTMirrorMixin — Extracted Mirror Tracking

The NFT mirror state machine (ownership transitions, service call dispatch) is extracted into `NFTMirrorMixin` (`typeclasses/mixins/nft_mirror.py`). This allows any Evennia object — whether a `DefaultObject` (items) or `DefaultCharacter` (pets/actors) — to participate in the NFT lifecycle.

The mixin provides:
- `token_id`, `chain_id`, `contract_address` — NFT identity attributes
- `at_post_move()` override — dispatches mirror transitions (spawn, pickup, drop, transfer, bank, unbank)
- `at_object_delete()` override — dispatches destruction transitions (despawn, craft_input, withdraw)
- `_resolve_owner()`, `_classify()`, `_is_same_owner()` — location classification helpers
- `_handle_creation()`, `_execute_transition()` — transition dispatch
- `_cascade_container_transition()` — container content cascade
- `assign_to_blank_token()`, `spawn_into()` — factory methods
- `_get_owner_wallet()`, `_get_character_key()`, `_log_error()` — utility helpers

`BaseNFTItem` composes this mixin with item-specific concerns (HeightAwareMixin, HiddenObjectMixin, ItemRestrictionMixin, DefaultObject).

```
NFTMirrorMixin (typeclasses/mixins/nft_mirror.py)
  ├── BaseNFTItem(NFTMirrorMixin, ..., DefaultObject) — items
  └── NFTPetMirrorMixin(NFTMirrorMixin) — pet-specific override (see below)
```

### NFTPetMirrorMixin — Pet-Specific Mirror Dispatch

Inherits from `NFTMirrorMixin` but overrides the core dispatch logic because pets live in rooms (not `character.contents`). NFTMirrorMixin resolves ownership by classifying LOCATIONS — a room is always WORLD. Pets need ownership resolved via `owner_key` attribute on the pet itself.

Overrides: `at_post_move()`, `at_object_delete()`, `_handle_creation()`, `_execute_transition()`, `_resolve_owner()` (raised NotImplementedError), `_is_same_owner()` (raised NotImplementedError). No-ops cascade methods.

Adds: `owner_key`, `transfer_ownership()`, `spawn_pet()`, `_get_owner_character()`, `_get_owner_wallet()`, `_classify_location()`, `at_pre_move` guard (blocks character.contents), `at_pre_get` guard.

Inherits unchanged: `token_id`, `chain_id`, `contract_address`, `_get_owner_wallet()`, `_get_character_key()`, `assign_to_blank_token()`, `_load_from_mirror()`, `get_nft_mirror()`, `_log_error()`.

```
NFTPetMirrorMixin(NFTMirrorMixin) (typeclasses/mixins/nft_pet_mirror.py)
  └── BasePet(NFTPetMirrorMixin, FollowableMixin, BaseNPC) — pets, mounts
```

Mirror transitions for pets: create → craft_output, stable → bank, retrieve → unbank, die → craft_input, trade → transfer. All via hooks, zero manual service calls from commands. See `design/PETS_AND_MOUNTS.md` for full details.

### OwnedWorldObjectMixin — World-Anchored Item Patterns

Extracted from `WorldAnchoredNFTItem`, provides patterns for items that live in `character.contents` but represent a world location (ships, property):
- `db.world_location` — room where the object is in the world
- `set_world_location(room)` — update Evennia attribute AND patch the NFT mirror metadata so the location is visible on external marketplaces and survives chain export/import (see § NFT Metadata Persistence below)
- `get_world_location_display()` — human-readable location
- `get_owned_display()` (subclass-defined) — single-line summary including berth info, used by `owned`, bank `balance`, OOC `bank`, and any other listing
- `at_pre_get()` / `at_pre_drop()` blocks — can't be picked up or dropped
- `at_pre_give()` allows — ownership transfer
- `get:false()` lock and zero weight on init

**Ownership transfer surfaces.** World-anchored items can change hands via three flows, all of which preserve `db.world_location`:
- `give` — synchronous, both characters in the same room
- `deposit`/`withdraw` at any bank room — asynchronous through the shared `AccountBank`. Cross-character transfer between characters of the same account is the primary use case. The bank `balance` command renders ships in their own `|wShips:|n` subsection by default (no `balance all` gate). For the broader cross-surface display rules, see `design/INTERZONE_TRAVEL.md` § Banking & Cross-Character Transfer.
- `export`/`import` via XRPL — chain round-trip; `world_location` survives via mirror metadata persistence (see § NFT Metadata Persistence below).

In all three flows, the new owner does NOT auto-relocate to the dock — they have to travel there themselves before using the ship.

**Note:** Pets do NOT use this mixin. Pets are actors in rooms, not items in character.contents. Ships use this mixin because they sit in character.contents with a `world_location` pointer. Pets have their own banking flow via stable rooms (`stable` / `retrieve` commands on `RoomStable`) and are deliberately hidden from bank-room commands.

```
OwnedWorldObjectMixin (typeclasses/mixins/owned_world_object.py)
  └── WorldAnchoredNFTItem(OwnedWorldObjectMixin, BaseNFTItem) — ships, property
```

### NFT Metadata Persistence

Some NFT-side state is mutable in-game and needs to be visible on external XRPL marketplaces *and* survive a chain export → re-import round-trip: a ship's berthed dock, an item's current durability, a lantern's remaining fuel, a district map's survey progress. Without a persistence path, a player who exported a half-burnt lantern and re-imported it would get a fresh one — an economic gap and a broken UX.

#### Reframing: the URI is a stable pointer, not storage

All game NFTs are minted with an on-chain URI of the form `https://nft.fcmud.world/{game_id}`. This URI is a stable pointer to the off-chain `nft_api` HTTP endpoint (`nft_api/app/metadata.py`), which reads `NFTGameState.metadata` live from the game DB and serves it back as XLS-24d JSON. External marketplaces fetching the URI see whatever is currently in the mirror row.

This eliminates three complications:
- No on-chain `NFTokenModify` transaction is required.
- No re-mint with `tfMutable` is required.
- `NFTGameState` rows are keyed by `nftoken_id` and persist across export/import cycles, so anything written to `metadata` automatically survives chain round-trips.

The "persist to chain URI" problem is really "patch the server-side `NFTGameState.metadata` JSON field from typeclass state, safely".

#### Write path

`NFTService.update_metadata(token_id, patch)` (`blockchain/xrpl/services/nft.py`) is the only code that writes the field directly. It does an atomic JSON merge under `select_for_update()`:
- Keys in `patch` overwrite matching keys in the row's metadata.
- Keys mapped to `None` in `patch` are deleted.
- Empty patch is a no-op.
- Values must be JSON-serializable (str, int, float, bool, list/dict of same). Room references must be flattened to `dbref` and/or `key` by the caller.

Game code never calls `NFTService` directly (per the service encapsulation rule). Instead, `NFTMirrorMixin.persist_metadata(patch)` on `typeclasses/mixins/nft_mirror.py` wraps the call and is the single entry point used from typeclasses:
- No-op if the instance has no `token_id` yet (e.g. during creation, before `assign_to_blank_token` has run).
- Catches exceptions via `_log_error()` — metadata persistence is best-effort and must not crash gameplay (e.g. mid-voyage).

Write from a state hook by calling `self.persist_metadata({...})` once the attribute has been updated. Example from `DurabilityMixin.reduce_durability()`:

```python
self.durability = max(0, self.durability - amount)
self._persist_durability()  # calls self.persist_metadata({...})
```

#### Read path (export → re-import round-trip)

On re-import, `NFTMirrorMixin.spawn_into(token_id, location)` materialises a fresh Evennia object from the mirror row. After the typeclass-specific attributes are applied and `move_to()` fires, the method iterates over the metadata dict and writes every key onto the new instance via `obj.attributes.add(key, value)`. That is enough to fully restore any attribute whose JSON representation matches its AttributeProperty shape:

- `durability`, `max_durability` → land directly on `DurabilityMixin`'s properties.
- `is_lit`, `fuel_remaining`, `max_fuel` → land directly on `LightSourceMixin`'s properties.

For attributes that were flattened on the write path (e.g. a room reference stored as `dbref + key`, or a set stored as a sorted list), the typeclass needs a small re-hydration step. After the generic attribute copy and `move_to()`, `spawn_into()` fires an optional hook:

```python
if hasattr(obj, "at_restore_from_metadata"):
    obj.at_restore_from_metadata(meta)
```

Typeclasses override this to convert the JSON-flat form back into live state. Current overrides:

- `ShipNFTItem.at_restore_from_metadata` — resolves `world_location_dbref` to a live room via `ObjectDB.objects.get(id=...)`, assigns to `db.world_location`, and clears the stale flat keys so `set_world_location()` remains the single source of truth. Gracefully sets `None` if the dbref no longer resolves.
- `DistrictMapNFTItem.at_restore_from_metadata` — converts `db.surveyed_points` from a list back into a set so `.add()` keeps working.

The hook runs *after* `move_to()` so it doesn't get clobbered by `at_post_move` logic. Errors are logged, not raised.

#### Flattening rules

When a value is not natively JSON-serializable, the typeclass must flatten on write and rehydrate on read. Conventions established so far:

| Python type | Flat form | Restore strategy |
|---|---|---|
| Room reference | `<attr>_dbref` (int) + `<attr>_name` (str) | Resolve via `ObjectDB.objects.get(id=dbref)`, fall back to `None` |
| `set[str]` | sorted `list[str]` | `set(list)` in `at_restore_from_metadata` |
| `int`, `float`, `str`, `bool` | direct | no override needed |
| `None` (clear) | explicit `None` in patch | auto-handled by `update_metadata` (deletes key) |

Keep the same top-level key name for the flat form only when the shape matches directly (durability, fuel). When flattening a room reference, use a distinct prefix (`world_location_dbref`) so the flat key doesn't shadow the live attribute name — that avoids the generic copy loop in `spawn_into` silently overwriting `self.db.world_location` with an integer.

#### Wired call sites

| Mutable attribute | Mixin / typeclass | Write site | Restore hook needed? |
|---|---|---|---|
| `db.world_location` (ship berth) | `OwnedWorldObjectMixin` / `ShipNFTItem` | `set_world_location(room)` via `_persist_location_metadata()` | Yes — dbref → room |
| `durability`, `max_durability` | `DurabilityMixin` | `reduce_durability()`, `repair_to_full()` via `_persist_durability()` | No (direct attribute copy works) |
| `is_lit`, `fuel_remaining`, `max_fuel` | `LightSourceMixin` | `light()`, `extinguish()`, `refuel()` via `_persist_light_state()` | No (direct attribute copy works) |
| `db.surveyed_points` + derived `completion_pct` | `DistrictMapNFTItem` | `record_survey_points(keys)` | Yes — list → set |

Deliberately *not* wired:
- Per-tick `LightBurnScript` fuel decrement — would churn the mirror DB every 30 seconds per lit light source in the game. The lantern-exhaust path is routed through `extinguish()` so the final `is_lit=False, fuel=0` snapshot does land in mirror state, and consumable torches delete (which already wipes metadata via `NFTService._reset_token_identity`).
- Container contents (`current_contents_weight`) — ephemeral / derived.
- Consumables (potions, scrolls, recipes) — strictly delete-on-use, no mid-life state.
- Immutable craft-time config (`base_damage`, `material`, `ship_tier`, etc.) — set at mint and never changed.

#### Adding a new persistence site

1. Identify an attribute that is (a) mutable in-game, (b) meaningful to a buyer looking at the NFT on a marketplace, and (c) meaningful to restore on re-import. If any of those three is false, don't persist.
2. If the value is JSON-native and the AttributeProperty name is fine to reuse, just call `self.persist_metadata({"field": value})` from the write site.
3. If the value needs flattening, pick a distinct flat key (e.g. `foo_dbref`) so the generic copy loop in `spawn_into` doesn't silently overwrite the live attribute. Add an `at_restore_from_metadata(metadata)` method on the typeclass to rehydrate.
4. Guard non-NFT users of a shared mixin (e.g. doors/chests using `DurabilityMixin`) via `getattr(self, "persist_metadata", None)` — the helper is only present on typeclasses that compose `NFTMirrorMixin`.
5. Add tests: one for the persist-on-change path (mock `NFTService.update_metadata`), one for the restore path (simulate the `spawn_into` metadata copy + call the restore hook).

#### Out of scope

- On-chain `NFTokenModify` — not needed while the URI remains a pointer to the HTTP API. A future phase could re-mint with `tfMutable` and have `update_metadata` also emit an `NFTokenModify` for tokens that support it, without changing any callsite.
- A dedicated `NFTMetadataLog` audit table — metadata edits are not transfers and do not currently need a separate audit trail. Add one if operational demand appears; don't overload `NFTTransferLog`.

#### Files

| File | Purpose |
|---|---|
| `blockchain/xrpl/services/nft.py` | `NFTService.update_metadata(token_id, patch)` — atomic JSON merge |
| `typeclasses/mixins/nft_mirror.py` | `NFTMirrorMixin.persist_metadata(patch)` write helper; `spawn_into()` fires `at_restore_from_metadata` hook |
| `typeclasses/mixins/owned_world_object.py` | `_persist_location_metadata()` — flattens room to dbref + key |
| `typeclasses/items/untakeables/ship_nft_item.py` | `ShipNFTItem.at_restore_from_metadata` — dbref → room |
| `typeclasses/mixins/durability.py` | `_persist_durability()` — guards non-NFT consumers via `getattr` |
| `typeclasses/mixins/light_source.py` | `_persist_light_state()` — called from light/extinguish/refuel |
| `typeclasses/scripts/light_burn.py` | Lantern exhaust routed through `obj.extinguish()` for consistent final snapshot |
| `typeclasses/items/maps/district_map_nft_item.py` | `record_survey_points()`; `at_restore_from_metadata` — list → set |
| `nft_api/app/metadata.py` | XLS-24d builder; exposes every key in `NFTGameState.metadata` as an `attributes` entry |

### Item Subclass Hierarchy — NFT Items (Player Economy)

```
BaseNFTItem (typeclasses/items/base_nft_item.py)
│   Composes: NFTMirrorMixin, HeightAwareMixin, HiddenObjectMixin, ItemRestrictionMixin
│
├── WearableNFTItem          — armor/clothing/jewelry
│   │   Composes: DurabilityMixin, WearableMixin
│   ├── WeaponNFTItem        — weapon base (at_wield bridge)
│   │   │   Composes: WeaponMechanicsMixin
│   │   ├── LongswordNFTItem(LongswordMixin, WeaponNFTItem)
│   │   ├── DaggerNFTItem(DaggerMixin, WeaponNFTItem)
│   │   ├── ... (one WeaponNFTItem subclass per weapon category, each composed with its identity mixin)
│   │   └── UnarmedWeapon    — pure Python singleton (NOT a DB object)
│   └── HoldableNFTItem      — shields/torches/orbs (at_hold)
├── ConsumableNFTItem        — single-use items (consume → delete → RESERVE)
│   ├── CraftingRecipeNFTItem — teaches recipe on consume
│   ├── PotionNFTItem         — mastery-scaled effects, anti-stacking
│   └── SpellScrollNFTItem    — transcribe command
├── ContainerNFTItem         — backpack/panniers (nested inventory)
└── ComponentNFTItem         — non-weapon crafting inputs
```

All NFT items inherit `ItemRestrictionMixin` via `BaseNFTItem` — any item can have usage restrictions set in its prototype.

### Item Subclass Hierarchy — Mob Items (Non-NFT, Deleted on Death)

Mirrors the NFT hierarchy but without blockchain tracking, durability, height awareness, hidden/invisible mechanics, or item restriction gates. Mob items are ephemeral — created when the mob spawns, deleted when it dies. They never enter the player economy or transfer to corpses.

```
MobItem (typeclasses/items/mob_items/mob_item.py)
│   Provides: weight, reduce_durability() no-op stub
│
├── MobWearable              — mob armour/clothing
│   │   Composes: WearableMixin
│   └── MobWeapon            — mob weapons
│       │   Composes: WeaponMechanicsMixin
│       ├── MobDagger(DaggerMixin, MobWeapon)
│       ├── MobLongsword(LongswordMixin, MobWeapon)
│       ├── ... (one-liner per weapon type, all mechanics from shared mixin)
│       └── MobHoldable      — mob shields/torches (future)
└── MobConsumable            — mob scrolls/potions (future)
```

**Agreed strips (mob items do NOT get):**
- **HeightAwareMixin** — mob's own height handles vertical position
- **HiddenObjectMixin** — mob hides with its gear; once visible, gear is visible
- **ItemRestrictionMixin** — builders enforce restrictions at build time
- **DurabilityMixin** — mob items are ephemeral, `reduce_durability()` is a no-op stub (combat calls it on every hit/parry)

**Corpse loot filtering** uses simple class-based check: `isinstance(obj, MobItem)` → delete. Everything else (NFT loot placed by spawn system, fungibles) transfers to the corpse.

### Container Cascading

When a container moves to a new owner, all contents cascade their ownership transition. Both NFT items and fungibles inside the container are transitioned via the appropriate service calls.

## Operation Summary

| Operation | Weight Impact | Stat Impact | Blockchain |
|-----------|--------------|-------------|------------|
| Pick up item | +weight (nuclear recalc) | None | NFTService.pickup() |
| Drop item | -weight (nuclear recalc) | None | NFTService.drop() |
| Equip item | None (stays in contents) | Nuclear stat recalc | None |
| Unequip item | None (stays in contents) | Nuclear stat recalc | None |
| Gain gold | +fungible weight (nuclear) | None | GoldService |
| Spend gold | -fungible weight (nuclear) | None | GoldService |
| Item into backpack | Net zero (item leaves direct, enters via container) | None | None (same wallet) |
| Delete item | Stale until next event/restart | at_remove if unequipped first | NFTService cleanup |

## Key Files

| File | Purpose |
|------|---------|
| `typeclasses/mixins/carrying_capacity.py` | CarryingCapacityMixin — weight tracking, nuclear recalculate |
| `typeclasses/mixins/wearslots/base_wearslots.py` | BaseWearslotsMixin — equip/unequip core logic |
| `typeclasses/mixins/wearslots/humanoid_wearslots.py` | HumanoidWearslotsMixin — 19 humanoid slots |
| `typeclasses/mixins/fungible_inventory.py` | FungibleInventoryMixin — gold/resource management |
| `typeclasses/mixins/container.py` | ContainerMixin — nested inventory, weight propagation |
| `typeclasses/mixins/effects_manager.py` | EffectsManagerMixin — conditions, named effects |
| `typeclasses/actors/base_actor.py` | BaseActor — _recalculate_stats(), conditions |
| `typeclasses/mixins/nft_mirror.py` | NFTMirrorMixin — NFT mirror state machine (extracted from BaseNFTItem) |
| `typeclasses/mixins/nft_pet_mirror.py` | NFTPetMirrorMixin — pet-specific mirror dispatch (owner_key based, inherits NFTMirrorMixin) |
| `typeclasses/mixins/owned_world_object.py` | OwnedWorldObjectMixin — world-anchored item patterns (ships, property — NOT pets) |
| `typeclasses/items/base_nft_item.py` | BaseNFTItem — composes NFTMirrorMixin + item-specific concerns |
| `typeclasses/items/wearables/wearable_nft_item.py` | WearableNFTItem — at_wear/at_remove hooks |
| `enums/wearslot.py` | HumanoidWearSlot, DogWearSlot enums |
| `enums/condition.py` | Condition flags (fly, water_breathing, etc.) |
| `utils/item_parse.py` | Shared parser for get/drop/give/put commands — supports dot syntax (`5.wheat`, `all.gold`) alongside space syntax (`5 wheat`, `all gold`) |
