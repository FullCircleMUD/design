# Dynamic Alignment System

## Overview

Alignment is a **continuous score** driven by player actions, not a static character-creation choice. Every significant act — killing mobs, completing quests, using certain abilities — nudges the score toward good or evil. Characters must actively manage their alignment to maintain access to alignment-restricted equipment, classes, and content.

---

## Score Model

| Range | Label | Colour |
|-------|-------|--------|
| +700 to +1000 | Pure Good | bright white |
| +300 to +699 | Good | green |
| -299 to +299 | Neutral | grey |
| -699 to -300 | Evil | red |
| -1000 to -700 | Pure Evil | dark red |

- **Score:** integer, clamped to [-1000, +1000]
- **New characters:** start at 0 (Neutral). No alignment selection in chargen.
- **Storage:** `alignment_score = AttributeProperty(0)` on `FCMCharacter`
- **Label:** computed property from score, displayed on score card and in spells

The existing `Alignment` enum (9-point D&D grid) is retained for backward compatibility. The `alignment` property on `FCMCharacter` derives an enum from the score so existing class/item restriction checks continue working:
- Score >= 300 maps to `NEUTRAL_GOOD`
- Score -299 to +299 maps to `TRUE_NEUTRAL`
- Score <= -300 maps to `NEUTRAL_EVIL`

The law/chaos axis is collapsed — alignment is a single good/evil dimension.

---

## Alignment Influence Sources

### Mob Kills (Primary Mechanism)

Alignment influence is a **derived property** on `CombatMob`, not a stored attribute. It's calculated from the mob's own alignment score:

```python
@property
def alignment_influence(self):
    """Kill impact on the killer's alignment. Derived from mob's alignment."""
    return -(self.alignment_score // 50)
```

**Formula:** `-(mob_alignment_score ÷ 50)`

- Killing an evil mob (negative score) gives a **positive** shift (good act)
- Killing a good mob (positive score) gives a **negative** shift (evil act)
- Killing a neutral mob (score 0) gives **no shift**

| Mob | alignment_score | alignment_influence | Rationale |
|-----|-----------------|---------------------|-----------|
| Skeleton | -1000 | +20 | Destroying pure evil undead |
| Footpad | -100 | +2 | Removing a petty criminal |
| Gnoll | -60 | +1 | Killing a savage raider |
| Kobold | -60 | +1 | Killing a hostile raider |
| Crow | -30 | +1 | Aggressive pest |
| Wolf, rat, rabbit, etc. | 0 | 0 | Neutral — survival |
| Town Guard | +600 | -12 | Murder of a good protector |

When a mob dies, alignment influence is applied to all allies on the killer's combat side (same distribution as XP). Applied in `CombatMob.die()`:

```python
influence = self.alignment_influence
if influence:
    for ally in recipients:
        if hasattr(ally, "shift_alignment"):
            ally.shift_alignment(influence)
```

### Quest Completion

Quests can specify an `alignment_influence` value in their definition. Applied when the quest is turned in.
- Help the temple, rescue villagers → positive
- Assassinate a target, steal an artifact → negative

### Other Sources (Extensible)

`shift_alignment(amount)` is the single entry point. Any system can call it:
- Crafting cursed items (negative)
- Donating to temples (positive)
- Casting necromancy spells (small negative per cast)
- Healing other players (small positive per heal)
- Pickpocketing NPCs (negative)

---

## Character Interface

```
alignment_score      — AttributeProperty(0), persisted integer
shift_alignment(n)   — clamp score to [-1000, +1000], then check equipment restrictions
alignment            — property: derives Alignment enum for backward compat
```

**Equipment check on shift:** After every alignment change, `shift_alignment()` calls `_check_alignment_equipment()` which scans all equipped items for alignment restrictions. Any item the character no longer qualifies for is forcibly unequipped with a "zapped" message.

---

## Remort

