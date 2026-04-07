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

Each mob type carries an `alignment_influence` attribute on `CombatMob`:

```python
alignment_influence = AttributeProperty(0)  # positive = good shift, negative = evil shift
```

| Mob Type | Influence | Rationale |
|----------|-----------|-----------|
| Undead (skeleton, zombie, lich) | +20 | Destroying undead is a good act |
| Demon / fiend | +30 | Banishing evil |
| Angel / celestial | -50 | Killing a holy being is deeply evil |
| City guard / town NPC | -30 | Murder of innocents |
| Bandit / highwayman | +5 | Removing a threat to society |
| Wild animal (wolf, bear) | 0 | Neutral — survival |
| Kobold / goblin | 0 | Neutral — self-defence |

When a mob dies, alignment influence is applied to all allies on the killer's combat side (same distribution as XP). Applied in `CombatMob.die()` alongside XP.

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
shift_alignment(n)   — clamp score to [-1000, +1000]
alignment_label      — property: "Pure Good" / "Good" / "Neutral" / "Evil" / "Pure Evil"
alignment            — property: derives Alignment enum for backward compat
```

---

## Migration (Existing Characters)

Existing characters have an `Alignment` enum stored. Converted to starting score:

| Current Alignment | Starting Score |
|-------------------|----------------|
| LAWFUL_GOOD, NEUTRAL_GOOD, CHAOTIC_GOOD | +500 |
| LAWFUL_NEUTRAL, TRUE_NEUTRAL, CHAOTIC_NEUTRAL | 0 |
| LAWFUL_EVIL, NEUTRAL_EVIL, CHAOTIC_EVIL | -500 |

Handled lazily: if `alignment_score` is 0 (default) and the character has the old enum, migrate on first access.

---

## Display

- **Score card:** alignment label with colour in header row (replaces raw enum)
- **Holy Insight spell:** shows alignment label (optionally numeric score for divine casters)

---

## Equipment & Class Checks

Existing `required_alignments` / `excluded_alignments` on items and classes work unchanged — the `alignment` property returns a compatible enum. No changes to `ItemRestrictionMixin` or `CharClassBase`.

For finer-grained control in the future, items could check `alignment_score` directly (e.g. "requires score >= 500" for a holy relic).

---

## Implementation Files

| File | Change |
|------|--------|
| `typeclasses/actors/character.py` | `alignment_score`, `shift_alignment()`, `alignment_label`, update `alignment` property |
| `typeclasses/actors/mob.py` | `alignment_influence` on CombatMob, apply in `die()` |
| `commands/all_char_cmds/cmd_score.py` | Display `alignment_label` |
| `server/main_menu/chargen/chargen_menu.py` | Remove alignment selection, start at 0 |
| Mob subclasses | Set `alignment_influence` values |
| Quest definitions | Add optional `alignment_influence` field |

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
