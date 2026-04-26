# Crafting & Processing System — Design & Architecture

## Overview

Crafting and processing are two related but distinct systems for transforming resources into goods. Processing converts raw resources into refined resources at a gold cost. Crafting creates NFT items from resources (and optionally NFT components) using character skill mastery.

> **Recipes & skills:** See `design/SPELL_SKILL_DESIGN.md` for the full recipe catalog per crafting skill and mastery tier. This document covers the system architecture.

---

## Recipe System (world/recipes/)

Recipes are Python dicts registered in `world/recipes/__init__.py`. One file per recipe, organised by skill subdirectory (carpentry, blacksmithing, leatherworking, tailoring, alchemy, jewellery, enchanting). 52+ recipes across 7 skill categories.

```python
# world/recipes/carpentry/training_longsword.py
RECIPE_TRAINING_LONGSWORD = {
    "recipe_key": "training_longsword",
    "name": "Training Longsword",
    "skill": skills.CARPENTER,
    "crafting_type": RoomCraftingType.WOODSHOP,
    "min_mastery": MasteryLevel.BASIC,
    "ingredients": {7: 3},          # 3 Timber (resource ID → quantity)
    "output_prototype": "training_longsword",
}

# Recipes can also consume NFT items as ingredients:
RECIPE_SPEAR = {
    "recipe_key": "spear",
    "name": "Spear",
    "skill": skills.BLACKSMITH,
    "crafting_type": RoomCraftingType.SMITHY,
    "min_mastery": MasteryLevel.BASIC,
    "ingredients": {5: 1},          # 1 Iron Ingot
    "nft_ingredients": {"shaft": 1},  # 1 Shaft (carpenter component NFT)
    "output_prototype": "spear",
}
```

**Helpers:** `get_recipe(key)`, `get_recipes_for_crafting_type(type)`, `get_recipes_for_skill(skill)`, `get_recipe_by_output_prototype(prototype_key)` (reverse lookup), `compute_repair_cost(recipe)` (auto-compute or explicit `repair_ingredients`)

---

## Enchanting System (world/recipes/enchanting/)

Enchanting is a **mage-only** crafting skill (`skills.ENCHANTING`) that transforms vanilla items into enchanted variants with magical effects. Key design decisions:

**Recipes auto-granted by mastery level** — no recipe scrolls for enchanting (unlike other crafting skills). Implemented via `_get_auto_granted_enchanting_recipes()` in [`typeclasses/mixins/recipe_book.py`](../src/game/typeclasses/mixins/recipe_book.py) — `knows_recipe()` checks both the explicit recipe book AND the auto-granted set, returning every enchanting recipe at or below the character's `class_skill_mastery_levels[ENCHANTING].mastery`. No scrolls, no migration entries, no manual learn step.

**Item split — vanilla vs enchanted:**
- **Vanilla items** (tailored/leathered): simple names (Bandana, Kippah, Cloak, Veil, Scarf, Sash, Leather Cap, Leather Gloves), no effects, no class restrictions. Crafted by tailors/leatherworkers.
- **Enchanted items**: named variants (Rogue's Bandana, Sage's Kippah, etc.) with effects and optional class restrictions. Created by enchanters transforming vanilla items.
- **Non-enchantable items** keep their effects baked in: Gambeson (+1 AC, excludes mage), Coarse Robe (+10 mana), Brown Corduroy Pants (+10 move), Warrior's Wraps (+10 HP, excludes mage/cleric/thief).

**Three tiers of enchanting ingredients (planned):**
- BASIC/SKILLED: Arcane Dust (resource ID 16) — 2 per recipe
- EXPERT/MASTER: mid-game ingredient (TBD)
- GRANDMASTER: late-game ingredient (TBD)

**Enchanting scope:**
- **Wearables**: deterministic (fixed recipe per item — vanilla + dust → named enchanted variant). Effects baked into the enchanted prototype; do not scale by enchanter mastery.
- **Gems**: pre-disclosed multi-effect cascade (compliance-driven slot model). Enchanter turns raw gem + dust into "Enchanted Ruby/Emerald/Diamond" with effects and restrictions visible BEFORE the player commits. See *Gem Enchanting Architecture* below and [ECONOMY.md § Compliance-Driven Design](ECONOMY.md) for the full model.
- **Weapons**: NOT directly enchanted — get enchanted gems inset by jeweller instead.

**Effects on enchanted gems are intentionally hidden in inventory display** — drives demand for identify spells/scrolls. See `project_gem_identity_intentional.md`.

