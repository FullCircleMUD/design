# Survival System — Hunger & Thirst

> Per-character upkeep meters that depleting over time and can kill via the regen system. Both meters tick on the same global cadence and live in the same Evennia script (`SurvivalService`). Designed so future meters (sleep, sanity, fatigue) can plug into the same loop body without architectural changes.

---

## Overview

Survival meters are a deliberate friction tax on play — they exist to (a) give the resource economy a constant baseline demand, (b) anchor inn rooms and town fountains as load-bearing infrastructure, and (c) reward players for thinking ahead before leaving town. The design rules them by these constraints:

- **Naggy is the failure mode.** A meter that pesters the player every few minutes feels worse than no meter at all. Wider numerical ranges are preferred over faster tick cadences.
- **Death is real but slow.** Both meters can kill, but only after a long depletion + bleed window. Players can recover with food, water, healing, or fleeing back to town.
- **One global tick.** All meters share `SURVIVAL_TICK_INTERVAL` (currently 1200s = 20 minutes). Different rates are achieved by giving each meter a different number of stages.
- **Puppeted-only.** Meters tick only on characters that have an active session. Logged-out characters and superusers are exempt.

---

## SurvivalService

`typeclasses/scripts/survival_service.py` — a single global persistent Evennia script.

- Created at boot by `at_server_startstop._ensure_global_scripts`.
- Ticks every `settings.SURVIVAL_TICK_INTERVAL` seconds (20 min default).
- On each tick, walks all puppeted characters via `SESSION_HANDLER.get_sessions()`.
- For each character: skips superusers, then dispatches to one helper per meter — currently `_tick_hunger(char)` and `_tick_thirst(char)`. New meters drop in as additional helpers in the same loop.
- Each helper is independent: it reads the meter's current state, applies its own decrement rules, and writes back. No cross-meter coupling at the tick layer.
- The script was renamed from `HungerService` once the second meter (thirst) shipped. A one-shot cleanup in `at_server_start` (`_cleanup_renamed_scripts` + `_RENAMED_SCRIPT_KEYS`) deletes the stale `"hunger_service"` script row on the first boot post-rename so we don't end up with ghost services.

---

## Hunger

**Enum:** `HungerLevel` (`enums/hunger_level.py`) — 8 stages. Two intermediate no-penalty stages (NOURISHED, CONTENT) sit between SATISFIED and PECKISH to widen the pre-damage runway without accelerating the bleed phase.

| Stage | Value | Effect |
|---|---|---|
| FULL | 8 | No effect; just-eaten state |
| SATISFIED | 7 | Regen normal |
| NOURISHED | 6 | Regen normal |
| CONTENT | 5 | Regen normal |
| PECKISH | 4 | Regen normal (last no-penalty stage) |
| HUNGRY | 3 | **Regen halts** (no penalty otherwise) |
| FAMISHED | 2 | **HP/MP/MV bleed** at ~30 min to death |
| STARVING | 1 | **Faster bleed** at ~15 min to death |

Death cause string: `"starvation"` (sent through `character.die()`).

**Decrement model:** one stage per `SurvivalService` tick. The constant decrement is applied via `HungerLevel.get_level(value - 1)`. STARVING is the floor — death is delivered through `RegenerationService.degenerate`, not by the survival tick itself.

**Free-pass-on-FULL:** when a character eats to FULL, `hunger_free_pass_tick` is set True. The next survival tick consumes the flag without decrementing — this prevents the awkward edge case where a player eats and immediately drops back to SATISFIED on the very next tick if the timing is unlucky.

**Sources of food (player-facing):**
- **Bread (resource ID 3)** — eaten directly via `eat` from inventory. The primary economic upkeep item.
- **Stew at an inn** — `stew` command in a `RoomInn`. Buys 1 bread via the AMM pool at the current market price, immediately consumes it, raises hunger by ONE level. Stew is a thin wrapper around the bread economy — it pulls liquidity from the AMM pool.
- **`forage`** (SURVIVALIST general skill, Druid/Ranger only) — restores hunger directly with no bread spend. Mastery scales yield (1–5 stages). 15-min cooldown matching the survival cycle. Deliberately does NOT set the free-pass tick, so bread retains its economic advantage. Requires forageable terrain (not urban/underground/dungeon/water).

---

## Thirst

**Enum:** `ThirstLevel` (`enums/thirst_level.py`) — 12 stages, twice the range of hunger so the meter feels half as naggy at the same tick cadence.