When a character remorts, they keep the same alignment score they had at the time of remort. Alignment is a persistent aspect of the character's moral journey — it does not reset.

---

## Display

- **Score card:** alignment label with colour in header row
- **Detect Alignment spell:** self-buff that shows alignment auras on characters/mobs in the room (red = Evil, gold = Good, white = Neutral)
- **Holy Insight spell:** utility spell that reveals a target's alignment as coloured text

---

## Equipment & Class Checks

### Item Restrictions (Score-Based)

Items can define `min_alignment_score` and `max_alignment_score` fields via `ItemRestrictionMixin`. Characters automatically unequip items they no longer qualify for when alignment shifts.

### Class Requirements (Score-Based)

- **Paladin:** `min_alignment_score=500` (Good required after remort)

---

## Implementation Files

| File | Role |
|------|------|
| `typeclasses/actors/character.py` | `alignment_score`, `shift_alignment()`, `alignment` property, `_check_alignment_equipment()` |
| `typeclasses/actors/mob.py` | `alignment_score` on CombatMob, `alignment_influence` derived property, applied in `die()` |
| `typeclasses/mixins/item_restriction.py` | `min_alignment_score` / `max_alignment_score` fields |
| `enums/alignment.py` | `Alignment` enum (5-tier: Pure Good → Pure Evil) |
| `commands/all_char_cmds/cmd_score.py` | Display alignment label |
| `world/spells/divine_revelation/detect_alignment.py` | Detect Alignment spell |
| `world/spells/divine_revelation/holy_insight.py` | Holy Insight spell |
| Mob definitions (e.g. `skeleton.py`, `town_guard.py`) | Set `alignment_score` values per mob type |

---

## Future Work

### Alignment Decay
Slow drift toward neutral (e.g. -1 per hour played) so maintaining extreme alignment requires ongoing commitment. Without this, players max out early and never think about alignment again.

### Paladin Power Loss
Paladins require `min_alignment_score=500` (Good) to take the class. If their alignment drops below 500 after class selection, they should lose access to divine abilities (smite, lay on hands, divine spells). Regain when score rises back above 500 — atonement mechanic. Details TBD: hard cutoff vs warning at 600 then loss at 500.

### Other Class Ability Gating
- **Necromancers** lose summoning power if alignment rises above Neutral. Dark magic requires dark commitment.
- **Druids** strongest at True Neutral — abilities weaken at extremes.

### PvP Alignment Impact
- Killing an Evil player: reduced penalty or small positive shift
- Killing a Good player: heavy negative shift
- Killing a Neutral player: moderate negative shift
- Creates a bounty-hunter dynamic for good-aligned PvPers

### Faction Reputation (Separate System)
Alignment measures moral standing. Faction reputation measures standing with specific groups (Thieves Guild, Temple of Light, City Watch). These are orthogonal — a good character could have bad standing with the Thieves Guild. Would be a separate `{faction_key: int}` dict.

### Quest & Zone Gating
- Temple quests require Good alignment
- Necromancer guild quests require Evil alignment
- Guards attack evil characters on sight; undead ignore evil characters

### Alignment-Reactive NPCs
- Shopkeepers adjust prices based on alignment
- LLM NPCs receive alignment context in their prompt, adjusting dialogue tone
- Guards become suspicious of evil-aligned characters

### Alignment-Based Spell Scaling
- Smite damage scales with alignment extremity (more good = more smite damage vs evil)
- Detect Evil / Detect Good spells read `alignment_score` directly
- Protection from Evil/Good spells check score for targeting

### Visible Aura
At extreme alignments (+/-900), characters gain a visible aura:
- Pure Good: faint golden glow in room description
- Pure Evil: shadowy aura, NPCs react with fear

### Threshold Notifications
Messages when alignment changes category:
- "You feel a darkness creeping into your soul..." (crossed from Neutral to Evil)
- "A sense of righteousness fills you." (crossed from Neutral to Good)
