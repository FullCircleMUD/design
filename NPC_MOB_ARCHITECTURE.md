# NPC & Mob Architecture — Class Hierarchy & Composition

> For combat mechanics, see [COMBAT_SYSTEM.md](COMBAT_SYSTEM.md). For mob spawning, see [SPAWN_MOBS.md](SPAWN_MOBS.md). For NPC services and quests, see [NPC_QUEST_SYSTEM.md](NPC_QUEST_SYSTEM.md). For vertical movement, see [VERTICAL_MOVEMENT.md](VERTICAL_MOVEMENT.md). For combat AI memory and the strategy bot system, see [COMBAT_AI_MEMORY.md](COMBAT_AI_MEMORY.md). For embedded world knowledge, see [LORE_MEMORY.md](LORE_MEMORY.md).

## Overview

Every non-player entity in the game — from shopkeepers to dragons — inherits from `BaseNPC`. Capabilities are composed in via mixins: combat, aggression, flight, swimming, equipment, AI. This means any NPC *can* be made fightable, flyable, or equipment-bearing — the architecture doesn't force a hard split between "service NPCs" and "combat mobs." In practice, most service NPCs don't compose in `CombatMixin` (making them effectively immortal), and most fightable entities use the `CombatMob` convenience class. But the boundary is permeable by design.

---

## Architecture — Composition over Inheritance

`BaseNPC` is the foundation. Everything else is a mixin composed in when needed:

```
BaseActor
│   Foundation mixins (on BaseActor itself):
│    + HeightAwareMixin               ← room_vertical_position
│    + EffectsManagerMixin            ← conditions, stat effects, named effects
│    + DamageResistanceMixin          ← resistance/vulnerability tracking
│
│   Auto-init: BaseActor.at_object_creation() detects and initialises
│   composed mixins via hasattr checks. Any mixin with an init method
│   (at_fungible_init, at_wearslots_init, at_carrying_capacity_init,
│   at_recipe_book_init, at_spellbook_init) is auto-called if present.
│   All init methods are idempotent. No explicit calls needed in subclasses.
│
│   Capability mixins (compose into any BaseActor subclass):
│    + CombatMixin                    ← combat commands (attack, flee), handler integration
│    + AggressiveMixin                ← attack on sight, mid-combat height matching
│    + InnateRangedMixin              ← innate ranged attack (breath, bolts, venom)
│    + FlyingMixin                    ← innate flight, preferred_height
│    + SwimmingMixin                  ← innate swimming, preferred_depth
│    + HumanoidWearslotsMixin         ← humanoid wearslots, full weapon pipeline
│    + FungibleInventoryMixin         ← gold/resource inventory (on CombatMob, not BaseNPC)
│    + StateMachineAIMixin            ← tick-driven AI state machine
│    + LLMAIMixin (future)            ← LLM tactical decisions
│    + AdaptiveLLMAIMixin (future)    ← LLM + embedding memory
│
├── FCMCharacter(CombatMixin, HumanoidWearslotsMixin, ..., BaseActor)
│       ← player character, gets combat from CombatMixin
│       ← die() override: player death (corpse, purgatory, XP penalty)
│
└── BaseNPC(BaseActor)                   ← is_pc=False, level, _EmptyNPCCmdSet
    │                                      (no FungibleInventoryMixin — service NPCs
    │                                       don't hold gold/resources directly)
    │
    ├── Service NPCs (no CombatMixin — effectively immortal)
    │   ├── TrainerNPC(BaseNPC)
    │   ├── GuildmasterNPC(QuestGiverMixin, BaseNPC)
    │   ├── ShopkeeperNPC(BaseNPC)
    │   ├── LLMRoleplayNPC(LLMMixin, BaseNPC)
    │   │   ├── BartenderNPC
    │   │   └── QuestGivingShopkeeper
    │   │       └── BakerNPC
    │   └── QuestGivingLLMTrainer
    │       └── OakwrightNPC
    │
    ├── CombatMob(CombatMixin, StateMachineAIMixin, FungibleInventoryMixin, FollowableMixin, BaseNPC)  ← convenience class for common mobs
    │   ├── die() override: mob death (corpse, loot, XP, delete/respawn)
    │   └── Composed further with AI, aggression, flight, swimming, body type
    │
    └── LLMCombatMob(LLMMixin, CombatMob)         ← combat mob with LLM-driven dialogue
        ├── Townfolk(LLMCombatMob)                                     ← anonymous townspeople
        ├── GuardSergeant(BashAbility, WeaponMasteryMixin, HumanoidWearslotsMixin, LLMCombatMob)
        ├── MeleeGuard(BashAbility, WeaponMasteryMixin, MobFollowableMixin, HumanoidWearslotsMixin, LLMCombatMob)
        ├── RangedGuard(BashAbility, WeaponMasteryMixin, MobFollowableMixin, HumanoidWearslotsMixin, LLMCombatMob)
        └── CityWatch(MeleeGuard)
```

**Hybrid combat+dialogue pattern:** Mobs that can both fight and hold a real conversation compose `LLMMixin` *into* `CombatMob` via the `LLMCombatMob` convenience class. The dialogue layer is `LLMMixin`; combat behaviour stays on `CombatMob`. This is the inverse of "compose CombatMixin into a service NPC" — in practice, the combat side is the heavier composition, so it sits at the bottom of the MRO and the LLM dialogue rides on top.

### NPC vs Mob — The Distinction

With composable combat, the NPC/mob distinction is no longer about who can fight. It's about **interaction breadth**:

- **NPC** = entity you can interact with beyond combat — dialogue, trade, training, quests. May or may not be fightable (with `CombatMixin`).
- **Mob** = entity that only fights you. No services, no dialogue beyond combat barks. Always has `CombatMixin`.

A city guard is an **NPC** — you can talk to them, they have dialogue, they happen to also be fightable. A wolf is a **mob** — all you can do is fight it.

**Why composable combat?** A city guard is an LLM mob who can talk, de-escalate, *and* fight. These aren't mobs pretending to be NPCs or NPCs pretending to be mobs — they're combat mobs with LLM dialogue composed in via `LLMCombatMob`. `LLMMixin` drives dialogue; the underlying `CombatMob` machinery handles what happens when swords come out.

**The common case stays simple:** the typical fightable entity is a wolf, kobold, or rat — they extend `CombatMob` (via `AggressiveMob`) and never think about the architecture. The composability is there for the cases that need it.

---

## Composition Axes

Every NPC capability is an independent axis composed via mixins. The same mixin set works for common mobs, boss encounters, and hybrid combat-dialogue NPCs.

| Axis | Options | Mechanism |
|------|---------|-----------|
| **Combat** | Can participate in combat (commands, handler) | `CombatMixin` presence |
| **AI** | State machine vs LLM vs Adaptive LLM | AI mixin choice |
| **Body type** | Animal (`damage_dice`) vs Humanoid (wearslots + weapons) | `HumanoidWearslotsMixin` presence |
| **Aggression** | Aggressive (attacks on sight) vs Passive (fights back when attacked) | `AggressiveMixin` presence |
| **Behavior** | Pack courage, rampage, tactical dodge, etc. | Mob behavior mixins (`mob_behaviours/`) |
| **Innate ranged** | Melee-only vs Innate ranged attack | `InnateRangedMixin` presence |
| **Flight** | Grounded vs Flying | `FlyingMixin` presence |
| **Swimming** | Land-only vs Aquatic | `SwimmingMixin` presence |

### FungibleInventoryMixin Placement

