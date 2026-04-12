# Language System

> Design doc covering the language system — how characters, NPCs, mobs, and animals speak and understand each other. The character-facing system is fully implemented. NPC/mob integration and animal language are designed but not yet built.

## Languages

8 languages, defined in `enums/languages.py`:

| Language | Racial | Learnable | Notes |
|----------|--------|-----------|-------|
| Common | — | Yes | Universal, all characters get it |
| Dwarven | Dwarf | Yes | |
| Elfish | Elf | Yes | |
| Halfling | Halfling | Yes | |
| Celestial | Aasimar | Yes | |
| Goblin | — | Yes | Spoken by goblins |
| Dragon | — | Yes | Spoken by kobolds and dragons |
| Animal | — | **No** | Non-learnable; granted only by spell/potion/item |

**Storage:** `db.languages` — a `set` of strings on any actor (e.g. `{"common", "dwarven"}`).

## Garble Engine

**File:** `utils/garble.py`

When a listener doesn't understand a language, the spoken text is replaced with language-flavoured gibberish.

**How it works:**
- Deterministic: same `(word, language)` pair always produces the same garbled output (MD5-seeded)
- Each language has a 16-syllable palette (dwarven: khor/grim/dur/thok..., elfish: ael/ith/lora/ven...)
- Punctuation, spacing, and capitalisation are preserved
- Word length is roughly matched (±2 characters)

**Example:**
- Input: `"There is treasure in the cave."` (language: dwarven)
- Listener who knows dwarven: `"There is treasure in the cave."`
- Listener who doesn't: `"Khor grim drekthok bor thurmak."`

## Speech Commands

Three commands support language switching:

### say
`say [/language] <message>` or `say [/alias] <message>`

Speaks to the room. Supports directed speech: `say to <target> <message>`.

### whisper
`whisper [/language] <target> = <message>`

Private speech to one or more targets (comma-separated). Only the target(s) hear it.

### shout
`shout [/language] <message>`

Room-wide speech that carries to adjacent rooms. Adjacent rooms hear a muffled version (first 3 words + "...") with a directional indicator ("from the north").

### Language Switches

All three commands accept a language switch via `/` syntax:
- Full name: `say/dwarven hello`
- 2-char alias: `say/dw hello`

Aliases are built from the first 2 characters of each language name.

## Comprehension Logic

Applied identically in say, whisper, and shout — per-listener:

```
understands = (language is Common)
           OR (language in listener.db.languages)
           OR (listener has COMPREHEND_LANGUAGES condition)
```

- **Understands:** listener receives the original text
- **Doesn't understand:** listener receives garbled text via `garble(text, language)`
- **Exception:** animal language uses species-specific vocalisation instead of garble (see below)

## Conditions

| Condition | Effect |
|-----------|--------|
| `COMPREHEND_LANGUAGES` | Understand all languages including animal. Wired into say/whisper/shout. No spell grants it yet. |
| `DEAF` | Receive no speech at all — skipped entirely in the message loop |
| `SILENCED` | Cannot produce speech — command blocked with error message |

## Character Language Assignment

At character creation (chargen step 9):

1. **Common** — all characters receive this automatically
2. **Racial languages** — added by race selection:
   - Dwarf → dwarven
   - Elf → elfish
   - Halfling → halfling
   - Aasimar → celestial
   - Human → none
3. **Bonus picks** — from INT modifier: `floor((INT_final - 10) / 2)`
   - INT 10 → 0 picks
   - INT 12 → 1 pick
   - INT 14 → 2 picks
   - INT_final includes racial bonuses
4. Player chooses bonus languages from the learnable set (excludes already-known and "animal")

**Files:** `typeclasses/actors/races/race_base.py` (racial assignment), `server/main_menu/chargen/chargen_menu.py` (bonus pick UI)

## LanguageMixin (TO BUILD)

Extract language attributes into a composable `LanguageMixin` applied at the base actor level so all actors — characters, NPCs, mobs, animals — share the same language infrastructure.

### Attributes

```python
class LanguageMixin:
    languages = AttributeProperty({"common"}, autocreate=False)       # set of strings
    preferred_language = AttributeProperty("common", autocreate=False) # default speaking language
```

### Actor Types

| Actor | languages | preferred_language | Notes |
|-------|-----------|-------------------|-------|
| Character | Set at chargen | "common" | Populated by race + INT bonus picks |
| NPC | Set in class/prototype | Racial language | e.g. dwarf merchant: `{"common", "dwarven"}`, preferred: "dwarven" |
| Mob | Set in class/prototype | Racial language | e.g. goblin warrior: `{"goblin", "common"}`, preferred: "goblin" |
| Animal | `{"animal"}` | "animal" | Non-learnable, species-specific rendering |

### Where to Apply

Compose into `BaseActor` (or equivalent base class) so the mixin is available to `FCMCharacter`, `BaseNPC`, `CombatMob`, `AggressiveMob`, `BasePet`, etc.

## NPC/Mob Language Behaviour (TO BUILD)

### Default Speech
NPCs and mobs speak in their `preferred_language`. A goblin NPC addresses players in Goblin. A dwarf merchant speaks Dwarven.