| Stage | Value | Effect |
|---|---|---|
| REFRESHED | 12 | No effect; just-drunk state |
| HYDRATED | 11 | No effect |
| QUENCHED | 10 | No effect |
| SLAKED | 9 | No effect |
| COMFORTABLE | 8 | No effect |
| AWARE | 7 | No effect; first nag message |
| DRY | 6 | **Regen halts** (parallel to HUNGRY) |
| THIRSTY | 5 | Regen halted |
| VERY_THIRSTY | 4 | Regen halted |
| PARCHED | 3 | **HP/MP/MV bleed** at ~35 min to death |
| DEHYDRATED | 2 | Bleed at ~25 min to death |
| CRITICAL | 1 | **Faster bleed** at ~15 min to death |

Death cause string: `"dehydration"`.

**Decrement model:** identical to hunger — one stage per `SurvivalService` tick, CRITICAL is the floor. Same `thirst_free_pass_tick` flag for the just-drunk edge case.

**Why 12 stages instead of 6:** keeps the tick cadence shared with hunger, but means a full container or a refresh from the inn buys the player twice as much real-time before they need to think about water again. Fewer interruptions.

**Sources of water (player-facing):**
- **Canteen** — small leather water container. Crafted by leatherworkers at BASIC mastery. Capacity: 5 drinks. Carried in inventory (not the HOLD slot — the player can drink while wielding a weapon and holding a torch). Refillable.
- **Cask** — wooden water container, twice a canteen's capacity. Crafted by carpenters at SKILLED mastery. Capacity: 10 drinks. Heavier than a canteen. Carried in inventory.
- **Fountain** in town — `FountainFixture` (`typeclasses/world_objects/fountain_fixture.py`). A `WorldFixture` with `is_water_source = True`. The `refill` command finds these in the room and tops up any held water container. First instance: Millholm Townsquare (sq_ne).
- **`Create Water` spell** — Conjuration school, BASIC mastery, 1 mana. Targets a held water container and refills it to capacity. Free water for any party with a mage. Cooldown: 0 (utility, spammable).
- **Inn ale** — `ale` command in a `RoomInn`. Costs 1 gold (sink). Raises thirst by ONE stage per mug, parallel to how stew raises hunger by one level per bowl. Drink multiple ales to fully refresh. Sets the free-pass tick only when the drink lands the player at REFRESHED.

**A drink restores one stage.** So a full canteen (5 drinks) carries the player from CRITICAL all the way to AWARE; a full cask (10 drinks) carries them from CRITICAL through to QUENCHED. Either is plenty for most ventures out of town.

---

## Container Mechanics — `WaterContainerMixin`

`typeclasses/mixins/water_container.py` — the shared mixin all water containers compose with `BaseNFTItem`. Same shape as `LightSourceMixin`:

- `max_capacity` (drinks)
- `current` (drinks remaining)
- `is_water_container` class flag — used by `drink`/`refill` to find containers
- `drink_from(character)` — decrements current, steps the character's `thirst_level` up by one (capped at REFRESHED), sets the free-pass tick on REFRESHED, persists state
- `refill_to_full()` — sets current = max_capacity, persists state
- `is_empty` / `is_full` properties
- `_persist_water_state()` — pushes `{current, max_capacity}` to `NFTGameState.metadata` via the standard `persist_metadata()` helper, so a half-full canteen survives bank/withdraw and chain export/import. Silently no-ops on non-NFT consumers. See `design/INVENTORY_EQUIPMENT.md` § NFT Metadata Persistence for the round-trip story.

Concrete typeclasses live in `typeclasses/items/water_containers/` (a new folder, distinct from `containers/` which is for backpacks-holding-items, and `consumables/` which is for delete-on-use items):
- `canteen_nft_item.py` — `CanteenNFTItem`, capacity 5, weight 0.5 kg
- `cask_nft_item.py` — `CaskNFTItem`, capacity 10, weight 2 kg, requires SKILLED carpenter

Both have `tracking_token` proxy tokens (`PCanteen`, `PCask`) registered in the NFTItemType migration alongside other proxy-tradeable items.

---

## Penalty Pipeline — `RegenerationService`

`typeclasses/scripts/regeneration_service.py` — the global script that runs every 20s and combines hunger + thirst into a single regen/degen/bleed decision.

- **Both meters must permit regen** for healing to fire. If hunger says PECKISH-or-better AND thirst says AWARE-or-better, regen runs normally (HP/MP/MV recover at base rate × position multiplier).
- **Either meter triggering degen** is enough to bleed. If hunger is FAMISHED-or-worse OR thirst is PARCHED-or-worse, the degen path runs once per minute (every 3rd 20s tick).
- **Worst-meter wins** for the bleed rate and death cause. Each meter contributes a `(cycles_to_death, death_cause)` candidate; the smallest cycles_to_death (i.e., fastest death) is used. Hunger wins ties, so "starved while also parched at the same lethal rate" is recorded as starvation.
- **Per-meter cycles_to_death:**

