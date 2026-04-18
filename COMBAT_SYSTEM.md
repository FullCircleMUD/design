# Combat System — Design & Architecture

## Overview

FCM uses a **real-time combat system** with a fixed 4-second tick interval. Each combatant gets their own combat handler script with an independent ticker. Within each tick, initiative determines action order — faster weapons and higher DEX act first. Combat feels like classic MUDs: you type `attack dire wolf` and your character auto-attacks on a timer until you stop, the target dies, or you leave.

---

## Core Architecture: Per-Combatant Handlers

Each combatant gets their own `CombatHandler` (Evennia Script) rather than a single shared combat instance.

**Why per-combatant?**
- "In combat" = has a `combat_handler` script. No script = bystander. Simple boolean check.
- Each combatant has their own action queue, their own ticker, their own advantage/disadvantage state.
- Combat ends naturally: when `get_sides()` finds no enemies, each ally's handler cleans itself up.
- Evennia persists scripts automatically — handlers survive `evennia restart`.
- Scales trivially from 1v1 to group combat — just more handlers.

---

## Combat Flow

```
Player types "attack dire wolf"
    ↓
CmdAttack.func()
    ↓
enter_combat(player, dire_wolf)
    ├── _get_or_create_handler(player)      → CombatHandler script on player
    ├── _get_or_create_handler(dire_wolf)   → CombatHandler script on dire wolf
    ├── Pull in player's group members      → handlers on all followers in room
    ├── Pull in target's group members      → handlers on all mob allies in room
    └── Roll initiative for all new combatants:
        roll_initiative(obj) = d20 + effective_initiative + weapon_speed
        calculate_initiative_delays() → stagger first tick across 75% of interval
    ↓
handler.queue_action({"key": "attack", "target": dire_wolf, "dt": 4, "repeat": True})
    ↓
TICKER_HANDLER fires every 4 seconds (COMBAT_TICK_INTERVAL)
    ↓
handler.execute_next_action()
    ├── obj.at_combat_tick(handler)          → mob AI decision point (dodge? special attack?)
    ├── Check skip_actions counter           → stun/prone, dodger skip, decrements each tick
    ├── tick_combat_round() on actor         → decrements all combat_round named effects
    ├── decrement_advantages()               → count-based advantage tracking
    └── execute_attack(attacker, target)     → full attack resolution with all weapon hooks
        (fires effective_attacks_per_round times + off-hand attacks if dual-wielding)
```

---

## Unified Player/Mob Command Path

Mobs use the exact same command interface as players:

```
Player: types "attack wolf"           → CmdAttack → enter_combat → CombatHandler
Mob AI: execute_cmd("attack player")  → CmdAttack → enter_combat → CombatHandler
```

This is intentional. Future LLM AI just swaps the decision-maker — the LLM outputs command strings identical to what a player would type. One attack resolution path means one set of bugs, one set of weapon hooks, consistent behaviour for all combatants.

---

## Attack Resolution: 14 Weapon Hooks

`execute_attack()` fires all weapon hooks in a fixed order. Both attacker's weapon and defender's weapon get hook calls.

```
1.  weapon.at_pre_attack(attacker, target)               → hit modifier (enchantment, blurred)
2.  defender_weapon.at_pre_defend(target, attacker)       → AC modifier
3.  d20 roll (with advantage/disadvantage if applicable)
4.  Compare total_hit vs total_ac → hit or miss

BEFORE DAMAGE CALCULATION (new):
4b. Parry attempt: if defender has parries remaining and attacker uses melee weapon,
    defender rolls d20+DEX+mastery vs attacker's total hit. Success → block.
    Both weapons lose 1 durability on parry.

ON HIT:
5.  Roll damage dice (weapon.get_damage_roll() or innate mob damage_dice)
6.  weapon.at_hit(attacker, target, damage, dmg_type)    → modify damage, trigger effects
7.  weapon.at_crit(attacker, target, damage, dmg_type)   → crit bonus (if natural ≥ crit_threshold)
8.  defender_weapon.at_wielder_receive_crit(...)          → defender crit reaction (CRIT_IMMUNE)
8b. Protect intercept: if an ally is protecting target, intercept check fires here
9.  defender_weapon.at_wielder_hit(...)                   → defender hit reaction (reactive spells)
9b. Riposte: if parry succeeded and weapon has_riposte(), fire free counter-attack
10. Apply damage resistance → target.take_damage(damage, damage_type)
11. weapon.at_kill(attacker, target)                      → if target dies (Rampage, Cleave, etc.)

ON MISS:
12. weapon.at_miss(attacker, target)
13. defender_weapon.at_wielder_missed(target, attacker)

ALWAYS:
14. weapon.at_post_attack(attacker, target, hit, damage_dealt)
```

**Why 14 hooks?** Weapon subclasses override these hooks to implement mastery-scaled special effects. A longsword at MASTER mastery parries on `at_pre_defend`; a greatsword's Cleave fires in `at_kill`. The hooks are the extension points — the resolution function never changes.

---

## Weapon Types and Mastery Paths

All 24 weapon types are implemented with unique mastery mechanics:

