# Cartography & Maps Design

> This is the canonical design document for the cartography skill's **intra-zone mapping system** (`survey`, `map` commands). For interzone travel design — the `explore` command, cartography mastery gates, route map NFTs, sea routes, and the BaseGate architecture — see `design/INTERZONE_TRAVEL.md`. For world-building context see `design/WORLD.md`.

---

## Overview

**CARTOGRAPHY** is a general skill available to all character classes. Trainable to BASIC in Millholm — enough to leave the starting zone.

Two distinct systems under one skill:
- **System 1: Inter-Zone Route Discovery** — `explore` command gates zone-to-zone travel. See `design/INTERZONE_TRAVEL.md`.
- **System 2: Intra-Zone District Mapping** — `survey` command produces tradeable map NFTs *(current implementation — documented below)*

---

## Command Vocabulary

| Command | System | Purpose |
|---|---|---|
| `explore` | System 1 | Move party to an unknown zone via a gateway. Mastery-gated. Auto-produces a route map NFT on arrival. See `design/INTERZONE_TRAVEL.md`. |
| `survey` | System 2 | Timed per-room action. District scale: reveals current + adjacent map cells. Region scale: reveals current map cell only. |
| `map` | System 2 | View a map from inventory, rendered server-side with unexplored cells hidden. `map <name>` does a case-insensitive substring match against the map's display name. |
| `chart` | — | **REMOVED.** Originally intended for manual map creation. Superseded by auto-map-on-explore (System 1) and `survey` (System 2). |

`explore` will eventually move from the general skill cmdset to a cartographer-specific cmdset.

---

## System 2: Intra-Zone Mapping

> *Implemented. Millholm Town is the reference implementation; other maps (region, sewers, mine) use the same system.*

### Core Mechanic

No mastery gate — any cartographer (CARTOGRAPHY BASIC+) can map any area they can physically reach.

**Predefined ASCII maps.** Each mappable area has a pre-drawn ASCII template stored in the map registry. Rooms are tagged with map point keys. Rendering is **server-side only** — unsurveyed cells are replaced with blank space before the string is sent to the client. The client never receives hidden content, preventing cheating via text selection. Structural characters (dashes) connecting unsurveyed cells are also hidden — the map reveals nothing until the player surveys.

### Tag Convention

```python
room.tags.add("<map_key>:<point_key>", category="map_cell")
```

A room can carry multiple `map_cell` tags to appear on more than one map simultaneously. Surveying a room updates all matching maps in inventory at once.

### Auto-Creation

When a character with CARTOGRAPHY BASIC+ enters any room with a `map_cell` tag and doesn't hold a map for that map key in inventory, a **blank district map NFT (0%)** auto-spawns in their pack. No command needed.

### The `survey` Command

- **Active, timed** (~3 seconds per room)
- Surveys the **current room plus adjacent rooms** (district scale) or **current room only** (region scale) — see "Survey Behaviour by Map Scale" below
- If the room has multiple map tags and the character holds multiple matching maps, all are updated simultaneously
- Cancelled by movement before timer fires
- Blocked in combat
- Completion % recalculates after each survey
- **Smart skip:** if the current room AND all adjacent cells are already surveyed, the command reports "You've already mapped everything notable here" rather than running the timer
- Creates tactical tension in dangerous areas — standing still while mobs wander. The party may need to protect the cartographer.

*Flavour: "You kneel down, unfurling your map. The scratch of charcoal on parchment echoes quietly..."*

#### Survey Behaviour by Map Scale

Survey behaviour depends on the map's `scale` field (`"district"` or `"region"`), set at map registration time. The same `survey` command handles both — it checks each matching map's scale to decide what to reveal.

**District scale** (`scale: "district"`) — used for town maps, sewer maps, mine maps:
- Reveals the current room's cell **plus all adjacent rooms** (via exit graph)
- Adjacent room discovery means surveying a road reveals the shops that open onto it — you can see the bakery entrance from the street without going inside
- A straight road can be fully mapped by surveying every other room
- **Tagging:** 1 room → 1 cell (1:1 mapping). Every mappable room gets its own point_key.

