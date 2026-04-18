# Effects System — Design & Architecture

## Overview

The `EffectsManagerMixin` (`typeclasses/mixins/effects_manager.py`) is the unified effect system for all actors. It replaces the old ConditionsMixin and provides three composable layers for applying, tracking, and removing effects. Mixed into `BaseActor` alongside `DamageResistanceMixin`.

> **Combat integration:** See `design/COMBAT_SYSTEM.md` for named effect specifications (values, durations, stacking rules) and combat-specific effect tables. See `design/SPELL_SKILL_DESIGN.md` for spell-granted effects.

**MANDATORY: The EffectsManagerMixin is the ONLY approved system for applying, tracking, and removing effects on actors. Use convenience methods when one exists; fall back to `apply_named_effect()` only for data-driven effects (e.g. potions). Use `apply_effect()`/`remove_effect()` only for equipment `wear_effects`. Do NOT create ad-hoc solutions (custom scripts, manual HP/stat manipulation, direct condition flag manipulation outside of equipment wear_effects, or any other pattern that bypasses this system). If you believe a deviation is necessary, you MUST discuss it with the developer and get explicit human approval before proceeding.**

---

## Layer 1 — Condition Flags (ref-counted)

Reference-counted condition flags. Multiple sources of the same condition don't collide — DARKVISION from a racial innate (count=1) plus a spell (count=2) won't be lost when the spell expires (count back to 1).

- `_add_condition_raw(condition)` → silent internal use, returns `True` if newly gained
- `_remove_condition_raw(condition)` → silent internal use, returns `True` if fully removed
- `add_condition(condition)` → public API, returns `True` if newly gained. BaseActor overrides for messaging + side effects
- `remove_condition(condition)` → public API, returns `True` if fully removed. BaseActor overrides for messaging + side effects
- `has_condition(condition)` → `bool` (count > 0)
- `get_condition_count(condition)` → raw ref count
- Item-granted conditions: `{"type": "condition", "condition": "darkvision"}` in `wear_effects`
- Condition enum: `enums/condition.py` — 11 actively-used condition flags with start/end messages. Cross-references `enums/named_effect.py`.

---

## Layer 2 — Stat Effect Dispatch (backward compatible, mostly superseded)

- `apply_effect(effect_dict)` → applies a single effect incrementally (stat_bonus, damage_resistance, condition, hit_bonus, damage_bonus)
- `remove_effect(effect_dict)` → symmetric reversal
- **Mostly superseded by nuclear recalculate:** Equipment `at_wear()`/`at_remove()` and named effects now only use `apply_effect()`/`remove_effect()` for condition-type effects (ref-counting). All numeric stat effects are handled by `_recalculate_stats()` instead.
- Still used directly for: condition application in at_wear/at_remove loops, backward compatibility in tests

---

## Layer 3 — Named Effects

Tracked, timed, anti-stacking effects that compose from the three building blocks (condition flag + stat effects + lifecycle). One call sets everything up; removal reverses everything cleanly.

### Effect Registry (Single Source of Truth)

Each `NamedEffect` enum member carries its associated condition and duration type via properties on the enum:

- `NamedEffect.INVISIBLE.effect_condition` → `Condition.INVISIBLE`
- `NamedEffect.SHIELD.effect_duration_type` → `"combat_rounds"`

Registry dicts (`_EFFECT_CONDITIONS`, `_EFFECT_DURATION_TYPES`) in `enums/named_effect.py` define these mappings. When `apply_named_effect()` is called, condition and duration_type auto-fill from the registry unless explicitly overridden.

**Sentinel semantics** (`_UNSET` sentinel in `effects_manager.py`):
- `condition=_UNSET` (default) → auto-fill from registry
- `condition=None` (explicit) → override registry, no condition
- `condition=Condition.X` (explicit) → override registry with this condition

### Convenience Methods (Preferred API)

**ALWAYS use convenience methods when one exists for the effect.** Each method is the single entry point for its effect — any source (spell, weapon, potion, scroll, trap) that applies the effect calls the same method. This guarantees consistency.

