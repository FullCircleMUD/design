# Interzone Travel System Design

> This is the canonical design document for all zone-to-zone travel — overland and sea. For the complete route table (which zones connect to which, with discovery gates and food costs) see `design/WORLD.md`. For intra-zone district mapping (the `survey` command and map NFTs) see `design/CARTOGRAPHY.md`.

---

## Overview

Zone-to-zone travel uses **gateway rooms** (`RoomGateway`) placed at zone boundaries — trailheads, docks, mountain passes, desert crossings. Every route between zones passes through a pair of gateway rooms.

Two travel modes:

| Mode | Command | Requires | Notes |
|---|---|---|---|
| Overland — explore | `explore` | Cartographer of sufficient mastery + food | Discovers the route, produces a route map NFT |
| Overland — travel | `travel` | Route map in party inventory + food | No cartographer needed |
| Sea — explore | `explore` | Cartographer of sufficient mastery + qualifying ship + food | Discovers the route, produces a route map NFT |
| Sea — sail | `sail` | Route map in party inventory + qualifying ship + food | No cartographer needed |

`explore` is the only command that requires a cartographer. `travel` and `sail` require only the map.

---

## Discovery — The `explore` Command

Every route starts hidden. Players cannot travel a route until it has been discovered.

### How Discovery Works (current implementation)

The cartographer triggers `explore` at a gateway. The command:

1. Filters destinations to those that are still hidden, not already known to the party (via map or chart item), pass non-food/non-gold conditions, and meet the cartography mastery gate.
2. Picks one explorable destination at random as the target.
3. Deducts the destination's `gold_cost` upfront (if any).
4. Loops over the cartographer's bread, deducting one per iteration and rolling against `explore_chance` (default 20%). The first successful roll discovers the destination, spawns the route map NFT, and triggers a delayed travel sequence to the destination gateway.
5. If all bread is consumed without success, the cartographer "limps back with empty stores" — they remain at the departure gateway (no teleport home).

### Planned: Two-Phase Food Model

> **Status: Planned — not yet implemented.** The current code uses a single bread-per-roll loop with no journey minimum.

The intended future model splits food into two phases:

**Phase 1 — The journey (hard minimum).** The destination's `food_cost` is the minimum bread required to reach the area. Deducted upfront before any rolls happen.

**Phase 2 — On station (exploration rolls).** Bread beyond the journey minimum becomes exploration days. Each extra bread = one roll against `explore_chance`. Rolls continue until success or bread runs out.

**On failure (all extra bread consumed):** No discovery. The party returns to the gateway they left from, arriving in a high hunger state. The return journey costs no additional food — the game handles it narratively.

**Example (solo character, planned model):** A 3-bread route. The character has 5 bread.
- 3 bread deducted immediately (the journey).
- Roll 1 — fail — 4th bread deducted.
- Roll 2 — success — character arrives at destination.

### Planned: Food Pooling Across the Party

> **Status: Planned — not yet implemented.** Currently only the cartographer's own bread is read and deducted; party inventories are not pooled or debited.

Food for `explore` will be **totalled and shared across the whole party**, then deducted from each member proportionally as days pass.

- **Journey cost:** `food_cost × party_size` bread must be present in the party collectively.
- **On-station cost:** each additional exploration day will cost `1 bread × party_size`.

**Example (planned):** Party of 4, 3-bread route, wanting 2 days on station. Total bread required: (3 + 2) × 4 = **20 bread** across the party. Each player needs 5 bread. As days pass, 1 bread is deducted from each party member per day.

### Cartography Mastery Gate

The cartographer's mastery level determines which routes can even be attempted. Routes outside the cartographer's tier are invisible — they don't appear as explorable options at all.

| Cartography Tier | Zones Discoverable (Overland) | Notes |
|---|---|---|
| BASIC | Ironback Peaks, Cloverfen (from Millholm) | Random which is found first — first "I need a cartographer" moment |
| SKILLED | The Shadowsward, Saltspray Bay, The Bayou | Second continental ring |
| EXPERT | Shadowroot, Scalded Waste, Kashoryu (via Bayou overland) | Third ring. Dangerous territory |
| MASTER | Aethenveil, Zharavan, Kashoryu (via Aethenveil), Atlantis (dive, no ship) | Deep exploration. Remote and dangerous |
| GRANDMASTER | Vaathari | Ultimate discovery — from Guildmere Island only |