**Region scale** (`scale: "region"`) — used for zone overview maps:
- Reveals **only the current room's cell** (no adjacent cell revelation)
- Multiple rooms may be tagged to the same cell — surveying any one of them reveals it
- The cartographer must physically travel to each area of the map and survey separately
- **Tagging:** Multiple rooms → 1 cell (many:1 mapping). The map builder decides which rooms along a path to tag to each cell. Tag 3-4 rooms in the southern woods to the same `~` cell — the player surveys any one to reveal it, then moves on to the next cluster.

This means region maps require genuine exploration — you can't map the zone from the town square. District maps are faster because adjacency does the work, matching the "I can see down the street" feel.

#### Map Registration

The `scale` field controls survey behaviour:

```python
register_map({
    "key": "millholm_town",
    "display_name": "Millholm Town",
    "scale": "district",        # survey reveals current + adjacent cells
    "template": _TEMPLATE,
    "point_cells": _POINT_CELLS,
})

register_map({
    "key": "millholm_region",
    "display_name": "Millholm Region",
    "scale": "region",          # survey reveals current cell only
    "template": _TEMPLATE,
    "point_cells": _POINT_CELLS,
})
```

No new tag categories needed — both scales use `category="map_cell"` with the same `"<map_key>:<point_key>"` format. The difference is purely in how rooms are tagged (1:1 vs many:1) and how the survey command interprets adjacency.

### The `map` Command

- `map` — lists all district maps in inventory with completion %
- `map <name>` — renders the ASCII map with server-side masking (unsurveyed cells → blank space). Case-insensitive substring match against the display name (e.g. `map town`, `map mill`). Auto-generated legend at the bottom shows only POI types visible on the current render.
- If no match in inventory: *"You don't have a map of that area."*

### Inventory Display

Map items always display as `Map: Millholm Town 37%` — in inventory, on the ground, in trade/auction. Buyers always see completion % before purchasing.

### Ownership Rules

**One map per map_key in inventory.** Cannot hold two maps of the same area in pack at once.

**Bank has no limit.** The cartography business model:
1. Enter a mapped area → blank map auto-creates
2. Survey rooms until satisfied with coverage
3. Bank the map
4. Re-enter the area → new blank map forms
5. Repeat to build stock for sale/trade/auction

### Transfer Rules

- **Non-cartographer buys a map** → static snapshot. Completion % never improves. They can view and navigate with it.
- **Cartographer buys a map** → can resume surveying. Builds on previous cartographer's work.

### Map Value

- **Simple maps** (Millholm Town) — low value, easy to complete, mostly convenience
- **Complex maps** (Bayou swamp, mountain passes, sewer systems) — high value, easy to get lost without one
- **Dungeon/sewer maps** — premium goods. A 100% sewer map is genuinely valuable to a party heading underground.
- **Procedural dungeon maps** — technically unmappable (see below), but entrance markers on district maps retain value

---

## District Map Design Conventions

### POI Symbol Registry

All map symbols are defined in a single central registry (`world/cartography/poi_symbols.py`). Map definitions reference POI types by string key; `render_map()` looks up the display character at render time. Changing a symbol in the registry updates every map.

**Current symbol table:**

| Symbol | POI Type | Description |
|---|---|---|
| `#` | road | Streets, paths, open outdoor space |
| `@` | gate | Town gates, zone boundaries |
| `C` | cemetery | Graveyards |
| `I` | inn | Inns, taverns |
| `S` | smithy | Blacksmith workshops |
| `$` | bank | Banks |
| `+` | temple | Temples, shrines |
| `G` | guild | Guild halls |
| `g` | gaol | Prisons, gaols |
| `*` | shop | General stores |
| `B` | bakery | Bakeries |
| `H` | stable | Stables |
| `W` | woodshop | Carpenter workshops |
| `T` | tailor | Tailor shops, textile shops |
| `L` | leathershop | Leatherworking shops |
| `A` | apothecary | Apothecaries, herbalists |
| `J` | jeweller | Jeweller workshops |
| `P` | post_office | Post offices |
| `h` | house | Residential houses |
| `D` | distillery | Distilleries |
| `X` | zone_exit | Exits to other zones/districts |

**Underground/dungeon symbols:** `~` tunnel, `O` chamber, `v` shaft, `x` dead end, `m` mine, `!` lair.

**Region overview symbols:** `T` town, `F` farm, `W` woods, `d` district, `?` unknown.

### Map Layout Rules

**Squares and markets are roads.** The 3×3 market square in Millholm Town maps to 9 road cells (`#`). From a navigation perspective, the square is open outdoor space that characters walk through. Shops line the edges — they get their own cells with POI symbols adjacent to the road cells.