**Room type:** `RoomCraftingType.WIZARDS_WORKSHOP` — enchanting-specific crafting rooms.

**Command alias:** `enchant` (full-word alias on `cmd_craft.py`; abbreviations stripped — players create their own shortcuts via `alias`).

---

## Gem Enchanting Architecture (world/recipes/enchanting/gem_tables.py)

**Two orthogonal axes drive the system:**

- **Gem quality** (ruby / emerald / diamond) → magnitude per atom and which atoms are in the pool
- **Enchanter skill (relative to gem complexity)** → cascade probability (how many effects fire) and restriction probability (how restricted the result is)

A GM enchanter using a ruby is at the top of their craft (max cascade, zero restrictions). The same GM using a diamond is at the bottom of diamond enchanting (cascade matches an EXPERT ruby). Future tiers below GM diamond would extrapolate the same pattern.

### Output: standard wear_effects + ItemRestrictionMixin fields

The rolling system produces output that maps **directly** onto the existing item systems — no parallel infrastructure, no custom storage:

- **wear_effects** (list) — same dict shape used by every other item's `wear_effects` field (see e.g. `veil_of_grace`, `runeforged_chain`, `n95_mask`). On inset, this list extends the weapon's existing `wear_effects` array; the standard equip pipeline (`_recalculate_stats`, condition ref-counting, resistance accumulation) handles everything.
- **restrictions** (dict) — keys are `ItemRestrictionMixin` field names (`required_classes`, `excluded_classes`, `required_races`, `excluded_races`, `min_alignment_score`, `max_alignment_score`). Applied to the gem's mixin fields directly; on inset, merged into the weapon's same fields. Existing `can_use()` enforcement does the rest.

### Data tables (rebalance by editing data only)

`gem_tables.py` exposes:

- **`WEAR_EFFECT_TEMPLATES`** — catalog of partial wear_effect dicts (stat_bonus on each ability/pool/combat stat, damage_resistance for each of 13 damage types, condition flags). Magnitude (where applicable) is filled in per gem type at roll time.
- **`GEM_POOL_WEIGHTS`** — which templates each gem type can roll, with weights (uniform = 1 today). Ruby pool = stat bonuses + resistances + 5 utility conditions. Emerald pool = ruby + movement/sense conditions + regen. Diamond pool = emerald + hasted.
- **`GEM_MAGNITUDES`** — per-gem-type integer magnitudes. Ability scores +1/+2/+3, pool maxes 10/15/20, combat stats +1/+3/+5, save_bonus +1/+2/+3 (ability-scale), crit_chance -1/-2/-3 (negative = wider crit), resistances 10/20/30 (integer percent matching n95_mask convention), regen 3x/5x multiplier (emerald/diamond only).
- **`CASCADE_PROBABILITIES`** and **`RESTRICTION_PROBABILITIES`** — keyed by `(gem_type, mastery_level)`. Cascade climbs with mastery and degrades with rarer gem type; restrictions decline with mastery and worsen with rarer gem type.

### wear_effect rolling (`roll_wear_effects`)

Procedural cascade. Primary template always rolls; secondary and tertiary by per-(gem, mastery) probability. Duplicate template = no-op (slot still advances). Each rolled template has its magnitude filled in from `GEM_MAGNITUDES` and is emitted as a fully-formed wear_effect dict ready to drop into `wear_effects`.

### Restriction rolling (`roll_restrictions`)

Procedural cascade up to 3 categories drawn without replacement from {class, race, alignment}. Each category appears at most once. For class/race: 60% MUST_NOT_BE / 40% MUST_BE polarity (mapped to `excluded_*` / `required_*` fields), value drawn from base classes/races. For alignment: weighted mode pick (no_evil 30% / no_good 30% / evil_only 15% / good_only 15% / neutral_only 10%) translated to `min/max_alignment_score` bounds with `>=` boundary semantics (`good_only` admits alignment 300, `evil_only` admits -300, `neutral_only` is strictly between). Restriction count is intentionally NOT decoupled from gem complexity — restrictions create item identity; rare gems with restrictions are different valuable items, not worse ones.

### Item handling

- One `NFTItemType` per gem tier (Enchanted Ruby, Enchanted Emerald, Enchanted Diamond) — NOT one per outcome variant. The rolled outcome is per-instance state.
- After spawn, the gem's `wear_effects` (via `db.wear_effects` since `BaseNFTItem` doesn't carry `WearableMixin`) and ItemRestrictionMixin fields are populated directly from the rolled outcome. No custom `gem_effects` / `gem_restrictions` attributes.
- Recipe declares `output_table` (e.g. `"enchanted_ruby"`) which `OUTPUT_TABLE_TO_GEM_TYPE` maps to a gem_type for the rolling functions.

