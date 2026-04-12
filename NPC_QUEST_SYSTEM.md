# NPC & Quest System — Design & Architecture

## Overview

NPCs and quests form the primary interaction layer between players and the game world. NPCs provide services (training, shopping, quest-giving, dialogue) while the quest system tracks multi-step objectives with rewards.

> **New player experience:** See `design/NEW_PLAYER_EXPERIENCE.md` for onboarding design intent and tutorial structure. **Economy:** See `design/ECONOMY.md` for AMM pricing model and trade accounting.

---

## NPC Architecture (typeclasses/actors/npc.py + npcs/)

> **For the full NPC/mob class hierarchy, mixin composition system, and mob AI tiers, see [NPC_MOB_ARCHITECTURE.md](NPC_MOB_ARCHITECTURE.md).**

NPCs share the full `BaseActor` infrastructure (ability scores, HP/mana/move, conditions, damage resistance, combat stats, effect system) plus `FungibleInventoryMixin` (gold and resources — classified as "WORLD" for blockchain service dispatch). They do NOT include other player-specific mixins (carrying capacity, wearslots, recipe book, spellbook, remort).

### CmdSet Visibility Pattern

NPCs use a two-tier system for command visibility:

1. **`call:true()` lock** on `BaseNPC.at_object_creation()` — required for Evennia's cmdhandler to merge the NPC's CmdSets into nearby characters' command pools. Without it, NPC commands are invisible to players. (`DefaultCharacter` does NOT set this lock by default.)

2. **`_EmptyNPCCmdSet`** — replaces the inherited character CmdSet as the default. Without this, player commands (stats, skills, look, etc.) from the NPC would leak into nearby characters' command pools.

Role-specific CmdSets (TrainerCmdSet, etc.) are added separately and contain only the NPC's interaction commands.

---

## TrainerNPC + Training System

TrainerNPC teaches skills and weapons to characters. Configuration per instance:

| Attribute | Type | Purpose |
|---|---|---|
| `trainable_skills` | `list[str]` | Skill keys this trainer teaches |
| `trainable_weapons` | `list[str]` | WeaponType value strings this trainer teaches |
| `trainer_class` | `str` | Class key (determines skill point pool) |
| `trainer_masteries` | `dict` | `{skill_key: mastery_int}` — max teachable level (hard cap, no random component) |
| `recipes_for_sale` | `dict` | `{recipe_key: gold_cost}` for recipe purchases |

**Training flow (deterministic — no random outcomes):**
1. Validate skill/weapon access via enum-driven lookup (`_CLASS_MAPPINGS_LOOKUP` for skills, `_WEAPON_CLASSES` for weapons)
2. Check trainer mastery ≥ target level (`trainer_masteries[skill]` is the hard cap on what this trainer can teach)
3. Verify the character has enough skill points to spend
4. Calculate gold cost with CHA modifier discount/surcharge: `final = max(1, round(base * (1 - cha_mod * 0.05)))`
5. Y/N confirmation showing cost, target mastery level, and training time — the player knows exactly what they will receive before paying
6. Deduct gold and skill points, start progress bar with `delay()` chain
7. On completion: advance mastery level — **always succeeds**

**Compliance note:** Training is fully deterministic. There is no random failure roll, no "lose your gold to a d100" mechanic, and no per-trainer cooldown. Pre-payment disclosure is total, satisfying the gambling-law compliance requirement that consideration must be paid only for outcomes the player has been told about in advance. See [COMPLIANCE.md](COMPLIANCE.md) and `ops/COMPLIANCE_LEGAL.md` § 9.5.

**Commands (injected via TrainerCmdSet):** `train` (list/train skills), `buy recipe` (list/buy recipes)

**Key files:** `typeclasses/actors/npcs/trainer.py`, `commands/npc_cmds/cmdset_trainer.py`, `tests/command_tests/test_cmd_train.py`

---

## ShopkeeperNPC + AMM Trading

ShopkeeperNPC buys and sells resources with prices driven by live XRPL AMM pools. Configuration per instance:

| Attribute | Type | Purpose |
|---|---|---|
| `tradeable_resources` | `list[int]` | Resource IDs this shop trades (e.g. `[1, 2, 3]` for wheat, flour, bread) |
| `shop_name` | `str` | Display name shown in list/quote output |