| Category | Method | Parameters |
|---|---|---|
| **Combat conditions** | `apply_stunned()` | `duration_rounds, source=None` |
| | `apply_prone()` | `duration_rounds, source=None` |
| | `apply_slowed()` | `duration_rounds, source=None` |
| | `apply_paralysed()` | `duration_rounds, source=None, save_dc=None, save_stat="wisdom", save_messages=None, messages=None` |
| | `apply_entangled()` | `duration_rounds, source=None, save_dc=None, save_stat="strength", save_messages=None, messages=None` |
| | `apply_blinded()` | `duration_rounds, source=None, save_dc=None, save_stat="constitution", save_messages=None, messages=None` |
| | `apply_frightened()` | `duration_rounds, source=None, save_dc=None, save_stat="wisdom", save_messages=None, messages=None` |
| | `apply_blurred()` | `duration_rounds` |
| **Combat stat effects** | `apply_shield_buff()` | `ac_bonus, duration_rounds, mana_cost=0` |
| | `apply_staggered()` | `hit_penalty, duration_rounds, source=None` |
| | `apply_sundered()` | `ac_penalty, duration_rounds, source=None` |
| **Seconds-based buffs** | `apply_invisible()` | `duration_seconds` |
| | `apply_sanctuary()` | `duration_seconds` |
| | `apply_armor_buff()` | `ac_bonus, duration_seconds` (alias: `apply_mage_armor`, shared by Mage Armor + Divine Armor) |
| | `apply_shadowcloaked()` | `stealth_bonus, duration_seconds, source=None` |
| | `apply_blessed()` | `hit_bonus, save_bonus, duration_seconds` |
| | `apply_true_sight()` | `duration_seconds, detect_invis=False` |
| | `apply_holy_sight()` | `duration_seconds, detect_invis=False` |
| | `apply_darkvision_buff()` | `duration_seconds` (shared by Darkvision + Divine Sight) |
| | `apply_water_breathing_buff()` | `duration_seconds` |
| | `apply_resist_element()` | `element, resistance_pct, duration_seconds, source=None` |
| **Script-managed** | `apply_poisoned()` | `ticks` |
| | `apply_acid_arrow_dot()` | `dot_rounds` |
| | `apply_vampiric()` | `source=None` |
| **Stances** | `apply_offensive_stance()` | `effects, source=None` |
| | `apply_defensive_stance()` | `effects, source=None` |

### break_effect() — Force Removal Without Messages

`break_effect(named_effect)` removes an effect immediately: zeros condition refs, calls `_recalculate_stats()` to rebuild numeric stats, stops timers. Does NOT send end messages — the caller handles context-specific messaging (e.g. "You attack and your invisibility shatters!").

```python
target.break_effect(NamedEffect.INVISIBLE)  # or break_invisibility() alias
target.break_effect(NamedEffect.SANCTUARY)  # or break_sanctuary() alias
```

### Low-Level API

- `apply_named_effect(key, source=None, effects, condition, duration, duration_type, messages, save_dc, save_stat, save_messages)` → accepts `NamedEffect` enum or string key. Auto-fills condition and duration_type from registry via `_UNSET` sentinel. Applies conditions incrementally, then calls `_recalculate_stats()` for numeric stat effects. Returns `True` if applied, `False` if already active (anti-stacking).
- `remove_named_effect(key)` → clears condition flag, calls `_recalculate_stats()` to rebuild numeric stats without removed effect, sends end messages, cleans up timers. Returns `True` if removed.

**Use `apply_named_effect()` directly only when:** no convenience method exists (e.g. data-driven potions where effect key, stat bonuses, and condition come from item data baked at craft time).

**Anti-stacking rule of thumb:** Effects with stat bonuses (AC, damage, etc.) MUST anti-stack — double Shield = double AC is broken. Effects that only set a boolean condition flag (INVISIBLE, DETECT_INVIS) with no stat impact do NOT need anti-stacking — conditions are ref-counted, so multiple sources just increment the count. `has_condition()` is a boolean gate (count > 0), so ref count 1 and ref count 50 behave identically. Multiple sources = safe redundancy. When the condition needs to be broken (e.g. attacking breaks invisibility), zero the entire ref count rather than decrementing.