### Compliance: pre-disclosure via EnchantmentSlot (shipped)

Gem outcomes are NOT rolled at craft time. The system maintains one `EnchantmentSlot` row per `(output_table, mastery_level)` pair holding the next pre-disclosed outcome. `cmd_craft.py` previews the slot before the player confirms; on confirm, `EnchantmentService.consume_slot()` atomically commits the previewed outcome and rolls the next one — so the displayed slot is exactly what gets applied. Race-loss costs the player nothing (materials only consume after slot consumption succeeds). Full design and SQL pattern in [ECONOMY.md § Compliance-Driven Design: Pre-Disclosure with Slot Consumption](ECONOMY.md).

### Implementation status

| Piece | Status |
|---|---|
| `EnchantmentSlot` model + migration + service (`preview_slot`/`consume_slot`) | Shipped |
| Slot-based preview/commit wired into `cmd_craft.py` (race-loss aborts cleanly) | Shipped |
| Atom catalog + cascade rolling + procedural restriction generator | Shipped |
| Per-gem pools, magnitudes, cascade/restriction probability tables (uniform weights) | Shipped |
| Display of multi-effect outcomes and structured restrictions in preview prompt and inset | Shipped |
| Engine wiring of effects: regen multiplier, crit_chance stat, detect_traps/detect_hidden conditions | **Pending** — atoms reference these by name and silently no-op until the underlying mechanics are added (separate body of work) |
| Restriction enforcement (refusing to equip a restricted gem if alignment/class/race doesn't match) | **Pending** — gems carry the data; equip-side enforcement is the equipment system's responsibility |
| Per-atom weighting rebalance | **Future** — structure supports it; all weights initialised at 1 |
| Emerald and diamond recipes (`enchant_emerald`, `enchant_diamond`) | **Pending** — only `enchant_ruby` exists today |
| Mid/late-game ingredients (replacing Arcane Dust at EXPERT+ / GM) | **Pending** |
| LLM name generation for inset weapons | **Stubbed** — returns hardcoded "LLMName" |

---

## Gem Insetting (cmd_inset.py — jeweller skill)

Standalone command `inset <gem> in <weapon>` (aliases: `ins`) at `RoomCraftingType.JEWELLER` — NOT a recipe through `cmd_craft`.

The jeweller consumes the enchanted gem and **merges its standard fields into the weapon** — no custom plumbing:

- **`weapon.wear_effects`** ← extended with `gem.db.wear_effects` (concat). Standard equip pipeline picks them up on wield.
- **`weapon.required_classes` / `excluded_classes` / `required_races` / `excluded_races`** ← unioned with the gem's same fields (deduplicated).
- **`weapon.min_alignment_score`** ← takes `max(weapon, gem)` (more restrictive lower bound wins).
- **`weapon.max_alignment_score`** ← takes `min(weapon, gem)` (more restrictive upper bound wins).
- **`weapon.is_inset = True`** — the existing AttributeProperty on `WeaponMechanicsMixin`. Used to gate against double-insetting.
- All updated mixin field values + `wear_effects` are persisted to `NFTGameState.metadata` (using mixin field names — no custom keys) so the weapon survives despawn/respawn.

Other behavior:
- LLM name generator stub (`llm/name_generator.py`) — returns hardcoded "LLMName" until LLM integration.
- Mastery requirements by gem tier: Ruby → BASIC, Emerald → EXPERT, Diamond → GRANDMASTER.
- Single gem per weapon (no double-insetting), weapon must not be wielded.
- Progress bar (2 ticks × 3 seconds), 10 XP per inset.

---

## RecipeBookMixin (typeclasses/mixins/recipe_book.py)

Mixed into `FCMCharacter`. Stores known recipes in `self.db.recipe_book` dict for O(1) lookup.

- `learn_recipe(recipe_key)` → `(bool, str)` — validates recipe exists, skill requirement met
- `knows_recipe(recipe_key)` → `bool`
- `get_known_recipes(skill=None, crafting_type=None)` → filtered list

---

## RoomProcessing (typeclasses/terrain/rooms/room_processing.py)

Resource refinement rooms (windmill, bakery, smelter, tannery, sawmill, textile mill). Converts input resources → output resource for a gold fee. Supports multi-recipe rooms (e.g. smelter handles iron ore → iron ingot, copper ore → copper ingot, alloys) and multi-input recipes (e.g. bakery: flour + wood → bread). Each recipe can have its own cost override or fall back to the room default.

Commands: `process <resource>` (aliases: `mill`, `bake`, `smelt`, `saw`, `tan`, `weave`) — auto-selects recipe by input match. `rates` — shows all available conversions and costs. Configurable delay with progress bar.

---

## RoomCrafting (typeclasses/terrain/rooms/room_crafting.py)

Skilled NFT item crafting rooms (smithy, woodshop, tailor, apothecary, etc.). Each room has a `crafting_type` and `mastery_level` that gates which recipes can be made there.

Commands: `craft` (aliases: `forge`, `carve`, `sew`, `brew`, `enchant` + prefix abbreviations like `cr`, `cra`, `fo`, `br`, `enc`, `ench`, `en`, etc.), `available`, `repair` (aliases: `rep`, `repa`, `repai`). Configurable delay with progress bar scaled by mastery level. Craft spawns NFTs via `BaseNFTItem.assign_to_blank_token()` + `spawn_into()`. Potions use `mastery_tiered: True` on the recipe — `cmd_craft.py` checks for this flag, computes the brewer's mastery, and builds the tier-specific `NFTItemType` name via `PotionQuality(mastery).prefix` (e.g. "Watery Potion of the Bull"). Effects are baked into each tier's prototype and `NFTItemType.default_metadata` — no post-spawn scaling code. Potions work identically to every other NFT item through the standard `assign_to_blank_token` → `spawn_into` pipeline. For gem enchanting, the recipe's `output_table` triggers a two-phase preview→consume flow against `EnchantmentSlot`: the player sees the next pre-disclosed outcome before committing materials; on confirm, `EnchantmentService.consume_slot()` atomically claims the slot's outcome and advances to a freshly-rolled next outcome. The claimed outcome is applied to the spawned gem (`db.gem_effects` and `db.gem_restrictions` are both lists). Race-loss aborts cleanly with no materials consumed. Repair restores durability to max at reduced material cost (dual mode: auto-compute `total_materials - 1` or explicit `repair_ingredients` on recipe), awards 50% craft XP.

---

## Consumable Items (typeclasses/items/consumables/)

- `ConsumableNFTItem` — base for single-use NFT items. `consume(consumer)` calls `at_consume()`, deletes item on success (returned to RESERVE via standard hooks).
- `CraftingRecipeNFTItem` — teaches a recipe when consumed. `recipe_key` AttributeProperty matches `world.recipes` registry.
- `PotionNFTItem` — potion with `potion_effects` list, `duration`, `named_effect_key`, and `mastery_tier`. `at_consume()` applies instant restore effects directly, then timed effects (stat_bonus, condition, damage_resistance) via `apply_named_effect()` (duration_type auto-filled from the NamedEffect registry). Anti-stacking via `has_effect(key)` — keyed by effect (e.g. `"potion_strength"`, `"barkskin"`), blocks consumption when effect is already active (potion saved). Supports dice-based restore (`"dice": "2d4+1"`) and int-based (`"value": 8`). Effects are baked into per-tier prototype files and `NFTItemType.default_metadata` — no post-spawn scaling code. Potions work through the standard NFT spawn pipeline like all other items.
- `SpellScrollNFTItem` — spell scroll with `spell_key`. Consumed via `transcribe` command to learn spells.

---

## Commands

| Command | Location | Purpose |
|---|---|---|
| `learn` | all_char_cmds | Consume recipe NFT to learn recipe (Y/N confirmation) |
| `recipes` | all_char_cmds | Show all known recipes grouped by skill |
| `craft`/`forge`/`carve`/`sew`/`brew`/`enchant` | room_specific (crafting) | Craft NFT items from recipes |
| `available` | room_specific (crafting) | Show craftable recipes in current room |
| `repair`/`rep` | room_specific (crafting) | Repair damaged item (reduced material cost, 50% XP) |
| `inset`/`ins` | room_specific (jeweller) | Inset enchanted gem into weapon |
| `process`/`mill`/`bake`/`smelt`/`saw`/`tan`/`weave` | room_specific (processing) | Convert resources |
| `rates` | room_specific (processing) | Show conversion rates and costs |

---

## Prototype Structure (world/prototypes/)

One file per item. Prototypes define item stats, wear effects, damage, and crafting metadata. Organized by category: `weapons/`, `wearables/`, `consumables/`, `components/`, `ships/`.

> **Design:** See `design/ECONOMY.md` for resource types and pricing. Individual prototype files are the source of truth for item stats.