### Language Switching
When a character speaks to an NPC in a language the NPC knows (but isn't its preferred language), the NPC switches to that language for the remainder of the conversation.

**Example flow:**
1. Player enters goblin camp. Goblin says something → player hears garbled Goblin
2. Player `say/goblin Hello, I come in peace` → goblin understands
3. Goblin switches its conversation language to Goblin (which the player speaks)
4. Alternatively, if the player speaks Common and the goblin knows Common, player `say Hello` → goblin switches to Common

### Conversation Language Tracking
- **Transient (default):** NPC stores `ndb.conversation_language[character_key] = "common"` — resets on server restart
- **Persistent (with ai_memory):** NPC remembers "this player speaks Dwarven" across sessions via the memory system. On next meeting, the NPC greets the player in the language they last used.

### Language Barriers
NPCs that don't share a language with the player cannot be interacted with for trade, quests, or training. The NPC speaks in its preferred language, the player hears garble, and interaction is blocked until the player either:
- Learns the NPC's language
- Uses COMPREHEND_LANGUAGES (spell/potion/item)
- Finds a shared language

## Animal Language (TO BUILD)

### The "animal" Language
A special non-learnable language. All animal mobs and pets have `languages={"animal"}`, `preferred_language="animal"`. Animals don't know Common or any other language (unless they're magical/special).

### Species-Specific Vocalisation

When an animal "says" something and the listener **doesn't** understand animal, the output is **not** garbled syllables. Instead, each animal species defines its own vocalisation list — curated per-species descriptions that look like ambient flavour:

```python
# On the animal mob/pet class or prototype
animal_vocalisations = [
    "barks excitedly.",
    "whines softly.",
    "tilts its head and barks.",
    "wags its tail and yaps.",
    "sniffs the air and woofs.",
]
```

**Non-speaker sees:**
```
The dog barks excitedly.
```

**Speaker of animal sees (normal say format):**
```
The dog says: "I am hungry, do you know where there is something to hunt?"
```

The key insight: **players don't realise the animal is actually saying something** until they acquire Speak With Animals. What looked like ambient flavour turns out to be real dialogue. This creates a meaningful reveal moment when the spell/potion activates.

### Example Vocalisation Lists

**Dog:** barks excitedly, whines softly, tilts its head and barks, wags its tail and yaps, sniffs the air and woofs, pants happily, growls low in its throat

**Cat:** meows insistently, purrs contentedly, hisses, flicks its tail and mews, yawns and stretches, stares at you and meows

**Wolf:** growls menacingly, throws back its head and howls, snarls and bares its teeth, sniffs the air and whines, paces restlessly

**Horse:** whinnies, snorts and stamps a hoof, nickers softly, tosses its mane, paws at the ground

**Hawk:** screeches, ruffles its feathers and cries out, tilts its head and shrieks, clicks its beak

**Rat:** squeaks, chitters nervously, scratches at the ground, squeaks and darts about

**Crow:** caws loudly, croaks, ruffles its feathers and caws, clicks its beak

### Speak With Animals

Spell, potion, or magic item that temporarily adds "animal" to the character's `db.languages` set.

- **While active:** animal speech renders as normal say — `The dog says: "I smell food nearby. Follow me!"`
- **While inactive:** species-specific vocalisation — `The dog barks excitedly.`

This naturally integrates with the existing comprehension logic — no special cases needed in say/whisper/shout beyond the different rendering for animal vocalisations.

### Animal Intelligence Tiers (future)

Not all animals are equally articulate:
- **Basic** (rats, insects): simple phrases — hunger, fear, territory
- **Moderate** (dogs, horses): can follow instructions, report what they've seen
- **High** (dolphins, whales): in-depth conversations, lore, quest information
- **Legendary** (dragons, phoenixes): full intelligence, may be the only source of certain quest info

Intelligence tier affects what the animal actually *says* when you can understand it, not the vocalisation output (that's always species-specific for non-speakers).

## Implementation Files

### Existing (implemented)
- `enums/languages.py` — Languages enum (7 languages, add "animal")
- `utils/garble.py` — garble engine (add animal vocalisation path)
- `commands/all_char_cmds/cmd_say.py` — say with language switches
- `commands/all_char_cmds/cmd_whisper.py` — whisper with language switches
- `commands/all_char_cmds/cmd_shout.py` — shout with language switches
- `commands/all_char_cmds/cmd_languages.py` — list known languages
- `typeclasses/actors/races/race_base.py` — racial language assignment
- `server/main_menu/chargen/chargen_menu.py` — chargen language selection
- `enums/condition.py` — COMPREHEND_LANGUAGES, DEAF, SILENCED

### To Build
- `typeclasses/mixins/language_mixin.py` — LanguageMixin (extract from character)
- Add "animal" to Languages enum with `learnable=False` flag
- Add animal vocalisation rendering to say/whisper/shout (override garble for animal language)
- Add `animal_vocalisations` attribute to animal mob/pet classes
- Add `preferred_language` and language-switching logic to NPC speech
- Comprehend Languages spell (`world/spells/divination/`)
- Translation magic items (Amulet of Translation, Amulet of Beast Speech)
- Wire Speak With Animals spell stub (`world/spells/nature_magic/speak_with_animals.py`)

### Tests
- `tests/command_tests/test_cmd_say.py` — 11+ tests
- `tests/command_tests/test_cmd_whisper.py` — 6+ tests
- `tests/command_tests/test_cmd_shout.py` — 6+ tests
- `tests/command_tests/test_cmd_languages.py` — 5 tests
- New tests needed for: NPC language switching, animal vocalisation rendering, LanguageMixin