`FungibleInventoryMixin` (gold/resource inventory) is on `CombatMob`, NOT on `BaseNPC`. Only combat mobs need it — for loot that transfers to corpses on death. Service NPCs transact via AMMs, training mechanisms, and quest rewards, never through direct inventory. Future pickpocketable NPCs can compose `FungibleInventoryMixin` individually.

### Composition Examples

```
Animal mobs (no wearslots, use damage_dice):

Wolf(AggressiveMob)                                                                       ← aggressive animal
Rabbit(CombatMob)                                                                         ← passive animal, scripted flee
Crow(FlyingMixin, PackCourageMixin, AggressiveMob)                                        ← flying pack predator, needs 3+ to attack
DireWolf(TacticalDodgeMixin, AggressiveMob)                                               ← 25% dodge per combat tick
CellarRat(AggressiveMob)                                                                  ← simple aggressive animal

Humanoid mobs (HumanoidWearslotsMixin + WeaponMasteryMixin, equip MobWeapons/MobWearables):

Kobold(PackCourageMixin, WeaponMasteryMixin, HumanoidWearslotsMixin, AggressiveMob)       ← pack fighter with weapon mastery
KoboldChieftain(...)                                                                       ← boss variant
Gnoll(RampageMixin, WeaponMasteryMixin, HumanoidWearslotsMixin, AggressiveMob)            ← on-kill chain attacker with weapon mastery
GnollArcher(Gnoll)                                                                         ← Gnoll subclass equipped with a bow
GnollWarlord(...)                                                                          ← boss variant of Gnoll
Skeleton(HumanoidWearslotsMixin, AggressiveMob)                                            ← undead with wearslots

LLM-driven hybrid combat mobs (LLMMixin composed into CombatMob via LLMCombatMob):

Townfolk(LLMCombatMob)                                                                                              ← anonymous townspeople
GuardSergeant(BashAbility, WeaponMasteryMixin, HumanoidWearslotsMixin, LLMCombatMob)                                ← squad leader
MeleeGuard(BashAbility, WeaponMasteryMixin, MobFollowableMixin, HumanoidWearslotsMixin, LLMCombatMob)               ← auto-follows sergeant
RangedGuard(BashAbility, WeaponMasteryMixin, MobFollowableMixin, HumanoidWearslotsMixin, LLMCombatMob)              ← auto-follows sergeant, ranged
CityWatch(MeleeGuard)                                                                                               ← city-specific guard variant

Future examples:

Dragon(AggressiveMixin, FlyingMixin, InnateRangedMixin, AdaptiveLLMAIMixin, CombatMob)    ← boss, flies, breathes fire, learns
BanditChief(AggressiveMixin, HumanoidWearslotsMixin, LLMAIMixin, CombatMob)               ← humanoid, LLM tactics
```

---

## Mob Ability System — Command-Based & Event-Based

Mobs can use the same combat skill commands as players (bash, stab, dodge, etc.) without needing a character class. Abilities are mixins — compose them onto any mob.

### Two Ability Types

| Type | Mechanism | AI Selection | Examples |
|---|---|---|---|
| **Command-based** | Merges cmdset, registers in `db.combat_commands`, sets mastery | AI selects via weighted random or LLM decision | BashAbility, PummelAbility, DodgeAbility, StabAbility |
| **Event-based** | Fires from hooks (at_kill, at_combat_tick, at_new_arrival) | Automatic — not AI-selected | RampageMixin, PackCourageMixin, TacticalDodgeMixin |

### Command-Based Ability Hierarchy

```
MobSkillAbility                    ← base: cmdset merge, AI registry, mastery write
├── MobClassSkillAbility           ← writes to db.class_skill_mastery_levels
│   ├── BashAbility, PummelAbility, StabAbility, TauntAbility, ProtectAbility
└── MobGeneralSkillAbility         ← writes to db.general_skill_mastery_levels
    ├── DodgeAbility

WeaponMasteryMixin                 ← data-only: writes to db.weapon_skill_mastery_levels
```

**Files:** `typeclasses/mixins/mob_abilities/mob_skill_ability.py` (base classes), `combat_abilities.py` (concrete abilities), `weapon_mastery.py` (weapon mastery data).

### How It Works

1. Each ability mixin merges a tiny cmdset containing the player command (e.g. `CmdBash`) onto the mob
2. Registers the command key + weight in `db.combat_commands` for AI selection
3. Sets mastery in the correct dict so `CmdSkillBase` dispatches at the right mastery tier
4. `CmdSkillBase.func()` already handles mobs with mastery — same dispatch as players, zero changes needed

### AI Integration

The state machine AI reads `db.combat_commands` on each combat tick:
- Normal attack chance (e.g. 60%)
- Weighted random command selection from registry (e.g. 40%)
- If selected command is on cooldown or blocked, action fails gracefully (same as player)

Future LLM AI receives the registry as context and makes tactical decisions.

### Composition Examples

```python
# Kobold warrior — bashes, uses dagger at SKILLED
class KoboldWarrior(BashAbility, WeaponMasteryMixin, PackCourageMixin,
                    HumanoidWearslotsMixin, AggressiveMob):
    default_weapon_masteries = {"dagger": 2}

# Gnoll berserker — bashes + pummels, longsword at EXPERT
class GnollBerserker(BashAbility, PummelAbility, WeaponMasteryMixin,
                     RampageMixin, HumanoidWearslotsMixin, AggressiveMob):
    default_weapon_masteries = {"long_sword": 3}
```

### Spawn JSON Override

Mastery defaults from mixins can be overridden per spawn rule:
```json
{
    "typeclass": "typeclasses.actors.mobs.kobold.KoboldWarrior",
    "attrs": {
        "class_skill_mastery_levels": {"bash": 3},
        "weapon_skill_mastery_levels": {"dagger": 3}
    }
}
```

---

## Mob Equipment — MobItem Hierarchy

Humanoid mobs can equip weapons and armour that share identical combat mechanics with player NFT items, without consuming NFT tokens or entering the player economy. This is achieved through extracted composition mixins (see `design/INVENTORY_EQUIPMENT.md` § Extracted Composition Mixins).

### Design Principle

Mirror the player/NFT item hierarchy for mobs. Wherever the NFT hierarchy uses inheritance instead of composition, refactor to composition so both paths share the same mechanics. Strip only what's explicitly unnecessary for mobs.

### MobItem Hierarchy

```
MobItem(DefaultObject)                              ← base for all mob-only items
├── MobWearable(WearableMixin, MobItem)              ← mob armour, clothing
│   ├── MobWeapon(WeaponMechanicsMixin, MobWearable) ← mob weapons
│   └── MobHoldable(MobWearable)                     ← mob shields, torches (future)
└── MobConsumable(MobItem)                           ← mob scrolls, potions (future)
```

Concrete mob weapons compose a weapon identity mixin with `MobWeapon`:

```python
class MobDagger(DaggerMixin, MobWeapon):
    pass  # all mechanics inherited from DaggerMixin
```

The `DaggerMixin` is the same class used by `DaggerNFTItem` — single source of truth for dagger combat mechanics. Change the dagger's crit table and both player and mob versions get the update.

### What Mob Items Strip (vs NFT Items)

| Feature | NFT Items | Mob Items | Reason |
|---|---|---|---|
| **HeightAwareMixin** | Yes | No | Mob's own height handles vertical position |
| **HiddenObjectMixin** | Yes | No | Mob hides with its gear; once visible, gear is visible |
| **ItemRestrictionMixin** | Yes | No | Builders enforce restrictions at build time |
| **DurabilityMixin** | Yes | No | Mob items are ephemeral; `reduce_durability()` is a no-op stub |
| **NFT tracking** | Yes | No | No `token_id`, no NFTService calls, no blockchain state |