Sea routes also require Seamanship and a ship — see Sea Travel below.

**Failure message:** *"The path fades into unmarked wilderness. You'd need a cartographer with at least [tier] mastery to explore this way."*

### Party Rules

- Only **one** party member needs the required cartography mastery — the cartographer leads.
- Food is pooled across the party — the total required is `food_cost × party_size` for the journey, plus `party_size` per on-station day. Each member is debited 1 bread per day as it passes.
- If no eligible cartographer is present, the route cannot be discovered.

---

## Overland Travel — The `travel` Command

Any character or party holding a route map can `travel` that route — no cartographer required.

```
travel                  — list available destinations from this gateway
travel <destination>    — travel to the named destination
```

Travel is gated by:
- **Route map** — a party member must hold a map for this route
- **Food cost** — bread consumed per character (see route table in WORLD.md)
- **Level required** — minimum total_level (route-specific, rarely used)

Travel uses a **delayed narrative sequence**. `_delayed_travel()` (in `cmd_travel.py`) fires staggered messages on a 2-second tick via `evennia.delay()` before teleporting the party. The caller is locked out of other actions while in transit (`ndb.is_processing` flag, same pattern as crafting). Message lists are currently hardcoded constants per command type (`_TRAVEL_MESSAGES`, `_SEA_MESSAGES`, `_EXPLORE_MESSAGES`) — see Future Work for the planned per-destination configurable narrative.

---

## Sea Travel — The `sail` Command

Sea routes require a ship. The ship type is the **hard gate** — destinations specify a minimum ship tier and you simply cannot sail there without a qualifying vessel. Two other skills are involved but play different roles:

| Skill | Role | Hard Gate? | Notes |
|---|---|---|---|
| **Cartography** | Discover the route | Yes — `explore` only | Mastery tier determines which routes can be explored. Not required for `sail` on a known route. |
| **Ship ownership** | Reach the destination | Yes — travel gate | You must own (or be a passenger on) a ship of sufficient tier. Ship type is set by `boat_level` on the destination. |
| **Seamanship** | Sail without sinking | No — risk modifier | Lower mastery = chance of shipwreck per voyage. Anyone may attempt any voyage; the risk is theirs to take. |
| **Shipwright** | Build ships | No — crafting skill | Not a travel gate. Determines what ships a player can craft. The ship itself is what matters, not who built it. |

No single character needs cartography, seamanship, and a ship. A guild where one member is the cartographer, another owns the Galleon, and a third is the GM sailor can reach Vaathari — party composition matters.

```
sail                        — list sea routes from this dock
sail <destination>          — show qualifying ships (auto-sails if only one)
sail <destination> <#>      — sail with the chosen ship
```

### Ship Tiers

Five ship types map 1:1 to mastery tiers. Ships are NFT items implemented as `ShipNFTItem` (a `WorldAnchoredNFTItem` → `BaseNFTItem` subclass). They live in `character.contents` with zero weight and are hidden from the normal `inventory` listing — players view their ships via the `owned` command. They are tradeable and exportable via the same NFT pipeline as any other NFT item.

| Ship | Tier | Mastery Level | Routes Reached |
|---|---|---|---|
| Cog | 1 | BASIC | Closest coastal destinations (Teotlan Ruin, Amber Shore) |
| Caravel | 2 | SKILLED | Mid-range islands (Calenport, Port Shadowmere, coastal Saltspray Bay ↔ Kashoryu) |
| Brigantine | 3 | EXPERT | Deeper islands (The Arcane Sanctum, Oldbone Island) |
| Carrack | 4 | MASTER | Far islands (Guildmere Island) |
| Galleon | 5 | GRANDMASTER | Cross-ocean (Vaathari, Solendra) |

Ship ownership queries are currently handled via `BaseNFTItem.get_best_ship_tier(character)`, `get_qualifying_ships(character, min_tier)`, and `get_character_ships(character)`. **This API is subject to change** — see the Owned Objects System backlog item (`ops/PLANNING/0_BACKLOG`). The planned rework stores ships at specific dock locations rather than in character inventory; a qualifying ship check will need to confirm the player owns a ship of sufficient tier **berthed at the current dock**, not just owned anywhere.

### Sea Route Gates

Each sea route destination config carries:
- `required_cartography_tier` — minimum cartography mastery for **discovery** only (int, 1–5). Does not gate `sail` on already-known routes.
- `boat_level` — minimum ship tier required to travel (int, 1–5). Hard gate — voyage cannot begin without a qualifying ship.
- `food_cost` — bread per character (consumed on departure)