| Weapon | Mastery Path |
|---|---|
| Longsword | Hit bonuses, parries, +1 attack at MASTER, parry advantage at GM |
| Rapier | Finesse, parries, riposte at EXPERT, parry advantage at GM |
| Greatsword | Cleave AoE cascade, Executioner (free attack on any kill) at GM |
| Battleaxe | Nerfed cleave + stacking Sunder (AC penalty per hit) |
| Handaxe | Lighter Sunder + extra attack at MASTER/GM |
| Dagger | Speed + crit threshold reduction, extra attacks, dual-wield off-hand |
| Shortsword | Dual-wield specialist, light parry |
| Bow | Slowing Shot (SLOWED debuff on contested roll), extra attack |
| Crossbow | Knockback/prone on hit |
| Sling | Concussive Daze (STUNNED) |
| Shuriken | Multi-throw, crit scaling, consumable (moves to target/floor on throw) |
| Blowgun | Poison DoT + CON-save paralysis |
| Bola | Entangle CC, save-each-round STR escape |
| Spear | Reach Counter (counter-attack when allies are hit) |
| Staff | Parry specialist (most parries, earliest parry advantage), riposte |
| Lance | Mounted powerhouse (devastating mounted, terrible on foot) |
| Mace | Crush (bonus damage vs high-AC targets) + extra attack |
| Club | Light Stagger (hit penalty debuff) + extra attack |
| Greatclub | Heavy Stagger (bigger penalty/duration, two-handed) |
| Hammer | Devastating Blow (crit damage multiplier) |
| Ninjatō | Pure offense — extra attacks + crit scaling + dual-wield. Ninja's signature sword. |
| Nunchaku | Contested DEX vs CON stun on hit, PRONE at MASTER+ |
| Sai | Disarm (unequips target weapon) + parry + riposte at GM |
| Unarmed | Damage/hit scaling, stun/knockdown at SKILLED+, extra attacks at MASTER/GM |

### Weapon Architecture — Composition via Mixins

Weapon mechanics are extracted into shared mixins so both player NFT weapons and mob non-NFT weapons use identical combat behaviour from a single source of truth.

**Three layers of composition:**

1. **WeaponMechanicsMixin** (`typeclasses/items/weapons/weapon_mechanics_mixin.py`) — defines all weapon AttributeProperties, `get_damage_roll()`, mastery helpers, 14 combat hooks (default to no-op), and 10 mastery-scaled query methods (default to 0/False). This is the combat interface that `execute_attack()` calls.

2. **Weapon identity mixins** (e.g. `DaggerMixin` in `dagger_nft_item.py`) — override specific methods from WeaponMechanicsMixin with mastery-tier lookup tables and special mechanics. A `DaggerMixin` defines what makes a dagger a dagger: crit scaling, extra attacks, off-hand attacks. Everything it doesn't override falls through to the base defaults via MRO.

3. **Carrier classes** — `WeaponNFTItem` (player, NFT-tracked, durable) or `MobWeapon` (mob-only, no NFT, no durability, deleted on death). These provide the item lifecycle; the weapon identity mixin provides the combat behaviour.

```
WeaponMechanicsMixin       ← base combat interface (14 hooks, 10 mastery queries, damage resolution)

Weapon identity mixins (single source of truth for each weapon type):
├── LongswordMixin         ← parry + extra attack mastery path
├── RapierMixin            ← finesse + parry + riposte mastery path
├── GreatswordMixin        ← cleave AoE + executioner mastery path
├── BattleaxeMixin         ← nerfed cleave + stacking sunder mastery path
├── AxeMixin               ← reduced sunder + extra attack mastery path
├── DaggerMixin            ← finesse + extra attacks + crit threshold + dual-wield off-hand
├── ShortswordMixin        ← dual-wield specialist + light parry
├── BowMixin               ← contested slowing shot + extra attack mastery path
├── CrossbowMixin          ← knockback/prone mastery path (no extra attacks)
├── SlingMixin             ← concussive daze/stun mastery path
├── ShurikenMixin          ← multi-throw + crit scaling + consumable
├── BlowgunMixin           ← poison DoT + paralysis mastery path
├── BolaMixin              ← entangle CC mastery path
├── SpearMixin             ← reach counter mastery path
├── StaffMixin             ← parry specialist (most parries, early parry advantage, riposte)
├── LanceMixin             ← mounted powerhouse (crit, prone, extra attacks; nerfed on foot)
├── MaceMixin              ← anti-armor crush + extra attack mastery path
├── ClubMixin              ← light stagger + extra attack mastery path
├── GreatclubMixin         ← heavy stagger, two-handed
├── HammerMixin            ← devastating blow (crit damage multiplier)
├── NinjatoMixin           ← pure offense + crit + dual-wield (ninja signature)
├── NunchakuMixin          ← contested stun + PRONE at MASTER+
└── SaiMixin               ← disarm + parry + riposte at GM (ninja defensive)

Composed into carrier classes:
  Player:  DaggerNFTItem(DaggerMixin, WeaponNFTItem)       ← NFT-tracked, durable, player economy
  Mob:     MobDagger(DaggerMixin, MobWeapon)               ← ephemeral, deleted on mob death

UnarmedWeapon              ← pure Python singleton (NOT a DB object)
                             Mimics WeaponMechanicsMixin interface. Returned by
                             get_weapon() when PC has no wielded weapon.
                             Stun/knockdown at SKILLED+.
```

### How Mastery Works — MRO Override Pattern

Each weapon identity mixin overrides mastery-related methods (`get_mastery_hit_bonus`, `get_parries_per_round`, `get_extra_attacks`, `get_offhand_attacks`, `get_offhand_hit_modifier`, `get_parry_advantage`, `has_riposte`, `get_mastery_crit_threshold_modifier`, `get_reach_counters_per_round`, `get_stun_checks_per_round`, `get_disarm_checks_per_round`) with mastery-tier lookup tables. The combat handler and `execute_attack()` call these methods generically — no weapon-specific branching in combat code.

**MRO enables this without the identity mixin inheriting from WeaponMechanicsMixin.** For `MobDagger(DaggerMixin, MobWeapon)`, Python's MRO is:

```
MobDagger → DaggerMixin → MobWeapon → WeaponMechanicsMixin → MobWearable → ...
```