- `has_effect(key)` → `bool` check if a named effect is active
- `get_named_effect(key)` → full effect record or `None`
- `tick_combat_round()` → decrements all `combat_rounds` effects by 1, auto-removes expired. Called by combat handler each tick.
- `clear_combat_effects()` → removes ALL `combat_rounds` effects. Called on combat end by `stop_combat()`.

### Size Effects

Actor size follows the same base/active pattern as ability scores: `base_size` (permanent racial/creature size) and `size` (active, rebuilt by `_recalculate_stats()`). Enlarge/shrink spells modify `size` during effect accumulation; on expiry the recalculate resets it to `base_size`. Size changes have gameplay consequences beyond stats — exit `max_size` gating blocks actors that are too large for a passage (see [EXIT_ARCHITECTURE.md](EXIT_ARCHITECTURE.md) § Size gating).

### Named Effect Record

Persisted in `active_effects` AttributeProperty dict:

```python
active_effects = {
    "shield": {
        "condition": None,           # optional Condition flag
        "effects": [{"type": "stat_bonus", "stat": "armor_class", "value": 4}],
        "duration": 2,               # rounds/seconds remaining
        "duration_type": "combat_rounds",  # or "seconds" or None
        "messages": {
            "start": "A barrier forms!",
            "end": "The barrier fades.",
            "start_third": "{name} is surrounded by a barrier!",
            "end_third": "The barrier around {name} fades.",
        },
    },
}
```

### Lifecycle Types

| `duration_type` | Mechanism | Cleanup |
|---|---|---|
| `"combat_rounds"` | Combat handler calls `actor.tick_combat_round()` each tick | `clear_combat_effects()` on combat end |
| `"seconds"` | EffectTimerScript (one-shot timer) auto-removes on expiry | Timer calls `remove_named_effect(key)` |
| `None` | Permanent until explicitly removed | Caller responsible |

---

## Decision Tree — When to Use What

| Scenario | Method | Example |
|---|---|---|
| Equipment bonuses | `apply_effect()` / `remove_effect()` in `wear_effects` | Sword +1 AC |
| Combat condition (stun/prone/slow/etc.) | Convenience method: `target.apply_stunned(rounds)` | Unarmed stun (1 round) |
| Timed combat buff (AC/hit/etc.) | Convenience method: `target.apply_shield_buff(ac, rounds)` | Shield (+4 AC, 2 rounds) |
| Seconds-based buff | Convenience method: `target.apply_invisible(seconds)` | Invisibility spell (300s) |
| Script-managed effect | Convenience method: `target.apply_poisoned(ticks)` | Blowgun poison DoT |
| Force-remove without messages | `target.break_effect(NamedEffect.INVISIBLE)` | Attack breaks invisibility |
| Data-driven effect (potions) | `apply_named_effect()` directly (auto-fill from registry) | STR potion (+2 STR, 300s) |
| Direct condition (legacy) | `add_condition()` / `remove_condition()` | Equipment `wear_effects` |

---

## NamedEffect Enum (enums/named_effect.py)

All named effect keys are validated against the `NamedEffect` enum — unknown keys are rejected with a `ValueError`. To add a new named effect:

1. Check the **unsorted effects list** at the bottom of `enums/named_effect.py`
2. Classify the effect: NamedEffect (lifecycle-managed), Condition (ref-counted flag), or both
3. Add a member to the appropriate enum(s) with detailed usage comments
4. Remove the entry from the unsorted list
5. Add start/end messages to the message registries
6. Add entry to `_EFFECT_CONDITIONS` if the effect has an associated Condition
7. Add entry to `_EFFECT_DURATION_TYPES` (`"combat_rounds"`, `"seconds"`, or `None`)
8. Add a convenience method on EffectsManagerMixin (e.g. `apply_my_effect()`)

**Current NamedEffect members:** STUNNED, PRONE, SLOWED, PARALYSED, ENTANGLED, POISONED, ACID_ARROW, SHIELD, MAGE_ARMORED, BLURRED, INVISIBLE, TRUE_SIGHT, SHADOWCLOAKED, SANCTUARY, VAMPIRIC, STAGGERED, SUNDERED, OFFENSIVE_STANCE, DEFENSIVE_STANCE, RESIST_FIRE, RESIST_COLD, RESIST_LIGHTNING, RESIST_ACID, RESIST_POISON, POTION_STRENGTH, POTION_DEXTERITY, POTION_CONSTITUTION, POTION_INTELLIGENCE, POTION_WISDOM, POTION_CHARISMA, POTION_TEST