**Commands (injected via ShopkeeperCmdSet):**
- `list` / `browse` — batch-queries AMM pools for all tradeable resources, shows buy/sell prices per unit
- `quote buy/sell <amount> <item>` — gets live AMM price, stores pending quote on `caller.ndb.pending_quote`
- `accept` — executes the pending quote (validates player still in room, still has funds)
- `buy <amount> <item>` — instant buy at current market price (no quote step)
- `sell <amount> <item>` / `sell all <item>` — instant sell at current market price

**Pricing:** Constant product formula (x * y = k) with AMM trading fee. Buy prices ceil-rounded, sell prices floor-rounded — all gold amounts are integers. Favorable slippage goes to the game. See **design/ECONOMY.md** for the full AMM trade accounting model.

**Superuser commands:**
- `amm_check` / `amm_check <resource>` — query AMM pool states (reserves, fees). Read-only, no trades
- `reconcile` / `recon` — compare vault on-chain balances vs game-state DB per currency. Shows Reserve, Distributed, Sink, Delta
- `sync_reserves` — recalculate RESERVE from on-chain vault state: `RESERVE = on_chain - (SPAWNED + ACCOUNT + CHARACTER + SINK)`. Always run `reconcile` first
- `sync_nfts` — sync on-chain NFTs with game DB (placeholder → real NFToken IDs)

**Key files:** `typeclasses/actors/npcs/shopkeeper.py`, `commands/npc_cmds/cmdset_shopkeeper.py`, `blockchain/xrpl/services/amm.py`, `blockchain/xrpl/xrpl_amm.py`, `commands/account_cmds/cmd_amm_check.py`, `commands/account_cmds/cmd_reconcile.py`, `blockchain/xrpl/management/commands/test_amm_trades.py`, `tests/xrpl_tests/test_amm_service.py`, `tests/command_tests/test_cmd_shopkeeper.py`

---

## QuestGiverMixin (typeclasses/mixins/quest_giver.py)

Shared mixin for any NPC that offers quests. Provides:
- `quest_key` AttributeProperty — the quest this NPC offers
- `CmdNPCQuest` command (`quest` / `quest accept` / `quest abandon`) — view, accept, abandon, turn-in via `progress()` on view
- `QuestGiverCmdSet` — Union-merged cmdset with the quest command
- `get_quest_completion_message(caller, quest)` hook — override for custom completion text

Used by: GuildmasterNPC, BartenderNPC, QuestGivingShopkeeper (and its subclass BakerNPC).

---

## GuildmasterNPC + Guild System

GuildmasterNPC manages multiclassing and class level advancement. Uses QuestGiverMixin for quest commands.

**Attributes:**

| Attribute | Type | Purpose |
|---|---|---|
| `guild_class` | `str` | Class key this guild serves (e.g. `"warrior"`) |
| `multi_class_quest_key` | `str\|None` | Quest required before joining as multiclass |
| `max_advance_level` | `int` | Highest class level this guildmaster can grant (default 40) |
| `next_guildmaster_hint` | `str\|None` | RP flavour text pointing to the next guildmaster |

**Commands:**
- `quest` / `quest accept` / `quest abandon` — via QuestGiverMixin
- `guild` — shows guild info, class description, requirements, character progress, quest status (GuildmasterCmdSet)
- `join` — join guild class with full requirement checks (race/alignment/remort/ability/quest) (GuildmasterCmdSet)
- `advance` — spend pending level, guildmaster-specific level cap with RP redirect message (GuildmasterCmdSet)

**Level cap system:** Each guildmaster has a `max_advance_level`. When a character reaches it, they get an RP message directing them to the next guildmaster (via `next_guildmaster_hint`). Creates natural world progression — starter guildmasters cap at low levels, forcing exploration.

**Key files:** `typeclasses/actors/npcs/guildmaster.py`, `commands/npc_cmds/cmdset_guildmaster.py`, `typeclasses/mixins/quest_giver.py`, `tests/command_tests/test_quests.py`

---

## BartenderNPC (typeclasses/actors/npcs/bartender_npc.py)

Quest-aware bartender for the Harvest Moon Inn. Uses QuestGiverMixin (quest command for rat_cellar quest) + LLMRoleplayNPC (LLM dialogue). Injects `{quest_context}` into the LLM prompt based on the player's tutorial and quest state.

**State machine** (level gate at 3): new player → tutorial/quest pitch → quest active → tutorial suggest → generic bartender.

**Key files:** `typeclasses/actors/npcs/bartender_npc.py`, `llm/prompts/bartender.md`, `tests/typeclass_tests/test_bartender_npc.py`

---

## QuestGivingShopkeeper + BakerNPC