When combat calls `weapon.get_extra_attacks(wielder)`, Python finds it on `DaggerMixin` first (the override). When `DaggerMixin` internally calls `self.get_wielder_mastery(wielder)`, Python doesn't find it on `DaggerMixin`, continues down the MRO, and finds it on `WeaponMechanicsMixin`. Any hook the identity mixin doesn't override (e.g. `at_hit`) falls through to the WeaponMechanicsMixin default. This means a weapon identity mixin only needs to define what makes it different — everything else inherits the base behaviour automatically.

```python
class LongswordNFTItem(WeaponNFTItem):
    def get_parries_per_round(self, wielder):
        mastery = self.get_wielder_mastery(wielder)
        return _LONGSWORD_PARRIES.get(mastery, 0)
```

### Mastery vs Individual Weapon Power

- **Mastery effects** = weapon TYPE skill (spear technique). Same across ALL spears — bronze spear, iron spear, steel spear all get the same mastery behaviour (reach counter, extra attacks at higher tiers)
- **Individual weapon power** = prototype data. All weapons of a type share the same `base_damage` key (all spears = d8, all daggers = d4). The `material` tier (wood, bronze, iron, steel, adamantine) determines how that base damage scales — higher material = more damage at every mastery level. Additional prototype fields (`wear_effects`, `max_durability`) further differentiate individual weapons

```python
BRONZE_SPEAR = {
    "prototype_key": "bronze_spear",
    "typeclass": "typeclasses.items.weapons.spear_nft_item.SpearNFTItem",
    "key": "Bronze Spear",
    "base_damage": "d8",       # weapon type determines base die (all spears = d8)
    "material": "bronze",      # material tier determines damage scaling
    "speed": 1,
    "max_durability": 3600,
}

IRON_SPEAR = {
    "prototype_key": "iron_spear",
    "typeclass": "typeclasses.items.weapons.spear_nft_item.SpearNFTItem",
    "key": "Iron Spear",
    "base_damage": "d8",       # same base die — it's still a spear
    "material": "iron",        # higher material tier = more damage at every mastery level
    "speed": 1,
    "max_durability": 5400,
}
```

Both use `SpearNFTItem`, both get identical mastery effects (reach counter, extra attacks at higher tiers). Damage is resolved via a 3D lookup: `get_damage_dice(base_damage, material, mastery_level)` in `world/damage_tables.py`. The `base_damage` key is fixed per weapon type (all spears are d8, all daggers are d4). The `material` tier determines how damage scales with mastery. For example at BASIC mastery:
- Bronze Spear: `d8 + bronze + BASIC` → `1d7`
- Iron Spear: `d8 + iron + BASIC` → `1d8`

Higher material tiers scale damage faster with mastery — an iron weapon at GRANDMASTER (`2d8`) hits harder than a bronze one at the same mastery (`2d7`).

### Crit Threshold System

`base_crit_threshold` is an `AttributeProperty(20)` on BaseActor, modified by equipment on/off and spell/potion effects. `effective_crit_threshold` is a `@property` that computes `base_crit_threshold + weapon.get_mastery_crit_threshold_modifier(wielder)`. Combat uses `attacker.effective_crit_threshold` (not the old `crit_threshold` AttributeProperty, which was deleted).

### Dual-Wield System

**`can_dual_wield` flag:** Boolean `AttributeProperty(False)` on `WeaponNFTItem`. Weapons with `can_dual_wield = True` (shortsword, dagger) can be equipped via `cmd_hold` into the HOLD slot in addition to `cmd_wield` into the WIELD slot. No new wearslot enum value — the weapon's wearslot stays WIELD. `cmd_hold` just checks the flag.

**Main hand mastery drives everything:**
- All mastery bonuses (hit, damage, crit threshold, parries, extra attacks) come from the **main hand weapon** (WIELD slot) only
- The off-hand weapon's mastery bonuses are **completely ignored** — no crit bonus, no extra attacks, no mastery hit/damage bonus from the held weapon
- Off-hand attacks only occur if the main hand weapon grants them via `get_offhand_attacks(wielder)` AND a `can_dual_wield` weapon is equipped in HOLD

**Enchantment bonuses from off-hand DO apply:**
- Magical bonuses (+1 hit, +1 damage, etc.) from the off-hand weapon's `wear_effects` apply to the **character's stats** (total_hit_bonus, total_damage_bonus, etc.) as they already do via the wear_effects system
- This means a held +1 dagger affects hit and damage for both main hand and off-hand attacks
- But mastery-based bonuses (from `get_mastery_*` methods) are main hand only

**Off-hand attack flow:**
- Main hand weapon's `get_offhand_attacks(wielder)` determines how many off-hand attacks (0 if no method or weapon doesn't grant them)
- Main hand weapon's `get_offhand_hit_modifier(wielder)` determines the off-hand hit penalty
- Off-hand attacks use the **off-hand weapon's damage dice** (`get_damage_roll()`) — so weapon quality matters for off-hand
- Off-hand attacks take durability from the off-hand weapon

**Dual-wield weapons:** Shortsword, Dagger (others may be added later).

### Finesse Weapons

Weapons that use `max(STR, DEX)` for hit/damage rolls. `is_finesse = AttributeProperty(False)` on WeaponNFTItem base class. Subclasses like RapierNFTItem set `is_finesse = True`. `effective_hit_bonus`/`effective_damage_bonus` on BaseActor check the wielded weapon's `is_finesse` flag: finesse = `max(STR, DEX)`, missile = DEX, melee = STR.

### Unarmed Weapon Singleton

Pure Python singleton (`typeclasses/items/weapons/unarmed_weapon.py`) — NOT an Evennia DB object. Mimics `WeaponNFTItem` interface so combat code treats unarmed PCs identically to armed ones. Stateless — all mutable state on wielder/combat_handler.