**Current Condition members:** SILENCED, DEAF, HIDDEN, INVISIBLE, DETECT_INVIS, DARKVISION, FLY, WATER_BREATHING, HASTED, COMPREHEND_LANGUAGES, CRIT_IMMUNE, SANCTUARY, PARALYSED, SLOWED

SLOWED and PARALYSED appear in both — named effect for lifecycle, condition flag for gameplay checks (movement speed / Remove Paralysis spell targeting).

---

## Named Effect On-Apply Callbacks (enums/named_effect.py)

Side effects that must ALWAYS happen when a named effect is applied are registered as callbacks in `_ON_APPLY_CALLBACKS` at the bottom of `enums/named_effect.py`. This ensures consistency regardless of whether the effect is applied by a weapon, spell, command, or trap.

Callbacks receive `(target, source, duration)` and are called automatically by `apply_named_effect()` after the effect is recorded.

| Effect | Callback | Side Effect |
|---|---|---|
| PRONE | `_grant_advantage_to_enemies` | All enemies of target get advantage for duration rounds |
| ENTANGLED | `_grant_advantage_to_enemies` | All enemies of target get advantage for duration rounds |
| PARALYSED | `_grant_advantage_to_enemies` | All enemies of target get advantage for duration rounds |
| BLINDED | `_grant_advantage_to_enemies` | All enemies of target get advantage for duration rounds |
| SLOWED | `_apply_slowed` | Reduces target initiative / action economy for duration rounds |
| STUNNED | None | Action denial only — no advantage (key differentiator from prone) |
| All others | None | No mechanical side effects beyond core effect system |

To add a new callback: define the function in `named_effect.py` and add it to `_ON_APPLY_CALLBACKS`. Use lazy imports inside callbacks to avoid circular dependencies.

---

## Early Cancellation (Dispel Pattern)

To cancel a named effect early (e.g. dispel magic, cleanse, death):

```python
# Remove a single effect — reverses stats, clears condition, sends end messages
target.remove_named_effect("shield")

# Remove ALL combat-round effects (used by stop_combat)
target.clear_combat_effects()

# Check before removing (e.g. targeted dispel)
if target.has_effect("slowed"):
    target.remove_named_effect("slowed")
```

`remove_named_effect()` is fully symmetric — it reverses everything `apply_named_effect()` set up: stat effects, condition flags, timers, and messaging. Safe to call even if the effect isn't active (returns `False`).

---

## Two Code Paths (Both Correct)

| Path | Condition handling | Messaging | Used by |
|---|---|---|---|
| Named effects | `_add_condition_raw` (silent) | Effect's `messages` dict | Spells, abilities, stun/prone |
| Legacy | `add_condition` (BaseActor override) | Condition enum messages | Equipment `wear_effects`, direct condition calls |

---

## When NOT to Add a Condition Flag

A `Condition` flag is a ref-counted boolean for **gameplay mechanics that check state** — can this actor fly? are they hidden? can they breathe underwater? The game code calls `has_condition()` to make decisions.

**Do NOT add a Condition flag just for spell targeting or status display.** The named effect system already provides `has_effect()` which serves the same purpose. Adding a Condition flag that nothing checks for gameplay purposes creates dead processing — a flag that just gets set and cleared with no mechanic consuming it. This is exactly the fragmentation the unified effects system was built to eliminate.

**Example — Poison DoT (CORRECT):**
Poison is a **NamedEffect only**. A "Remove Poison" spell checks `has_effect("poisoned")`. The status display checks `has_effect("poisoned")`. There is no `Condition.POISONED` because no gameplay mechanic needs a ref-counted condition flag — nothing in the codebase calls `has_condition(Condition.POISONED)` to make a gameplay decision.