**One cell per room.** Each game room maps to exactly one cell position on the map. A room can occupy multiple cells only when it logically spans a large area (e.g. `sq_center` maps to 3 cells on the main road row to represent the wide market square).

**Shared physical space.** When two rooms occupy the same logical space (e.g. a house entered from the road and a shop entered from the square share the same building), map the more important one. Gareth's House is omitted from the Millholm Town map because the General Store occupies the same grid position and is more useful to players.

**Building interiors are not mapped.** Maps show the street layout and POI markers for buildings. The interior of a shop, inn, or guild hall does not appear on the map. This keeps maps clean and avoids mapping every room in multi-room buildings.

**Adjacent room revelation (district scale only).** On district-scale maps, surveying a room reveals it plus all adjacent rooms (via exit graph). This means standing on the main road and surveying reveals the shops on both sides without entering them. The design intent: a cartographer walks the streets and surveys at intervals, building a useful town map showing all landmarks, without visiting every building interior. Region-scale maps do not use adjacent revelation — see "Survey Behaviour by Map Scale" above.

### Legend Generation

The legend is auto-generated at render time. Only POI types that appear in visible (surveyed) cells are included. Sorted alphabetically by symbol. Format: `symbol=Name  symbol=Name  ...`

Example after surveying part of Millholm Town:
```
#=Road  +=Temple  $=Bank  @=Gate  B=Bakery  G=Guild  S=Smithy
```

### Map Scale — District (Zoomed In) vs Region (Zoomed Out)

Maps come in two scales with different visual conventions:

**District maps** (town, sewers, mine) — zoomed in, one cell per room:

- **Horizontal connectors:** single dash `-` between cells (e.g. `T-S-W-#-#-#-B-A-J`)
- **No vertical connectors.** North-south relationships are implied by column alignment across rows. Removing `|` pipes keeps the map clean — the eye follows the vertical alignment naturally.
- **Each row is a row of rooms.** No blank lines between rows needed (compact layout).
- **POI symbols are specific** — individual shops, services, and buildings get their own symbol from the town POI table (smithy=`S`, bank=`$`, inn=`I`, etc.)