**`get_weapon(actor)`** helper in `combat_utils.py` returns: wielded weapon if equipped, `UNARMED` singleton if actor has wearslots but no weapon (PC boxing), or `None` for animal mobs (no wearslots, use `damage_dice` attribute). All combat code uses `get_weapon()` instead of direct `get_slot("WIELD")`.

**`force_drop_weapon(target, weapon=None)`** — shared disarm utility in `combat_utils.py`. Conditional behaviour: mobs/NPCs drop weapon to room floor (`move_to(location)`), player characters unequip to inventory only (no item loss). Guards: UnarmedWeapon immune, must have wielded weapon. Used by both Command spell (Drop word) and Sai weapon disarm. Returns `(bool, weapon_name)`.

**Mastery progression:** Damage dice, hit/damage bonuses, and extra attacks scale with mastery tier.

**Stun/Knockdown (SKILLED+):** On-hit contested roll (d20 + STR + mastery vs d20 + CON). Size-gated: HUGE+ immune. Stun check count tracked via `stun_checks_remaining` on combat handler, reset each tick. Higher mastery tiers unlock PRONE (with advantage grant to enemies) and longer durations.

**Parry immunity:** Unarmed attacks cannot be parried (weapon-on-weapon only, gated by `isinstance(weapon, UnarmedWeapon)` check in `execute_attack()`).

**Multi-round skip:** `skip_actions` counter on combat handler replaces `skip_next_action` for stun/prone. Decrements each tick; condition cleared when counter reaches 0 via `_clear_stun_prone()`.

**Actor size:** All actors (PCs, mobs, pets) have `base_size` and `size` AttributeProperty on `BaseActor` (stored as `Size.X.value` strings for Evennia serialization). `base_size` is the permanent racial/creature size; `size` is the active value rebuilt from `base_size` by `_recalculate_stats()`. PCs get both set from `race.size` during chargen/remort. Mobs/pets override via their own `AttributeProperty(Size.X.value)`. `get_actor_size(actor)` in `combat/combat_utils.py` converts the string to a `Size` enum for comparison. Size enum: `enums/size.py` — TINY, SMALL, MEDIUM, LARGE, HUGE, GARGANTUAN (numeric values 1–6 via `size_value()`). Enlarge/shrink effects modify `size` during `_accumulate_effect()`; when the effect expires, `_recalculate_stats()` resets `size` back to `base_size`.

### Reaction System Architecture (Planned)

**Key architectural decision:** the combat reaction system (Shield spell on hit, Counter Spell, etc.) is NOT a separate system. Instead, weapon hooks serve as the universal reaction engine:

- **Unarmed weapon singleton:** every humanoid always has a "weapon" — real or a stateless `UnarmedWeapon` singleton. This means weapon hooks fire for ALL combatants, not just armed ones.
- **Base hooks handle universal reactions:** `WeaponNFTItem.at_wielder_hit()` checks if the defender has reaction spells configured (e.g. Shield). Weapon subclasses override hooks but **always call `super()`** to ensure universal reactions fire.
- **The 14 existing weapon hooks ARE the reaction pipeline.** No separate reaction system needed.
- **`spellconfig` menu command:** players configure reaction conditions via a template/menu UI. Single condition per rule only (no compound AND/OR). Storage: `db.reaction_rules` on character.
- **Reactions cost resources** (mana for casters, movement for martial) — not free.

---

## Parry and Riposte

**Parry:** On each incoming melee weapon attack, if the defender has `parries_remaining > 0`, they automatically attempt a parry. Roll: `d20 + DEX mod + mastery hit bonus` vs attacker's total hit. Success: damage blocked, both weapons lose 1 durability, `parries_remaining -= 1`. Parries reset each combat tick.

**Riposte:** If a parry succeeds and `weapon.has_riposte()` is True, fire a free `execute_attack(defender, attacker, _is_riposte=True)`. Riposte attacks skip the parry check (no infinite recursion) and cannot themselves be parried.

**Parry immunity:** Unarmed attacks cannot be parried (weapon-on-weapon only).

---

## Multi-Attack

`effective_attacks_per_round` is a `@property` on `BaseActor` composing:
- `attacks_per_round` (base + condition effects like HASTED)
- `weapon.get_extra_attacks(wielder)` (mastery bonus)

The combat handler reads this single property — weapon classes return different values at each mastery tier. Daggers get extra attacks at SKILLED/EXPERT/MASTER/GM; longswords only at MASTER/GM.

**Off-hand attacks:** Dual-wield weapons (dagger, shortsword) grant off-hand attacks via `weapon.get_offhand_attacks(wielder)`. Off-hand hits use the HOLD slot weapon's damage dice.

---

## Advantage/Disadvantage System

### In Combat (per-target, count-based)

Stored on `CombatHandler`: `advantage_against = {target_id: int}` rounds remaining.

- `set_advantage(target, rounds=N)` — takes max of existing vs new (not additive)
- `consume_advantage(target)` — called by `execute_attack()` on use, decrements by 1
- `decrement_advantages()` — called at end of each tick, minimum 1/round decrement for unused entries
- `has_advantage(target)` — used by execute_attack() to decide 2d20 pick highest

This is count-based, not one-shot. Multiple rounds of advantage from different sources work correctly.

### Out of Combat (boolean flags)

`db.non_combat_advantage` / `db.non_combat_disadvantage` on actors. Used by all non-combat d20 skill checks (picklock, hide, search, etc.). Both True → cancel out. Consumed after each check.

---

## Named Effects in Combat

The `EffectsManagerMixin` integrates directly with the combat handler:

