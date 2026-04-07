# WORLD.md

> **THIS FILE is for WORLD BUILDING only** — lore, narratives, zone designs, NPC concepts, quest ideas, brainstorms, and creative direction. For technical architecture, code patterns, and implementation details, see **src/game/CLAUDE.md**. For economic design — pricing models, market structures, spawn algorithms, and trade mechanics — see **design/ECONOMY.md**. Do not put technical implementation details or economic design content here.

---

## Purpose

This is the creative bible for FullCircleMUD. Everything that shapes the *world players experience* lives here: the history, the factions, the geography, the stories, the atmosphere. When building new zones, writing room descriptions, designing quests, or creating NPCs — this is the source of truth.

---

## Table of Contents

- [World Overview](#world-overview)
- [The Starting Continent](#the-starting-continent)
- [Zone Directory](#zone-directory)
- [Maritime Progression & The Island Chain](#maritime-progression--the-island-chain)
- [The Vaathari](#the-far-continent)
- [The Hall of Worlds](#the-hall-of-worlds)
- [Zone Design Principles](#zone-design-principles)
- [New Player Onboarding](#new-player-onboarding)
- [Exploration & Cartography](#exploration--cartography)
- [Progression & Resource Distribution](#progression--resource-distribution)
- [Factions & Powers](#factions--powers)
- [History & Lore](#history--lore)
- [NPCs & Key Characters](#npcs--key-characters)
- [Quests & Storylines](#quests--storylines)
- [Economy & Trade Narrative](#economy--trade-narrative)
- [Atmosphere & Tone](#atmosphere--tone)
- [World Builder's Toolset](#world-builders-toolset)
- [Brainstorm & Ideas](#brainstorm--ideas)

---

## World Builder's Toolset

*What we have to work with. Each system is marked as either a **fixed engine** (the mechanic itself) or an **expandable registry** (pour in more content). Current content counts show what exists today; the registries can grow indefinitely.*

### Characters & Identity

| System | Type | Current Content | Expandability |
|---|---|---|---|
| Races | Registry | 5 (Human, Dwarf, Elf, Halfling, Aasimar) | Drop a .py file in `races/` — auto-collected. Define ability bonuses, languages, size, remort gate. |
| Classes | Registry | 6 (Warrior, Thief, Mage, Cleric, Paladin, Bard) | Drop a .py file in `char_classes/` — auto-collected. Define level table, prime attr, skill access, multiclass reqs. |
| Languages | Registry | 8 (Common, Dwarven, Elfish, Halfling, Celestial, Kobold, Goblin, Dragon) | Add to Languages enum + garble syllable palette. New races/regions can bring new languages. |
| Alignment | Engine | Axis system | Gates class eligibility, item restrictions. |
| Multiclassing | Engine | Via guildmaster NPCs | Gated by race, alignment, remort, ability scores. |
| Remort | Engine | Perk selection on max level reset | Unlocks races, classes, equipment. Point buy budget grows. Long-term prestige loop. |
| Point buy stats | Engine | 6 stats, 27-point budget | Budget grows with remort. |

### Combat

| System | Type | Current Content | Expandability |
|---|---|---|---|
| Weapon types | Registry | 20+ (longsword, dagger, bow, staff, lance, ninjato, etc.) | Add a .py subclass of WeaponNFTItem. Define mastery path, hooks, damage dice. Each weapon plays differently. |
| Named effects | Registry | stunned, prone, slowed, staggered, entangled, paralysed, poisoned | Add to NamedEffect enum + message registry. Lifecycle, anti-stacking, save-each-round all built in. |
| Conditions | Registry | 14 (hidden, invisible, hasted, fly, water_breathing, darkvision, silenced, deaf, crit_immune, etc.) | Add to Condition enum + messages. Ref-counted — multiple sources stack safely. |
| Damage types | Registry | 10 (slashing, piercing, bludgeoning, fire, cold, lightning, acid, poison, magic, force) | Add to DamageType enum. Resistance/vulnerability system picks it up automatically. |
| Combat skills | Registry | bash, pummel, protect, taunt, offence, defence, retreat, dodge, assist, stab, flee | Add via CmdSkillBase with mastery dispatch. 24 more are scaffolded and ready to implement. |
| Combat engine | Engine | Real-time (twitch), weapon-speed intervals, group combat, advantage/disadvantage, parry/riposte, durability degradation | Fixed architecture — content plugs in via weapons, effects, and skills above. |

### Magic

| System | Type | Current Content | Expandability |
|---|---|---|---|
| Spell schools | Registry | 7 (Evocation, Abjuration, Necromancy, Divine Healing, Divination, Conjuration, Illusion) | Add a new school package in `world/spells/`. |
| Spells | Registry | ~15 implemented, ~20 scaffolded | Add a .py in the school folder, register with `@register_spell`. Full framework: cooldowns, memorisation, mastery scaling, AoE safe/unsafe, spell args, reactive hooks. |
| Spell scrolls | Registry | Scroll prototype per spell | Add a prototype — players learn spells from scrolls found/crafted/traded. |
| Spellbook system | Engine | Memorise/forget slots, cooldowns, mana costs | Fixed architecture — spells plug in. |

### Crafting & Economy

| System | Type | Current Content | Expandability |
|---|---|---|---|
| Crafting skills | Registry | 7 (Blacksmithing, Carpentry, Leatherworking, Tailoring, Alchemy, Jewellery, Enchanting) | Add to skills enum + recipe folder. |
| Recipes | Registry | 52+ across all skills | Add a .py recipe file. Defines ingredients, skill, mastery tier, output prototype, craft time. |
| Resource types | Registry | 36 (grains, ores, ingots, alloys, herbs, gems, hides, cloth, coal) | Add via DB seed migration. New zones can introduce zone-specific resources. |
| Item prototypes | Registry | Weapons, wearables, holdables, components, consumables, containers | Add a .py prototype file. Data-driven effects, restrictions, durability, weight. NFTs on-chain. |
| Potions | Registry | 9 (stat boost + restore for each ability) | Add recipe + prototype. Mastery-scaled at brew time, named effects for timed buffs. |
| Processing rooms | Engine | Smelter, windmill, sawmill | Place a RoomProcessing with recipe list. Multi-recipe support (one smelter does all ore→ingot). |
| Harvesting rooms | Engine | Wheat fields built | Place a RoomHarvesting with resource_id. Any zone can have gathering nodes. |
| AMM shops | Engine | On-chain XRPL liquidity pools | Place a ShopkeeperNPC — pool prices drive buy/sell. Real supply/demand. |
| Blockchain economy | Engine | Gold + resources as XRPL tokens, items as NFTs, import/export/bank | Fixed architecture — every new item/resource automatically inherits on-chain ownership. |
| Hunger | Engine | Bread every 15 min, starvation, forage | Fixed — creates demand for bread (wheat→flour→bread supply chain). |

### World Building Infrastructure

| System | Type | Current Content | Expandability |
|---|---|---|---|
| Zones | Registry | 1 (Millholm, 11 districts, ~400+ rooms) | Add a Python builder script. Zone/district/terrain tags. Soft rebuild support. |
| Terrain types | Registry | 12 (Urban, Rural, Forest, Mountain, Desert, Swamp, Coastal, Underground, Dungeon, Water, Arctic, Plains) | Add to TerrainType enum. Drives weather exposure, natural light, forage eligibility, future systems. |
| Climate zones | Registry | 5 (Temperate, Arctic, Desert, Tropical, Coastal) | Add to ClimateZone enum + weather tables. Per-zone mapping. |
| Weather tables | Registry | 20 transition matrices (5 climates x 4 seasons) | Add/tune probabilities. New climates get their own weather personality. |
| Dungeon templates | Registry | 1 test template (Cave of Trials) | Add a DungeonTemplate dataclass. Define room generators, boss depth, exit budget, lifetime. Per-group instanced. |
| Room types | Registry | RoomBase, RoomBank, RoomInn, RoomProcessing, RoomCrafting, RoomHarvesting, RoomCemetery, RoomPurgatory, RoomGateway, RoomPostOffice, PressurePlateRoom, DungeonRoom, RoomRecycleBin (inherits from DefaultRoom, not RoomBase) | Subclass RoomBase for new room behaviours. |
| Gateway conditions | Engine | Dock rooms with boat_level + seamanship check | Any room can gate travel on conditions, skills, items, quests. |
| Day/night | Engine | 4 phases, TIME_FACTOR=24 | Fixed cycle — rooms react via natural_light + terrain. |
| Seasons | Engine | 4 seasons, 360-day year | Fixed cycle — drives weather transitions. |
| Light sources | Registry | Torch (consumable), Lantern (reusable), LitFixture (permanent) | Add new light source types. LightSourceMixin is reusable on any object. |
| Vertical movement | Engine | Up/down exits, fly-gated air, underwater breath timers | Fixed — place exits and tag rooms. |

### NPCs & Mobs

| System | Type | Current Content | Expandability |
|---|---|---|---|
| Mob types | Registry | 4 (Rabbit, Wolf, Dire Wolf, Training Dummy) | Subclass CombatMob. Define AI behaviour, stats, loot, respawn. Area-restricted wandering via tags. |
| NPC roles | Registry | Trainer, Guildmaster, Shopkeeper (scaffolded), LLMRoleplayNPC | Subclass BaseNPC. Inject role-specific CmdSets. |
| Trainers | Registry | 8 placed in Millholm | Place a TrainerNPC with skill list, mastery caps, costs. |
| Guildmasters | Registry | 4 placed in Millholm (level 5 cap each) | Place with class, max_advance_level, quest requirements, next_guildmaster_hint for progression chain. |
| LLM dialogue | Engine | OpenRouter API, memory, engagement tracking | Toggle `llm_use_vector_memory=True` on any NPC. Prompt templates per NPC type. Any NPC can talk. |
| Mob AI | Engine | State machine, ticker-driven, area tags | Fixed architecture — mob behaviours plug in via state handlers. |

### World Objects & Interactions

| System | Type | Current Content | Expandability |
|---|---|---|---|
| Signs | Registry | 4 templates (post, hanging, wall, stone) | Place with text. Cheap atmospheric detail. |
| Chests | Registry | WorldChest, TrapChest | Place with contents, lock_dc, key_tag, trap params. |
| Doors | Registry | ExitDoor, TrapDoor | Place with lock_dc, key_tag, trap params, hidden (find_dc). |
| Traps | Registry | TrapChest, TrapDoor, TripwireExit, PressurePlateRoom | Configure damage, effects, alarm, reset timer, one-shot, DCs. Extensible via TrapMixin on any object. |
| Keys | Registry | KeyItem (consumable, matched by tag) | Place key + matching lock anywhere. |
| Hidden objects | Engine | HiddenObjectMixin with find_dc | Any object/exit/room can be hidden. Per-character discovery. |
| Containers | Registry | Backpack, Panniers | Add prototypes with capacity limits. |
| World fixtures | Engine | Immovable, untakeable base class | Subclass WorldFixture for any atmospheric/interactive object. |

### Player Systems

| System | Type | Current Content | Expandability |
|---|---|---|---|
| Quests | Registry | 4 guild initiations + 3 templates (collect, visit, multi-step) | Add quest .py + `@register_quest`. Step-based, any trigger conditions. |
| Sea routes | Registry | 1 test route (town dock ↔ beach dock) | Place RoomGateway docks, define routes with food_cost + boat_level. 5 ship tiers already exist. |
| Tutorial zones | Registry | 1 (Survival Basics, 11 rooms) | Hub has placeholder exits for tutorials 2 and 3. Per-player instanced, item-stripped on exit. LLM guide NPC walks player through (see [New Player Onboarding](#new-player-onboarding)). |
| Death/respawn | Engine | Corpse, purgatory timer, cemetery bind, XP penalty | Fixed — place RoomCemetery for new bind points. |
| Follow/Group | Engine | Chain-following, auto-follow on exits | Fixed — underpins group combat, strategy skills, dungeon entry. |
| Communication | Engine | say/whisper/shout, language-aware, garble | Fixed — new languages just work. |

### Existing World Content (Millholm) — REDESIGNING
- **Currently built:** ~330+ rooms across 8 districts: Town (3x3 market square with NS/EW crossroads), Farms (west), Woods (~93 rooms including northern woods row and deep woods entry), Sewers (beneath town), Abandoned Mine (17 rooms, copper/tin), Faerie Hollow (5 rooms, hidden), Deep Woods (4 procedural passages connecting entry↔clearing↔miners_camp), Southern District (~30 rooms, rougher town → countryside → moonpetal fields → gnoll territory → barrow → shadowsward gate).
- **Remaining:** Mobs, NPCs, and quests for all districts. All room layouts are complete.
- **Named NPCs** — Sergeant Grimjaw, Warlord Thane, Whisper, Shadow Mistress Vex, Archmage Tindel, High Magus Elara, Brother Aldric, High Priestess Maren. LLM-driven NPCs planned: Bartender, Bron (baker), Old Hendricks (smithy), Elena Copperkettle (seamstress), Mara Brightwater (herbalist), Master Oakwright (carpenter), Town Guard.

---

### GAPS — What's Missing or Incomplete

**Unbuilt Systems** (skill/hook exists but no implementation):
- Mounts (ANIMAL_HANDLING skill, lance weapon references mounted combat)
- Pets/companions (ANIMAL_COMPANION skill)
- Shapeshifting (SHAPE_SHIFTING skill)
- Player housing/construction (SHIPWRIGHT, BUILD skills)
- Cartography/maps (CARTOGRAPHY skill)
- Reputation/faction tracking
- Formal player-to-player trade/auction
- Mail/offline messaging
- Smash command (SmashableMixin exists on chests/doors, no player command)

**Scaffolded Skills** (command stub exists, mastery dispatch prints placeholder):
- Frenzy (berserker rage), Assassinate (instant kill), all Bard skills (Performance, Inspiration, Debilitation, Manipulation, Misdirection), Nature Magic, most Divine spells (Protection, Judgement, Revelation, Dominion, Turn Undead)

**Scaffolded Spells** (file exists, class defined, not functional):
- Conjuration (Acid Arrow, Conjure Elemental, Dimensional Lock, Gate, Teleport), Illusion (Blur, Invisibility, Greater Invisibility, Phantasmal Killer, Mass Confusion), Divination (True Sight, Scry, Mass Revelation), Necromancy (Raise Dead, Raise Lich, Death Mark), Abjuration (Antimagic Field, Group Resist, Invulnerability, Shadowcloak)

**World Content Gaps:**
- Only one zone (Millholm) — no other settlements, coasts, mountains, deserts, swamps, arctic
- All Millholm districts built (room layouts complete). Next: populate with mobs, NPCs, and quests
- Dungeon entrances placed (4 deep woods passages) but no mobs in procedural rooms yet. Cave of Trials template exists but is unused — thief initiation uses the static Thieves' Gauntlet instead.
- Very few mob types — 3 animals + training dummy. No humanoid enemies, undead, magical creatures
- No boss encounters designed
- Limited quests — 4 guild initiations only. No side quests, story arcs, exploration quests
- No sea destinations beyond test route
- No NPCs placed in world (all designed, none built)

---

## World Overview

FullCircleMUD spans **two continents** separated by an ocean, connected by an **island chain** of increasing difficulty. Beyond the mortal world lies the **Hall of Worlds** — an interdimensional corridor leading to infinite planes of existence.

**The arc of the game:**

1. **The Starting Continent** — players begin in Millholm, a sleepy farming town in the northwest. They explore outward through increasingly dangerous zones: dwarven mountains, the Shadowsward, blighted wastelands, harsh deserts, a major port city, tropical coasts. Each zone introduces new resources, recipes, enemies, and cultures.

2. **The Ocean** — from the continent's port cities, players sail eastward through an island chain. Each island is a self-contained zone with unique resources and discoveries. Ship quality gates how far you can go. Cartographers map routes for repeat travel. The ocean is the mid-to-late game progression ladder.

3. **The Vaathari** — accessible only to grand master sailors, this is endgame territory. First access to the Hall of Worlds. The divine forge of Olympus. Top-tier content launches from here.

4. **The Hall of Worlds** — the true endgame. An interdimensional corridor with doors to infinite planes. Each door leads to a different world — some are deadly sprints, others are entire zones. Anything is possible here: prehistoric, futuristic, sci-fi crossovers, dying worlds, alien civilisations.

**Key design philosophy:** Zones are not stitched together room-by-room with invisible boundaries. They are connected via **gateway rooms** with narrative travel descriptions, food costs, skill requirements, and discovery mechanics. Travelling between zones *feels like a journey*. Some routes are hidden and must be discovered through exploration or earned through cartography.

**Remort discovery philosophy:** The world is deliberately layered so that each playthrough reveals new content. A player on their third remort should still be discovering zones, quests, and secrets they never knew existed. This is achieved through:
- **Hidden zones** that require high-level skills to discover (Zharavan beyond the waste, certain islands)
- **Remort-only content** (Aasimar race and the Celestial Threshold)
- **Mastery-gated routes** (Master cartography for the desert crossing, GM seamanship for the Vaathari)
- **Multiple viable paths** through the continent — a player who went east on playthrough 1 might go west on playthrough 2 and find entirely different content
- **Boss-dropped map NFTs** that unlock routes to places players didn't know existed
- **The Hall of Worlds** as infinite endgame — there is always another door

---

## The Starting Continent

### Layout

The continent is oriented roughly NW to SE, with Millholm in the northwest (toward the centre) and the coastline along the east and south. Climate trends from temperate (north) to tropical (south).

```
                              N
                              |
         Millholm -----------+----------- Ironback Peaks (Dwarven)
         (NW, starter)       / \                    |
              |             /   \                   |
              |            /     \          (foothills → coast)
         The Shadowsward   /       \                 |
         (W/SW)          /    Cloverfen       Saltspray Bay
              |         /     (Halfling)       (east coast)
         Shadowroot    /          |                 |
         (beyond      /      SE to Bayou    [Ocean → Islands]
          Shadowsward)            |                 |
              |              Bayou / Swamp     Southern Zones
         Scalded Waste          |                 |
         (desert)                 |          Kashoryu
              |    \              |          (Asian/tropical)
              |     \             /                  |
              |      Aethenveil          [Southern Ocean → Solendra]
              |      (link zone)
     Zharavan
     (far west, beyond waste)
```

*This is a conceptual layout, not a literal grid. Gateway rooms connect zones with narrative travel.*

**Racial Homeland Zones:** Each playable race has a cultural homeland on the continent. These zones serve as both cultural identity anchors and progression content. Players of that race feel a connection; all players benefit from the unique content.

| Race | Homeland | Position | Level |
|---|---|---|---|
| Human | Millholm (and Saltspray Bay, Kashoryu) | Throughout | Starter–Mid |
| Dwarf | Ironback Peaks | East of Millholm | Low-mid |
| Halfling | Cloverfen | Between Millholm and Saltspray Bay | Low–Low-mid |
| Elf | Aethenveil | Between Scalded Waste and Kashoryu | Mid |
| Aasimar | Celestial Threshold (Vaathari) | Base of Olympus | Endgame (remort only) |

### Zone Connections (Gateway Routes)

Discovery gate applies to all overland routes via the `explore` command. A party needs at least one member at or above the required cartography mastery — if no eligible cartographer is present, the route cannot be discovered.

Food costs are per character — every party member must carry the minimum bread for the route.

| From | To | Route Type | Discovery Gate | Food Cost (per character) | Notes |
|---|---|---|---|---|---|
| Millholm | Ironback Peaks | Overland | Cartography BASIC | 3 bread | Foothills, snowbound in winter (blocked passes force underground/fly) |
| Millholm | Cloverfen | Overland | Cartography BASIC | 2 bread | Plains road — gentle, low-level route. Random which of Ironback Peaks/Cloverfen is discovered first |
| Millholm | The Shadowsward | Overland | Cartography SKILLED | 4 bread | Trade road west/southwest — players must return when ready |
| Ironback Peaks | Cloverfen | Overland | Cartography BASIC | 2 bread | Mountain-to-lowland route, dwarves and halflings as trade neighbours |
| Ironback Peaks | Saltspray Bay | Overland | Cartography SKILLED | 4 bread | Foothills descent to coast |
| Cloverfen | Saltspray Bay | Overland | Cartography SKILLED | 3 bread | Short pastoral route |
| The Shadowsward | Shadowroot | Overland | Cartography EXPERT | 3 bread | Dangerous frontier crossing |
| The Shadowsward | Scalded Waste | Overland | Cartography EXPERT | 6 bread | Desert crossing |
| Saltspray Bay | The Bayou | Overland/Coastal | Cartography SKILLED | 2 bread | Trending tropical |
| Cloverfen | The Bayou | Overland | Cartography SKILLED | 2 bread | Southeast route |
| The Bayou | Kashoryu | Overland | Cartography EXPERT | 4 bread | Cultural transition |
| Scalded Waste | Aethenveil | Overland | Cartography MASTER | 4 bread | Desert edge to ancient forest |
| Scalded Waste | Zharavan | Overland | Cartography MASTER | 10 bread | Far western desert crossing — most players won't know it exists |
| Aethenveil | Kashoryu | Overland | Cartography MASTER | 3 bread | Forest to tropical coast |
| Saltspray Bay | Kashoryu | Sea route | Cartography SKILLED + Seamanship SKILLED + Caravel | 6 bread | Coastal route between the two port cities |
| Kashoryu | Saltspray Bay | Sea route | Cartography SKILLED + Seamanship SKILLED + Caravel | 6 bread | Coastal route between the two port cities |
| Saltspray Bay | Port Shadowmere | Sea route | Cartography SKILLED + Seamanship SKILLED + Caravel | 6 bread | — |
| Port Shadowmere | Saltspray Bay | Sea route | Cartography SKILLED + Seamanship SKILLED + Caravel | 6 bread | — |
| Saltspray Bay | The Arcane Sanctum | Sea route | Cartography EXPERT + Seamanship EXPERT + Brigantine | 8 bread | — |
| The Arcane Sanctum | Saltspray Bay | Sea route | Cartography EXPERT + Seamanship EXPERT + Brigantine | 8 bread | — |
| Saltspray Bay | Teotlan Ruin | Sea route | Cartography BASIC + Seamanship BASIC + Cog | 3 bread | — |
| Teotlan Ruin | Saltspray Bay | Sea route | Cartography BASIC + Seamanship BASIC + Cog | 3 bread | — |
| Saltspray Bay | Calenport | Sea route | Cartography SKILLED + Seamanship SKILLED + Caravel | 6 bread | — |
| Calenport | Saltspray Bay | Sea route | Cartography SKILLED + Seamanship SKILLED + Caravel | 6 bread | — |
| Saltspray Bay | Oldbone Island | Sea route | Cartography EXPERT + Seamanship EXPERT + Brigantine | 8 bread | — |
| Oldbone Island | Saltspray Bay | Sea route | Cartography EXPERT + Seamanship EXPERT + Brigantine | 8 bread | — |
| The Arcane Sanctum | Guildmere Island | Sea route | Cartography MASTER + Seamanship MASTER + Carrack | 10 bread | — |
| Guildmere Island | The Arcane Sanctum | Sea route | Cartography MASTER + Seamanship MASTER + Carrack | 10 bread | — |
| Oldbone Island | Guildmere Island | Sea route | Cartography MASTER + Seamanship MASTER + Carrack | 10 bread | — |
| Guildmere Island | Oldbone Island | Sea route | Cartography MASTER + Seamanship MASTER + Carrack | 10 bread | — |
| Saltspray Bay | Amber Shore | Sea route | Cartography BASIC + Seamanship BASIC + Cog | 3 bread | — |
| Amber Shore | Saltspray Bay | Sea route | Cartography BASIC + Seamanship BASIC + Cog | 3 bread | — |
| Guildmere Island | Vaathari | Sea route | Cartography GM + Seamanship GM + Galleon | 10 bread | Final crossing — only reachable from Guildmere. Lower seamanship = shipwreck risk |
| Vaathari | Guildmere Island | Sea route | Cartography GM + Seamanship GM + Galleon | 10 bread | Return crossing |
| Kashoryu | Port Shadowmere | Sea route | Cartography SKILLED + Seamanship SKILLED + Caravel | 6 bread | — |
| Port Shadowmere | Kashoryu | Sea route | Cartography SKILLED + Seamanship SKILLED + Caravel | 6 bread | — |
| Kashoryu | The Arcane Sanctum | Sea route | Cartography EXPERT + Seamanship EXPERT + Brigantine | 8 bread | — |
| The Arcane Sanctum | Kashoryu | Sea route | Cartography EXPERT + Seamanship EXPERT + Brigantine | 8 bread | — |
| Kashoryu | Teotlan Ruin | Sea route | Cartography BASIC + Seamanship BASIC + Cog | 3 bread | — |
| Teotlan Ruin | Kashoryu | Sea route | Cartography BASIC + Seamanship BASIC + Cog | 3 bread | — |
| Kashoryu | Calenport | Sea route | Cartography SKILLED + Seamanship SKILLED + Caravel | 6 bread | — |
| Calenport | Kashoryu | Sea route | Cartography SKILLED + Seamanship SKILLED + Caravel | 6 bread | — |
| Teotlan Ruin | Calenport | Sea route | Cartography SKILLED + Seamanship SKILLED + Caravel | 6 bread | — |
| Calenport | Teotlan Ruin | Sea route | Cartography SKILLED + Seamanship SKILLED + Caravel | 6 bread | — |
| Kashoryu | Oldbone Island | Sea route | Cartography EXPERT + Seamanship EXPERT + Brigantine | 8 bread | — |
| Oldbone Island | Kashoryu | Sea route | Cartography EXPERT + Seamanship EXPERT + Brigantine | 8 bread | — |
| The Arcane Sanctum | Oldbone Island | Sea route | Cartography EXPERT + Seamanship EXPERT + Brigantine | 8 bread | — |
| Oldbone Island | The Arcane Sanctum | Sea route | Cartography EXPERT + Seamanship EXPERT + Brigantine | 8 bread | — |
| Kashoryu | Amber Shore | Sea route | Cartography BASIC + Seamanship BASIC + Cog | 3 bread | — |
| Amber Shore | Kashoryu | Sea route | Cartography BASIC + Seamanship BASIC + Cog | 3 bread | — |
| Port Shadowmere | Amber Shore | Sea route | Cartography BASIC + Seamanship BASIC + Cog | 3 bread | — |
| Amber Shore | Port Shadowmere | Sea route | Cartography BASIC + Seamanship BASIC + Cog | 3 bread | — |
| Kashoryu | Solendra | Sea route | Cartography GM + Seamanship GM + Galleon | 10 bread | Secret destination — southern port only |
| Guildmere Island | Atlantis | Overland | Cartography MASTER | — | Swim off the beach at Guildmere Island and dive through an underwater cave system. Treated as overland discovery — no ship required. Water Breathing (or equivalent) needed to survive the descent. |

---

## Zone Directory

### 1. Millholm — *The Sleepy Farming Town* (REBUILDING)

**Level:** 1–5 starter zone | **Climate:** Temperate | **Position:** NW, toward continent centre

The starting zone. A prosperous farming community founded by four families (Stonefield, Brightwater, Goldwheat, Ironhand). Players learn the basics here: crafting, combat, guilds, economy. Full player journey documented in [New Player Onboarding](#new-player-onboarding).

#### Physical Layout

```
                                        LAKE (swim/sail practice)
                                        ducks, frogs, pike
                                        sailing club, boatyard
                                              |
                                        (proc. lake passage)
                                              |
                              DEEP WOODS      LAKE TRACK
                              (procedural)          |
                                    |         CEMETERY (west off road)
                              MINE  |         3 family tombs, skeletons
                                    |               |
                              DEEP WOODS ENTRY ---- NORTH ROAD
                              (funnel)              |
                              NORTHERN WOODS   MILLHOLM TOWN ---- INDUSTRIES ---- ZONE TRANSITION
                                    |          3x3 market sq       smelter         (BASIC cartography)
                                    |          crossroads EW/NS    sawmill
  FARMS (L1-2) ----  Old Trade Way West/East                 BASE WOODS
  rabbits, crows      |                                      (L1-2, south)
  wheat, cotton       |    ROOFTOPS (3x3 grid above craft quarter)
       |              |    drainpipe + fly access, Gareth's wardrobe
       |              |
       |        SOUTH DISTRICT (L3-5)        SEWERS (beneath town)
        \       harder mobs                  Gareth's thieves' lair
         `---'        |                      hidden doors dc 10-12
                 SHADOWSWARD GATE
                 (SKILLED cartography)
```

#### Districts

**Millholm Town** (centre)
- Old Trade Way West/East runs east-west as the main thoroughfare
- A smaller north-south local road crosses through the market square
- **3x3 market square** at the crossroads — shops, noticeboard, founding monument
- Harvest Moon Inn (bartender, tutorial entry, cellar → sewer access)
- Crafting shops: smithy, leathershop, tailor, apothecary, woodshop, bakery
- Bank (Order of the Golden Scale), general store
- Guild halls: Warriors (The Iron Company), Mages (Circle of the First Light), Temple (Shrine of the First Harvest)
- Gareth Stonefield's house (secret passage to abandoned house, upstairs bedroom with hidden wardrobe → rooftops)
- Residential area, stables
- NS road heads south to the Southern District, north to lake
- All outdoor streets max_height=2 (consistent fly ceiling), indoor rooms max_height=0

**Millholm Farms** (west of town, L1-2)
- Wheat farm (harvestable wheat fields), cotton farm
- Windmill (wheat → flour processing)
- Abandoned farm (quest hook — why was it abandoned?)
- **Light mobs:** rabbits, foxes — every player must come here for wheat to survive, so mobs are non-threatening
- Road heading south connects into the Southern District (side entrance)

**Industries** (east edge of town)
- Smelter (ores → ingots, alloys: copper+tin → bronze)
- Sawmill (wood → timber)
- The smelly end of town — these are noisy, dirty operations placed at the town's edge
- Gateway to the road east

**Base Woods** (south of the road east, L1-2)
- Main wood-gathering area (harvestable wood)
- Tanner's hut (hide → leather processing)
- **Medium mobs** — tougher than rabbits but won't murder a level 1 collecting wood. Boar? Badgers? Snakes? (TBD — something aggressive but survivable)
- Named POIs: spring-fed pool, stone cairn, game trails
- Persistent, mappable area

**Northern Woods** (north of main path, transition zone)
- 10 "Dense Woods" rooms along the north side of main path rooms 5-14 (mirrors southern grid placement)
- Connected east-west along the row, and north-south to the main path
- All 10 rooms funnel north into a single **Edge of the Deep Woods** entry room (many-to-one)
- Deep woods entry south exit returns to middle of the northern row (asymmetric — disorientation)
- Dense thickets block east/west from the entry room — only north (deeper) or south (back)
- Built in `millholm_woods.py` alongside the base woods

**Deep Woods** (north of the road east, L2-4, **PROCEDURAL**)
- Structure: **4 procedural dungeon passages** connecting 3 static rooms
  - **Deep Woods Entry** (built in `millholm_woods.py`) → proc passage 1 (north, inbound) → **Deep Woods Clearing** (built in `millholm_faerie_hollow.py`)
  - **Deep Woods Clearing** → proc passage 2 (west, outbound/return) → **Deep Woods Entry**
  - **Deep Woods Clearing** → proc passage 3 (east, inbound) → **Miners' Camp** (built in `millholm_mine.py`)
  - **Miners' Camp** → proc passage 4 (west, outbound/return) → **Deep Woods Clearing**
  - Passages are one-way and despawn after the group exits
- Procedural passages use `DungeonTriggerExit` + `deep_woods_passage` template (all 4 wired in `build_game_world.py`):
  - Auto-trigger on entry (no command needed) — `DungeonTriggerExit` intercepts `at_traverse`
  - Group instance mode, boss_depth=5 (5 rooms of forest), despawn after group exits
  - Template: `world/dungeons/templates/deep_woods_passage.py`, registered in `at_server_startstop.py`
- The static clearing is designed to look identical to the procedural deep woods rooms — players walk straight through unless paying attention
- **Invisible door** in the static clearing → **Faerie Hollow** district (requires DETECT_INVIS to see — "a shimmer in the air")
  - Built: `millholm_faerie_hollow.py` — 5 rooms: Deep Woods Clearing (static midpoint, tagged `millholm_deep_woods`), Shimmering Threshold (transition), Faerie Hollow (main chamber), Moonlit Glade (offering altar), Crystalline Grotto (RoomHarvesting — arcane dust resource 16)
  - Entrance is an ExitDoor with `is_invisible=True`, always open — walk through if you can see it
  - Return exit is visible (you can always leave once inside)
  - Arcane dust (resource 16) available here IF you are polite to the faeries — bring a gift, complete a small quest for them
  - Faeries are powerful enough to quickly kill all but a strong level 5 party if provoked
  - Knowledge gate, not level gate: a respectful level 2 with detect invis succeeds where an aggressive level 5 dies
  - Mages' guild quest sends players here for arcane dust (guild trainer provides detect invis scroll)
- Wolves and harder woodland mobs in procedural rooms
- Not mappable (entrance shown on district maps, interior is not)

**Abandoned Mine** (end of deep woods, L3-5)
- Accessible only through the deep woods procedural passages — no direct path
- **Abandoned Miners' Camp** — small static outdoor hub (arrival point from procedural passage 2)
  - Windroot Hollow nearby — **RoomHarvesting for Windroot** (resource 15, used in Potion of the Zephyr / +move potion)
  - Mine entrance leads to static mine dungeon interior
- **Mine interior is STATIC** (not procedural) — the procedural element is getting TO the mine, not inside it. Players who learn the mine layout are rewarded.
- **Copper ore and tin ore** mining (resources 23/25 — needed for bronze at the smelter)
- Infested with **kobolds** — small, dangerous in groups, territorial. They've moved in because the mine was abandoned.
- Persistent interior once reached (mappable)

**Southern District** (south of town, L3-5)
- Two entrances: south through town (via south_road → Rat Run) and side entrance from farms (south_fork_end → Countryside Road)
- **Rougher Town** (6 rooms): Rat Run, Low Market crossroads, Fence's Stall, Gaol, The Broken Crown tavern, South Gate through town wall
- **Countryside** (4 rooms): South Road (outside wall), Farmstead Fork, Bandit Holdfast (outlaws in ruined farmstead), Bandit Camp
- **Moonpetal Fields** (7 rooms): Moonpetal Approach + 2x3 RoomHarvesting grid (resource_id=12, gather command) — primary moonpetal supply for all potions
- **Gnoll Territory** (5 rooms): Wild Grasslands, Gnoll Hunting Grounds, Ravaged Farmstead, Gnoll Camp, Gnoll Lookout — gnolls are cowardly raiders, attack small groups, avoid town
- **Barrow Underground** (5 rooms, hidden): Barrow Hill (hidden entrance, find_dc=18), Barrow Entrance, Bone-Strewn Passage, Ancient Catacombs (Ancient Builders glyphs), Necromancer's Study — evil but pragmatic necromancer, future necromancy trainer
- **Shadowsward** (2 rooms): Southern Approach, Shadowsward Gate — zone exit placeholder (SKILLED cartography, future)
- **Shadowsward zone transition gate** at the far south — explorable with SKILLED cartography

**Rooftops** (above the western craft quarter, `millholm_rooftops`)
- 3x3 grid of rooftop rooms above Artisan's Way and Old Trade Way
- South row: Sagging Rooftop (abandoned workshop), Smithy Rooftop, Cottage Rooftop
- Middle row: Rain Gutter Walkway, Chimney Forest, Narrow Ridge
- North row: Flat Roof, Merchant's Rooftop (Gareth's, max_height=0), General Store Rooftop
- Access: drainpipe (ClimbableMixin) in back alley behind vacant workshop, fly from Artisan's Way W1 (height 1+), hidden wardrobe in Gareth's bedroom
- Merchant's Rooftop one level higher than neighbours — requires height 1 from adjacent rooms
- Dark at night, weather-exposed, no lampposts. Thieves' guild territory.
- Own builder: `rooftops.py`

**Cemetery** (west of north road, `millholm_cemetery`)
- 2x3 grid: Overgrown Graves, Untended Graves, Tended Graves (with bench), Paupers' Corner, Old Crypts + Millholm Cemetery (RoomCemetery bind room)
- Three family tombs off the west edge (doors to open):
  - **Stonefield Tomb** (3 rooms): trapped iron door (dc 8), tripwire (dc 8), burial chamber with 3 zone-spawned skeletons (L1, 10 HP, undead-tagged, 18min respawn) + lootable sarcophagus (10 gold max + basic scroll)
  - **Goldwheat Tomb** (1 room): tended, fresh flowers, family history
  - **Ironhand Crypt** (1 room): martial, weapons on walls, soldier's burials
- Outdoor rooms: RURAL terrain, weather-exposed, dark at night, max_height=1
- Tomb interiors: UNDERGROUND terrain, max_height=0
- Own builder: `cemetery.py`

**Lake** (north of town via procedural passage, `millholm_lake`)
- Lake Track (town side, `millholm_town` district) → procedural lake_passage (5 rooms, shared instance, hourly reset) → Lake Shore
- 3 shore rooms: Western Lake Shore (willows, heron), Lake Shore (jetty), Eastern Lake Shore (rocky platforms)
- Millholm Junior Sailing Club (RoomGateway — sail to Far Shore, BASIC cartography + Cog)
- Lakeside Boatyard (RoomCrafting shipyard, BASIC mastery cap for Cog)
- Far Shore Landing (RoomGateway stub for future expansion)
- 5 shallows rooms (max_depth=-1, WATER terrain — swim practice)
- 5 deep water rooms (max_depth=-2, WATER terrain — diving practice)
- Edge rooms loop back to themselves (lake feels endless)
- Atmospheric mobs: ducks, ducklings (shore+shallows), frogs (shallows), pike (deep water, BaseMarineMob)
- Old Barnacle Bob: eccentric sailing/shipwright trainer (BASIC mastery, LLM personality)
- Own builder: `northern.py`

**Sewers** (beneath town)
- Accessible via hidden door in Harvest Moon cellar (find_dc=10) and abandoned house (find_dc=10)
- Sewer tunnels, cistern branch with ancient pre-human stonework
- Hidden Thieves' Lair behind crumbling wall (find_dc=12) — 8 rooms
- **Gareth Stonefield** — thief guildmaster (LLMGuildmasterNPC, vector memory). Respectable merchant above, guild boss below.
- **Vex** — 2IC, runs the night shift (LLMRoleplayNPC, vector memory). Sardonic, dangerous, ambitious.
- Whisper — thief trainer
- Thieves' Gauntlet — static 3-room challenge off Training Alcove (traps, hidden lever, locked vault with guild token)

#### Zone Transitions

| Direction | Destination | Cartography Required | Notes |
|---|---|---|---|
| Far east (past woods) | Ironback Peaks OR Cloverfen | BASIC | Random which discovered first |
| Far south (through Southern District) | The Shadowsward | SKILLED | Higher tier — player must return when ready |

#### Resources

| Resource | Source | Processing |
|---|---|---|
| Wheat | Farms (harvest) | Windmill → flour → Bakery → bread |
| Cotton | Farms (pick) | Tailor → cloth goods |
| Wood | Base Woods (chop) | Sawmill → timber → Carpenter → wood goods |
| Hide | Mob drops | Tanner's hut → leather → Leatherworker → leather goods |
| Copper Ore | Abandoned Mine | Smelter → copper ingot |
| Tin Ore | Abandoned Mine | Smelter → tin ingot |
| Bronze Ingot | — | Smelter (copper + tin → bronze) → Smithy → bronze weapons/armor |
| Moonpetal | Farms, Southern District (most supply) | Apothecary → moonpetal essence (base for all potions) |
| Windroot | Abandoned Mine (windroot hollow) | + moonpetal essence → Potion of the Zephyr (+move) |
| Arcane Dust | Faerie Grove (hidden, deep woods) | + moonpetal essence → Potion of the Wellspring (+mana) |
| Other alchemy herbs | Various drops, gathering | Apothecary → potions |

#### Narrative Seeds

- Founding families (Stonefield, Brightwater, Goldwheat, Ironhand) — faction/quest lines
- Ancient stonework in deep sewers — pre-human civilisation mystery
- Thieves' Guild under Shadow Mistress Vex — organised crime below the surface
- Abandoned farm — unexplained failure, quest hook
- Bricked-up passage in sewers — sealed secret
- Faerie Grove — magical hidden enclave, hints at deeper magic in the world
- Kobold infestation of the mine — why did they come? What drove out the miners?

---

### 2. Ironback Peaks — *The Dwarven Homeland*

**Level:** Low-mid | **Climate:** Arctic/Mountain | **Position:** East of Millholm

Heavily populated by dwarves. The next tier of mining resources beyond Millholm's beginner ores. Deep mines, mountain halls, forges.

**Key features:**
- Mid-tier mining resources (silver, gems, better ores — specifics TBD)
- Dwarven culture, language, architecture
- **Seasonal weather effects:** Snowbound in winter — mountain passes become impassable on foot. Players must fly or use underground tunnel routes to navigate. Creates seasonal economic impact on ore supply.
- Underground route network (mines/tunnels) as alternative to surface passes
- Dwarven guildmasters for higher-level advancement

**Design notes:**
- Winter pass closures create demand for cartographers who've mapped the underground routes
- Dwarven language (already in Languages enum) becomes useful here
- Natural home for higher-level blacksmithing recipes
- Verticality: deep mines below, mountain peaks above

---

### 3. The Shadowsward — *The Military Frontier*

**Level:** Low-mid to mid | **Climate:** Temperate | **Position:** West/SW of Millholm

Inspired by the Borderlands from *The Wheel of Time*. A militaristic zone where the entire culture is defined by holding the line against the evil that pours from Shadowroot. Tough, disciplined people. Fortified settlements. Constant vigilance.

**Key features:**
- Military culture — fortresses, watchtowers, training grounds
- Hardened warrior NPCs, veteran trainers
- Higher-level combat training available
- Patrols, guard rotations, war councils
- The frontier itself — the boundary between civilisation and corruption

**Atmosphere:** Grim determination. No frivolity. Every person has a role in the defence. Children train with weapons. The land is scarred by generations of conflict. Hope is a luxury.

---

### 4. Shadowroot — *The Corrupted Wasteland*

**Level:** Mid to high | **Climate:** Corrupted (unique weather?) | **Position:** Beyond the Shadowsward

Inspired by Shadowroot from *The Wheel of Time*. A poisoned, twisted landscape where evil creatures spawn and pour forth to destroy all that is good. This is what the Shadowsward exists to contain.

**Key features:**
- High-level hostile zone — everything here wants to kill you
- Twisted mobs: corrupted animals, undead, dark creatures, things that shouldn't exist
- Environmental hazards: toxic terrain, darkness, corruption effects
- Source of dark/necromantic resources? Rare drops from Shadowroot creatures
- Boss encounters — the things that lead the hordes

**Design notes:**
- Players entering Shadowroot should feel genuine dread
- Could have corruption mechanic — the longer you stay, the more it affects you
- Natural home for necromancy-related content
- High-risk, high-reward resource gathering

---

### 5. The Scalded Waste — *The Harsh Desert*

**Level:** Mid | **Climate:** Desert | **Position:** Adjacent to Shadowroot, reachable from the Shadowsward

Inspired by the Aiel Waste from *The Wheel of Time*. A brutal desert landscape inhabited by fierce desert warrior people. The land itself is as deadly as any enemy.

**Key features:**
- Desert survival mechanics — heat, water scarcity, sandstorms
- Fierce warrior culture with unique combat traditions
- Desert-specific resources (exotic herbs, rare minerals exposed by erosion)
- Hidden oases as safe zones within the hostile terrain
- Ancient ruins half-buried in sand

**Atmosphere:** Stark beauty. The people here are hard because the land demands it. Honour and combat prowess are everything. Outsiders are treated with suspicion until they prove themselves.

---

### 5b. Zharavan — *Beyond the Waste*

**Level:** High | **Climate:** Desert/Oasis | **Position:** Far west — beyond the Scalded Waste, westernmost zone on the continent

Inspired by Shara from *The Wheel of Time*. An isolationist civilisation on the far side of the desert that most people on the continent don't believe exists. Opulent, ancient, secretive, and deeply suspicious of outsiders. This is late-game discovery content — most players won't find it on their first or even second playthrough.

**Key features:**
- **Isolationist culture** — they don't trade with the outside world. Entry is difficult. They may not want you there at all.
- **Unique magical tradition** — different from academic mages, divine clerics, shamanic bayou magic, or elven enchanting. Something entirely its own (silk-weaving magic? Pattern-based enchanting? Dream magic?)
- **Exclusive high-tier resources** — materials that don't exist anywhere else on the starting continent. Players who discover this place access crafting ingredients nobody else knows about.
- **Unique language** — not spoken anywhere else. Players need COMPREHEND_LANGUAGES or must learn it here. Adds to the sense of alien otherness.
- **Cultural shock** — after the harsh austerity of the Scalded Waste, stepping into a refined, opulent, self-contained civilisation. Silk, gold, perfume, art — but with an undercurrent of control and secrecy.

**Gateway design:**
- Route from Scalded Waste → Zharavan is **hidden** with low explore_chance
- Requires **Master-level cartography** to map (not GM — that tier is reserved for cross-ocean routes, but Master is still very high)
- High food cost — the crossing takes significant supplies
- The desert warriors of the Scalded Waste may know rumours of the city but have never been welcomed there — quest breadcrumbs from desert NPCs

**Atmosphere:** Opulent, controlled, ancient, watchful. Behind every silk curtain, someone is listening. Beautiful gardens fed by hidden aquifers. Architecture that seems to grow from the sand itself. The people are polite but reveal nothing. You are a guest, and guests are expected to leave.

**Design notes:**
- Deliberately designed as remort-2+ discovery content — players who reach it feel they've found something truly special
- A cartographer who maps the route across the waste holds one of the most valuable maps on the continent
- Potential connection to the Ancient Builders mystery? An isolationist civilisation that has preserved knowledge everyone else lost?
- Could house unique class advancement options or remort-only content
- The secrecy itself is the hook — what are they hiding?

---

### 6. Saltspray Bay — *The Great Harbour*

**Level:** Low-mid to mid | **Climate:** Temperate/Coastal | **Position:** SE of Ironback Peaks, east coast

The largest city on the starting continent. A major European-style port city — stone quays, tall ships, merchant guilds, naval tradition. Think Bristol, Amsterdam, or Lisbon. This is where the maritime game begins.

**Key features:**
- **Shipyards** — where shipwright-skilled players build ships (from basic boats to ocean-going vessels)
- **Docks / Gateway rooms** — departure point for the eastern island chain
- Merchant guilds, naval offices, chandlers, rope-walks
- Mid-level crafting facilities, diverse shops
- Larger and more complex than Millholm — proper city with districts
- Multiple guildmasters for mid-level class advancement

**Design notes:**
- This is where players first encounter the maritime progression system
- Shipwright trainers here
- Seamanship trainers here
- Cartography becomes critical from this point forward
- Named after its position — open to better name ideas

---

### 7. Cloverfen — *The Halfling Homeland*

**Level:** Low to low-mid | **Climate:** Temperate | **Position:** Between Millholm and Saltspray Bay, west of Saltspray Bay

A gentle, rolling countryside of green hills, burrow-homes, orchards, and breweries. The Halfling homeland — peaceful, pastoral, and deceptively comfortable. Connects to Millholm (plains road), Ironback Peaks (mountain-to-lowland trade route), and Saltspray Bay (short pastoral route), forming a triangle of accessible early-game zones.

**Key features:**
- Halfling culture — burrow-homes, communal feasting, suspicion of adventure
- **Alchemy (culinary tradition)** — rich gardens grow unique food-based alchemy ingredients found nowhere else. Halfling potion-brewing tradition uses herbs, roots, fruits, and garden-grown reagents rather than magical herbs. EXPERT alchemy training available. Players can build a whole career farming ingredients and brewing potions here.
- **Agricultural abundance** — richer farming than Millholm. Unique garden herbs and orchard ingredients for the culinary alchemy tradition.
- Trade relationship with Ironback Peaks dwarves — halflings farm, dwarves mine, they trade
- Halfling language (already in Languages enum) useful here
- Deceptively safe — but what's in the old barrow mounds on the hills? (dungeon hook)

**Atmosphere:** Cosy, well-fed, hospitable but provincial. Halflings are friendly but think adventurers are slightly mad. Good food, good ale, comfortable beds. The kind of place you don't want to leave — which is exactly the point.

**Design notes:**
- The "easy road" from Millholm — lower level than Ironback Peaks or the Shadowsward
- Natural home for culinary alchemy tradition — unique ingredient pool, unique recipes
- The dwarven trade route creates economic interplay (ore for food, tools for crops)
- Barrow mounds or old ruins as dungeon content for low-level parties
- Southeast route to The Bayou for players heading south

---

### 8. The Bayou — *The Murky Depths*

**Level:** Low-mid to mid | **Climate:** Tropical/Swamp | **Position:** South of Cloverfen, between Saltspray Bay and Kashoryu

A sprawling swamp/bayou zone — thick with mist, insects, and things that slither. Reachable from both Cloverfen (southeast) and Saltspray Bay (south), this is the transition zone between the temperate north and the tropical south.

**Key features:**
- Swamp terrain — difficult movement, visibility reduced by fog and canopy
- Poison ingredients, venomous mobs (snakes, gators, swamp creatures)
- Shamanic/voodoo flavour — a different magical tradition from the academic mages or divine clerics
- Unique alchemy herbs that only grow in swamp conditions
- Reptilian/amphibian mobs not found elsewhere
- Ruins overgrown by swamp (Ancient Builders connection? Jungle ruins swallowed by the bayou?)

**Atmosphere:** Oppressive, humid, alive with sounds you can't identify. The water is never still. Things watch from just below the surface. Beautiful in a dangerous way — bioluminescent fungi, fireflies, flowering vines that strangle.

**Design notes:**
- Poison-related alchemy ingredients as exclusive resources (return hook)
- Potential home for necromancy-adjacent content (swamp spirits, ancestor worship)
- Could house a voodoo/shaman NPC type with unique quest flavour
- The Ancient Builder ruins here could be a key lore stop on the mystery arc

---

### 9. The Aethenveil — *The Ancient Forest*

**Level:** Mid | **Climate:** Temperate/Forest | **Position:** Between Scalded Waste and Kashoryu

An ancient, deep forest homeland of the Elves. Positioned as a link zone between the harsh desert of the Scalded Waste and the tropical Kashoryu, the Aethenveil are a cultural bridge — refined, ancient, and deeply connected to the history of the world.

**Key features:**
- Elven culture — tree-top cities, ancient libraries, long memory, refined craftsmanship
- **Lore keepers** — if anyone knows about the Ancient Builders, it's the Elves. Their libraries hold fragments, their elders remember stories passed down through millennia. This is a critical stop on the mystery arc.
- Elven language (already in Languages enum) useful here
- **Enchanting focus** — natural home for advanced enchanting recipes and rare magical ingredients. Elven craftsmanship is legendary.
- Canopy verticality — ground-level paths and treetop walkways (uses vertical movement system)
- Ancient groves, sacred springs, ruins integrated respectfully into the forest

**Atmosphere:** Timeless, quiet, luminous. The forest feels aware. Light filters through canopy in shafts of gold and green. Everything is old — the trees, the stones, the paths worn smooth by centuries of elven feet. Outsiders are tolerated, not welcomed, until they prove worthy.

**Design notes:**
- Link zone: Scalded Waste ↔ Aethenveil ↔ Kashoryu creates a southern overland route
- Elven guildmasters for mid-to-high level advancement
- The lore connection to the Ancient Builders mystery makes this zone essential for the main quest arc
- Verticality (treetop routes) rewards fly or climbing, offers alternative paths
- Unique enchanting ingredients as exclusive resource (return hook)

---

### 10. The Kashoryu — *The Eastern Gateway*

**Level:** Mid | **Climate:** Tropical | **Position:** Southern coast

A tropical port city with **Asian/Oriental cultural theming** — a stark contrast to Saltspray Bay's European character. Wooden architecture, junks and sampans, lantern-lit night markets, spice trade, silk, martial arts tradition.

**Key features:**
- **Ninja Dojo** — hidden (requires subterfuge or a contact to find?). Trainers and guildmaster for the Ninja class. Players who want Ninja must travel here to access it — the journey is part of earning it.
- **Secondary port access** — once players reach Expert level seamanship, they can sail the coastal route between Kashoryu and Saltspray Bay
- **Unique island access** — some islands reachable only from here, not Saltspray Bay. Different flavour (Pacific/Asian themed?)
- **Solendra** — secret island accessible only at GM seamanship from Kashoryu. Something truly special. *Details TBD — leave this open.*
- Spice markets, tea houses, silk traders
- Unique tropical crafting ingredients

**Design notes:**
- Two ports with different cultural identities makes the world feel large and diverse
- Ninja guildmaster placement here forces meaningful travel for that class choice
- Solendra should be something players talk about in whispers — word of its existence spreads, but the route is known only to the greatest sailors

---

## Maritime Progression & The Island Chain

> For the full interzone travel system — triple skill gate mechanics, ship tiers, `explore`/`sail` commands, route map NFTs, and the BaseGate architecture — see `design/INTERZONE_TRAVEL.md`.

### Island Design (Partial List — Ideas)

Each island must have a **discovery hook** (reason to go the first time) and a **return hook** (reason to come back).

| Island | Tier | Theme | Discovery Hook | Return Hook |
|---|---|---|---|---|
| Teotlan Ruin | BASIC (Cog) | Dense tropical jungle has reclaimed the magnificent stone temples and stepped pyramids of a lost civilisation. Think D&D Isle of Dread. | TBA | TBA |
| Calenport | SKILLED (Caravel) | Pirate haven, lawless port | Black market, pirate contacts | Smuggler trade routes, PvP zone? |
| Port Shadowmere | SKILLED (Caravel) | A mysterious port where locals are pleasant and helpful by day, conducting normal trade and offering hospitality. As darkness falls, a sinister transformation occurs — ships that stay past sunset rarely depart | TBA | TBA |
| Amber Shore | BASIC (Cog) | Colonial disease in reverse — the island's inhabitants are immune/unaffected, but any visitor contracts the sickness | TBA | TBA |
| The Arcane Sanctum | EXPERT (Brigantine) | A mist-shrouded island where ancient hermit mages have withdrawn from worldly concerns to pursue pure magical study | Rare spell scrolls, magical knowledge | Unique alchemy ingredients (Shimmerleaf), spell research |
| Oldbone Island | EXPERT (Brigantine) | Jurassic Park analog — living dinosaurs | TBA | TBA |
| Guildmere Island | MASTER (Carrack) | Advanced mercantile society with strong seamanship and trade culture | TBA | TBA |
| Solendra | GM (Galleon, Kashoryu only) | Mystery — deliberately an easter egg. The entire island chain progression runs from Saltspray Bay. Nobody finds anything new exploring from Kashoryu through SKILLED, EXPERT, or MASTER. Then one day, a GM cartographer with a Galleon explores from Kashoryu and finds something that shouldn't exist. Theme TBA — the surprise is the point. | The discovery itself | TBA |

### Island Resource Exclusivity

Each island should have **at least one exclusive resource or recipe** that only spawns/is learnable there. This creates:
- Genuine economic value for the first players to reach an island
- Trade routes between islands and the mainland
- Reasons to return repeatedly (harvesting runs)
- Cartographer value — maps to resource-rich islands are highly sought after

---

## The Vaathari

**Level:** Endgame | **Climate:** Varied | **Position:** Across the ocean, east

Accessible only to grand master sailors. This is where the top-tier content lives.

**Key features:**
- **Hall of Worlds access** — the primary entrance is here. There may be exits (one-way doors *out* of the Hall) on the starting continent, but players can't enter until they reach the Vaathari. Players who stumble across these mysterious doors early in the game will spend the whole journey wondering about them.
- **Olympus** (or equivalent divine realm) — home of the **Forge of the Gods**, one of the only (possibly *the* only) Grand Master level forge. A GM blacksmith *must* come here to craft GM-level items. This makes the Vaathari essential even for non-combat endgame players.
- **The Celestial Threshold** — Aasimar homeland. A radiant city/settlement at the base of the mountain that leads to Olympus, or at the entrance to the divine realm itself. The Aasimar are servants and emissaries of the gods — their culture is shaped by proximity to the divine. Celestial language (already in Languages enum) is spoken here.
  - **Aasimar is a remort-only race** — players must have completed at least one full playthrough before unlocking the option. By the time a player can *become* Aasimar, they will likely have already visited the Vaathari and encountered Aasimar culture. The homeland is aspirational content that becomes *personal* on remort.
  - Aasimar NPCs, trainers, and unique celestial crafting recipes
  - Celestial-themed quests and lore connecting to Olympus and the divine hierarchy
- Top-tier resources, recipes, enemies, and content
- Full zone details TBD

---

## The Hall of Worlds

*Inspired by Raymond E. Feist's "Magician" trilogy.*

An interdimensional corridor that exists outside normal reality. A long north-south looping path that curves back on itself — it can hold infinite doors and always seems to go on forever.

### Structure
- **The Path** — a narrow walkway through void. N/S loop (technically a ring, but players perceive it as an endless corridor). New doors can be added to either side indefinitely.
- **The Void** — step off the side of the path and you float in nothing. Forever. Death. Players receive a Y/N warning before stepping off. This makes the Hall feel alien and dangerous, not just a fancy corridor.
- **The Doors** — periodically placed on either side of the path. One-way: some lead *into* a world, others lead *out of* a world. You enter through one door and must find the exit door on the other side.

### World Types

The worlds behind the doors can be *anything*. The Hall is the licence to break every tonal rule of the main game:

- **Sprint worlds** — 5-room dashes across toxic/lethal environments. You take damage every tick. Run from entry door to exit door. In lore, the exit is closer to your destination in the Hall than the entrance was — a shortcut through hell.
- **Full worlds** — entire zones with their own cultures, economies, quests. Prehistoric (dinosaurs), futuristic, dying worlds, alien civilisations.
- **Pop-culture worlds** — Red Dwarf crew, Doctor Who, Zaphod Beeblebrox and Arthur Dent, whatever can be imagined. Tone doesn't need to match the main game because these are explicitly *other realities*.
- **Challenge worlds** — puzzle zones, gauntlet runs, boss arenas
- **Resource worlds** — unique crafting materials only found in specific planes

### Design Notes
- The loop structure means content can be added forever without restructuring
- One-way doors create commitment — you enter, you must find the exit
- Sprint worlds are cheap to build (5 rooms) but create memorable experiences
- Pop-culture worlds are the long-tail creative playground — low stakes, high fun
- The Hall itself should feel *wrong* — not evil, just fundamentally alien. The void is not darkness, it's *nothing*.

---

## Exploration & Cartography

The world is explored through discovery, not a minimap. Zone connections exist but are hidden — players must find routes to unknown zones before they can travel them freely. Two systems underpin this:

- **Inter-zone route discovery** — zones are connected by gateway exits invisible until explored. A party needs a cartographer of sufficient mastery to discover a new route; once found, a tradeable route map lets anyone travel it.
- **Intra-zone district mapping** — within a zone, cartographers survey rooms to fill in pre-drawn ASCII district maps, which are tradeable as NFTs. A 100% sewer map is genuinely valuable to a party heading underground.

See **[design/CARTOGRAPHY.md](CARTOGRAPHY.md)** for the full mechanics design.

---

## Progression & Resource Distribution

*Big picture mapping of what becomes available where. This drives zone design — every zone's content follows from its position on this map.*

### Mastery Tier Geography

The game's six mastery tiers are distributed in geographic rings, but the cutover is **gradual, not clean**. Each ring introduces some training at the next tier while completing the previous tier's coverage. This means players always have a reason to explore the next ring — there's something they can't finish where they are.

| Ring | Mastery Range | Zones | What's Available |
|---|---|---|---|
| Starter | BASIC + some SKILLED | Millholm | All skills trainable to BASIC. Less in-demand skills (carpentry, tailoring, leatherworking, battleskills, alertness) trainable to SKILLED. High-demand skills (blacksmithing, enchanting, alchemy, jewellery) cap at BASIC — must travel for SKILLED. |
| Second ring | Rest of SKILLED + some EXPERT | Ironback Peaks, Shadowsward, Saltspray Bay, Cloverfen, The Bayou, Scalded Waste | Completes SKILLED for all skills. Zone specialties begin reaching EXPERT. Each zone excels at specific skills — players choose direction based on what they want to train. |
| Third ring | Rest of EXPERT + some MASTER | Aethenveil, Kashoryu, Zharavan, The Arcane Sanctum, Oldbone Island | Completes EXPERT for all skills. Zone specialties begin reaching MASTER. Remote, dangerous, or culturally exclusive — earned through journey. |
| Island tier | MASTER + some GM | Guildmere Island, Port Shadowmere, Calenport, Teotlan Ruin, Amber Shore | TBA — island chain deep progression. Only reachable via sea routes from third-ring ports. |
| Endgame | Rest of MASTER + GRANDMASTER | Vaathari, Solendra, Olympus | Completes MASTER. The only place in the game to reach GM. Requires crossing the ocean from Guildmere Island. |

#### The Buyers' Remorse Mechanic

The gradual cutover is deliberately designed to create a **buyers' remorse** effect. In Millholm, players have limited general skill points and can only see the skills available to them. The skills trainable to SKILLED in Millholm are useful but not the most exciting — carpentry, tailoring, leatherworking, basic combat. Players spend points on what's available.

Then they reach the second ring and discover blacksmithing, enchanting, alchemy, jewellery — the skills everyone *really* wants — are now trainable to SKILLED (and the zone specialty ones to EXPERT). But they've already spent points on the starter skills. This creates natural demand for **remort** (prestige reset) and gives players a reason to plan their second life differently.

**Early-available skills (SKILLED in Millholm):** Carpentry, Tailoring, Leatherworking, Battleskills, Alertness — solid, practical, but not glamorous.

**Deferred skills (BASIC only in Millholm):** Blacksmithing, Enchanting, Alchemy, Jewellery — the skills players will covet once they see what's possible in the second ring.

### Class Advancement (Guildmaster Chains)

Each class has a chain of guildmasters in increasingly distant zones. The `next_guildmaster_hint` system tells players where to go next. Multiple paths exist — a warrior can level through the Shadowsward military route OR the dwarven mountain route.

| Level Range | Zones | Notes |
|---|---|---|
| 1–5 | Millholm | All four base classes (Warrior, Thief, Mage, Cleric). Safe starter progression. The power gap between level 1 and 5 is significant — players outgrow Millholm quickly. |
| 6–10 | Second ring (Ironback Peaks, Cloverfen, Saltspray Bay, Shadowsward) | Class-appropriate: warriors→Shadowsward, mages→Saltspray Bay/Ironback Peaks, clerics→Shadowsward. Players choose their direction. |
| 11–20 | Third ring (Bayou, Kashoryu, Scalded Waste, Shadowroot, Aethenveil, closer islands) | Specialist zones: ninja→Kashoryu, druid→Bayou/Aethenveil, bard→Saltspray Bay/Islands. Cultural training traditions. |
| 21–30 | Further islands (Port Shadowmere, Calenport, Teotlan Ruin, Amber Shore, The Arcane Sanctum, Oldbone Island) | Remote island guildmasters. Maritime skills required to reach them. Deep specialisation. |
| 31–40 | Guildmere Island, Solendra | TBA — deep island endgame tier. Only reachable via MASTER+ sea routes. |
| 36–40 | Vaathari | Final advancement. The pinnacle trainers for every class live on Vaathari. |
| 40+ | Hall of Worlds / Olympus | Transcendent. Post-cap progression for those who've mastered everything. |

Remort-locked classes have their own chains:
- **Paladin** (remort 1): Guildmasters in the Shadowsward (military-divine tradition), Aethenveil, Vaathari
- **Bard** (remort 2): Guildmasters in Saltspray Bay (port culture), Kashoryu, Islands, Vaathari
- **Barbarian, Ranger, Druid, Ninja** (when implemented): Placed in culturally appropriate zones

### Crafting Skill Specialisation by Zone

Each zone has a crafting specialty — one or two skills it trains higher than anywhere else nearby. This creates a "where do I go to train X?" answer for every skill.

| Zone | Crafting Specialty | Max Mastery | Why (Narrative Reason) |
|---|---|---|---|
| Millholm | All 7 crafting skills | BASIC (some SKILLED) | Small-town craftspeople. Less in-demand skills to SKILLED, high-demand skills capped at BASIC (see buyers' remorse above). |
| Ironback Peaks | Blacksmithing, Jewellery | EXPERT | Dwarven mastery of metalwork and gemcraft. Best forges on the continent. |
| Cloverfen | Alchemy (culinary tradition) | EXPERT | Rich gardens, unique food-based alchemy ingredients. Same alchemy skill, different ingredient pool and recipes only learnable here. Halfling potion-brewing tradition. A viable full career — farm ingredients, brew potions, sell on AMMs. |
| The Shadowsward | Blacksmithing (weapons focus) | EXPERT | Military smiths — weapons and armor for war, not art. |
| Saltspray Bay | Carpentry, Shipwright (new) | EXPERT | Port city. Shipyards, timber trade, naval construction. |
| The Bayou | Alchemy (swamp tradition) | EXPERT | Swamp herbs, poison lore, shamanic brewing tradition. Unique ingredients, different recipes from Cloverfen. |
| Scalded Waste | Leatherworking | EXPERT | Desert warriors work in leather and hide. Exotic beast hides. |
| Aethenveil | Enchanting | MASTER | The elves are the continent's master enchanters. Ancient techniques. |
| Kashoryu | Tailoring | EXPERT | Silk, exotic fabrics, Asian textile tradition. |
| Zharavan | Enchanting (unique tradition) | MASTER | Different enchanting path — silk-weave magic? Pattern enchanting? Not the same as elven. |
| Calenport | TBA | TBA | TBA |
| Teotlan Ruin | TBA | TBA | TBA |
| Port Shadowmere | TBA | TBA | TBA |
| Amber Shore | TBA | TBA | TBA |
| The Arcane Sanctum | TBA | TBA | TBA |
| Oldbone Island | TBA | TBA | TBA |
| Guildmere Island | TBA | TBA | TBA |
| Solendra | TBA | TBA | TBA |
| Olympus (Vaathari) | All crafting | GRANDMASTER | The Forge of the Gods. The only place in the game to reach GM crafting. |

**Alchemy progression path:** Millholm (BASIC) → Cloverfen or Bayou (EXPERT, different traditions/recipes) → outer island (MASTER) → Vaathari (GRANDMASTER). Potions are single-use consumables — unlike other NFTs they are destroyed on use, creating permanent demand. Players can build an entire career around alchemy ingredient farming and potion crafting.

### Exploration Skill Training

These skills gate geographic access. Training them further out means you can *reach* further out — a self-reinforcing loop.

| Skill | BASIC | SKILLED | EXPERT | MASTER | GRANDMASTER |
|---|---|---|---|---|---|
| Cartography | Millholm | Saltspray Bay | Kashoryu | Guildmere Island (TBA) | Vaathari |
| Shipwright | — | Saltspray Bay | Kashoryu | Guildmere Island (TBA) | Vaathari |
| Seamanship | — | Saltspray Bay | Kashoryu | Guildmere Island (TBA) | Vaathari |
| Animal Handling | Cloverfen | Shadowsward | Scalded Waste | Aethenveil | Vaathari |

Maritime skills (Shipwright, Seamanship, Cartography) are concentrated in port cities and islands — you learn to sail where ships are. Animal Handling follows an overland path — pastoral Cloverfen → military Shadowsward → desert riders → elven beast-speakers.

### Combat Skill Training

| Zone | Combat Focus | Max Mastery | Narrative |
|---|---|---|---|
| Millholm | All base combat skills | SKILLED | Town militia and guild trainers. Competent, not elite. |
| Ironback Peaks | Bash, Pummel, heavy weapons | EXPERT | Dwarves fight in tight quarters with heavy weapons. |
| The Shadowsward | Warrior/Paladin skills, Strategy, Protect | EXPERT | Military academy. The best soldiers on the continent. |
| The Bayou | Survivalist, Nature Magic | EXPERT | Druid/Ranger wilderness training. Swamp survival. |
| Aethenveil | Archery, Nature Magic | EXPERT–MASTER | Elven combat tradition. Bow mastery, forest warfare. |
| Scalded Waste | Unarmed, Frenzy, Battleskills | EXPERT | Desert warrior combat tradition. Brutal, efficient. |
| Kashoryu | Stealth, Stab, Assassinate, all Ninja skills | EXPERT–MASTER | Ninja dojo. Shadow arts and martial discipline. |
| Zharavan | Unique martial tradition? | MASTER | Isolationist fighting style — something no one else teaches. |
| Calenport | TBA | TBA | TBA |
| Teotlan Ruin | TBA | TBA | TBA |
| Port Shadowmere | TBA | TBA | TBA |
| Amber Shore | TBA | TBA | TBA |
| The Arcane Sanctum | TBA | TBA | TBA |
| Oldbone Island | TBA | TBA | TBA |
| Guildmere Island | TBA | TBA | TBA |
| Solendra | TBA | TBA | TBA |
| Vaathari | All combat | MASTER–GM | The greatest fighters in the world train here. |

### Resource Tiers

Resources are distributed across zones to create a material progression that matches mastery tiers. Higher-tier materials require travelling to more distant zones.

#### Weapon/Armor Material Progression

| Mastery | Metal | Source Zone | Notes |
|---|---|---|---|
| BASIC | Bronze (Copper + Tin) | Millholm | Starter tier. Soft metal, adequate weapons. |
| SKILLED | Iron | Ironback Peaks | Standard military grade. Reliable. Iron ore found in the mountains, not Millholm — players must travel to the dwarves for the next tier of gear. |
| EXPERT | **Steel** (Iron + Coal) | Inner islands, south of Saltspray Bay | Dwarven smelting technique learned elsewhere. Third-tier zones have the advanced smelters and coal deposits. |
| MASTER | **Mithril** | Outer islands, Atlantis | Legendary. Light, strong, magically resonant. Very rare. Found in deep oceanic mines and Atlantean forges. |
| GRANDMASTER | **Adamantine** | Vaathari / Olympus | Divine material. Only workable at the Forge of the Gods. The hardest substance in existence. |

#### Jewellery Material Progression

| Mastery | Metal | Gems | Source Zone |
|---|---|---|---|
| BASIC | Copper | — | Millholm |
| SKILLED | Pewter (Tin+Lead), Silver | — | Millholm, Ironback Peaks |
| EXPERT | **Gold** | Ruby, Emerald, Diamond | Islands, deep Ironback Peaks |
| MASTER | Mithril, Gold | Rare gems | Islands, Aethenveil |
| GRANDMASTER | **Adamantine**, Gold | Divine gems | Vaathari |

#### Zone-Exclusive Resources

These are the return hooks — materials that can only be gathered in one zone, ensuring players return.

| Zone | Exclusive Resource(s) | Used For | Notes |
|---|---|---|---|
| Ironback Peaks | Coal, Silver Ore, **Mithril Ore** (deep, MASTER mining) | Steel alloy, silver jewellery, mithril crafting | Coal+Iron→Steel is the key EXPERT-tier unlock |
| Cloverfen | **Cloverfen herbs & garden ingredients** (food-based alchemy reagents) | Alchemy (culinary tradition) — potions, elixirs, tonics | Rich gardens grow ingredients found nowhere else. Potion staples for the whole game economy. |
| The Bayou | **Bayou Venom**, unique swamp herbs | Poison alchemy, advanced potions | Exclusive poison crafting line |
| Aethenveil | **Elven Heartwood** | MASTER enchanting, MASTER carpentry | Magically resonant ancient wood |
| Scalded Waste | **Desert-hardy herbs**, exotic hides | Advanced leatherworking, alchemy | Unique to the harsh climate |
| Zharavan | **Silksteel Thread** | MASTER tailoring, MASTER enchanting | Unique textile-magic material |
| Shadowroot | **Corrupted materials**, dark reagents | Necromancy ingredients, dark alchemy | High risk harvesting — the zone fights back |
| Calenport | TBA | TBA | TBA |
| Teotlan Ruin | TBA | TBA | TBA |
| Port Shadowmere | TBA | TBA | TBA |
| Amber Shore | TBA | TBA | TBA |
| The Arcane Sanctum | TBA | TBA | TBA |
| Oldbone Island | TBA | TBA | TBA |
| Guildmere Island | TBA | TBA | TBA |
| Solendra | TBA | TBA | TBA |
| Vaathari | **Adamantine Ore**, **Ambrosia**, **Aetheric Essence** | GM-tier all crafting | The rarest materials in the game |

#### Processing Room Distribution

Processing rooms (smelter, windmill, etc.) don't require crafting skill — anyone can use them for a gold fee. But higher-tier processing recipes exist only in specific zone smelters/workshops.

| Processing | Standard (Millholm) | Advanced (Zone-specific) |
|---|---|---|
| Smelting | Copper, Tin, Lead, Silver, Bronze, Pewter | **Iron** (Ironback Peaks+), **Steel** (third-tier smelters), **Mithril** (outer islands/Atlantis), **Adamantine** (Forge of the Gods only) |
| Milling | Wheat → Flour | Cloverfen may add specialty grain processing |
| Baking | Flour → Bread | Cloverfen adds advanced food recipes? |
| Sawmill | Wood → Timber | Standard everywhere. Aethenveil may process Heartwood. |
| Tannery | Hide → Leather | Scalded Waste may process exotic hides. |
| Textile | Cotton → Cloth | Zharavan processes Silksteel. |

### New Resources Needed (Summary)

Materials that don't exist yet but are required to fill the progression. Working names — final names TBD.

| Name | Source | Processing | Mastery Tier | Notes |
|---|---|---|---|---|
| Steel Ingot | Ironback Peaks smelter | Iron + Coal (alloy) | EXPERT | The key second-ring unlock |
| Gold Ore → Gold Ingot | Islands, deep Ironback Peaks | Smelter | EXPERT | Precious metal for jewellery |
| Mithril Ore → Mithril Ingot | Deep Ironback Peaks, specific islands | Dwarven deep forge | MASTER | Legendary metal — light, strong, magical |
| Elven Heartwood | Aethenveil | — (raw ingredient) | MASTER | Ancient wood with magical resonance |
| Shimmerleaf | The Arcanum (island) | — (alchemy ingredient) | EXPERT+ | Grows near arcane energy |
| Silksteel Thread | Zharavan | Zharavan textile process | MASTER | Textile-magic hybrid |
| Bayou Venom | The Bayou | — (alchemy ingredient) | EXPERT | Poison crafting exclusive |
| Adamantine Ore → Adamantine Ingot | Vaathari | Forge of the Gods | GM | Divine metal — the hardest substance in existence |
| Ambrosia | Olympus | — (alchemy ingredient) | GM | Divine potion ingredient |
| Aetheric Essence | Vaathari gathering | — (enchanting ingredient) | GM | Crystallised magical energy |
| Zone-specific herbs (various) | Cloverfen, Bayou, Desert, Islands | Per-zone alchemy/cooking | EXPERT+ | Each zone's unique flora |

*Resource IDs to be assigned when implemented. These fill the gaps between current Tier 1 (Millholm) resources and the endgame.*

### Economic Flow Between Zones

Zones are not self-sufficient — they produce and consume different things, creating natural trade routes.

| Zone | Produces | Needs | Trade Partners |
|---|---|---|---|
| Millholm | Food (bread), bronze goods, basic materials, all Tier 1 resources | Higher-tier training, advanced materials, iron+ gear | Everyone (food supplier to the continent) |
| Ironback Peaks | Metal (iron, steel, silver, mithril), gems, coal | Food, cloth, leather, wood | Millholm (food), Cloverfen (potions/food), Saltspray Bay (timber) |
| Cloverfen | Potions (alchemy), unique garden ingredients, superior food | Metal goods, weapons, armor | Ironback Peaks (metal for potions), Millholm (food basics). Potion economy hub — consumables create permanent demand via AMMs. |
| The Shadowsward | Military training, exotic hides (war beasts) | Food, weapons, metal, healing supplies | Millholm (food), Ironback Peaks (steel), Bayou (potions) |
| Saltspray Bay | Ships, timber goods, naval supplies | Metal, food, crew (seamanship training) | Ironback Peaks (metal), Millholm (food), Islands (exotic goods) |
| The Bayou | Potions, poisons, swamp herbs | Food, metal goods, cloth | Cloverfen (food), Millholm (basics) |
| Scalded Waste | Exotic hides, desert herbs, warrior training | Water/food (scarce in desert), metal, cloth | Shadowsward (military exchange), Aethenveil (culture) |
| Aethenveil | Enchanted items, Heartwood, lore | Metal, food, mundane materials | Ironback Peaks (metal), Cloverfen (food), Kashoryu (silk) |
| Kashoryu | Silk, spices, martial training, naval access | Metal, food, raw materials | Saltspray Bay (naval trade), Aethenveil (enchanting) |
| Zharavan | Silksteel, unique enchantments | Very little — isolationist | Scalded Waste (grudging border trade) |
| Calenport | TBA | TBA | TBA |
| Teotlan Ruin | TBA | TBA | TBA |
| Port Shadowmere | TBA | TBA | TBA |
| Amber Shore | TBA | TBA | TBA |
| The Arcane Sanctum | TBA | TBA | TBA |
| Oldbone Island | TBA | TBA | TBA |
| Guildmere Island | TBA | TBA | TBA |
| Solendra | TBA | TBA | TBA |
| Vaathari | GM-tier everything | N/A (endgame, self-sufficient?) | — |

### Multiple Progression Paths

A critical design principle: there is no single "correct" path through the zones. Players who go east and players who go west both progress — just differently.

**Eastern path:** Millholm → Ironback Peaks → Saltspray Bay → Islands → Vaathari
- *Strength:* Metalwork, maritime skills, shipbuilding, island exploration
- *Character:* Builder, trader, explorer, sailor

**Western path:** Millholm → Shadowsward → Shadowroot → Scalded Waste
- *Strength:* Combat, survival, leatherwork, desert lore
- *Character:* Fighter, survivor, warrior-scholar

**Southern path:** Millholm → Cloverfen → Bayou → Kashoryu → Aethenveil
- *Strength:* Food/alchemy, nature magic, stealth, enchanting
- *Character:* Alchemist, druid, ninja, enchanter

**Each path has resources and training the others don't.** A guild that sends members down all three paths covers more ground than one that follows a single road. This is by design — the party composition mechanic extends to the macro level.

---

## Zone Design Principles

Every zone built for FullCircleMUD should satisfy these requirements:

### 1. Discovery Hook
*Why would a player go here the first time?*
- New story content, mystery, quest chain
- Unique NPC, class trainer, or guildmaster
- Something they've heard about but haven't seen

### 2. Return Hook
*Why would a player come back?*
- Exclusive harvestable resource that only spawns here
- Repeatable dungeon with valuable drops
- Crafting facility (processing room) needed for specific recipes
- Trainer with higher mastery caps than previous zones
- Social hub / trade centre

### 3. Exclusive Content
*What can ONLY be found here?*
- At least one unique resource type
- At least one unique recipe or crafting ingredient
- Ideally a unique mob type, NPC, or quest

### 4. Economic Integration
*How does this zone fit the supply chain?*
- What resources enter the game here?
- What processing happens here?
- What do players need to bring with them? (food, gear, materials)
- How does this zone's output feed into other zones' inputs?

### 5. Cultural Identity
*What makes this zone feel different?*
- Distinct architectural style, language, customs
- NPCs with zone-appropriate attitudes and dialogue
- Zone-specific ambient echoes and room description flavour
- Weather and terrain that match the climate

### 6. Progression Role
*Where does this zone sit in the player journey?*
- What level range is it designed for?
- What skills/abilities does it assume players have?
- What does it prepare players for next?
- What gateway routes lead in and out, and what are their conditions?

---

## New Player Onboarding

### Design Goal

Two player types arrive in Millholm: the **new player** (disoriented, just finished character creation) and the **remorting/returning player** (knows the world, back for a reason). The priority is hooking the new player — making the first session so compelling they get addicted, tell their friends, and daydream about the game when they should be working.

### The Bartender — First Contact

The **Harvest Moon bartender** is an LLM-driven NPC backed by vector database / embeddings for persistent memory. He is the first NPC a new player interacts with.

**First visit (new account):**
- Bartender notices he hasn't seen them before
- Welcomes them to Millholm with a natural, conversational tone — like an experienced MUDder greeting a newbie, not a tutorial prompt
- Sizes them up, asks a question or two, offers help
- Points them toward the tutorial: "There's a training hall if you want to learn the ropes, or I can get one of my mates to walk you through it"
- **This replaces the hard static "would you like to go to the tutorial?" menu** with a warm, organic conversation

**Return visits:**
- Bartender remembers previous conversations (stored in embeddings)
- Asks how the tutorial went, references things the player said or asked about
- Reacts to milestones — first death, first level, first time exploring the sewers
- Becomes the player's **anchor NPC** — the one constant in a world full of unknowns
- Every return to the Harvest Moon feels like walking into your local pub

### The LLM-Guided Tutorial

The existing tutorial infrastructure (instanced rooms, items, training dummy, 11-room progression) stays. What changes is the **delivery**.

Instead of static `tutorial_text` that the player reads with a command, the bartender spawns an **LLM-driven guide NPC** — "one of his mates" — who walks alongside the player through the tutorial. The guide:

- **Talks them through each mechanic conversationally** — "Right, see that sword on the wall? Have a look at it — type `look sword`. That's how you examine things around here."
- **Reacts to what the player does** — if they pick something up without being told, the guide acknowledges it. If they're stuck, the guide nudges them.
- **Has personality** — not a help file with legs. Opinionated, funny, encouraging. A character.
- **Answers questions in real time** — the player can just talk to the guide
- **Adapts pacing** — fast learners get moved along quicker, struggling players get more explanation
- **Reports back to the bartender** — when the tutorial ends, the guide's observations feed into the bartender's memory. The bartender knows if they sailed through or struggled.

### Post-Tutorial: Starter Quest Chain

The bartender is the quest hub for the first hour of play. What he offers depends on the path the player took:

**Player took the tutorial → "Fresh Bread" quest**
They already know the mechanics. The bartender gives them their first real-world task:
"Now you've got the basics down, I could use a hand. The bakery's running low and the wheat fields are ripe..."
- Harvest wheat → mill at windmill → bake at bakery → deliver bread
- **Teaches:** the full harvest → process → craft loop in the real world (not tutorial instances)
- **Reward:** gold + XP. Baker now acknowledges you as someone useful.

**Player skipped the tutorial → "Rats in the Cellar" quest**
The bartender reads them as experienced or impatient. Classic MUD opener:
"Well if you're looking for work, there's rats in my cellar that need dealing with."
- Go to the Harvest Moon cellar → kill rats → report back
- **Teaches:** real combat, navigating vertical spaces, finding specific locations
- **Reward:** gold + XP.

**Both paths converge.** Whoever completed their first quest gets the other one next — tutorial players get the rats, skippers get the bread run. The bartender has work for both.

**First-time account bonus:** New accounts (first character, first time through) receive an extra 5 gold on their first quest completion, seeding their first crafting attempts. Tracked per-account so alts and remorts don't benefit.

**Quest tracking:** Completed quests are tracked per-character. No quest can be completed more than once per character.

#### Starter Quest Progression (All From the Bartender)

The bartender feeds quests in a loose chain — each one completed unlocks the next conversation. By quest 5, the player has touched most of Millholm's systems and should be approaching level 2 (~1 hour of play).

| # | Quest | System Taught | Location | Given After |
|---|---|---|---|---|
| 1a | **Fresh Bread** — harvest wheat, mill flour, bake bread, deliver to bakery | Harvest → process → craft chain | Farms, windmill, bakery | Tutorial completion |
| 1b | **Rats in the Cellar** — clear rats from the Harvest Moon cellar | Combat, vertical navigation | Inn cellar | Skipping tutorial |
| 2 | *(whichever of 1a/1b they didn't do first)* | *(see above)* | *(see above)* | Completing quest 1 |
| 3 | **Timber Order** — chop wood, saw into timber, deliver to carpenter | Second resource chain, woods district, sawmill | Woods, sawmill, carpenter | Completing quest 2 |
| 4 | **Supply Run** — deposit gold or goods at the bank for safekeeping | Banking, deposit/withdraw, wealth management | Bank (Order of the Golden Scale) | Completing quest 3 |
| 5 | **Something Below** — investigate strange noises in the sewers, report back | Exploration, danger escalation, knowing when to retreat | Sewers (first few rooms only) | Completing quest 4 |

Quest 5 is the **transition point**. The player has now harvested, processed, crafted, fought, banked, explored three districts, and peeked into the sewers. The bartender's starter chain is done — he tells them to **check the noticeboard just outside the inn door**: "I've got nothing else for you right now, but there's a board just outside — folk around town are always looking for a hand."

#### The Town Noticeboard — "Looking for Work"

The **noticeboard just outside the Harvest Moon** becomes the player's next quest source. This is the shift from guided to self-directed — multiple listings visible at once, the player picks what appeals to them.

**Design:**
- The noticeboard shows several NPC requests simultaneously — player chooses which to take
- Listings are from NPCs they've already met during the bartender chain (baker, carpenter, smithy, etc.) — familiar faces, feels like belonging
- Completed quests disappear from that character's view of the board (per-character tracking)
- Rewards are **practical, not just XP** — bread, basic gear, crafted items. Things that solve real problems (hunger, weak equipment)

**Example Noticeboard Quests:**

| Quest | NPC | Task | Reward | Design Purpose |
|---|---|---|---|---|
| **Baker's Wheat** | Bron (baker) | Deliver 3 wheat | 2 loaves of bread + XP | Reinforces harvesting, keeps player fed |
| **Baker's Firewood** | Bron (baker) | Deliver 3 wood | 2 loaves of bread + XP | Second resource, baker becomes a reliable bread source |
| **Smithy Needs Ore** | Old Hendricks | Deliver copper ore or tin ore | Bronze dagger or bronze cap + XP | Introduces mining, shows crafting output |
| **Cotton Harvest** | Elena Copperkettle (seamstress) | Deliver 3 cloth | Basic clothing item + XP | New resource, new NPC, new district (Brightwater farm). Teaches the cotton → loom → cloth processing chain. |
| **Herb Gathering** | Mara Brightwater (herbalist) | Deliver specific herb | Healing potion + XP | Introduces alchemy ingredients, potion as practical reward |
| **Guard Patrol** | Town guard | Kill X mobs in the woods | Gold + XP | Combat practice, woods exploration |

The noticeboard quests are **repeatable across different characters but one-time per character**. They exist to:
1. Prevent "what do I do now?" paralysis after the bartender chain
2. Reinforce resource chains with real choice about which to pursue
3. Keep the player fed and equipped through practical rewards
4. Introduce NPCs and locations the bartender chain didn't cover
5. Bridge the gap to guild initiation quests and longer storylines

### LLM-Driven NPCs — The First Hour

**Every NPC the player interacts with in the first hour is LLM-driven with persistent memory.** The bartender is the warmup — they got the wow from him. But when Bron the baker has opinions about the wheat quality and grumbles about the miller overcharging, that's when they realise it's not just one special NPC. The whole world is like this.

Each NPC has a **distinct personality and voice** loaded into their system prompt. Memories are stored in vector database embeddings per-NPC, per-player — so every NPC builds a relationship with the player over time.

#### Millholm NPC Personalities

| NPC | Role | Personality | Voice |
|---|---|---|---|
| **The Bartender** | Harvest Moon innkeeper, quest hub, first contact | Warm, worldly, slightly knowing. The veteran who's seen everything. Your anchor. | Calls everyone "friend" or "mate." Casual, reassuring. Drops hints without being obvious. |
| **Bron** | Baker, Goldencrust Bakery | No-nonsense, practical, a bit sharp. Judges you by usefulness. Softens once you've proven reliable. | "Well, it's not the worst wheat I've seen. Barely." Grudging praise is high praise. |
| **Old Hendricks** | Blacksmith | Gruff, perfectionist, few words. Respects hard work, dismisses talkers. Everything measured against his dead mentor. | Grunts approval rather than speaking it. Short sentences. "That'll do." is a compliment. |
| **Elena Copperkettle** | Seamstress | Gossip, warm, knows everyone's business. Talks while she works. Tells you things about other NPCs unprompted. | Chatty, conspiratorial. "Don't tell anyone I told you, but..." She's the social hub. |
| **Mara Brightwater** | Herbalist | Quiet, precise, slightly odd. Talks about plants like they're people. Notices things others miss. | Observant, unsettling. "You've been in the sewers. I can smell it on you." |
| **Town Guard** | Gate/patrol guard | Weary, dutiful, stretched thin. Grateful for help but won't admit it. Dry humour. | Understated, tired. "Another hero. Wonderful. The rats'll be thrilled." |
| **Master Oakwright** | Carpenter | Patient, methodical, teaching nature. Likes explaining how things work. Proud of his craft. | Deliberate, instructive. Talks about wood the way a sommelier talks about wine. |

#### NPC Memory & Cross-Talk

- Each NPC stores memories of interactions with each player character (via embeddings)
- Old Hendricks remembers you brought poor ore last time. Bron remembers you helped when she was short on wheat.
- **NPCs reference each other** — Elena mentions something she heard from Bron about you. The bartender asks if Old Hendricks warmed up to you yet. The guard mentions the baker was grateful for the wheat delivery.
- This creates the feeling that Millholm is a community that **talks about you when you're not there.**

#### NPC World-Knowledge — The Rumour Engine

Each NPC's system prompt includes **trade-appropriate references to the wider world** — zones, resources, and opportunities beyond Millholm. They never say "go to zone X at level Y." They grumble, wish, reminisce, and gossip in ways natural to their profession. The player absorbs a mental map of what's out there without ever reading a zone guide.

| NPC | What they hint at | Zone seeded |
|---|---|---|
| **Old Hendricks** | Grumbles that local bronze is soft compared to the iron the dwarves use in their mountain forges. Mentions his mentor trained in Ironback Peaks. | Ironback Peaks (dwarven mountains, iron) |
| **Mara Brightwater** | Wishes she had certain herbs that only grow in the gentle hills to the south. Mentions recipes she can't make without them. | Cloverfen (alchemy herbs, culinary tradition) |
| **Master Oakwright** | Talks about timber quality — local wood is fine, but the hardwoods from the southern forests are something else entirely. | Bayou / southern zones (rare timber) |
| **The Bartender** | Has stories from travellers — mentions the busy port to the east, the Shadowsward forts to the north, things he's heard from sailors. Widest range of hints. | Saltspray Bay, Shadowsward, islands, anywhere |
| **Elena Copperkettle** | Gossips about a seamstress in the port town who works with silks she's never even seen. Wonders where the cotton traders take their goods. | Saltspray Bay / Kashoryu (trade, silk, exotic cloth) |
| **Town Guard** | Mentions the Shadowsward watch is always recruiting, and that things have been getting worse up north. Talks about reinforcements that never arrive. | Shadowsward (military conflict, Shadowroot) |
| **Brother Aldric** | Speaks of distant temples and holy sites. Mentions pilgrims who pass through heading south or to the islands. | Celestial Threshold, island temples |

By level 5 the player has heard enough fragments to have **opinions about where they want to go first** — and the motivation is personal curiosity, not a quest arrow.

### The Wow Moments

The first hour has **two wow moments**:

1. **The bartender remembers you.** They log in expecting static text and scripted menus. Instead, a character talks to them like a person and remembers what they said. "Wait — this NPC actually knows who I am?"

2. **The SECOND NPC remembers them too.** They deliver wheat to Bron and she has a personality. They visit Old Hendricks and he grunts at them. They talk to Elena and she gossips about the bartender. That's when they realise: it's not one special NPC. **The whole town is alive.** That's the moment they're hooked.

### Millholm Progression Arc — Full Picture

| Level | Phase | What's Happening | Player State |
|---|---|---|---|
| **1** | Guided | Bartender → tutorial → starter quests → noticeboard → **level 2** | Learning mechanics, being welcomed, first wow moments |
| **2–3** | Directed | Guild initiation quest → start training class skills → explore all districts | Has a class, knows the map, beginning to understand the economy |
| **3–5** | Sandbox | Grind mobs, gather resources, craft gear, train skills, accumulate gold | Self-directed, building toward leaving Millholm |

#### Levels 3–5: The Sandbox

The hand-holding is over. The player knows the map, knows the NPCs, knows the economic loops. Millholm becomes a sandbox of interlocking systems:

**Combat & Loot**
- Mobs across all districts with escalating difficulty — farm pests (easy), woods creatures (medium), sewer denizens (harder)
- Loot drops include **crafting recipes** and **spell scrolls** as rare finds — gives combat an economic purpose beyond XP
- Some mobs may have associated quests, others are just part of the world

**Resource Mastery**
- Player knows WHERE resources are, now getting efficient at gathering and processing
- Wheat, cotton, wood, copper ore, tin ore — all harvestable in Millholm
- Processing chains: wheat→flour→bread, wood→timber, ore→ingots, hide→leather, cotton→cloth
- Bronze is the gear ceiling — copper + tin → bronze ingots → bronze weapons/armor at the smithy

**Skill Training**
- Guild trainers offer BASIC and SKILLED training in class skills
- General skills (battleskills, alertness, cartography) trainable by all
- Crafting skills trainable at the relevant workshops
- Time investment: pushing from BASIC toward SKILLED across multiple skills

**Gear Progression**
- Bronze weapons and armor (smithy)
- Leather armor (tannery/leatherworker)
- Cloth gear (tailor)
- Basic potions (apothecary, if alchemy ingredients found)
- Goal: be well-equipped in bronze-tier gear by level 5

**Economy**
- Selling bread, selling resources, selling crafted goods on AMMs
- Building a bankroll for the journey out of Millholm
- Understanding supply and demand through direct experience

#### Ready to Leave (Level 5)

By level 5 the player should have:
- Decent bronze-tier gear
- A class with skills trained toward SKILLED
- Food supply figured out (can bake their own bread or buy it)
- Gold in the bank
- BASIC cartography — enough to discover the route to the next zone
- Rumours and threads pulling them outward: the dwarven mines of Ironback Peaks, the rolling hills of Cloverfen, whatever the bartender and other NPCs have hinted at

The transition out of Millholm is gated by **cartography** (must discover the route) but motivated by **curiosity and ambition** — better resources, higher-tier trainers, new stories, and the knowledge that bronze gear won't cut it forever.

### Millholm Design Principles (New Player Lens)

1. **Arrival is mid-scene** — Millholm is already happening. Market noise, passing NPCs, the smell of bread. The world doesn't care that you just showed up.
2. **Sensory specificity** — Not "a busy market" but "a fishwife shouts prices over the clatter of cart wheels on cobblestone, the air thick with salt and woodsmoke." Text crushes graphics when descriptions trigger the player's own imagination.
3. **Discovery, not direction** — No quest markers, no exclamation points. Players find things by looking, listening, and talking. The world teaches by responding, not lecturing.
4. **Layered depth** — Surface level is obvious (market sells things, docks have ships). Underneath: scratched symbols, locked doors, NPCs who hint at things, sounds from below.
5. **Curiosity spiral** — Every room has something to look at closer. Every closer look reveals a detail pointing somewhere else. Every dead end leaves a thread.
6. **Early combat is environmental** — Rats in the cellar aren't a quest. The cellar has rats because it's a cellar. You went down there because you were curious.
7. **The economy is visible from minute one** — Players see others trading, see the auction board, see crafters at work. They don't understand it yet, but they see it's real.
8. **Danger is present but readable** — A guard fighting something at the edge of town. A dead adventurer in the sewers with better gear. Aspiration, not frustration.
9. **Unresolved threads create addiction** — The player logs off with things nagging them: something they couldn't access yet, something they started but didn't finish, something they don't understand. Incomplete tasks occupy the mind (Zeigarnik effect).

---

## Factions & Powers

*(To be developed — founding families of Millholm, Shadowsward military, Thieves' Guild, Dwarven clans, desert warriors, mage colony, pirate factions, etc.)*

---

## History & Lore

### The Ancient Builders

Something built the deep sewers beneath Millholm — carved stone older than the town by centuries, with deliberate symbols too weathered to read. This same civilisation may have left ruins elsewhere on the continent (jungle ruins in the southern zones?). Who were they? What happened to them? This is the continent-spanning mystery arc.

*(Further development needed — origin of the ancient builders, their relationship to the Hall of Worlds, what destroyed them)*

---

## NPCs & Key Characters

### Existing (Millholm)
- **Sergeant Grimjaw** — Warrior guild
- **Warlord Thane** — Warrior guild (higher)
- **Gareth Stonefield** — Thief guildmaster (LLMGuildmasterNPC, vector memory)
- **Whisper** — Thief trainer
- **Shadow Mistress Vex** — Thieves' Lair 2IC, cultured and dangerous
- **Archmage Tindel** — Mage guild
- **High Magus Elara** — Mage guild (higher)
- **Brother Aldric** — Temple / Cleric guild
- **High Priestess Maren** — Temple / Cleric guild (higher)

### Needed
- Founding family NPCs (Stonefield, Brightwater, Goldwheat, Ironhand)
- Millholm town NPCs (baker, blacksmith, shopkeepers, guards, farmers)
- Dwarven NPCs for Ironback Peaks
- Shadowsward military commanders
- Saltspray Bay harbour master, shipwright trainer, naval officers
- Kashoryu martial arts masters, ninja guildmaster
- Island-specific NPCs

---

## Quests & Storylines

### Starter Quests
- Bartender starter quest chain (5 quests, levels 1→2) — see [New Player Onboarding](#new-player-onboarding)
- Town noticeboard quests (self-directed, post-bartender chain) — see [New Player Onboarding](#new-player-onboarding)

### Guild Initiation Quests

Joining any guild (multiclassing) requires completing an initiation quest. Only 4 beginner classes are available in Millholm: **Warrior, Thief, Mage, Cleric**. Each guild has a trainer who can train BASIC and SKILLED levels in all skills related to that class. Guild initiation quests are LLM-driven — the guildmaster NPCs have personalities and persistent memory.

**General rule:** The world acknowledges what the player has already done. If a quest objective was already completed (e.g. rats already cleared), the LLM guildmaster recognises this and can skip or shorten the quest.

| Guild | Guildmaster | Quest | Design Purpose |
|---|---|---|---|
| **Warriors** (The Iron Company) | Sergeant Grimjaw | **Rat Clearing** — clear the rats from the Harvest Moon cellar. If already done via bartender quest, Grimjaw says: "The bartender told me you took care of the rats for him. You're exactly the kind of person we need." Quest skipped. | Direct problem, direct solution. Combat competence. Also: the world remembers what you've done. |
| **Thieves** (Sewers, hidden) | Gareth Stonefield | **The Shadow Trial** — navigate the Thieves' Gauntlet, a static 3-room challenge off the Training Alcove in the Thieves' Lair. Find the hidden entrance, survive traps (dart trap, tripwire), discover a hidden lever and key, unlock the vault chest, and retrieve the shadow guild token. Return the token to Gareth. | Resourcefulness, observation, danger. Teaches search, trap detection, and lockpicking. Static rooms with full reset between aspirants. |
| **Mages** (Circle of the First Light) | Archmage Tindel | **Arcane Dust** — collect 3 arcane dust. Found in an elusive faerie grove hidden in the woods, along the path toward the abandoned mine. | Seek rare materials in hidden places. Curiosity and exploration lead to power. Introduces the faerie grove as a mysterious location with depth. |
| **Clerics** (Shrine of the First Harvest) | Brother Aldric | **Feed the Hungry** — bake bread and give it to a beggar living on the street outside the temple. | Compassion as qualification. You prove yourself through service, not combat. Quietly reinforces that bread has real survival value in this world. |

**Each quest teaches the class philosophy:**
- Warriors: strength solves problems. Step up and handle it.
- Thieves: navigate danger alone, find what others can't reach.
- Mages: knowledge and curiosity are the path to power.
- Clerics: serve the vulnerable. Compassion is strength.

**New locations implied:**
- **Faerie Grove** — hidden area in Millholm Woods near the path to the abandoned mine. Elusive, magical, arcane dust spawns here. Discovery moment for the player — the woods are more than just a logging area.
- **Thieves' Gauntlet** — static 3-room challenge (Narrow Corridor → Damp Chamber → The Vault) accessed via hidden entrance in Training Alcove. Fully built and resettable.
- **Beggar's Spot** — street outside the Shrine of the First Harvest. A permanent NPC (LLM-driven?) who lives rough and is grateful for food.

### Planned Arcs
- **The Ancient Builders** — continent-spanning mystery. Breadcrumbs start in Millholm sewers, lead through ruins across multiple zones. Who built the ancient stonework? What are the symbols? What's behind the bricked-up passage?
- **The Abandoned Farm** — early quest hook. Why was the Goldwheat homestead abandoned? Disease? Raids? Something from below?
- **Thieves' Guild** — can players join? Infiltrate? Oppose? Shadow Mistress Vex has plans.
- **The Shadowsward War** — ongoing conflict against Shadowroot. Players can join the defence, run supply missions, push into enemy territory.
- **Maritime Discovery** — the quest to map the ocean. First to chart a route to a new island earns fame and fortune.

---

## Economy & Trade Narrative

The economy is **on-chain** — gold and resources are issued currencies on the XRPL, items are NFTs. Everything is transparent and auditable. This means:

- **No free loot** — everything that enters the game is drawn from a finite, managed supply
- **Supply and demand are real** — AMM pools drive prices based on actual scarcity
- **Cartography has economic value** — maps are NFTs, route knowledge is tradeable
- **Zone-exclusive resources create trade routes** — Shimmerleaf only grows on the Arcanum island, so whoever controls that supply route has real economic power
- **Seasonal effects on economy** — winter closes Ironback Peaks passes, reducing ore supply, driving up prices (though real economic forces dominate)
- **Crafting tiers are geographically gated** — beginner ores in Millholm, mid-tier in Ironback Peaks, high-tier on islands/Vaathari, GM forge in Olympus

The economy isn't a game mechanic bolted onto a world — it *is* the world. Trade routes exist because resources are scarce. Cities exist because crafting facilities are there. Ships exist because islands have things the mainland doesn't.

---

## Atmosphere & Tone

### Main World
**Serious fantasy with warmth.** The world is dangerous but not grimdark. Millholm is cosy. The Shadowsward is grim. The ocean is vast and mysterious. Each zone has its own emotional register, but the baseline is: *a world worth exploring, with real stakes.*

### The Hall of Worlds
**Anything goes.** Once through a door, tone is whatever the world behind it demands. A sprint through a toxic wasteland. A chat with Zaphod Beeblebrox. A horror survival on a dying world. The Hall itself is **alien and unsettling** — not evil, just fundamentally *wrong*. The void is not darkness, it's nothing.

### Writing Style
- Multi-sensory room descriptions (sight, sound, smell, touch)
- Evocative but not purple — earn every adjective
- Each zone should have a distinct vocabulary and rhythm
- Millholm: warm, domestic, agricultural
- Ironback Peaks: stone, metal, deep, old
- Cloverfen: cosy, abundant, gentle, well-fed
- Shadowsward: sharp, military, terse
- Desert: vast, dry, spare, proud
- Saltspray Bay: salt, timber, commerce, bustle
- The Bayou: humid, murky, alive, unsettling
- Aethenveil: ancient, luminous, quiet, aware
- Kashoryu: silk, spice, lantern-light, discipline
- Zharavan: opulent, controlled, perfumed, watchful
- Celestial Threshold: radiant, serene, divine, humbling

---

## Brainstorm & Ideas

*(Running scratchpad for ideas, fragments, and creative sparks)*

### Island Ideas (Unsorted)
- Isle of Dread (classic D&D homage)
- The Arcanum (mage research colony)
- Nassau (pirate haven)
- Cannibal Isle (survival horror)
- Moonhaven (werewolf secret — "stay indoors at night" rule)
- Isle of Druids (mid-level island, druidic tradition, nature magic training, wild beasts)
- Atlantis (deep underwater, high-level, requires water breathing)
- Volcanic forge island (rare smithing materials)
- Ghost ship graveyard (undead maritime zone)
- Turtle island (the island is alive and moves)

### Vaathari Ideas
- Hall of Worlds primary entrance
- Olympus / divine realm with Forge of the Gods (GM-only crafting)
- Completely different cultural flavour from starting continent
- Top-tier everything: resources, mobs, quests, recipes
- Details TBD — this is endgame, build toward it

### Hall of Worlds Ideas
- Red Dwarf (meet the crew, Lister's curry, Cat's wardrobe)
- Doctor Who (TARDIS interior? Daleks?)
- Hitchhiker's Guide (Zaphod, Arthur, Ford, the whale)
- Prehistoric world (dinosaurs, primitive survival)
- Dying sun world (everything is ending, melancholy tone)
- Cyberpunk city (futuristic contrast to main fantasy world)
- Mirror world (everything is reversed/opposite)
- Tiny world (5-room sprint, everything is miniaturised)
- Gravity-reversed world (ceiling is floor)

### Naming Ideas
- ~~Eastport~~ → **Saltspray Bay** (renamed). Also considered: Tidemark, Breakwater, Helmrest, Saltmere, Crosshaven
- ~~Ironreach Mountains~~ → **Ironback Peaks** (renamed)
- ~~Southern Port~~ → **Kashoryu** (renamed)
- Cloverfen: needs an original name (too Tolkien as-is) — TBD
- The Bayou: working name, may want something more evocative — TBD
- ~~Elven Lands~~ → **Aethenveil** (renamed)
- Zharavan
- Celestial Threshold: working name — TBD
- Vaathari: "the counterweight continent" (Pratchett nod) — or something original
- ~~The Blight: may want an original name to distance from WoT~~ → **Shadowroot** (renamed)
- ~~The Borderlands: may want an original name~~ → **The Shadowsward** (renamed)
- ~~The Threefold Lands~~ → **The Scalded Waste** (renamed)

### Open Questions
- How many islands in the chain? (Enough for meaningful progression tiers, not so many it feels padded)
- What is Solendra? (Secret island, GM seamanship, Kashoryu only — details TBD)
- What civilisation built the ancient sewers under Millholm? How does their story arc resolve?
- Does the Vaathari have multiple zones or is it one large endgame zone?
- Should there be PvP zones? (Nassau as a lawless PvP island?)
- What specific alchemy garden ingredients does Cloverfen produce? (Unique food-based reagents for culinary alchemy tradition)
- What's in the halfling barrow mounds? (Low-level dungeon content)
- What specific poison ingredients come from the Bayou?
- How much do the Elves know about the Ancient Builders? Are they descendants? Rivals? Inheritors?
- What is the Aasimar relationship to Olympus — servants, guardians, children of the gods?