Example (Millholm Town — 14 rows, Artisan's Way district in south):
```
          X              row 0:  lake track (north exit)
        C-@              row 1:  cemetery, cemetery gates
          #              row 2:  north road
        I-#-H            row 3:  inn, square north, stables
  *-*-*-#-#-#-B-*-*     row 4:  north shops, square top
X-#-#-#-#-#-#-#-#-#-X   row 5:  Old Trade Way (main road)
  h-h-*-#-#-#-$-P-*     row 6:  south side shops
        +-#-G            row 7:  shrine, south road, mages guild
        #-#-G            row 8:  beggars alley, mid south, warriors guild
    h-W-*-#-W-W-h       row 9:  artisan's way north side
    #-#-#-#-#-#-#        row 10: artisan's way lane
    W-*-W-#-g-*-W        row 11: artisan's way south side
        I-#-g            row 12: broken crown, far south, gaol cell
          @              row 13: south gate
```

**Region maps** (zone overview) — zoomed out, one cell per area/district:

- **No connectors by default.** Layout is double-spaced horizontally, single-spaced vertically, no structural characters. Adjacency is implied by proximity alone.
- **Manual `--` connectors only where a physical road or path exists.** The map builder adds dashes where a built road connects two areas (e.g. The Old Trade Way between farms, town, and woods). Wilderness adjacency doesn't get connectors — forests, mountains, and open terrain sit next to each other with spacing only.
- **Room scale is variable.** A cell in town represents a smaller physical area than a cell in the wilderness. The 3×3 town block covers a few streets; a single `~` cell in the woods covers a large exploration grid.
- **POI symbols are generalised.** Region maps use a broader, less specific set: `T` town, `F` farm/harvesting, `~` wilderness, `R` resource processing, `M` mine, `C` cemetery, `D` dungeon, `?` point of interest, `Z` zone exit, `#` road. Some symbols intentionally cover multiple meanings (e.g. `F` = wheat farm, cotton farm, herb field, or any harvesting area).
- **Template structure differs from district maps.** No automatic dash-fill — the builder places `--` manually where roads exist. This keeps wilderness feeling open and organic rather than gridded.

Example contrast:
```
R--#--#--#--#--T--T--T--#--#--#--#--#--Z    ← road: dashes connect
      #        T  T  T  R  ~  ~  ~         ← road left, wilderness right: no dashes in woods
                              ~  R  ~       ← deep wilderness: spacing only
```

- See `maps/drafts/millholm_region.md` for the working draft

### Template Structure

Map templates use `.` as placeholder characters at cell positions. Structural characters (dashes `-`) connect cells horizontally. At render time, `.` characters are replaced with the POI symbol from the registry (if surveyed) or blank space (if unsurveyed). Structural characters adjacent to unsurveyed cells are also hidden.

```python
# District-scale template (millholm_town excerpt)
_TEMPLATE = (
    "          .          \n"  # row 0: cemetery
    "          .          \n"  # row 1: gate
    "          .          \n"  # row 2: north road
    "        .-.-.        \n"  # row 3: inn, sq_n, stables
    "  .-.-.-.-.-.-.-.-.  \n"  # row 4: north shops
    ".-.-.-.-.-.-.-.-.-.-.\n"  # row 5: main road
    "  .-.-.-.-.-.-.-.-.  \n"  # row 6: south shops
)
```

Cell positions use `.` placeholders at even columns. Dashes `-` fill odd columns on horizontally connected rows. Vertical connections are implied by column alignment — no `|` characters in templates. Region-scale templates omit automatic dashes; the builder places `--` manually where roads exist.

### Point Cell Format

Each point cell declares its grid position(s) and POI type:

```python
_POINT_CELLS = {
    "cemetery":   {"pos": [(0, 10)], "poi": "cemetery"},
    "sq_center":  {"pos": [(5, 8), (5, 10), (5, 12)], "poi": "road"},
    "smithy":     {"pos": [(4, 4)], "poi": "smithy"},
}
```

A single room can map to multiple positions when it spans a wide area (e.g. the market square centre).

---

## Predefined Maps — Millholm

### `millholm_town`

Streets, road segments, POI markers for shops and services, cemetery, stables, guild halls. Market square presented as road cells with shops flanking both sides. Gareth's House omitted (shares grid position with General Store).

**NOT included:** building interiors, the secret passage (Gareth→Abandoned House), NPC back rooms (hendricks_house, mara_house, priest_quarters, arcane_study, barracks).

**Target layout** (fully surveyed):
```
          X              lake track (north exit)
        C-@              cemetery, cemetery gates
          #              north road
        I-#-H            inn, square north, stables
  *-*-*-#-#-#-B-*-*     north shops, square top
X-#-#-#-#-#-#-#-#-#-X   Old Trade Way (main road)
  h-h-*-#-#-#-$-P-*     south side shops
        +-#-G            shrine, south road, mages guild
        #-#-G            beggars alley, mid south, warriors guild
    h-W-*-#-W-W-h       artisan's way north side
    #-#-#-#-#-#-#        artisan's way lane
    W-*-W-#-g-*-W        artisan's way south side
        I-#-g            broken crown, far south, gaol cell
          @              south gate
```

**Legend:** `#`=Road `@`=Gate `C`=Cemetery `I`=Inn `H`=Stable `B`=Bakery `W`=Workshop `*`=Shop `$`=Bank `P`=Post Office `+`=Temple `G`=Guild `g`=Gaol `X`=Zone Exit `h`=House

**Key spatial decisions:**
- The 3×3 market square (sq_nw, sq_n, sq_ne, sq_w, sq_center, sq_e, sq_sw, sq_s, sq_se) maps as 9 road cells
- sq_center occupies 3 horizontal positions on the main road row to represent the wide square
- sq_n occupies 2 vertical positions (row 3 + row 4) to give the north side breathing room
- North shops use generic `*` (shop) symbol — weapons, armour, clothing, magical supplies, jeweller
- Artisan's Way (rows 9-11) is a new southern district with workshops (`W`) for smithy, apothecary, textiles, leathershop, jeweller, woodshop
- South road branches have shrine, warriors guild, mages guild, broken crown, gaol flanking the road
- Zone exits (`X`) mark connections: lake track north, woods east, southern district south
- Cemetery connects to the main north road via cemetery gates (`@`)

### `millholm_region`
Region-scale (`scale: "region"`) zone overview. Each cell represents an area or district, not individual rooms. Multiple rooms tagged to the same cell — surveying any one reveals it. No adjacent cell revelation. Uses generalised POI symbols (`T` town, `F` farm/harvesting, `~` wilderness, `R` processing, `M` mine, `#` road, `Z` zone exit, `?` point of interest, `C` cemetery, `D` dungeon). Road connections shown with `--` dashes; wilderness adjacency implied by spacing only.

**Deep woods treatment:** The procedural deep woods passage rooms are NOT tagged (dynamically created/destroyed). The static clearing at the end of the passage tags to `millholm_region:deep_woods`. Players see "something is out there" but not how to get there — unsurveyed cells show as blank space.

The miners camp / mine entrance tags to BOTH `millholm_mine:mine_entrance` AND `millholm_region:deep_woods` — surveying it updates both maps simultaneously. The faerie hollow entry and the mine entrance both resolve to the same `deep_woods` cell on the region map, so neither destination is differentiated — just a shared mystery marker.

See `maps/drafts/millholm_region.md` for the working layout draft.

### `millholm_sewers`
All sewer rooms. The crumbling wall room (thieves lair entrance) IS tagged — the map shows something is there. The thieves lair interior is NOT tagged.

### `millholm_mine`
All mine rooms. The miners camp / mine entrance tags to BOTH `millholm_mine:mine_entrance` AND `millholm_region:deep_woods`. Surveying that room updates both maps simultaneously if held.

---

## Procedural Areas

Procedurally generated areas (deep woods passage, rat cellar, cave of trials, etc.) are **not mappable**. Their rooms are created fresh each instance and do not persist.

Cartographers can mark the **entrance** to a procedural area on a district map — the entrance room gets a map_cell tag. The interior cannot be surveyed or mapped. This makes procedural areas permanently dangerous: no shortcuts, no bought maps, no memorised routes. Every visit is fresh exploration.

---

## Technical Architecture (System 2)

### POI Symbol Registry (`world/cartography/poi_symbols.py`)
- `POI_SYMBOLS` — dict of POI type string → single display character
- `POI_NAMES` — dict of POI type string → human-readable legend name
- Single source of truth for all map rendering. Change here → every map updates.

### Map Registry (`world/cartography/map_registry.py`)
- `MAP_REGISTRY` dict keyed by `map_key`
- Each entry: `key`, `display_name`, `scale` (`"district"` or `"region"`), `template` (ASCII layout string with `.` placeholders), `point_cells` (dict of `point_key → {"pos": [(row, col), ...], "poi": "<type>"}`)
- `get_map(key)` — lookup
- `get_map_keys_for_room(room)` — returns `[(map_key, point_key)]` from room's `map_cell` tags
- `render_map(map_def, surveyed_points)` — server-side rendering. Replaces `.` placeholders with POI symbols from `poi_symbols.py` for surveyed cells, blank space for unsurveyed. Hides structural characters (`-`) adjacent to unsurveyed cells. Returns `(rendered_ascii, legend_string)`.

### Map Files (`world/cartography/maps/`)
One file per map definition. Each registers via `register_map()` on import. Point cells declare both grid position and POI type — the template character at the position is ignored (`.` placeholder).

### `DistrictMapNFTItem` (`typeclasses/items/maps/district_map_nft_item.py`)
- `map_key` — which predefined map this NFT represents
- `db.surveyed_points` — set of `point_key` strings
- `completion_pct` — `@property`, `round(len(surveyed) / len(all_points) * 100)`
- `get_display_name()` — returns `"Map: {display_name} {pct}%"`

### Auto-creation (`typeclasses/actors/character.py`)
`_check_map_autocreation()` called from `at_post_move()`. Checks CARTOGRAPHY BASIC+, reads room tags, spawns blank NFT for any unowned map key.

### NFT Seed Data
`NFTItemType` entry: `item_type_name="DistrictMap"`, `typeclass="typeclasses.items.maps.district_map_nft_item.DistrictMapNFTItem"`, `prototype_key="district_map"`.

### Self-Healing Key
Old map NFTs created before the key fix had `key="DistrictMap"` making them impossible to drop/search by name. `get_display_name()` now self-heals the key to `f"map of {display_name}".lower()` (e.g. `"map of millholm town"`) on first access. Players can drop/search maps by name fragment (e.g. `drop town`, `drop region`).