**Example — SLOWED (CORRECT dual-system):**
SLOWED is **both** a NamedEffect AND a Condition because a future movement speed system needs `has_condition(Condition.SLOWED)` to reduce movement speed. The named effect manages lifecycle; the condition flag exists because a concrete gameplay mechanic will check it.

**Rule of thumb:** If the only consumers of the flag would be "Remove X" spells or status display, use `has_effect()` — that's what it's for. Only add a `Condition` when a **separate gameplay system** needs to check the state (movement, visibility, breathing, speech, etc.).

---

## Poison Timing Fork

When an effect could be applied in or out of combat, fork `duration_type` at apply time based on context rather than building hybrid timing:

```python
# In blowgun at_hit(): check target's combat state at the moment of application
if target.scripts.get("combat_handler"):
    duration_type = "combat_rounds"
else:
    duration_type = "seconds"

target.apply_named_effect(
    key="poisoned",
    duration=poison_ticks,
    duration_type=duration_type,
    ...
)
```

This keeps poison within the unified effect system. The 99% case is combat (attack → combat starts). The edge case (invisible attacker, no combat) uses seconds. Both paths use `apply_named_effect()` — no custom scripts, no hybrid mechanisms.

---

## Examples

```python
# ── Convenience methods (PREFERRED) ──────────────────────────
# Combat condition — weapon stun
target.apply_stunned(1, source=attacker)

# Combat buff — reactive shield
wielder.apply_shield_buff(ac_bonus=4, duration_rounds=2, mana_cost=5)

# Seconds-based buff — invisibility spell
caster.apply_invisible(duration_seconds=300)

# Combat debuff — weapon sunder
target.apply_sundered(ac_penalty=-2, duration_rounds=2, source=attacker)

# Elemental resistance — resist spell
target.apply_resist_element("fire", resistance_pct=40, duration_seconds=30, source=caster)

# Force-remove without messages — attack breaks invisibility
target.break_invisibility()  # alias for break_effect(NamedEffect.INVISIBLE)

# ── Low-level apply_named_effect (data-driven potions) ───────
# Potion: effect key, stats, condition come from item data
consumer.apply_named_effect(
    key=effect_key,          # e.g. "potion_strength"
    effects=stat_effects,    # from potion_effects data
    condition=condition,     # from potion_effects data (explicit override)
    duration=self.duration,  # duration_type auto-fills from registry
    messages={"end": f"The effects of {self.key} wear off."},
)

# Early cancellation (e.g. dispel magic) — reverses everything + sends end messages
target.remove_named_effect("shield")

# Remove ALL combat effects (on combat end)
target.clear_combat_effects()
```

---

## Automatic Condition Messaging (BaseActor overrides)

`BaseActor.add_condition()` and `remove_condition()` override the mixin to send messages:
- **First person:** `self.msg(cond_enum.get_start_message())` / `get_end_message()`
- **Third person:** `self.location.msg_contents(cond_enum.get_start_message_third_person(self.key), ...)` / `get_end_message_third_person()`

Visibility-aware timing prevents conditions from filtering their own announcements:
- **add_condition:** snapshots `was_hidden`/`was_invisible` BEFORE incrementing — gaining INVISIBLE itself is seen by everyone
- **remove_condition:** checks `has_condition(HIDDEN)`/`has_condition(INVISIBLE)` AFTER decrementing — losing INVISIBLE itself is seen by everyone

### Condition-Specific Side Effects (BaseActor.add/remove_condition)

Beyond messaging, certain conditions trigger gameplay effects on gain/loss:
- **FLY removal:** `_check_fall()` — if `room_vertical_position > 0`, character falls to ground (position reset to 0) with 10 HP damage per height level. Water rooms (`max_depth < 0`) absorb the first 20 HP of fall damage (`WATER_FALL_ABSORB`) — low falls into water are harmless (splash message), high falls still hurt
- **WATER_BREATHING gain:** `stop_breath_timer()` — cancels active underwater breath timer
- **WATER_BREATHING removal:** `start_breath_timer()` — starts breath timer if `room_vertical_position < 0` (underwater)

---

## DamageResistanceMixin (typeclasses/mixins/damage_resistance.py)

