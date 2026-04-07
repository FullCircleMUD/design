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

### How Discovery Works

Food is used in two distinct phases:

**Phase 1 — The journey (hard minimum).** The destination's `food_cost` is the minimum bread required to even reach the area. A party attempting to explore a 3-bread route must have at least 3 bread per character before `explore` can begin. If they don't, the command is blocked. This bread is deducted upfront — it is the cost of getting there.

**Phase 2 — On station (exploration rolls).** Any bread beyond the journey minimum becomes exploration days. Each extra bread = one day on station = one roll against `explore_chance` (default 20%). Rolls continue until success or bread runs out.

**On success:** The party arrives at the destination gateway. A **route map NFT** (departure ↔ destination) auto-creates in the cartographer's inventory. The map is the knowledge — keep it to travel the route again, trade it, or give it away. New maps are only produced by exploration — traveling a known route never produces a copy.

**On failure (all extra bread consumed):** No discovery. The party returns to the gateway they left from, arriving with a high hunger state. The return journey costs no additional food — the game handles it narratively.

**Example (solo character):** A 3-bread route. The character has 5 bread.
- 3 bread deducted immediately (the journey).
- Roll 1 — fail — 4th bread deducted (1 bread per roll for a solo character).
- Roll 2 — success — character arrives at destination.
- (If roll 2 had failed: 5th bread deducted, roll 3. Fail = return home hungry, all 5 bread spent.)

> **Note:** This example shows a solo character (party size 1). Each exploration roll costs 1 bread per character — see Party Rules below for how this scales with party size.

### Food Pooling Across the Party

Food for `explore` is **totalled and shared across the whole party**, then deducted from each member proportionally as days pass.

- **Journey cost:** `food_cost × party_size` bread must be present in the party collectively.
- **On-station cost:** each additional exploration day costs `1 bread × party_size`.

**Example:** Party of 4, 3-bread route, wanting 2 days on station. Total bread required: (3 + 2) × 4 = **20 bread** across the party. Each player needs 5 bread. As days pass, 1 bread is deducted from each party member per day (i.e., each exploration roll costs 1 bread per character — the same mechanic as the solo example above, scaled by party size).

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

Travel is currently **instant** — see Future Work for the planned travel delay mechanic.

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

Five ship types map 1:1 to mastery tiers. Ships are **NFTs** — owned by players, tradeable, not spawned as in-game objects (`prototype_key=None`).

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
    "key": str,                  # unique route ID
    "label": str,                # display name
    "destination": RoomGateway,  # target room object
    "travel_description": str,   # narrative on arrival
    "conditions": {
        "food_cost": int,        # bread consumed per character
        "gold_cost": int,        # gold consumed
        "level_required": int,   # minimum total_level
        "boat_level": int,       # ShipType tier (1–5)
        # stubs: mounted, fly, water_breathing
    },
    "hidden": bool,              # invisible until discovered
    "discover_item_tag": str,    # item tag that also reveals this dest
    "explore_chance": int,       # % chance per bread roll (default 20)
}
```

**Condition checking** lives in `cmd_travel.py` as `CONDITION_CHECKS` — a flat dict of validators imported by `cmd_explore.py`. Currently missing: `required_cartography_tier`, `required_seamanship_tier`.

**Commands:**
- `cmd_travel.py` — overland/dock travel, validates conditions, consumes costs, teleports
- `cmd_sail.py` — sea travel, two-pass ship selection, SEAMANSHIP-gated
- `cmd_explore.py` — discovery rolls, bread-per-roll mechanic, spawns route map NFT on success

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

This introduces `required_cartography_tier` to the destination schema. `required_seamanship_tier` is NOT added — seamanship is never a hard gate.

**Files to touch:** `room_gateway.py` (split into base + two subclasses), `cmd_travel.py` (remove condition dict, thin out), `cmd_sail.py`, `cmd_explore.py`.

---

## Seamanship Risk

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

### Travel Delay and Narrative

Travel is currently instant. Planned: a configurable delay with staggered narrative messages.

- Add `travel_time` (seconds) and `travel_messages` (list of str) to destination config
- Use Evennia `delay()` to fire messages during transit
- Player unable to act during transit (temporary cmdset lockout or limbo room)
- Longer journeys (more bread) = more narrative beats

Example for a 3-bread journey:
```
"You set off along the narrow bush track..."
[3s] "You grow weary from days of travel."
[6s] "The scrub thins and the ground turns sandy beneath your feet."
[9s] "After 3 days of travel you arrive at a beach with a small cabin."
```
