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

**Recipes auto-granted at mastery level-up** — no recipe scrolls needed (unlike other crafting skills). This means:
- No recipe scroll prototypes in `world/prototypes/consumables/recipes/` for enchanting
- No recipe scroll entries in migration 0007
- Enchanters learn recipes automatically when they reach the required mastery tier

**Item split — vanilla vs enchanted:**
- **Vanilla items** (tailored/leathered): simple names (Bandana, Kippah, Cloak, Veil, Scarf, Sash, Leather Cap, Leather Gloves), no effects, no class restrictions. Crafted by tailors/leatherworkers.
- **Enchanted items**: named variants (Rogue's Bandana, Sage's Kippah, etc.) with effects and optional class restrictions. Created by enchanters transforming vanilla items.
- **Non-enchantable items** keep their effects baked in: Gambeson (+1 AC, excludes mage), Coarse Robe (+10 mana), Brown Corduroy Pants (+10 move), Warrior's Wraps (+10 HP, excludes mage/cleric/thief).

**Three tiers of enchanting ingredients (planned):**
- BASIC/SKILLED: Arcane Dust (resource ID 16) — 2 per recipe
- EXPERT/MASTER: mid-game ingredient (TBD)
- GRANDMASTER: late-game ingredient (TBD)

**Enchanting scope:**
- **Wearables**: deterministic (fixed recipe per item — vanilla + dust → named enchanted variant)
- **Gems**: probabilistic (d100 roll tables) — enchanter turns raw gem + dust into "Enchanted Ruby/Emerald/Diamond" with random effects and optional race/class restrictions. Effects hidden until examined/identified.
- **Weapons**: NOT directly enchanted — get enchanted gems inset by jeweller instead

**Room type:** `RoomCraftingType.WIZARDS_WORKSHOP` — enchanting-specific crafting rooms.

**Command aliases:** `enchant`/`enc`/`ench`/`en` (active — added to `cmd_craft.py` aliases and verb/gerund maps).

---

## Gem Enchanting Architecture (world/recipes/enchanting/gem_tables.py)

- Roll tables keyed by gem type → mastery level → d100 ranges with effects
- Separate restriction table: 1-20 = random race restriction, 21-50 = random class restriction, 51-100 = no restriction
- One `NFTItemType` per gem tier (Enchanted Ruby, Enchanted Emerald, Enchanted Diamond) — NOT one per effect variant
- Effects stored as `db.gem_effects` and `db.gem_restrictions` on the spawned item (regular Evennia attributes)
- Ruby available at BASIC (5 mastery tables planned), Emerald at EXPERT (3 tables), Diamond at GM only (1 table)
- `cmd_craft.py` detects `output_table` field in recipe and calls `roll_gem_enchantment()` after spawn

---

## Gem Insetting (cmd_inset.py — jeweller skill)

Standalone command `inset <gem> in <weapon>` (aliases: `ins`) at `RoomCraftingType.JEWELLER` — NOT a recipe through `cmd_craft`.

- Jeweller consumes enchanted gem and transfers its effects to a weapon
- Host weapon keeps its original `NFTItemType`/typeclass but gets effects + name override in NFTMirror metadata
- Extends `weapon.wear_effects` with gem effects (preserves original prototype effects)
- Stores `gem_effects` and `gem_restrictions` as db attrs on the weapon
- Persists name, wear_effects, gem_effects, gem_restrictions to NFTMirror metadata for despawn/respawn survival
- LLM name generator stub (`llm/name_generator.py`) — returns hardcoded "LLMName" until LLM integration
- Mastery requirements by gem tier: Ruby → BASIC, Emerald → EXPERT, Diamond → GRANDMASTER
- Single gem per weapon (no double-insetting), weapon must not be wielded
- Progress bar (2 ticks × 3 seconds), 10 XP per inset

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

Commands: `craft` (aliases: `forge`, `carve`, `sew`, `brew`, `enchant` + prefix abbreviations like `cr`, `cra`, `fo`, `br`, `enc`, `ench`, `en`, etc.), `available`, `repair` (aliases: `rep`, `repa`, `repai`). Configurable delay with progress bar scaled by mastery level. Craft spawns NFTs via `BaseNFTItem.assign_to_blank_token()` + `spawn_into()`. For potions, post-spawn mastery scaling overrides `potion_effects` and `duration` from `potion_scaling.py` tables based on the brewer's alchemy mastery. For gem enchanting, post-spawn roll table sets `gem_effects` and `gem_restrictions` from `gem_tables.py` based on the enchanter's mastery. Repair restores durability to max at reduced material cost (dual mode: auto-compute `total_materials - 1` or explicit `repair_ingredients` on recipe), awards 50% craft XP.

---

## Consumable Items (typeclasses/items/consumables/)

- `ConsumableNFTItem` — base for single-use NFT items. `consume(consumer)` calls `at_consume()`, deletes item on success (returned to RESERVE via standard hooks).
- `CraftingRecipeNFTItem` — teaches a recipe when consumed. `recipe_key` AttributeProperty matches `world.recipes` registry.
- `PotionNFTItem` — potion with `potion_effects` list, `duration`, and `named_effect_key`. `at_consume()` applies instant restore effects directly, then timed effects (stat_bonus, condition) via `apply_named_effect()` (duration_type auto-filled from the NamedEffect registry). Anti-stacking via `has_effect(key)` — keyed by stat (e.g. `"potion_strength"`), blocks consumption when effect is already active (potion saved). Supports dice-based restore (`"dice": "2d4+1"`) and int-based (`"value": 8`). Mastery scaling applied post-spawn by `cmd_craft.py` from `potion_scaling.py` tables.
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