QuestGivingShopkeeper combines QuestGiverMixin + LLMRoleplayNPC + ShopkeeperCmdSet. Provides LLM dialogue with quest-aware context injection + AMM shop commands. Injects `{quest_context}` and `{shop_commands}` template variables into the LLM prompt.

BakerNPC is a QuestGivingShopkeeper subclass for Bron at the Goldencrust Bakery. Trades flour (ID 2) and bread (ID 3). Quest-aware prompt states: pitch → active → grateful → generic (level gate at 3).

**Key files:** `typeclasses/actors/npcs/quest_giving_shopkeeper.py`, `typeclasses/actors/npcs/baker_npc.py`, `llm/prompts/baker.md`, `tests/typeclass_tests/test_quest_giving_shopkeeper.py`, `tests/typeclass_tests/test_quest_giver_mixin.py`

---

## Other Quest-Giving NPCs (typeclasses/actors/npcs/)

The starter quest chain involves several additional NPCs beyond Rowan and Bron, each implemented as a quest-aware LLM NPC. They follow the same pattern as `BakerNPC` (quest-state-driven `{quest_context}` injection plus role-specific prompt) but specialise on a different production loop.

| NPC file | Character | Quest | Role |
|---|---|---|---|
| `mara_npc.py` | Mara Brightwater | `mara_moonpetal` | Apothecary at The Mortar and Pestle (alchemy ingredients) |
| `elena_npc.py` | Elena Copperkettle | `elena_cloth` | Weaver at her cottage (textile chain) |
| `hendricks_npc.py` | Old Hendricks | `hendricks_ore` | Blacksmith (metals chain) |
| `oakwright_npc.py` | Master Oakwright | `oakwright_timber` | Carpenter at the woodshop |
| `torben_npc.py` | Torben | — | Currently flavour-only NPC |
| `tutorial_guide_npc.py` | Tutorial guide | — | NPC used inside the Tutorial Hub instances |

Additional NPC variants used elsewhere in the world:

| NPC file | Purpose |
|---|---|
| `quest_giving_llm_trainer.py` | Trainer + LLM dialogue + quest-giving combo (used by Oakwright-style NPCs) |
| `llm_guildmaster_npc.py` | LLM-driven variant of GuildmasterNPC for guildmasters that hold real conversations |
| `llm_shopkeeper_npc.py` | LLM-driven variant of ShopkeeperNPC for shopkeepers that hold real conversations |
| `nft_shopkeeper.py` | NFT-trading shopkeeper (sells/buys NFT items rather than fungible resources) |
| `llm_roleplay_npc.py` | Base class for non-combat LLM dialogue NPCs |

All quest-giving NPCs share the `QuestGiverMixin` machinery (quest command, completion hook, account cap check) so adding a new quest-NPC is a matter of subclassing the closest existing variant and pointing `quest_key` at the new quest.

---

## Quest System (world/quests/)

Step-based quest engine with registry pattern (same as spells/races/classes).

### Architecture

- `FCMQuest` base class (`world/quests/base_quest.py`) — `step_<name>()` methods, `progress()` dispatches to current step, status tracking (active/completed/failed), help text per step
- `FCMQuestHandler` (`world/quests/quest_handler.py`) — lazy_property on FCMCharacter, stores quest state in Evennia attributes (category `fcm_quests`). API: `add()`, `remove()`, `get()`, `has()`, `is_completed()`, `active()`, `completed()`, `all()`
- `QuestTagMixin` on rooms — `quest_tags` AttributeProperty + `fire_quest_event()` for location-triggered progression
- Quest registry with `@register_quest` decorator, `get_quest()` lookup, auto-imports from `world.quests.guild`

**Quest debt integration:** When `FCMQuest.complete()` awards gold or bread, it calls `_register_quest_debt()` to notify the unified spawn system. This deducts the reward amount from the next spawn cycle's budget, preventing quest rewards from inflating the economy. The call is a graceful no-op if the spawn service isn't running. See **UNIFIED_ITEM_SPAWN_SYSTEM.md** § Quest Reward Integration.

**Quest templates** (`world/quests/templates/`): CollectQuest, VisitQuest, MultiStepQuest — reusable bases for common patterns.

### Implemented Quests