- **`tick_combat_round()`** called each tick — decrements all `combat_rounds` effects, auto-removes expired. Includes save-each-round checks (ENTANGLED, PARALYSED, etc.)
- **`clear_combat_effects()`** called on combat end — removes all `combat_rounds` effects

**Active combat effects with gameplay impact:**

| Effect | Impact |
|---|---|
| STUNNED | `skip_actions` counter set — actor loses that many actions |
| PRONE | `skip_actions` set + all enemies get advantage for duration |
| SLOWED | Caps `attacks_per_round` to 1, blocks off-hand attacks |
| PARALYSED | `skip_actions` set + enemies get advantage, save-each-round WIS |
| ENTANGLED | `skip_actions` set + enemies get advantage, save-each-round STR |
| BLURRED | BlurScript sets 1 disadvantage on all enemies per round |
| SHIELD | AC bonus for duration |
| STAGGERED | Hit penalty via `stat_bonus` on `total_hit_bonus` |
| SUNDERED | Stacking AC penalty on target |
| OFFENSIVE_STANCE | Hit/damage bonus, AC penalty |
| DEFENSIVE_STANCE | AC bonus, hit penalty |

---

## Reactive Spells

Two reactive spells auto-trigger during combat without any action cost:

**Shield (Abjuration):**
- Triggers via `at_wielder_about_to_be_hit` weapon hook (step 9 in the 14-hook pipeline)
- Check in `combat/reactive_spells.py`: `check_reactive_shield(wielder, attacker)`
- Gates: `shield_active` player preference toggle + memorised + mana available
- Mana cost 3/5/7/9/12 per trigger (scaling with mastery)
- Effect: +4/+4/+5/+5/+6 AC for 1/2/2/3/3 rounds
- Anti-stacking: won't trigger if already shielded

**Smite (Divine Judgement — paladin only):**
- Triggers via `at_post_attack` weapon hook on successful hit
- Check: `check_reactive_smite(wielder, attacker)`
- Gates: `smite_active` player preference toggle + memorised + mana
- Mana cost 3/5/7/9/12 per trigger
- Effect: +1d6/+2d6/+3d6/+4d6/+5d6 bonus radiant damage on the triggering hit
- Respects radiant resistance on target

---

## Combat Skill Commands

All class combat skills are implemented:

| Command | Class | Mechanic |
|---|---|---|
| `flee` | All | DEX check DC 10 — success exits through random exit, failure gives enemies 1 round advantage |
| `dodge` | All | Give up action, give all enemies disadvantage (1 round) |
| `assist` | All | Give up action, grant ally advantage against all enemies |
| `bash` | Warrior/Paladin | High-risk: contested STR. Success: target PRONE 1 round. Failure: basher may fall prone |
| `pummel` | Warrior/Paladin | Low-risk: contested STR vs DEX. Success: target STUNNED 1 round |
| `stab` | Thief/Ninja | Sneak Attack — bonus damage dice (2d6–10d6) when attacker has advantage |
| `protect` | Warrior/Paladin | Toggle — intercept attacks aimed at an ally (40–80% flat chance) |
| `taunt` | Warrior/Paladin | Opener or in-combat: contested CHA vs WIS to redirect mob's target |
| `offence` | Warrior/Paladin | Group leader stance: hit/damage bonus, small AC penalty |
| `defence` | Warrior/Paladin | Group leader stance: AC bonus, small hit penalty |
| `retreat` | Warrior/Paladin | Group withdrawal: INT+CHA roll. Success: move whole group through chosen exit |

---

## Movement Cost for Combat Actions

Physical combat actions consume **movement points** (the `move` attribute). This creates a secondary resource alongside HP and mana — extended combat drains stamina, forcing tactical decisions about when to use powerful skills vs conserving energy.

Movement regenerates via RegenerationService every 60 seconds: `ceil(level/4) + CON_bonus × position_multiplier` (standing 1×, resting 2×, sleeping 3×, fighting 0×).

| Action | Move Cost | Rationale |
|--------|-----------|-----------|
| Room movement | 1 | Base traversal cost |
| `bash` | 2 | Heavy physical charge |
| `pummel` | 1 | Lighter rapid strikes |
| `stab` | 2 | Explosive burst movement |
| `dodge` | 1 | Evasive repositioning |
| `retreat` | 2 | Group withdrawal effort |
| `taunt` | 1 | Positioning and posturing |

**Not costed (already resource-gated):** `attack` (free basic action), `cast` (mana), `flee` (costs 1 via room move), `protect`/`offence`/`defence` (toggles), parry/riposte (automatic reactions), `assist` (costs attack action).

When move points are insufficient, the skill is blocked with "You are too exhausted to [action]."

---

## Protect Mechanic

`protect <ally>` toggles a tanking stance. Per tick, before `take_damage()` is called on the protected ally, the system checks each active protector. On intercept (flat % by mastery: 40/50/60/70/80%), the protector takes the full hit instead — using their own AC/resistances. Multiple protectors for one target each roll independently; first success wins.

---

## Group Combat

**Auto-join on attack:** `enter_combat()` creates handlers on the attacker, target, all members of attacker's follow group (same room), and all members of target's follow group (same room).

**Stances (offence/defence):** Leader-driven buff that applies to the entire group in the same room. Uses named effects with `duration_type=None` (permanent until combat ends). `clear_combat_effects()` removes stances on combat end. Mutually exclusive — activating one removes the other.

**Retreat:** Single leader skill check, whole group moves on success — captures all enemy references before the move to correctly stop combat.

---

## CmdSkillBase Mastery Branch