| Meter stage | Cycles to death (minutes) |
|---|---|
| HungerLevel.FAMISHED | 30 |
| HungerLevel.STARVING | 15 |
| ThirstLevel.PARCHED | 35 |
| ThirstLevel.DEHYDRATED | 25 |
| ThirstLevel.CRITICAL | 15 |

Bleed amounts: `max(1, round(stat_max / cycles_to_death))` per minute on each of HP, MP, MV. When HP hits zero, the character dies with the chosen cause string.

**Messaging:** `send_hunger_messages` and `send_thirst_messages` push first-person nags to the affected character (once they cross the no-penalty threshold) and third-person observation messages to the room (once they enter the bleed zone). The first/third person message strings live in the enum modules.

---

## Design Constraints (intentional gotchas)

- **Containers are inventory items, not held.** A canteen does NOT compete with weapons / torches for the HOLD slot. The model is "drink while you fight, like you eat bread while you fight" — friction without UX punishment.
- **Containers are NFTs.** They participate in the full NFT lifecycle: bank/withdraw, give, export to XRPL, re-import. The mirror metadata persistence layer carries `current` through every round-trip, so a half-full canteen sold on a marketplace shows the buyer exactly how much water they're getting.
- **No survivalist free-pass for thirst.** Forage gives a hunger free pass for druids/rangers but not for thirst. The reasoning: thirst already has free supply via fountains and the Create Water spell, so a third bypass would erode the supply chain entirely. If a "find water" survival skill ever lands, it should produce drinks-into-a-container rather than direct meter restoration, mirroring how forage produces hunger restoration but not bread.
- **Stew is hunger only; ale is thirst only.** Inns serve both — the player chooses which meter to top up. Don't bundle them. Stew prices off the bread AMM; ale is a flat 1-gold sink (subject to change once a hops/ale resource exists).
- **Death by either meter routes through `character.die()`.** No special-case bypass. A starved corpse and a dehydrated corpse drop their loot the same way.

---

## Files

| File | Purpose |
|---|---|
| `enums/hunger_level.py` | HungerLevel enum + first/third person message dicts |
| `enums/thirst_level.py` | ThirstLevel enum + message dicts |
| `typeclasses/scripts/survival_service.py` | The global tick dispatcher (renamed from HungerService) |
| `typeclasses/scripts/regeneration_service.py` | Regen/degen/bleed pipeline; consumes both meters |
| `typeclasses/mixins/water_container.py` | `WaterContainerMixin` shared by canteen / cask / future containers |
| `typeclasses/items/water_containers/canteen_nft_item.py` | Canteen typeclass |
| `typeclasses/items/water_containers/cask_nft_item.py` | Cask typeclass |
| `typeclasses/world_objects/fountain_fixture.py` | `FountainFixture` (a WorldFixture with `is_water_source`) |
| `commands/all_char_cmds/cmd_drink.py` | `drink [container]` command |
| `commands/all_char_cmds/cmd_refill.py` | `refill [container]` command |
| `commands/room_specific_cmds/inn/cmd_ale.py` | Ale (steps thirst up by one) |
| `commands/room_specific_cmds/inn/cmd_stew.py` | Stew (steps hunger up by one) |
| `world/recipes/leatherworking/canteen.py` | BASIC leatherworker recipe |
| `world/recipes/carpentry/cask.py` | SKILLED carpenter recipe |
| `world/prototypes/water_containers/` | Canteen + cask prototypes |
| `world/spells/conjuration/create_water.py` | Create Water spell |
| `blockchain/xrpl/migrations/0001_initial.py` | NFTItemType seed rows + PCanteen / PCask proxy tokens |

---

## Future Work

- **Sleep / fatigue meter** — third survival meter. Same `SurvivalService` loop, same shape, different penalty curve (probably no death, just regen halts). Likely tied to inn rooms (rest at an inn to clear it).
- **Climate-driven thirst rate** — desert zones tick thirst faster than temperate. Implementation: per-tick rate multiplier read from the room's terrain/climate before applying the decrement. Out of scope until desert content exists.
- **Natural water sources** — rivers, springs, the spring-fed pool tile in southern woods, and wells should all set `is_water_source = True` so the `refill` command picks them up. Trivial to add in zone builders; deferred until the second town is built.
- **Find Water survival skill** — analogous to forage but for thirst. Produces a drink credit, NOT direct meter restoration, to preserve the canteen/cask supply chain.
- **Ale economy** — once a hops/ale resource exists, the ale command should price off an AMM pool (mirroring stew's bread pool) instead of the flat 1-gold sink.
- **Dehydration-specific death narrative** — the death system currently uses a generic cause string. A future pass could give dehydration its own dying-thoughts narrative the way starvation has flavour.