### Corpse Loot Filtering

On mob death, `_create_corpse()` uses a simple class-based check:

```python
for obj in list(self.contents):
    if isinstance(obj, MobItem):
        obj.delete()       # mob equipment — always destroyed
    else:
        obj.move_to(corpse) # NFT loot + fungibles → corpse
```

No tags needed. MobItem instances are destroyed; NFT items placed by the spawn system and fungibles (gold, resources) transfer to the corpse for players to loot.

### Spawning Mob Equipment

Mob weapons are created from the same prototype data as NFT weapons via `MobItem.spawn_mob_item()`. Each prototype dict defines a `mob_typeclass` field alongside the NFT `typeclass`:

```python
IRON_DAGGER = {
    "typeclass": "typeclasses.items.weapons.dagger_nft_item.DaggerNFTItem",
    "mob_typeclass": "typeclasses.items.mob_items.mob_weapons.MobDagger",
    "base_damage": "d4", "material": "iron", ...
}
```

Usage in mob creation or spawn scripts:

```python
from typeclasses.items.mob_items.mob_item import MobItem

weapon = MobItem.spawn_mob_item("iron_dagger", location=self)
self.wear(weapon)
```

`spawn_mob_item()` reads the prototype, swaps `typeclass` for `mob_typeclass`, strips NFT-only fields (max_durability, excluded_classes), and creates the object with all stat fields applied. The result is a `MobDagger` with identical `base_damage`, `material`, `speed`, and `weight` to the NFT version — single source of truth.

### Animal vs Humanoid Mobs

| Mob Type | Damage Source | Hit Messages | Equipment |
|---|---|---|---|
| **Animal** (wolf, rat, crow) | `damage_dice` attribute, raw dice roll | `damage_type` + `get_descriptor()` — severity-scaled verbs | None — no wearslots |
| **Humanoid** (kobold, gnoll, bandit) | Equipped `MobWeapon` via `get_weapon()` | Weapon's `damage_type` + `get_descriptor()` | Full wearslots, weapon mastery hooks |

Both animal and humanoid mobs use the same damage descriptor system for hit messages — `get_descriptor(damage_type, damage_dealt, target_hp_max)` selects a verb whose capitalization conveys severity (lowercase → Title Case → ALL CAPS). Animal mobs set `damage_type` on the class (e.g. wolf → PIERCING, jagular → SLASHING); humanoid mobs inherit it from their equipped weapon. Miss messages use the mob's `attack_message` attribute instead (e.g. "bites at you but misses").

---

## Mob Group Combat — FollowableMixin + MobFollowableMixin

Mobs use the same follow/group system as player characters for group combat. When any mob in a group is attacked, `enter_combat()` pulls in the entire group — identical to how player groups work.

### Follow System on Mobs

`FollowableMixin` is composed into `CombatMob`, giving all combat mobs the follow/group infrastructure. A `followable` tag (category `system`) is set via `at_followable_init()` for efficient DB queries in `get_followers()`.

```
FollowableMixin                    ← base: following, nofollow, get_group_leader, get_followers
├── (used directly by FCMCharacter — players use `follow` command)
├── (composed into CombatMob — all mobs can participate in groups)
└── MobFollowableMixin             ← extends with auto-reacquire logic
    └── (used by squad member mobs — guards, future pack mobs, escorts)
```

### MobFollowableMixin — Auto-Reacquire Pattern

`MobFollowableMixin` (`typeclasses/mixins/mob_behaviours/mob_followable_mixin.py`) extends `FollowableMixin` with AI-driven leader reacquisition. On each AI idle tick, if `following is None`, the mob scans the room for `squad_leader_typeclass` and follows them.

This handles all group reformation cases:
- **Staggered respawn** — guards spawn before or after sergeant, group forms as each arrives
- **Partial kills** — surviving guards stay grouped, new spawns join on their first tick
- **Leader death and respawn** — guards find the new sergeant automatically

The check is trivial (one `getattr` per tick when already following) — negligible overhead even across hundreds of mobs.

### Example: Town Guard Squad

```python
class GuardSergeant(BashAbility, WeaponMasteryMixin, HumanoidWearslotsMixin, CombatMob):
    # Leader — gets FollowableMixin from CombatMob, no auto-reacquire needed

class MeleeGuard(BashAbility, WeaponMasteryMixin, MobFollowableMixin, HumanoidWearslotsMixin, CombatMob):
    squad_leader_typeclass = GuardSergeant
    # Auto-follows sergeant on each idle tick if not already following
```

Player attacks any guard → `enter_combat()` pulls in target's group → all guards + sergeant enter combat.

---

## CombatMixin — Unified Combat for All Actors

`CombatMixin` is the single combat capability for **all combatants** — players, mobs, and hybrid NPCs alike. It provides the combat mechanism: commands, handler integration, and the shared combat path. Composed into `FCMCharacter`, `CombatMob`, and any NPC that needs to fight.

**Provides:**
- `CombatCmdSet` injection — `CmdAttack`, `CmdFlee`, `CmdHold`, and other combat commands
- `CombatHandler` integration — enter/exit combat hooks
- Shared combat methods used by the `execute_attack()` pipeline

**Does NOT provide (separate concerns):**
- Death behavior — polymorphic via `die()` overrides (see below)
- Mob-specific attributes — `damage_dice`, loot tables, respawn logic stay on `CombatMob`
- Stats — HP, AC, ability scores stay on `BaseActor`
- Equipment — wearslots stay on `HumanoidWearslotsMixin`

**Where it's composed in:**

```python
# Player character — gets combat handler accessors from CombatMixin
FCMCharacter(CombatMixin, CarryingCapacityMixin, ..., BaseActor)

# Common mobs — CombatMixin for combat, StateMachineAIMixin for AI, FollowableMixin for groups, BaseNPC for NPC base
CombatMob(CombatMixin, StateMachineAIMixin, FungibleInventoryMixin, FollowableMixin, BaseNPC)

# Hybrid combat+dialogue — LLMMixin composed into CombatMob
LLMCombatMob(LLMMixin, CombatMob)
GuardSergeant(BashAbility, WeaponMasteryMixin, HumanoidWearslotsMixin, LLMCombatMob)
```

**Actual implementation:**
- `get_combat_handler()` — safe accessor (`scripts.get("combat_handler")`)
- `is_in_combat` — @property
- `hp_fraction` — HP/effective_hp_max ratio
- `is_low_health` — below `aggro_hp_threshold`
- `enter_combat(target)` — wraps `combat_utils.enter_combat()`
- `exit_combat()` — stops combat handler
- `initiate_attack(target)` — programmatic attack via `execute_cmd("attack ...")` (compat alias: `mob_attack`)
- `at_object_creation()` — adds `CmdSetMobCombat` + `call:false()` lock for non-PCs

Players get `CmdAttack`/`CmdFlee` via `CmdSetCharacterCustom`. Mobs use `initiate_attack()` → `execute_cmd("attack ...")`. Both route through the same `execute_attack()` pipeline and `CombatHandler` script.

### NPC Command Visibility and `call:true()`

Evennia's `call:true()` lock is binary per object — ALL CmdSets on an NPC are either visible or invisible to nearby players. You cannot selectively hide individual CmdSets. This means command-based service NPCs (trainer, shopkeeper) that use `call:true()` would leak combat commands to players if CombatMixin is composed in.

**This is a non-issue for the cases that matter:**