All skill commands inherit `CmdSkillBase`, which branches on whether `skill_mastery_levels` exists:
- **Has mastery data** (players, humanoid NPCs) → full mastery dispatch (unskilled/basic/skilled/expert/master/grandmaster)
- **No mastery data** (animal mobs) → `mob_func()` fallback with innate ability flavour

This means humanoid boss NPCs fight with mastery benefits matching their tier. Wolves just dodge like wolves.

---

## Side Detection

Sides are assigned **dynamically at combat entry time** based on who attacked who and group membership — not actor type. Each combatant's `CombatHandler` stores a `combat_side` (1 or 2).

**Rules:**
1. When A attacks B, they are placed on **opposite sides** (side 1 vs side 2).
2. **Group members** (follow system) auto-join on the same side as their groupmate already in combat.
3. A **bystander** who attacks a combatant joins the **opposite side** of that combatant.
4. A combatant who directly attacks a bystander pulls the bystander onto the **opposite side**.
5. PvP rooms remain FFA (everyone is their own side, ignoring `combat_side`).
6. Everyone without a combat handler is a bystander — not on any side.

**Key functions** (`combat/combat_utils.py`):
- `get_sides(combatant)` → `(allies, enemies)` — reads `combat_side` from each handler in the room.
- `_get_combat_side(obj)` → int — reads side from an object's handler (0 if none).
- `_determine_side_from_group(actor, room)` → int — checks if any group member is already in combat, returns their side.
- `_opposite_side(side)` → int — returns the other side (1↔2).

**This enables:** NPC-NPC fights (e.g. goblins vs guards), PC-NPC alliances (guard following a PC joins their side), and any grouping where the follow system connects actors.

---

## Stealth & HIDDEN Condition

Hide command (`commands/all_char_cmds/cmd_hide.py`) — available to ALL characters (not class-gated). Contested check: `d20 + effective_stealth_bonus` vs best passive perception in room (`10 + effective_perception_bonus`). Binary outcome — hidden from everyone or nobody. Unskilled characters can attempt but suffer -2 penalty from UNSKILLED mastery bonus. `best_passive_perception(room, exclude)` helper accepts single object or set/list of exclusions.

**Racial Skill Advantages:** Races can have permanent advantage on specific skill checks via `racial_skill_advantages` frozenset on `RaceBase` (e.g. halflings have `{"stealth"}`). This grants advantage (roll 2d20, take best) on all stealth checks — hide, movement stealth rechecks, and stash. Racial advantage is permanent and does NOT consume the one-shot `non_combat_advantage` flag from `assist` — both can be active, but advantage doesn't stack beyond "you have advantage". The `racial_skill_advantages` field is extensible for future races (e.g. elves could get `{"perception"}`).

**Movement while HIDDEN:** `FCMCharacter.at_post_move()` automatically rolls stealth vs best perceiver in destination room on entry. Success = stay hidden. Fail = revealed with messaging. Check is on **entry only**, not exit. No separate sneak command — movement while HIDDEN *is* sneaking.

**Search reveals hidden characters:** `cmd_search.py` rolls active perception (`d20 + effective_perception_bonus`) vs passive stealth (`10 + effective_stealth_bonus`).

**Attack from hide:** `cmd_attack.py` checks HIDDEN before combat — breaks hide, grants 1 round advantage via `combat_handler.set_advantage(target, rounds=1)`.

**Restrictions:** Cannot hide in combat (`scripts.get("combat_handler")`). Aggressive/noisy actions (attacking, speaking) break hide.

### Stash Command (Object Concealment)

Stash command (`commands/class_skill_cmdsets/class_skill_cmds/cmd_stash.py`) — STEALTH class skill (thief, ninja, bard). Two branches:

- **Object stashing:** `stash <object>` — physically hides objects in the current room. Rolls `d20 + effective_stealth_bonus`; result becomes the object's `find_dc`. Object disappears from room display via existing `is_visible_to()` filtering. Found via `search` command (uses `discover()` to reveal).
- **Actor stashing:** `stash <ally>` — hides an ally using the stasher's stealth roll. Rolls `d20 + effective_stealth_bonus` vs `best_passive_perception(room, exclude={caller, target})`. On success, applies HIDDEN condition to the target. Target then follows normal hidden rules — they use their own stealth if they move, and all standard hidden-breaking triggers work. Cannot stash yourself (use `hide` instead), cannot stash someone already hidden or in combat.

**Stash vs Conceal:** Two distinct commands for hiding things:
- `stash` (STEALTH, thief/ninja/bard) — physical concealment. Tucks something behind stones, buries under debris. DEX-based.
- `conceal` (MISDIRECTION, bard-only) — magical glamour. Makes something unremarkable via bardic magic. CHA-based. Scaffold only, not yet implemented.

**HiddenObjectMixin coverage:** All three item base classes support hiding:
- `BaseNFTItem` — all NFT items (weapons, armor, consumables, containers)
- `WorldItem` — non-NFT takeables (keys, novelty items)
- `WorldFixture` — immovable world objects (signs, chests) — had the mixin originally

All default to `is_hidden=False`. Room display filtering (`get_display_things()`) already handles `is_visible_to()`.

### Case & Pickpocket (SUBTERFUGE Skill)

Two linked commands forming a scout-then-steal workflow for thieves/ninjas/bards:

**Case** (`commands/class_skill_cmdsets/class_skill_cmds/cmd_case.py`) — passive observation of a target's inventory. Each item has a mastery-dependent % chance of being revealed (BASIC 50%, SKILLED 60%, EXPERT 70%, MASTER 80%, GM 90%). Gold shown as vague tiers ("a few coins", "some gold", "a decent purse", "a heavy coin purse", "a fortune"). Resources show type but not quantity ("some wheat"). Results cached for 5 minutes — repeat shows same results. Does NOT break HIDDEN (purely observational).

