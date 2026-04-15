# Character Progression

Races, character classes, point buy, and the remort long-term progression loop. This is player-facing progression — NPC/mob architecture lives in `NPC_MOB_ARCHITECTURE.md`.

## Ability Score Point Buy

All six ability scores (`strength`, `dexterity`, `constitution`, `intelligence`, `wisdom`, `charisma`) start at **8**. Players spend points from a budget to raise them during character creation. Standard 5e cost table:

| Score | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 |
|-------|---|---|----|----|----|----|----|----|
| Cost  | 0 | 1 | 2  | 3  | 4  | 5  | 7  | 9  |

- Default budget: **27 points** (`FCMCharacter.point_buy = AttributeProperty(27)`).
- Range: 8–15 before racial bonuses. Racial bonuses (from `RaceBase.ability_score_bonuses`) apply after point buy and can push scores above 15 or below 8.
- `point_buy` persists on the character — it is NOT consumed during creation. It records the character's point budget, which can grow over time via remort perks.

## Remort

When a character reaches max level, they can **remort** — reset to level 1 while keeping accumulated advantages. `num_remorts` (AttributeProperty on `FCMCharacter`) tracks how many times a character has remorted.

On remort, the player chooses from a set of perks:

- Additional point buy points (increases `point_buy` for future stat rebuilds)
- Bonus base HP, Mana, or Move
- Other advantages TBD

**Remort gates access to content:**

- Some races, classes, equipment, and items have `min_remorts` requirements.
- `ItemRestrictionMixin` already supports `min_remorts` checks: `character.num_remorts >= value`.
- `CharClassBase` already has a `min_remorts` field for class eligibility.
- Example: a legendary weapon might require `min_remorts: 5`.

This creates a long-term progression loop — characters grow more powerful across remort cycles, unlocking content that first-life characters cannot access. Infrastructure exists (num_remorts on character, min_remorts on classes/items/races); the actual remort flow UI is not yet implemented.

## Race System (`typeclasses/actors/races/`)

Auto-collecting registry pattern. Each race is a frozen `RaceBase` dataclass instance in its own file. `__init__.py` imports all race files via `from ... import *`, scans the module namespace for `RaceBase` instances, and builds `RACE_REGISTRY`. A `Race` enum is auto-generated from registry keys (`Race.HUMAN`, `Race.DWARF`, `Race.ELF`, …).

**Key fields:**

- `key`, `display_name`, `description`
- `ability_score_bonuses` (`Dict[Ability, int]`)
- `racial_weapon_proficiencies` (`List[WeaponType]`)
- `required_alignments` / `excluded_alignments` (`List[Alignment]`)
- `racial_languages` (e.g. Dwarf → dwarven, Elf → elfish, Aasimar → celestial)
- `min_remort` (int)

**Methods:**

- `at_taking_race(character)` — applies ability score bonuses via `setattr`.
- `get_valid_alignments()` — returns filtered alignment list.

**Adding a new race:**

1. Create `typeclasses/actors/races/my_race.py` with a `RaceBase` instance.
2. Add `from typeclasses.actors.races.my_race import *` to `__init__.py`.
3. Done — `Race.MY_RACE` auto-generates.

**Lookup API:** `get_race(key)`, `list_races()`, `get_available_races(num_remorts)`, `Race` enum.

## Character Class System (`typeclasses/actors/char_classes/`)

Same auto-collecting registry pattern as races. Each class is a frozen `CharClassBase` dataclass in its own file. `CLASS_REGISTRY` and the `CharClass` enum are auto-generated.

**Key fields:**

- `key`, `display_name`, `description`
- `prime_attribute` (`Ability`)
- `level_progression` (`Dict[int, Dict]` for levels 1–40)
- `multi_class_requirements` (`Dict[Ability, int]`)
- `min_remort`
- `required_races` / `excluded_races` (`List[str]`)
- `required_alignments` / `excluded_alignments` (`List[Alignment]`)
- `class_cmdset` (`Optional[Type[CmdSet]]`)

**Level progression per level:** `weapon_skill_pts`, `class_skill_pts`, `general_skill_pts`, `hp_gain`, `mana_gain`, `move_gain`.

**Methods:**

- `char_can_take_class(character)` — checks race, alignment, remort requirements.
- `at_char_first_gaining_class(character)` — adds cmdset, applies level 1 progression, initialises `db.classes[key]`.
- `at_gain_subsequent_level_in_class(character)` — increments level, applies progression, deducts `levels_to_spend`.
- `get_valid_alignments()` — filtered alignment list.

**Multiclassing:** Characters track classes in `db.classes = {"warrior": {"level": 1, ...}, "thief": {"level": 2, ...}}`. Stats stack additively.

**Lookup API:** `get_char_class(key)`, `list_char_classes()`, `get_available_char_classes(num_remorts)`, `CharClass` enum.

## Ability enum

`Ability` (`enums/abilities_enum.py`) is used across race and class systems for validation and typo prevention — reference it in `ability_score_bonuses`, `prime_attribute`, and `multi_class_requirements`.

## Character creation wizard

EvMenu-based guided flow: race → class → alignment → point buy → weapon skills → starting skills → languages → starting knowledge → name → confirm → create. Shows CON modifier in HP display and effective HP at creation.