- **Players** — combat commands are in their own CmdSet. No leaking concern.
- **Mobs** — `execute_cmd()` resolves against the mob's own CmdSet. `call:true()` is not involved.
- **LLM hybrid NPCs** (city guard, drunk) — interaction is via speech, not commands. These NPCs can use `call:false()` because players talk to them, not type commands at them. Combat commands stay internal.
- **Command-based service NPCs** (trainer, shopkeeper) — the rare edge case. If one needs CombatMixin, the duplicate commands are harmless in practice (player's version wins the merge).

**Future direction — LLM-driven NPC interaction:** The architectural north star is to abstract ALL NPC interaction behind conversation. Instead of `quote buy 3 bread`, players say "how much for 3 bread?" and the LLM interprets intent, runs `execute_cmd()` internally, and responds naturally. This moves every NPC to `call:false()` — commands exist only for the NPC's internal use. Combat commands, service commands, everything stays internal. The `call:true()` leaking concern disappears entirely. See `ops/PLANNING` AI NPC System for details.

### Death Behavior — One System, Polymorphic

Death is not a combat concept — entities can die from drowning, fall damage, hunger, or any non-combat cause. `BaseActor.die()` is the single entry point for all death, overridden at each level that needs different behavior:

| Entity | `die()` override | Behavior |
|--------|-----------------|----------|
| **BaseActor** | Base stub | Sets HP to 0 — foundation for all death |
| **BaseNPC** (no CombatMixin) | Immortal guard | Resets HP to max, returns. NPC shrugs off damage |
| **CombatMob** | Mob death | Stops AI → creates corpse with loot → awards XP → deletes or respawns |
| **FCMCharacter** | Player death | Creates corpse with all items → XP penalty → purgatory timer → respawn at home |
| **Hybrid NPC** | Custom | Depends on the class — could match CombatMob behavior or define its own |

This is already one unified system — `take_damage()` on BaseActor calls `die()` when HP reaches 0, and Python's MRO dispatches to the correct override. CombatMixin itself does not handle death. Death behavior belongs to the concrete class.

### CombatMob — Convenience Class

```python
class CombatMob(CombatMixin, StateMachineAIMixin, FungibleInventoryMixin, FollowableMixin, BaseNPC):
    """Convenience class for common fightable mobs — wolves, rats, kobolds, etc."""
```

Adds mob-specific attributes on top of CombatMixin's combat capability and StateMachineAIMixin's AI:
- `damage_dice` — innate damage for non-humanoid mobs (e.g. `"1d4"`, `"2d6"`)
- `damage_type` — `DamageType` enum (default `BLUDGEONING`). Used by `get_descriptor()` to select severity-scaled hit verbs and by `calculate_damage()` for resistance checks.
- `attack_message` — used for miss messages only (e.g. "bites at you but misses"). Hit messages use the damage descriptor system.
- `initiate_attack(target)` — inherited from CombatMixin (`mob_attack` kept as compat alias)
- `_roll_damage()`, `_create_corpse()`
- Loot: `loot_resources`, `loot_gold_max`, `spawn_scrolls_max`, `spawn_recipes_max`
- HP condition text (`get_condition_text()`)
- Area-restricted wandering via `area_tag` tags
- `max_per_room` anti-stacking
- `die()` override — mob death (corpse, loot, XP, delete/respawn)

This is what most mobs use. All existing mob subclasses inherit from `CombatMob` unchanged — backward compatible.

**Passive mobs** (no `AggressiveMixin`) only fight when a player uses `attack` on them — `enter_combat()` creates handlers on both sides. Their AI can still include custom behaviors (Rabbit flees threats).

---

## Mob AI — Three Tiers (Composable)

All mobs use the same command interface as players — `execute_cmd("attack player")`, `execute_cmd("flee")`, etc. This is the critical design decision: the **decision-maker** is swappable via mixin, but the **action execution path** is always identical to what a player would type.

AI is a composable axis — each mob picks one AI mixin. All three share the same `execute_cmd()` interface, so weapons, mastery, height, combat hooks all work automatically regardless of which AI drives the mob.

### StateMachineAIMixin

The current system (refactored from the existing `AIMixin`). A tick-driven state machine (`AIHandler`) that fires every `ai_tick_interval` seconds and dispatches to state-specific methods (`ai_wander`, `ai_retreating`, etc.).

**Used for:** Simple/commodity mobs — wolves, kobolds, rats, crows. Predictable behavior patterns defined in code. Fast, zero latency, zero cost.

**How it works:**
- `TICKER_HANDLER` fires `ai_tick()` every N seconds
- `AIHandler` dispatches to current state method
- State methods call `execute_cmd()` to perform actions
- Special behaviors composed in via **mob behavior mixins** (`mob_behaviours/`): PackCourageMixin (Kobold, Crow), TacticalDodgeMixin (DireWolf), RampageMixin (Gnoll)

### LLMAIMixin (Future)

An LLM agent that reasons about the situation and generates command strings. The mob's available actions, current context (room, enemies, allies, HP, conditions), and combat state feed into the LLM prompt. The LLM outputs commands like `"attack mage"`, `"flee"`, `"bash tank"`.

**Used for:** Mid-tier named NPCs, mini-bosses, mobs that need tactical flexibility but don't need long-term memory. Smarter than scripted AI but without the overhead of persistent memory.

**Latency consideration:** Combat decisions need to be fast. The state machine AI can serve as a fallback when LLM latency is too high — e.g. use scripted AI for the first tick while the LLM response arrives.

### AdaptiveLLMAIMixin (Future)

Extends `LLMAIMixin` with embedding-based persistent memory. The full vision for boss and important mobs.

> **Full design:** See [COMBAT_AI_MEMORY.md](COMBAT_AI_MEMORY.md) for the complete combat memory and strategy bot architecture — three-layer system (persistence, pre-fight planning, in-fight execution), early warning pattern, unified briefings, memory scope, and ML training data.

**Used for:** Named bosses, important encounters, key story mobs. Creates emergent gameplay through adaptive behavior.

**Three-layer architecture:**
1. **Combat Memory** (`CombatMemory` table in `ai_memory`) — structured encounter summaries with embeddings, recorded after every fight. Shared per mob type or per named instance.
2. **Strategy Bot** — fires when players approach (not at combat start), queries both combat memory and interaction memory, synthesises a tactical plan via a single LLM call. Latency absorbed by player traversal time.
3. **Execution** — state machine reads the tactical plan from `ndb` and follows instructions each tick. Zero LLM calls per tick.

**Adaptive learning via embeddings:** The infrastructure is ready — `ai_memory` uses pgvector with HNSW-indexed `VectorField(1536)` on PostgreSQL for sub-linear semantic search, falling back to numpy on SQLite for local dev (see `DATABASE.md § pgvector for AI Memory`). The `CombatMemory` model uses the same dual-backend pattern.

**Memory scope:** Per-mob-type (all gnolls share learnings), per-instance (this specific dragon remembers), or hybrid (instance memories first, type fallback) — configurable per mob.

**Unified briefing:** For bosses that are also dialogue NPCs, the strategy bot merges interaction history (`NpcMemory`) with combat history (`CombatMemory`) into a single briefing. The boss's disposition emerges from the full relationship — always-hostile players get pre-buffed aggression, peaceful visitors get dialogue. Players can change a boss's behavior through repeated interaction.

**Social memory:** Beyond combat, important mobs remember player interactions — gifts, conversations, repeated visits. A boss might gift a rare item to a player who's built rapport over time rather than forcing every encounter to be combat.

---

## Three Memory Systems

All NPC intelligence draws from three independent embedding-backed memory systems that compose at prompt time. Each answers a different question:

| System | Table | Question it answers | Design doc |
|---|---|---|---|
| **Lore Memory** | `LoreMemory` | "What do I know about the world?" | [LORE_MEMORY.md](LORE_MEMORY.md) |
| **Interaction Memory** | `NpcMemory` | "What do I know about *you*?" | [DATABASE.md](DATABASE.md) § pgvector |
| **Combat Memory** | `CombatMemory` | "What do I know about *fighting* you?" | [COMBAT_AI_MEMORY.md](COMBAT_AI_MEMORY.md) |

All three live in the `ai_memory` database (same router, same pgvector/SQLite dual-backend, same HNSW indexing on PostgreSQL). They are queried independently and composed into a unified context at prompt-build time.

---

## NPC Intelligence Tiers

Every NPC/mob in the game falls into one of four intelligence tiers. The tier determines which memory systems the entity uses, how it makes decisions, and what kind of encounters it creates. Moving an NPC between tiers is a configuration change (attribute toggles + mixin composition), not a class hierarchy change.

### Tier 1: Commodity Mob

**Examples:** Wolf, cellar rat, kobold, crow, rabbit.

| System | Setting |
|---|---|
| Lore | None |
| Interaction memory | None |
| Combat memory | None |
| AI | `StateMachineAIMixin` — hardcoded state machine |
| Strategy bot | None |

Meat for the grinder. They attack (or flee), they die, players level up. Behavior comes from state machine AI and composable behavior mixins (PackCourageMixin, RampageMixin, TacticalDodgeMixin). No LLM calls, no memory, no knowledge. Fast, cheap, predictable.

**Configuration:**
```python
class Wolf(AggressiveMob):
    # StateMachineAIMixin inherited from CombatMob
    # No LLMMixin, no memory flags
```

### Tier 2: Generic Named-Role

**Examples:** "a townsperson", "a town guard", a generic farmer, a street vendor, a stable hand. Possibly animals with `speak with animals` potential.

| System | Setting |
|---|---|
| Lore | Local + regional + continental (via `lore_scope_tags`) |
| Interaction memory | Short-term rolling list only (`llm_use_vector_memory=False`) |
| Combat memory | None (state machine if fightable) |
| AI | `LLMMixin` for dialogue, `StateMachineAIMixin` if fightable |
| Strategy bot | None |

These are interchangeable NPCs without a persistent individual identity. A player can ask "a townsperson" for directions to the blacksmith, and the townsperson should know — they live here. But if the player talks to a different townsperson later, there's no expectation that it's the same person or that they remember the first conversation.

**What the lore system gives them:** Local knowledge appropriate to their role — directions to landmarks, common knowledge about the town, regional geography, continental history. A town guard knows the law and the district layout. A farmer knows the fields and the seasons. No per-NPC authoring needed — the lore scope tags pull the right knowledge dynamically.

**Why short-term memory is enough:** These NPCs are anonymous. The rolling list in `db` attributes gives them conversational continuity within a session ("you just asked me about the inn — it's that way, like I said") but no cross-session persistence. This is correct — "a townsperson" today is not necessarily the same "a townsperson" tomorrow.

**Configuration:**
```python
class Townsperson(LLMRoleplayNPC):
    llm_use_vector_memory = False       # rolling list only
    llm_use_lore = True                 # pull from LoreMemory
    lore_scope_tags = ["millholm", "millholm_town"]  # or set from room tags
```

### Tier 3: Named Individual (Non-Combat)

**Examples:** Rowan (bartender), Bron (baker), Archmage Tindel, Old Silas (beggar), Merchant Harlow, Brother Aldric.

| System | Setting |
|---|---|
| Lore | Local + regional + continental + faction (via `lore_scope_tags`) |
| Interaction memory | Long-term (`NpcMemory` with embeddings, `llm_use_vector_memory=True`) |
| Combat memory | None |
| AI | `LLMMixin` for dialogue |
| Strategy bot | None |

Named individuals with persistent identities. Bron the baker remembers you bought bread last week. Archmage Tindel recalls your interest in ley lines from three conversations ago. The Mages Guild guildmaster knows guild secrets that the bartender doesn't.

**What separates this from Tier 2:** The player has an expectation of continuity. When they return to Bron, Bron should recognise them and remember past interactions. This requires long-term embedding-backed memory (`NpcMemory` with `llm_use_vector_memory=True`).

**What the lore system adds beyond Tier 2:** Faction-scoped knowledge. Archmage Tindel knows Mages Guild secrets (tagged `faction:mages_guild`). Shadow Mistress Vex knows Thieves Guild safe houses (tagged `faction:thieves_guild`). A bartender knows only what any local would know. The same lore base, different scope access.

**Configuration:**
```python
class BartenderNPC(QuestGiverMixin, LLMRoleplayNPC):
    llm_use_vector_memory = True        # long-term NpcMemory
    llm_use_lore = True                 # pull from LoreMemory
    lore_scope_tags = ["millholm", "millholm_town"]

class ArcmageTindel(GuildmasterNPC):
    llm_use_vector_memory = True
    llm_use_lore = True
    lore_scope_tags = ["millholm", "millholm_town", "mages_guild"]
```

### Tier 4: Named Combatant

**Examples:** Gnoll Warlord boss, named dragon, dungeon boss, FightableCityGuard, a warlord you can negotiate with.

| System | Setting |
|---|---|
| Lore | Local + regional + continental + faction/tribal |
| Interaction memory | Long-term (`NpcMemory` with embeddings) |
| Combat memory | Full (`CombatMemory` + strategy bot) |
| AI | `StateMachineAIMixin` (executing tactical plan) + `LLMMixin` (dialogue) |
| Strategy bot | Pre-fight briefing + post-fight logging |

Everything Tier 3 has, plus the combat intelligence systems. These are the richest encounters in the game — a boss that knows the world, remembers you, and has learned how to fight you.

**What separates this from Tier 3:** Two agentic systems that bookend every combat encounter (see below). The pre-encounter strategist synthesises a unified briefing from all three memory systems. The post-encounter recorder logs what happened for future learning.

**Configuration:**
```python
class GnollWarlord(CombatMixin, AggressiveMixin, LLMRoleplayNPC):
    llm_use_vector_memory = True        # long-term NpcMemory
    llm_use_lore = True                 # pull from LoreMemory
    llm_use_combat_memory = True        # CombatMemory + strategy bot
    lore_scope_tags = ["millholm_southern", "gnoll"]
    combat_memory_scope = "per_type"    # all gnoll warlords share learnings
```

### Tier Summary

| | Tier 1 | Tier 2 | Tier 3 | Tier 4 |
|---|---|---|---|---|
| **Identity** | Anonymous | Anonymous role | Named individual | Named individual |
| **Lore** | — | Local/regional/continental | + Faction | + Faction |
| **Interaction memory** | — | Rolling list (short-term) | NpcMemory (long-term) | NpcMemory (long-term) |
| **Combat memory** | — | — | — | CombatMemory + strategy bot |
| **Combat AI** | State machine | State machine (if fightable) | — | State machine executing tactical plan |
| **Dialogue AI** | — | LLMMixin | LLMMixin | LLMMixin |
| **LLM calls per encounter** | 0 | Per-message (dialogue only) | Per-message | Per-message + 1 strategy call |
| **Config flags** | No LLMMixin | `llm_use_lore=True` | + `llm_use_vector_memory=True` | + `llm_use_combat_memory=True` |

---

## Combat Encounter Agents (Tier 4)

Tier 4 NPCs are served by two agentic systems that bookend every combat encounter:

### Agent 1: Pre-Encounter Strategist

Fires when players approach the mob (early warning pattern — see [COMBAT_AI_MEMORY.md](COMBAT_AI_MEMORY.md) § Early Warning Pattern). One LLM call per encounter, using a capable model.

**Input** (queries all three memory systems):

| Source | What it provides |
|---|---|
| `LoreMemory` | World context — territorial knowledge, faction history |
| `NpcMemory` | Relationship history — past conversations, disposition, topics |
| `CombatMemory` | Tactical history — past fights, effective tactics, party comp patterns |
| Current context | Party composition now, levels, visible equipment, HP state |

**Output:** A unified briefing stored on `ndb.tactical_plan`:

*Peaceful history:*
> "This player has visited 3 times, always peaceful. Last conversation: interested in gnoll tribal history. Has never attacked. Recommend: greet, continue discussion. If provoked: warn twice, then engage — target the mage."

*Hostile history:*
> "This party has attacked 4 times. Warrior+cleric composition. Lost 3 of 4. Cleric healing sustained the warrior every time. Best tactic from the 1 win: silence cleric round 1, melee focus. Recommend: pre-cast shield, silence cleric on sight."

The state machine reads this plan each combat tick and follows its priority list. No per-tick LLM calls needed.

### Agent 2: Post-Encounter Recorder

Fires after `stop_combat()`. Asynchronous, non-blocking.

**Input:**
- Combat log: actions taken per round, damage dealt/received, spells cast
- Party composition from `get_sides()`
- Outcome: who won, HP remaining, rounds survived
- Any dialogue that occurred during the encounter

**Actions:**
1. Generates a structured summary of the encounter
2. Embeds the summary and writes to `CombatMemory`
3. If dialogue occurred before or during combat, also writes to `NpcMemory` (the interaction memory) — capturing the relationship context, not just the tactical data

**What it decides:** Not everything is worth remembering in detail. A routine fight against a solo player might get a brief entry. A dramatic fight against a full party with novel tactics gets a rich summary. The recorder agent judges what's relevant — this is where the LLM adds value over a simple template.

### Agents and Tiers

| Agent | Tier 1 | Tier 2 | Tier 3 | Tier 4 |
|---|---|---|---|---|
| Pre-encounter strategist | — | — | — | Full unified briefing |
| Post-encounter recorder | — | — | — | Combat + interaction logging |

Tiers 1–3 do not use these agents. Tier 1 mobs fight and die without recording anything. Tier 2/3 NPCs that happen to be fightable (e.g. a town guard who's attacked) use state machine combat with no strategic memory — they don't learn from past fights.

---

## AggressiveMixin

Adds "attacks on sight" behavior. Extracted from the current `AggressiveMob` class.

**Provides:**
- `is_aggressive_to_players` attribute (default True, can be toggled)
- `at_new_arrival(arriving_obj)` — aggro when a player enters the room
- `_try_reach_and_attack(target)` — height-aware attack initiation
- `_try_match_height(target)` — adjust vertical position to reach target (respects FlyingMixin/SwimmingMixin bounds if present)
- `_schedule_attack(target)` — delayed attack with random interval (`attack_delay_min`/`attack_delay_max`)
- `_execute_attack(target)` — validate target still alive and reachable, then `initiate_attack()`
- `ai_wander()` — scan for players each tick, attack if found, wander if not

**On-aggro target preference:** When multiple players are in the room, prefers targets at reachable height. If no reachable targets exist, attempts height-matching before giving up.

**Mid-combat retargeting (in `combat_handler.execute_next_action()`):** When the current target becomes unreachable (player flies up, dives), the combat handler calls `_try_match_height()` first. If the mob can match (flying mob chases upward, swimming mob dives), it broadcasts a room message and attacks. If it can't match (ground mob vs flying target), it scans enemies via `_find_reachable_target()` for an alternative at the same height. If no reachable enemies exist, the mob flees (auto-succeeds when no melee threat at same height). See `design/COMBAT_SYSTEM.md` § Height Combat for the full per-tick flow.

---

## Mob Behavior Mixins (`mixins/mob_behaviours/`)

Reusable plug-and-play behaviors that compose into any `AggressiveMob` subclass. Each mixin adds a single, well-defined behavioral pattern with configurable attributes for per-mob flavour.

### PackCourageMixin

Pack-fighting behavior — mobs only attack when enough allies of the **same type** are present. Solo mobs flee. Cornered mobs (no valid exit) fight regardless.

**Provides:**
- `min_allies_to_attack` (default 1) — minimum other living mobs of `self.__class__` in room
- `flee_message` (default `"{name} panics and flees!"`) — configurable per mob type
- `_count_allies()` — counts `isinstance(obj, self.__class__)` in room (cross-species don't count)
- `_has_pack_courage()` / `_is_cornered()` — decision helpers
- `at_new_arrival()` override — courage-gated aggro, else flee
- `ai_wander()` override — mid-combat courage check (flee if allies die), courage-gated target scan

**Used by:** Kobold (`min_allies_to_attack=1`), Crow (`min_allies_to_attack=2`)

### RampageMixin

On-kill chain attack — when the mob slays a target it immediately attacks the next living player in the room, bypassing the normal attack delay.

**Provides:**
- `rampage_message` — configurable announcement string with `{name}` and `{target}` placeholders
- `at_kill(victim)` override — find next living PC, announce, `execute_attack()` instantly

**Used by:** Gnoll

### TacticalDodgeMixin

Random dodge during combat ticks — the mob uses the same `CmdDodge` command available to players, sacrificing its attack for a round of disadvantage on incoming hits.

**Provides:**
- `dodge_chance` (default 0.25) — probability per combat tick
- `at_combat_tick(handler)` — roll against `dodge_chance`, execute `"dodge"` command if lucky

**Used by:** DireWolf (`dodge_chance=0.25`)

### Adding New Behaviors

Create a new mixin in `typeclasses/mixins/mob_behaviours/`, add it to `__init__.py`, and compose it into the mob class:

```python
class NewBehaviorMixin:
    some_attr = AttributeProperty(default_value)
    def at_kill(self, victim): ...   # or at_combat_tick, ai_wander, at_new_arrival, etc.

class NewMob(NewBehaviorMixin, AggressiveMob):
    some_attr = AttributeProperty(custom_value)
```

**MRO rule:** Behavior mixins go before `AggressiveMob` in the class definition so they override the correct methods. `FlyingMixin`/`SwimmingMixin` go before behavior mixins.

---

## FlyingMixin

Grants innate flight. Available to any NPC — not just mobs. A crow, a dragon, a flying merchant on a magic carpet.

**Provides:**
- `Condition.FLY` granted at object creation (permanent — not a spell effect)
- `preferred_height` AttributeProperty (default 2) — mob's idle/patrol altitude
- Mob spawns at `preferred_height`
- Returns to `preferred_height` when combat ends, target dies, or idle AI tick
- **Height bounds:** can descend to ground (0), **cannot go underwater** (< 0)

**Combat behavior (with AggressiveMixin):**
- Prefers targets at its own height (other flyers)
- If no airborne targets, descends to ground to engage
- With `InnateRangedMixin`: can attack from the air without descending (dragon breath)

---

## SwimmingMixin

Grants innate aquatic ability. Available to any NPC — not just mobs. A shark, a merfolk trainer, an underwater quest giver.

**Provides:**
- `Condition.WATER_BREATHING` granted at object creation (permanent)
- `preferred_depth` AttributeProperty (default -1) — mob's idle depth
- Mob spawns at `preferred_depth`
- Returns to `preferred_depth` when idle
- **Height bounds:** can surface to ground (0), **cannot fly** (> 0)

**Combat behavior (with AggressiveMixin):**
- Prefers targets at its own depth
- Can surface to fight ground-level targets
- Cannot pursue flying targets

---

## Humanoid Mobs — HumanoidWearslotsMixin

Humanoid mobs compose in `HumanoidWearslotsMixin` — the **same mixin** `FCMCharacter` uses. This gives them full equipment capability with zero new code.

**Gets from the mixin:**
- The full humanoid wearslot set (head, chest, wield, hold, etc. — see [INVENTORY_EQUIPMENT.md](INVENTORY_EQUIPMENT.md) § Wearslot Architecture for the canonical list)
- `wear()` / `remove()` / `get_slot()` / `get_all_worn()`
- The full weapon hook pipeline in combat
- Parry, riposte, dual-wield if weapon supports it

### Weapon Equipping

Done in `at_object_creation()` — create the weapon inside the mob, call `self.wear()`:

```python
class GnollArcher(AggressiveMixin, HumanoidWearslotsMixin, StateMachineAIMixin, CombatMob):
    def at_object_creation(self):
        super().at_object_creation()
        self.at_wearslots_init()  # initialize humanoid wearslots
        bow = create_object(
            "typeclasses.items.weapons.bow_nft_item.BowNFTItem",
            key="a crude shortbow", location=self,
            attributes=[("damage_dice", "1d6"), ("speed", 1.0)]
        )
        self.wear(bow)
```

(Note: `at_wearslots_init()` is auto-called from `BaseActor.at_object_creation()` — the explicit call above is illustrative, not required.)

### Weapon Mastery

Set via `db.weapon_skill_mastery_levels`:

```python
self.db.weapon_skill_mastery_levels = {"bow": 2}  # SKILLED
```

Mobs without this dict default to UNSKILLED (0) — base weapon damage and hooks fire, but no mastery bonuses (no extra attacks, no special effects like Slowing Shot).

Example mastery levels by mob difficulty:
- Town guard: BASIC longsword (0 parries, no special)
- Gnoll warrior: SKILLED battleaxe (sunder on hit)
- Boss knight: EXPERT longsword (parries + extra attack)

### Loot on Death

`CombatMob._create_corpse()` unequips worn items and transfers contents to the corpse. The class-based filter (see § Corpse Loot Filtering above) decides what survives:

- **`MobItem` instances** (mob-only weapons, armour, consumables) — deleted on death. They never enter the player economy.
- **NFT items** (placed by the spawn system as loot) — transferred to the corpse.
- **Gold and resources** — always transferred to the corpse via `transfer_gold_to()` / `transfer_resource_to()`.

**Why this works:** Mob weapons and armour are `MobItem` subclasses (`MobDagger`, `MobLongsword`, `MobWearable`, etc.) that share combat mechanics with the NFT versions but carry no NFT identity. A gate guard wields a `MobLongsword` for the stat bonus and combat hooks, and on death the weapon disappears — no economy flood. A gnoll carrying a scroll dropped by the spawn system carries an actual `BaseNFTItem` (not a MobItem), so the `isinstance(obj, MobItem)` filter lets it transfer to the corpse for players to loot.

The rule is purely class-based — there is no `loot` tag involved. The spawn system controls what becomes loot by choosing whether to place an NFT item or a MobItem on the mob.

### get_weapon() Integration and the Ranged Model

Already works — `get_weapon(mob)` checks `hasattr(mob, "get_slot")`:
- Humanoid mobs with wearslots → return wielded weapon (full hook pipeline)
- Animal mobs without wearslots → return None (use `damage_dice` directly)

**Three-tier ranged model:**
1. **Humanoid mobs** — wield actual weapon objects (melee sword or ranged bow). `get_weapon()` returns the weapon; weapon type determines reach.
2. **Non-humanoid mobs (default)** — melee only. `get_weapon()` returns None; `mob_weapon_type` defaults to `"melee"`.
3. **Non-humanoid mobs with `InnateRangedMixin`** — innate ranged attack. `get_weapon()` returns None; `mob_weapon_type = "missile"` from the mixin.

---

## InnateRangedMixin

Grants an innate ranged attack without wielding a weapon object. For non-humanoid mobs with natural ranged abilities — dragon breath, magic bolts, spitting venom.

**Provides:**
- `mob_weapon_type = "missile"` — tells the height system this mob can attack at range
- Innate ranged flag checked by `can_reach_target()` when `get_weapon()` returns None

```python
# In can_reach_target() — when weapon is None (animal mob):
weapon_type = getattr(attacker, "mob_weapon_type", "melee")
```

This is **separate from** humanoid mobs wielding actual bow objects — it covers innate abilities only. A dragon breathes fire at range via `InnateRangedMixin`; a gnoll archer shoots via a BowNFTItem in its WIELD slot.

**Combat behavior (with FlyingMixin):** Can attack from the air without descending — no need to swoop down to engage ground targets.

---

## Hybrid NPCs — CombatMixin + Service Role

When an NPC needs both service capabilities (dialogue, quests, trade) and combat capability, compose `CombatMixin` into the service NPC class. The NPC retains all its service functionality and gains the ability to be fought and killed.

### City Guard (LLM + Combat + Aggressive)

```
FightableCityGuard(AggressiveMixin, CombatMixin, HumanoidWearslotsMixin, LLMRoleplayNPC)
  Level: 5  |  HP: 40  |  Weapon: LongswordNFTItem (1d8, melee)
  weapon_skill_mastery_levels: {"longsword": 2}  (SKILLED)

  Behavior:
  - LLM drives dialogue: "Halt! You're under arrest. Come quietly."
  - If player complies: LLM continues, escorts to jail
  - If player attacks: CombatMixin handles the fight
  - AggressiveMixin: attacks criminals on sight (players with bounty)
  - On death: drops weapon, respawns after delay
```

### Provocable Drunk (LLM + Combat, No Aggression)

```
TavernDrunk(CombatMixin, LLMRoleplayNPC)
  Level: 1  |  HP: 15  |  damage_dice: "1d3"  |  damage_type: BLUDGEONING

  Behavior:
  - LLM drives bar conversation — mostly harmless
  - Player insults trigger LLM deciding to fight → execute_cmd("attack player")
  - No AggressiveMixin — only fights when LLM decides to
  - Comedic death message, respawns after short delay
```

### Design Notes for Hybrids

**Service continuity on death:** When a hybrid NPC dies, their service role goes away until they respawn. Kill the shopkeeper? No shopping until they're back. This is intentional — consequences for violence against service NPCs.

**Not every service NPC needs CombatMixin.** Most don't. The architecture enables it; the design decides when it makes sense. A trainer in a safe city probably doesn't need to be killable. A shady merchant in the wilderness probably does.

---

## Example Mob Profiles

### Crow — Flying Pack Predator
```
Crow(FlyingMixin, PackCourageMixin, AggressiveMob)
  Level: 1  |  HP: 4  |  damage_dice: "1d2"  |  damage_type: PIERCING
  size: "tiny"  |  AC: 13  |  STR: 3  |  DEX: 15  |  CON: 4
  preferred_height: 1  |  min_allies_to_attack: 2 (needs 3 total)
  loot: none  |  respawn: 30s  |  ai_tick: 5s
  Behavior: spawns airborne, descends to attack via _try_match_height(),
    re-ascends to preferred_height when idle, flees when pack breaks up
  Spawns: Millholm Woods (15, max 4/room), Millholm Farms abandoned (8, max 4/room)
```

### Shark — Aggressive Swimming Animal
```
Shark(AggressiveMixin, SwimmingMixin, StateMachineAIMixin, CombatMob)
  Level: 4  |  HP: 35  |  damage_dice: "2d6"  |  damage_type: PIERCING
  preferred_depth: -2
  Behavior: patrols at depth -2, surfaces to attack swimmers, can't pursue flyers
```

### Gnoll Archer — Aggressive Humanoid with Ranged Weapon
```
GnollArcher(AggressiveMixin, HumanoidWearslotsMixin, StateMachineAIMixin, CombatMob)
  Level: 3  |  HP: 25  |  Weapon: BowNFTItem (1d6, missile)
  weapon_skill_mastery_levels: {"bow": 1}  (BASIC)
  Behavior: attacks on sight, can hit targets at any height, -4 penalty at same height
```

### Dragon — Aggressive Flying Boss with Innate Ranged
```
Dragon(AggressiveMixin, FlyingMixin, InnateRangedMixin, AdaptiveLLMAIMixin, CombatMob)
  Level: 10  |  HP: 200  |  damage_dice: "3d8"  |  damage_type: FIRE
  preferred_height: 3
  InnateRangedMixin: mob_weapon_type = "missile" — attacks from the air without descending
  Behavior: LLM-driven, remembers past encounters, adapts tactics
```

### Town Guard — Passive Humanoid
```
TownGuard(HumanoidWearslotsMixin, StateMachineAIMixin, CombatMob)
  Level: 5  |  HP: 40  |  Weapon: LongswordNFTItem (1d8, melee)
  weapon_skill_mastery_levels: {"longsword": 2}  (SKILLED — gets parries)
  Behavior: does NOT attack on sight, only fights when attacked, patrols area
```

### Bandit Chief — Humanoid with LLM Tactics
```
BanditChief(AggressiveMixin, HumanoidWearslotsMixin, LLMAIMixin, CombatMob)
  Level: 5  |  HP: 50  |  Weapon: BattleaxeNFTItem (1d8, melee)
  weapon_skill_mastery_levels: {"battleaxe": 2}  (SKILLED — sunder on hit)
  Behavior: LLM picks targets tactically, uses bash/pummel, no persistent memory
```

---

## Integration Points

| System | How It Interacts |
|--------|------------------|
| **Combat (execute_attack)** | Humanoid mobs with weapons → full weapon hook pipeline. Animal mobs → `damage_dice` fallback |
| **Height combat (height_utils)** | `can_reach_target()` checks wielded weapon type OR `InnateRangedMixin.mob_weapon_type` for animals |
| **Spawning (ZoneSpawnScript)** | Weapon creation in `at_object_creation()`, not spawn JSON. JSON `attrs` can set mastery levels |
| **Loot (corpse)** | `_create_corpse()` transfers worn equipment to corpse — already implemented |
| **Flee** | Height advantage flee checks enemy weapon types — works for armed humanoid mobs and innate ranged mobs |
| **Mastery (CmdSkillBase)** | `mob_func()` fallback for animal mobs, full mastery dispatch for humanoids with mastery dicts |
| **AI memory (ai_memory app)** | `AdaptiveLLMAIMixin` stores/retrieves combat encounter embeddings for adaptive behavior |

---

## Migration from Previous Architecture (COMPLETED)

The refactor was **backward-compatible** — no database migration needed. All steps completed:

1. ~~Rename `AIMixin` → `StateMachineAIMixin`~~ — done (backward-compat alias kept)
2. ~~Extract `CombatMixin`~~ — done (handler access, enter/exit combat, initiate_attack, hp_fraction, is_low_health)
3. ~~Compose `CombatMixin` into `FCMCharacter`~~ — done (first in MRO)
4. ~~Compose `CombatMob(CombatMixin, StateMachineAIMixin, FungibleInventoryMixin, BaseNPC)`~~ — done
5. ~~Extract `AggressiveMixin` from `AggressiveMob`~~ — done (7 methods moved)
6. ~~`AggressiveMob` → `class AggressiveMob(AggressiveMixin, CombatMob)`~~ — done (pure convenience class)
7. ~~Create `FlyingMixin`~~ — done (grants Condition.FLY, preferred_height, ascend/descend)
8. ~~Create `SwimmingMixin`~~ — done (grants Condition.WATER_BREATHING, preferred_depth, dive/surface)
9. ~~Create `InnateRangedMixin`~~ — done (mob_weapon_type="missile", height_utils integration)
10. All existing subclasses (Wolf, DireWolf, Gnoll, Kobold, etc.) work unchanged via normal inheritance
11. `LLMAIMixin`, `AdaptiveLLMAIMixin` — future, purely additive
12. ~~Extract mob behavior mixins~~ — done: `PackCourageMixin` (from Kobold), `RampageMixin` (from Gnoll), `TacticalDodgeMixin` (from DireWolf) into `mixins/mob_behaviours/`
13. ~~Create Crow mob~~ — done: `Crow(FlyingMixin, PackCourageMixin, AggressiveMob)`, first flying mob in game

---

## Files

```
typeclasses/
├── actors/
│   ├── base_actor.py                   ← BaseActor (foundation, composes Height/Effects/DamageResistance)
│   ├── ai_handler.py                   ← AIHandler + StateMachineAIMixin
│   ├── character.py                    ← FCMCharacter (player)
│   ├── npc.py                          ← BaseNPC
│   ├── mob.py                          ← CombatMob + LLMCombatMob convenience classes
│   ├── npcs/                           ← service NPC subclasses (bartender, baker, trainer, shopkeeper, etc.)
│   └── mobs/
│       ├── aggressive_mob.py           ← AggressiveMob(AggressiveMixin, CombatMob)
│       ├── base_marine_mob.py          ← passive aquatic base (SwimmingMixin, CombatMob)
│       ├── animal mobs                 ← wolf, dire_wolf, rabbit, crow, owl_bird, cellar_rat,
│       │                                  bee_swarm, jagular, heffalump, woozle, wild_mule, ...
│       ├── humanoid mobs               ← kobold, kobold_chieftain, gnoll, gnoll_warlord,
│       │                                  skeleton, footpad, rat_king, ...
│       └── LLM combat mobs             ← townfolk, town_guard, city_watch, librarian, training_dummy
├── mixins/
│   ├── combat_mixin.py                 ← CombatMixin (handler access, enter/exit/initiate)
│   ├── aggressive_mixin.py             ← AggressiveMixin
│   ├── innate_ranged_mixin.py          ← InnateRangedMixin
│   ├── flying_mixin.py                 ← FlyingMixin
│   ├── swimming_mixin.py               ← SwimmingMixin
│   ├── llm_mixin.py                    ← LLMMixin (dialogue layer used by LLMCombatMob and LLMRoleplayNPC)
│   ├── mob_behaviours/                 ← reusable mob behavior mixins
│   │   ├── pack_courage_mixin.py       ← PackCourageMixin (Kobold, Crow)
│   │   ├── rampage_mixin.py            ← RampageMixin (Gnoll)
│   │   ├── tactical_dodge_mixin.py     ← TacticalDodgeMixin (DireWolf)
│   │   └── mob_followable_mixin.py     ← MobFollowableMixin (auto-reacquire squad leader)
│   └── wearslots/
│       ├── base_wearslots.py           ← BaseWearslotsMixin
│       └── humanoid_wearslots.py       ← HumanoidWearslotsMixin
```

**Future mixins (not yet built):**
- `llm_ai_mixin.py` — LLMAIMixin (per-tick LLM tactical decisions, replaces state machine for tactical mobs)
- `adaptive_llm_ai_mixin.py` — AdaptiveLLMAIMixin (LLMAIMixin + CombatMemory persistent learning)