**Warrior Initiation** (`world/quests/guild/warrior_initiation.py`): Rat cellar check quest. Sergeant Grimjaw sends player to clear rats from Harvest Moon cellar. If player already completed `rat_cellar` quest, instant induction on accept. Otherwise, `step_clear_rats` checks on return. Acceptance gated on: levels_to_spend > 0, not already warrior, race/alignment/remort/multiclass requirements. On completion, deducts 1 level, auto-grants warrior level 1. reward_xp=100.

**Thief Initiation** (`world/quests/guild/thief_initiation.py`): "The Shadow Trial" — Gareth Stonefield sends the aspirant to navigate the Thieves' Gauntlet, a static 3-room challenge off the Training Alcove (Narrow Corridor → Damp Chamber → The Vault). Find the hidden entrance (find_dc=5), survive traps (dart trap and tripwire, DC 6), discover a hidden lever and key, unlock the vault chest, and retrieve a shadow guild token. Return the token to Gareth to complete. Acceptance gated on: levels_to_spend > 0, not already thief, race/alignment/remort/multiclass requirements (DEX 14 for multiclass). On completion, deducts 1 level, auto-grants thief level 1. reward_xp=100. Gauntlet fully resets (re-arms traps, re-hides objects, re-locks chest) after each completion.

**Mage Initiation** (`world/quests/guild/mage_initiation.py`): Single-step resource delivery (1 Ruby, ID 33). On completion, auto-grants mage level 1. Guildmaster is turn-in point.

**Cleric Initiation** (`world/quests/guild/cleric_initiation.py`): "Feed the Hungry" — Brother Aldric sends player to give bread to Old Silas (beggar) in Beggar's Alley behind the temple. Beggar's Alley has `quest_tags=["cleric_initiation"]`. Step checks for bread (resource ID 3) in inventory. On completion, auto-grants cleric level 1. Evil alignments excluded.

**Rat Cellar** (`world/quests/rat_cellar.py`): First combat quest. key="rat_cellar", quest_type="main", reward_xp=100, reward_gold=10, repeatable=False. QuestDungeonTriggerExit gates cellar entrance. Single step completes on `boss_killed` event (fired by RatKing.die()). No-death: defeated players teleport to inn at 1 HP.

**Baker's Flour** (`world/quests/bakers_flour.py`): Starter delivery quest. key="bakers_flour", quest_type="side", reward_xp=100, reward_gold=4, reward_bread=1. Bring 3 Flour (resource ID 2) to Bron at the Goldencrust Bakery. Consumes flour on turn-in.

**Oakwright's Timber** (`world/quests/oakwright_timber.py`): Starter delivery quest. key="oakwright_timber", quest_type="side", reward_xp=100, reward_gold=5, reward_bread=1. Bring 4 Timber (resource ID 7) to Master Oakwright at the woodshop.

**Mara's Moonpetal** (`world/quests/mara_moonpetal.py`): Starter alchemy ingredient quest. key="mara_moonpetal", quest_type="side", reward_xp=150, reward_gold=5, reward_bread=1. Bring moonpetal to Mara Brightwater at The Mortar and Pestle. Teaches the alchemy ingredient supply chain.

**Elena's Cloth** (`world/quests/elena_cloth.py`): Starter textile production quest. key="elena_cloth", quest_type="side", reward_xp=100, reward_gold=5, reward_bread=1. Cotton → loom → cloth, deliver to Elena Copperkettle at her cottage. Teaches the parallel textile chain alongside wheat.

**Hendricks' Bronze** (`world/quests/hendricks_ore.py`): Hardest starter quest. key="hendricks_ore", quest_type="side", reward_xp=250, reward_gold=10 (no bread — gold reflects difficulty). Mine copper and tin from the abandoned mine, smelt to bronze ingots, deliver to Old Hendricks at the smithy. Teaches the full metals chain and introduces the mine zone.

All six starter quests have `account_cap = 10` (account-level completion cap — see [NEW_PLAYER_EXPERIENCE.md](NEW_PLAYER_EXPERIENCE.md) § Account-Level Quest Caps).

### Adding a New Quest

1. Create `world/quests/guild/my_quest.py` (or appropriate subfolder)
2. Decorate with `@register_quest`
3. Define `key`, `name`, `desc`, `start_step`, step methods
4. Import in the subfolder's `__init__.py`

**Character command:** `CmdQuests` (key=`"quests"`, aliases=`["quest log", "questlog"]`) — view quest log, show details for specific quests. No conflict with `CmdNPCQuest` (key=`"quest"`) from QuestGiverMixin.

Comprehensive test coverage in `tests/command_tests/test_quests.py` plus per-quest test modules under `tests/quest_tests/`.