Seamanship is **not** in the destination config as a gate. It is consulted at sail time to calculate shipwreck probability — see Seamanship Risk below.

For full route details including specific costs see the Zone Connections table in `design/WORLD.md`.

### Shipwright Skill — Building Ships

Ships are crafted via the Shipwright skill. Mastery tier determines the largest ship buildable. Materials scale with tier — higher ships require cross-crafter collaboration.

| Mastery | Ship | Materials (approx.) |
|---|---|---|
| BASIC | Cog | 50 Timber, 10 Cloth |
| SKILLED | Caravel | 100 Timber, 25 Cloth, 20 Copper |
| EXPERT | Brigantine | 200 Timber, 50 Cloth, 40 Copper, 20 Iron Ingots |
| MASTER | Carrack | 300 Timber, 50 Hardwood, 75 Cloth/Silk, 60 Copper, 30 Iron Ingots |
| GRANDMASTER | Galleon | 400 Timber, 100 Hardwood, 100 Silk, 100 Copper, 40 Iron Ingots, 20 Steel Ingots |

GM ships require collaborative effort across multiple skilled crafters (Carpenter builds hull sections, Blacksmith forges anchors, Tailor sews sails).

**Economic note:** Ships are NFTs. A player who owns the only GM-level Galleon on the server controls access to Vaathari. They can charge for passage. Passenger limits apply — food costs per character still apply to all passengers.

### Banking & Cross-Character Transfer

Ships can be deposited and withdrawn at any bank room using the standard `deposit` / `withdraw` commands — the same flow as any other NFT item. This is the asynchronous counterpart to `give` (which requires both characters to be in the same room at the same time): a player can `deposit` a ship from one character, log out, switch to another character on the same account, walk into a bank, and `withdraw` it. Ownership effectively transfers between characters via the shared `AccountBank`.

The ship's berth location (`db.world_location`) is preserved across the bank cycle. The mirror-metadata persistence layer (see `design/INVENTORY_EQUIPMENT.md` § NFT Metadata Persistence) writes `world_location_dbref` and `world_location_name` into `NFTGameState.metadata` on every dock arrival, and `ShipNFTItem.at_restore_from_metadata` rehydrates it on respawn — so the location also survives an XRPL export → re-import round-trip.

A new owner who withdraws a ship at a bank does **not** auto-relocate to its dock. They have to travel to that dock themselves before they can `sail` it. This is intentional friction matching the design note in `ship_nft_item.py:9-13` ("New owner must travel to that dock to sail the ship") and applies equally to ships acquired via `give`, via XRPL marketplace purchase, or via the bank.

**Display rules across the three command surfaces:**

| Command | Where | Shows |
|---|---|---|
| `balance` | bank room | gold, resources, items, ships (under `\|wShips:\|n` with berth info) |
| `stabled` | stable room | pets only |
| `bank` (OOC) | account-level | everything — gold, resources, items, pets, ships |