**Pickpocket** (`commands/class_skill_cmdsets/class_skill_cmds/cmd_pickpocket.py`) — steal something revealed by case. Syntax: `pickpocket <thing> from <target>`. Contested roll: `d20 + DEX mod + SUBTERFUGE mastery bonus` vs `10 + target.effective_perception_bonus`. HIDDEN gives advantage (roll twice, take best). HIDDEN always breaks after attempt.

Gate chain: parse "from" syntax → find target → not self → not in combat → room allows combat → PvP room for player targets → BASIC+ mastery → not immortal NPC → must have cased → target still has item → 60s per-target cooldown.

Success: gold (`1d6 + mastery_bonus`, capped), resource (`1d4 + mastery_bonus//2`, capped), or item (moved to thief). Failure: target alerted, room message, aggressive CombatMobs aggro via `mob_attack()`.

**Thief combo:** `hide` → `case` (stays hidden) → `pickpocket` (advantage from hidden, then revealed).

---

## Durability in Combat

Items take wear during combat:
- **Weapon:** -1 durability per hit landed
- **Body armor:** -1 durability when the wearer takes a hit
- **Both weapons:** -1 each on a parry
- **Helmet:** -1 when CRIT_IMMUNE condition downgrades an incoming crit to a normal hit

Items at 0 durability can no longer be used. `DurabilityMixin` on all item types.

---

## Height Combat

Height introduces a reachability dimension to combat. Actors at different `room_vertical_position` values cannot melee each other by default.

### Reachability Rules (`combat/height_utils.py`)

```
can_reach_target(attacker, target, weapon):
  Same height           → always reachable
  Different heights:
    Wielded missile weapon  → reachable
    InnateRangedMixin       → reachable (optional range limit)
    Otherwise               → NOT reachable
```

**Ranged at same height:** -4 to-hit penalty (melee range, awkward for ranged) via `get_height_hit_modifier()`.

### Per-Tick Height Check (combat_handler.py)

Every combat tick in `execute_next_action()`, before attack resolution:

1. `can_reach_target()` check against current target
2. If unreachable and combatant is a **mob (not PC)**:
   - **Step 1 — Chase:** Call `_try_match_height(target)` (from `AggressiveMixin`). Flying mobs ascend, swimming mobs dive. If height matched → broadcast message → attack proceeds.
   - **Step 2 — Retarget:** If can't match → `_find_reachable_target(weapon)` scans enemies for a reachable alternative.
   - **Step 3 — Flee:** If no reachable targets → `execute_cmd("flee")`. Auto-succeeds when no enemy can melee at the same height.
3. If unreachable and combatant is a **PC**: "You can't reach X from your current position." → skip tick (keep combat active).

### Height Change Broadcast

`_broadcast_height_change(actor, old_height, new_height)` sends contextual room messages:

| Transition | Message |
|---|---|
| Ascending from water to surface | "X surfaces from the water!" |
| Ascending within water | "X rises through the water!" |
| Ascending in air | "X flies upward!" |
| Descending from surface to water | "X dives beneath the surface!" |
| Descending within water | "X dives deeper!" |
| Descending in air | "X swoops down!" |

### Non-Repeating Retarget

After a non-repeating action completes, the auto-retarget code also prefers height-reachable targets (uses `can_reach_target()` filter before falling back to first available enemy).

### Flee Height Advantage

Flee auto-succeeds (no DEX check) when no enemy at the same height has a melee weapon. A flying character with no grounded melee enemies below can always escape.

### Spell Height Gating

Spells use a separate `spell_range` system (not `can_reach_target()`, which is weapon-specific):

```
spell_range attribute on Spell base class:
  "self"   → no height check (self-targeted)
  "melee"  → same height required (blocked across heights, no mana spent)
  "ranged" → any height (default — most spells)
```

Height validation runs in `Spell.cast()` before mana deduction. Spells that override `cast()` (e.g. Vampiric Touch) must include the same check.

| Spell | spell_range | Reason |
|---|---|---|
| Vampiric Touch | melee | Touch attack |
| Cure Wounds | melee | Laying on hands |
| Shield | self | Self-only reactive |
| Mage Armor | self | Self-only buff |
| All others | ranged (default) | Projectile/AoE magic |

**AoE and height:** `get_room_enemies()` and `get_room_all()` return targets at all heights. Height-filtered variants (`get_room_enemies_at_height()`, `get_room_all_at_height()`) are available for future spells that need same-height AoE targeting.

### Files

- `combat/height_utils.py` — `can_reach_target()`, `get_height_hit_modifier()`
- `combat/combat_handler.py` — per-tick height check, `_broadcast_height_change()`, `_find_reachable_target()`
- `typeclasses/mixins/aggressive_mixin.py` — `_try_match_height()`
- `commands/all_char_cmds/cmd_flee.py` — height advantage flee
- `world/spells/base_spell.py` — `spell_range` attribute, height gate in `cast()`
- `world/spells/spell_utils.py` — `get_room_enemies_at_height()`, `get_room_all_at_height()`

### Tests

- 25 tests in `tests/command_tests/test_combat_height.py` (melee blocking, ranged cross-height, flee height advantage)
- 5 tests in `tests/typeclass_tests/test_combat_height_retarget.py` (mob chase, retarget, flee, broadcast)
- 13 tests in `tests/typeclass_tests/test_spells.py` (spell height gating, AoE height helpers)

---

## Combat Position

Characters have a `position` AttributeProperty (standing/sitting/resting/sleeping/fighting):
- Combat sets position to "fighting"
- Combat end sets position to "standing"
- Resting and sleeping affect regeneration multipliers
- Movement is blocked unless position is "standing" or "fighting"