Provides damage resistance/vulnerability tracking for any typeclass — characters, NPCs, mobs, pets, mounts, destructible objects.

- `damage_resistances` — AttributeProperty dict of raw integer percentages (e.g. `{"piercing": 50, "fire": -25}`)
- `get_resistance(damage_type)` — returns effective value clamped to [-75, 75]. Returns 0 for missing types.
- `apply_resistance_effect(effect)` / `remove_resistance_effect(effect)` — add/subtract from raw dict
- Raw values stored unclamped to prevent drift (see module docstring for detailed explanation)

Mixed into BaseActor: `class BaseActor(EffectsManagerMixin, DamageResistanceMixin, DefaultCharacter)` — EffectsManagerMixin replaces the old ConditionsMixin and absorbs `apply_effect`/`remove_effect`.

## Damage Pipeline — Split Architecture (typeclasses/actors/base_actor.py)

The damage system is split into three methods for correct message ordering:

### `calculate_damage(raw_damage, damage_type=None, ignore_resistance=False) → int`

Pure calculation — returns final damage after resistance/vulnerability. **No side effects.** Does not modify HP, does not trigger death.

Use when you need the damage amount before applying it (e.g. to broadcast a hit message before the target dies and gets deleted from the DB).

### `apply_damage(damage, cause="combat", killer=None) → int`

Applies pre-calculated damage. Subtracts HP, triggers wimpy auto-flee, triggers `die()` on death. **Side effects happen here.**

### `take_damage(raw_damage, damage_type=None, cause="combat", ignore_resistance=False, killer=None) → int`

Convenience wrapper — calls `calculate_damage()` then `apply_damage()` in one call. **Backward compatible** — all existing callers (spells, traps, DoTs, fall damage, drowning, cleave) use this and require zero changes.

**When to use which:**
- **Most callers:** Use `take_damage()` — simple, one call does everything.
- **`execute_attack()` in combat:** Uses the split methods. Calculates damage first, broadcasts the hit message, then applies damage. This ensures "Bob hits a rabbit for 5 damage!" appears before "a rabbit has been slain!" — critical because mobs are deleted from the DB on death and can't be referenced afterward.

**Resistance/Vulnerability Rules:**
- Positive resistance: `reduction = max(1, floor(damage * resistance / 100))` — even 1% always saves at least 1 HP
- Negative resistance (vulnerability): `extra = max(1, floor(damage * abs(resistance) / 100))` — even -1% always adds at least 1 HP
- Final damage: `max(1, damage)` — minimum 1 HP always dealt
- If min-1-damage and min-1-resistance collide, min damage wins

**Parameters (take_damage / calculate_damage):**
- `damage_type` — string (e.g. `"fire"`, `"piercing"`) for `get_resistance()` lookup. `None` skips resistance.
- `cause` — passed to `die()` on death (`"combat"`, `"spell"`, `"fall"`, `"drowning"`)
- `ignore_resistance` — `True` for environmental damage (fall, drowning) that bypasses all resistances
- `killer` — the entity that dealt the killing blow (for XP, kill hooks)

**Callers using `take_damage()` (backward compat — no changes needed):**
- `apply_spell_damage()` in `world/spells/spell_utils.py`
- `_check_fall()` in `base_actor.py` — fall damage
- `BreathTimerScript` — drowning damage
- `PoisonDoTScript`, `AcidDoTScript` — DoT ticks
- `TrapMixin.trigger_trap()` — trap damage
- Greatsword/Battleaxe cleave — AoE follow-up hits

**Caller using split methods:**
- `execute_attack()` in `combat/combat_utils.py` — calculates first, broadcasts hit, then applies

---

## Files

```
typeclasses/mixins/
├── effects_manager.py      ← EffectsManagerMixin (3 layers, convenience methods, named effects)
├── damage_resistance.py    ← DamageResistanceMixin (resistance/vulnerability tracking)

typeclasses/actors/
├── base_actor.py           ← BaseActor (add_condition/remove_condition overrides, take_damage)

enums/
├── condition.py            ← Condition enum (ref-counted flags)
├── named_effect.py         ← NamedEffect enum, registry dicts, on-apply callbacks
├── damage_type.py          ← DamageType enum
```