The bank room and stable room are deliberately separate gateways. A bank room hides pets (they're managed via `stable` / `retrieve` in a `RoomStable` — see the design notes in `world_anchored_nft_item.py` and the pet-specific dispatch in `NFTPetMirrorMixin`) so the player isn't misled into thinking they can retrieve a pet from a bank. The OOC `bank` command exists so a player can see their entire account state in one place before deciding what to import or export.

In all three views, ships render via `ShipNFTItem.get_owned_display()`, e.g. `"  The Grey Widow (Caravel) — berthed at Saltspray Bay Docks"`, so the berth location is always visible inline.

---

## Route Map NFTs

On successful `explore`, a **route map NFT** auto-creates in the cartographer's inventory. This is the tangible product of exploration — the cartographer's economic output.

- **Type:** Unique NFT — not fungible, not AMM-tradeable
- **Trade:** Player-to-player only (direct trade, auction house)
- **Function:** Any party that contains a member holding the map may `travel`/`sail` that route without a cartographer and without having previously explored it themselves
- **No duplication on travel:** Using the map to travel never produces another copy. Maps are produced only by exploration.
- **If sold or given away:** The cartographer no longer holds a map for that route. To travel it again they must `explore` again to produce a new one.
- **Boss loot:** Route maps can drop from bosses (a pirate boss drops a chart to their hidden cove) — access as loot, no cartographer required

Cartographers are economically valuable because they are the only source of new maps. A Vaathari route map is worth a great deal to a party without a GM cartographer.

---

## Where to Train Maritime Skills

Training locations follow the geography — you must reach a zone before you can train further there.

| Mastery Tier | Cartography | Seamanship | Shipwright |
|---|---|---|---|
| BASIC | Millholm | — | — |
| SKILLED | Saltspray Bay | Saltspray Bay, Kashoryu | Saltspray Bay, Kashoryu |
| EXPERT | Kashoryu | Calenport | Calenport |
| MASTER | Guildmere Island (TBA) | Guildmere Island (TBA) | Guildmere Island (TBA) |
| GRANDMASTER | Spread across Guildmere Island, Atlantis, and one EXPERT island (TBA which skill trains where) | Spread across Guildmere Island, Atlantis, and one EXPERT island (TBA which skill trains where) | Spread across Guildmere Island, Atlantis, and one EXPERT island (TBA which skill trains where) |

> **Design constraint:** GM training for all three maritime skills must be fully available before reaching Vaathari, because GM Cartography + Galleon (GM Shipwright) are hard requirements, and GM Seamanship is highly advisable to avoid shipwreck. Seamanship is not a hard gate — any player can attempt any voyage — but sailing a Galleon without GM Seamanship carries significant shipwreck risk (see Seamanship Risk above). GM training is intentionally scattered across the deepest pre-endgame locations — players must piece it together before they can attempt the final voyage.

---

## Technical Architecture

### Current Implementation

**`RoomGateway`** (`typeclasses/terrain/rooms/room_gateway.py`) — the room typeclass. Stores a list of destination dicts via `AttributeProperty`.

**Destination config schema (current):**

```python
{
    "key": str,                          # unique route ID
    "label": str,                        # display name
    "destination": RoomGateway,          # target room object
    "travel_description": str,           # narrative on arrival
    "conditions": {
        "food_cost": int,                # bread consumed (caller only — see #2)
        "gold_cost": int,                # gold consumed
        "level_required": int,           # minimum total_level
        "boat_level": int,               # ShipType tier (1–5)
        # stubs: mounted, fly, water_breathing
    },
    "hidden": bool,                      # invisible until discovered
    "discover_item_tag": str,            # item tag that also reveals this dest
    "explore_chance": int,               # % chance per bread roll (default 20)
    "required_cartography_tier": int,    # min cartography mastery for explore (1–5)
}
```

**Condition checking** lives in `cmd_travel.py` as `CONDITION_CHECKS` — a flat list of validators imported by `cmd_explore.py` and `cmd_sail.py`. The `required_cartography_tier` field is gated directly inside `cmd_explore.py` via `_best_party_skill(caller, CARTOGRAPHY)`, not via a `CONDITION_CHECKS` validator. `required_seamanship_tier` is intentionally absent — seamanship is never a hard gate (see Seamanship Risk).

**Commands:**
- `commands/room_specific_cmds/gateway/cmd_travel.py` — overland/dock travel, validates conditions, consumes costs, teleports via `_delayed_travel`
- `commands/class_skill_cmdsets/class_skill_cmds/cmd_sail.py` — sea travel (subclass of `CmdSkillBase`, SEAMANSHIP-skill-gated to require at least BASIC mastery), two-pass ship selection, reuses `validate_conditions` / `consume_costs` / `_delayed_travel` from `cmd_travel.py`
- `commands/room_specific_cmds/gateway/cmd_explore.py` — discovery rolls, bread-per-roll mechanic, cartography-tier gate, spawns route map NFT on success

**Test routes:** Town Dock ↔ Beach Dock in `world/test_world/test_area_gateway.py` (`boat_level: 1, food_cost: 1`).

---

### Planned Refactor — BaseGate Class Hierarchy

Current problem: condition checking is in commands, not the room — adding a new gate type requires touching command files.

Proposed structure moves validation into the typeclass:

```
BaseGate(RoomGateway)
  ├── destination schema (adds required_cartography_tier, required_seamanship_tier)
  ├── check_conditions(caller, dest) → list[str]    — calls _get_condition_checkers()
  ├── consume_costs(caller, dest)
  ├── spawn_route_map(caller, dest_key)               — creates route map NFT on exploration success
  └── get_discoverable_destinations(caller)          — filters by cartography tier + caller has no map for this route

OverlandGate(BaseGate)
  └── _get_condition_checkers() → [level, cartography_tier, food, gold]

SeaborneGate(BaseGate)
  ├── _get_condition_checkers() → [cartography_tier (discovery), boat_level, food, gold]
  └── get_shipwreck_chance(caller, dest) → int   — seamanship risk calculation
```

Commands become thin: `cmd_explore` calls `gate.get_discoverable_destinations(caller)`, `cmd_travel` and `cmd_sail` call `gate.check_conditions()` and `gate.consume_costs()` — no knowledge of which subclass. `cmd_sail` additionally calls `gate.get_shipwreck_chance()` to warn the player before they confirm.

`get_discoverable_destinations(caller)` filters by:
1. `hidden=True`
2. Caller does not already hold a route map for this destination
3. `required_cartography_tier` ≤ caller's current cartography mastery

`required_cartography_tier` is already in the destination schema today (gated inside `cmd_explore.py`); the refactor formalises it as a `BaseGate` validator. `required_seamanship_tier` is NOT added — seamanship is never a hard gate.

**Files to touch:** `room_gateway.py` (split into base + two subclasses), `cmd_travel.py` (remove condition dict, thin out), `cmd_sail.py`, `cmd_explore.py`.

---

## Seamanship Risk

> **Status: Planned — not yet implemented.** `cmd_sail.py` does not currently read seamanship mastery for risk calculation, does not roll for shipwreck, and does not warn the player. Any character with at least BASIC seamanship and a qualifying ship sails safely today. The model below describes the intended end state.

Seamanship is **not** a hard gate. Any character may attempt any voyage, regardless of their seamanship level — the risk is theirs to accept.

**Safe sailing:** A sailor whose seamanship mastery is at or above the ship's tier sails with 0% failure chance. A BASIC sailor on a Cog sails safely. A GM sailor on any ship sails safely.

**Undersailed voyages:** When a sailor's mastery is below the ship's tier, each voyage (exploration or travel) carries a shipwreck chance. Formula: `(ship_tier − sailor_tier) × 15%`.

| Sailor | Cog (BASIC) | Caravel (SKILLED) | Brigantine (EXPERT) | Carrack (MASTER) | Galleon (GM) |
|---|---|---|---|---|---|
| GM | 0% | 0% | 0% | 0% | 0% |
| MASTER | 0% | 0% | 0% | 0% | 15% |
| EXPERT | 0% | 0% | 0% | 15% | 30% |
| SKILLED | 0% | 0% | 15% | 30% | 45% |
| BASIC | 0% | 15% | 30% | 45% | 60% |
| UNSKILLED | 15% | 30% | 45% | 60% | 75% |

**Shipwreck outcome:** Ship is lost, all passengers die, all carried inventory is lost. This is a meaningful consequence — the risk must be real to make seamanship valuable.

**Player-facing:** Before confirming a voyage where there is shipwreck risk, `sail` shows the failure percentage: *"Warning: your seamanship (BASIC) gives you a 60% chance of losing this ship and your life. Sail anyway? (yes/no)"*

**Why this model works:**
- The ship type determines reachability — you cannot reach Vaathari without a Galleon, period.
- The sailor determines safety — once you have the Galleon, a party without a GM sailor can still attempt the voyage, but the stakes are high.
- Creates real demand for skilled sailors as crew even for players who own high-tier ships.
- A GM sailor is a valuable party member regardless of their combat ability.

**Future modifier:** Weather/storm events could temporarily increase the failure chance for active voyages.

---

## Future Work

### Per-Destination Travel Narrative

Travel delay itself is implemented (`_delayed_travel` with 2-second ticks and `ndb.is_processing` lockout). What's still planned: making the narrative configurable per route instead of using the three hardcoded message lists (`_TRAVEL_MESSAGES`, `_SEA_MESSAGES`, `_EXPLORE_MESSAGES`).

- Add `travel_time` (seconds) and `travel_messages` (list of str) to destination config
- Per-destination message lists drive the existing `_delayed_travel` loop
- Longer journeys (more bread) = more narrative beats tuned to the specific route

Example for a 3-bread journey:
```
"You set off along the narrow bush track..."
[3s] "You grow weary from days of travel."
[6s] "The scrub thins and the ground turns sandy beneath your feet."
[9s] "After 3 days of travel you arrive at a beach with a small cabin."
```