`ndb.combat_target` on the character tracks current target for room display ("Bob is here, fighting a gnoll.").

---

## Creature Type Tags

Mobs can be tagged with `category="creature_type"` to classify them for ability targeting (Turn Undead, nature spells, etc.). Tags are set in the mob's `at_object_creation()` method.

**Current creature types:**
- `"undead"` — skeletons, zombies, wraiths. Affected by Turn Undead, divine abilities.

**Implementation:** Simple Evennia tag — no enum yet. Check via:
```python
if target.tags.get("undead", category="creature_type"):
    # affected by Turn Undead
```

**Skeleton typeclass** (`typeclasses/actors/mobs/skeleton.py`): `AggressiveMob` subclass that auto-tags as undead. Stats set via spawn JSON. First instance: Millholm Cemetery Stonefield Tomb (3 skeletons, L1, 10 HP, 1d4).

**Future types** (add as needed): beast, humanoid, demon, construct, elemental, fey, dragon.

---

## XP and Loot

**XP:** `CombatMob` awards XP on death: `10 * mob.level`. The kill hook path is `CombatMob.die()` → `weapon.at_kill()` in `combat_utils.py` → `CombatMob.at_kill(victim)` (mob's own override for special abilities like Rampage).

**Loot:** Common mobs (deleted on death) transfer contents to a `Corpse` object before deletion. Players use the `loot` command on the corpse. Corpses despawn after 300 seconds (default `corpse_despawn_delay`).

Corpse transfer follows the **loot tag rule**: fungibles (gold, resources) always transfer. NFT items only transfer if tagged `loot` (category `item`) — untagged NFTs (mob equipment like wielded weapons or worn armour) are deleted on death. The spawn system tags all NFT loot it places on mobs (scrolls, recipes, rare drops). Items created as mob equipment at spawn time are not tagged and do not drop. This allows humanoid mobs to wield real weapons for combat stats without those weapons entering the player economy. See **UNIFIED_ITEM_SPAWN_SYSTEM.md** and **NPC_MOB_ARCHITECTURE.md** for details.

---

## Files

```
combat/
├── __init__.py           ← package init
├── combat_handler.py     ← CombatHandler script (per-combatant), _broadcast_height_change()
├── combat_utils.py       ← execute_attack(), enter_combat(), get_sides(), get_weapon()
│                           force_drop_weapon(), _check_reach_counters()
├── height_utils.py       ← can_reach_target(), get_height_hit_modifier()
└── reactive_spells.py    ← check_reactive_shield(), check_reactive_smite()

commands/
├── all_char_cmds/
│   ├── cmd_attack.py     ← CmdAttack (attack/kill)
│   ├── cmd_flee.py        ← CmdFlee (flee)
│   └── cmd_assist.py      ← CmdAssist (assist)
├── general_skill_cmds/
│   ├── cmdset_general_skills.py  ← CmdDodge
├── npc_cmds/
│   └── cmdset_mob_combat.py  ← CmdSetMobCombat (CmdAttack + CmdDodge for mobs)
└── class_skill_cmdsets/
    ├── cmdset_warrior.py    ← CmdSetWarrior (bash, pummel, retreat, protect, taunt, offence, defence)
    ├── cmdset_thief.py      ← CmdSetThief (..., stab, ...)
    ├── cmdset_paladin.py    ← CmdSetPaladin (protect, taunt, turn)
    └── class_skill_cmds/
        ├── cmd_skill_base.py  ← CmdSkillBase (mastery dispatch + mob_func branch)
        ├── cmd_dodge.py
        ├── cmd_bash.py
        ├── cmd_pummel.py
        ├── cmd_stab.py
        ├── cmd_protect.py
        ├── cmd_taunt.py
        ├── cmd_offence.py
        ├── cmd_defence.py
        └── cmd_retreat.py

typeclasses/
├── actors/
│   ├── base_actor.py         ← effective_attacks_per_round, take_damage(), effective stats
│   └── mob.py                ← CombatMob — die(), _respawn(), at_kill()
└── mixins/
    └── effects_manager.py    ← EffectsManagerMixin — tick_combat_round(), clear_combat_effects()

tests/
└── command_tests/
    └── test_cmd_attack.py    ← 74 combat tests
```

---

## Design Notes

**Combat tick** is a fixed 4-second interval (`COMBAT_TICK_INTERVAL = 4.0` in settings). All combatants act once per tick. **Weapon speed** (0-3) affects **initiative** (turn order within a tick), not attack frequency. Initiative roll: `1d20 + effective_initiative + weapon_speed`. Higher speed = earlier action in the round. Combatants are staggered across the first 75% of the tick window based on initiative rank. **Attacks per round** come from class base + weapon mastery extra attacks, not weapon speed. Example: a longsword at MASTER grants +1 extra attack per round regardless of speed.

**Room gating:** `allow_combat = True` required for combat. `allow_pvp = True` required for PvP. Banks, inns, shops are safe zones by default.

**SLOWED special case:** Caps `attacks_per_round` to 1 at the `effective_attacks_per_round` property level and blocks off-hand attack generation — implemented as a named effect with a Condition flag (for future movement systems too).

**Save-each-round convention:** Save DC is always the caster's full contested total (`d20 + ability + mastery`), not just the raw d20 roll. Contested save makes the DC dynamic — better casters are harder to resist.

**Crit threshold:** `effective_crit_threshold = base_crit_threshold + weapon.get_mastery_crit_threshold_modifier(wielder)`. A natural roll at or above `effective_crit_threshold` is a critical hit (doubles damage before other modifiers). Default 20 for all weapons except mastery-scaled exceptions (dagger reduces threshold at EXPERT/MASTER/GM).
