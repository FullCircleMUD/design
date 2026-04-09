# Pets & Mounts System

## Overview

Pets and mounts are **persistent NFT objects** that follow a character, assist in combat, carry items, provide movement bonuses, and add gameplay depth. Both share a common foundation — a mount is a pet whose special ability is that it can be ridden.

All pets use the same NFT lifecycle as ships and equipment: owned, tradeable, destroyable. Magically created pets (familiars, raised undead) are NFTs that are destroyed on death, freeing the creator's slot.

---

## Architecture

### Class Hierarchy

Pets are **actors** (have HP, fight, display in rooms, follow) that are also **NFTs** (tracked on-chain, tradeable, destroyable). This is achieved through mixin composition:

```
NFTPetMirrorMixin(NFTMirrorMixin) — pet-specific mirror dispatch via owner_key
  │  Overrides at_post_move, at_object_delete with pet ownership logic
  │  Inherits token_id, factory methods, helpers from NFTMirrorMixin

BasePet(NFTPetMirrorMixin, FollowableMixin, BaseNPC)
  ├── Mule — medium pack animal (no combat)
  ├── WarDog(CombatCompanionMixin, BasePet) — medium combat pet
  └── Horse(CombatCompanionMixin, MountMixin, BasePet) — large mount + combat
```

For comparison, the existing item NFT chain:
```
BaseNFTItem(NFTMirrorMixin, HeightAwareMixin, HiddenObjectMixin, ItemRestrictionMixin, DefaultObject)
  └── WorldAnchoredNFTItem(OwnedWorldObjectMixin, BaseNFTItem) — ships, property
```

**Key difference:** Items live in `character.contents` — NFTMirrorMixin resolves ownership by walking the location chain (item → character → CHARACTER). Pets live in `room.contents` — NFTPetMirrorMixin resolves ownership via `owner_key` attribute on the pet, because a room always classifies as WORLD regardless of who owns the pet.

Behavioural mixins compose capabilities onto any pet type:
- `CombatCompanionMixin` — enters combat, attacks, uses abilities
- `MountMixin` — can be mounted/dismounted, movement bonuses
- `FamiliarMixin` — remote control (see through eyes, move room-to-room, return, scouting state)
- `CarryMixin` — wearable containers (panniers/saddlebags) — future
- `ScoutMixin` — can be sent to adjacent rooms, reports contents — future

A warhorse: `Horse(CombatCompanionMixin, MountMixin, BasePet)`.
A war dog: `WarDog(CombatCompanionMixin, BasePet)`.
A mule: `Mule(BasePet)` — no combat, no mount.
A familiar rat: `FamiliarRat(FamiliarMixin, BasePet)` — remote control.
A familiar hawk: `FamiliarHawk(FamiliarMixin, FlyingMixin, CombatCompanionMixin, BasePet)` — remote + fly + fight.

### NFT Model

Pets are NFTs but follow a different ownership model than items:

**Pets can only be in two places:**
1. A room (`room.contents`) — active in the world, mirror state = CHARACTER
2. An AccountBank (`bank.contents`) — stabled, mirror state = ACCOUNT

**Pets can NEVER be in `character.contents`.** Enforced by `NFTPetMirrorMixin.at_pre_move`.

Ownership is tracked via `owner_key` on the pet actor, NOT by the object's location in a character. The NFTPetMirrorMixin overrides dispatch logic so all mirror transitions (bank, unbank, craft_output, craft_input, transfer) are called automatically via `at_post_move` and `at_object_delete` hooks — same single-source-of-truth principle as items, just different ownership resolution.

- Has `token_id`, tracked in `NFTGameState` mirror DB
- Can be traded via `transfer_ownership()` (updates owner_key + mirror)
- Can be exported (stable first → ACCOUNT → export to chain)
- Destroyed on death → token returned to unallocated

**Creator tracking:**
```
creator_key = AttributeProperty(None)  # character_key of the caster who created this pet
```

Spells like Find Familiar check: "does an NFT with `creator_key == my_key` and `pet_type == familiar` still exist in the game?" If yes, can't cast again — regardless of who currently owns the familiar.

---

## Pet Lifecycle

### Creation Methods

| Method | Skill/Spell | Result | Tradeable | Limit |
|--------|-------------|--------|-----------|-------|
| **Find Familiar** | Conjuration spell | Familiar pet | No | 1 per caster (global) |
| **Raise Dead** | Necromancy spell | Undead pet | No | Per spell tier (1-5) |
| **Conjure Elemental** | Conjuration spell | Elemental pet | No | 1 per caster |
| **Tame** | ANIMAL_HANDLING skill | Tamed wild animal | Yes | Based on mastery |
| **Purchase** | Gold from trainer/stable | Horse, dog, mule | Yes | No limit |
| **Quest reward** | Quest completion | Unique pet | Varies | 1 per quest |

### States

Pets are always physical objects in the world — never teleported to the owner.

```
STABLED (at a stable, safe, no hunger)
  ↓ owner visits stable, retrieves pet
FOLLOWING (in world, following owner room to room)
  ↓ owner walks away / pet can't fit through door
WAITING (in world, at a location, hunger ticking)
  ↓ owner returns to the room, tells pet to follow
FOLLOWING
  ↓ owner visits a stable
STABLED
  ↓ HP reaches 0 (combat or starvation)
DEAD (NFT destroyed, returned to RESERVE)
```

There is no dormant/inventory state. The owner must physically go to where the pet is to collect it. Stabling is a safe parking spot, not magical storage.

Summoned pets (find familiar, raise dead) are different only in **creation** — the spell creates the NFT and places the animal in the caster's room. After creation, it behaves identically to any other pet.

### Death

When a pet's HP reaches 0:
1. Pet "dies" — room message, death animation
2. NFT object is deleted (returned to RESERVE)
3. Creator's slot is freed (for summoned pets)
4. No corpse, no loot (pets don't drop items)
5. Owner is notified: "Your familiar has been slain!"

---

## Pet Command — Routing Layer

A single `pet` command routes instructions to the active pet:

```
pet <command> [args]           — if only one active pet
pet.<name> <command> [args]    — target a specific pet by name
```

### Examples

```
pet attack goblin              — pet attacks the goblin
pet follow                     — pet follows you (default)
pet stay                       — pet stops following, stays in room
pet.horse mount                — mount the horse
pet.horse dismount             — dismount
pet.dog attack goblin          — dog attacks goblin
pet.owl scout north            — owl scouts the room to the north
pet stay                       — pet stops following, waits here
pet inventory                  — show what pet is carrying (if CarryMixin)
```

### Implementation

The `pet` command parses the prefix, finds the target pet from the owner's active pets, and delegates to the pet's own command handler. Each mixin adds commands to the pet's available actions:

- **All pets:** follow, stay
- **Summoned pets only:** dismiss (destroys the NFT — spell must be recast to create a new one)
- **CombatCompanionMixin:** attack, guard (protect owner)
- **MountMixin:** mount, dismount
- **CarryMixin:** inventory, give (transfer items to/from pet)
- **ScoutMixin:** scout <direction>

---

## Mounts

### What Makes a Mount Special

A mount is a pet with `MountMixin` that can be ridden. When mounted:

- Character's movement cost per room is drastically reduced (mount's stamina used instead)
- Character gains the mount's movement conditions (FLY for wyvern, WATER_BREATHING for dolphin)
- Character cannot enter incompatible rooms (auto-dismount or blocked)
- Character's room description changes: "rides a black stallion"

### Mount Types

| Mount | Size | Movement | Terrain | Equipment Slots |
|-------|------|----------|---------|-----------------|
| Pony | Large | Ground (moderate) | Roads, fields, forests | Saddle, Bridle, Saddlebags |
| Horse | Large | Ground (fast) | Roads, fields, forests | Saddle, Bridle, Barding, Saddlebags |
| Warhorse | Large | Ground (fast) | Same + combat bonus | Saddle, Bridle, Barding |
| Mule | Medium | Ground (slow) | Goes anywhere (indoor OK) | Bridle, Panniers (extra carry) |
| Wyvern | Huge | Flying | Air + ground | Saddle, Barding |
| Giant Eagle | Huge | Flying | Air + ground | Saddle |
| Dolphin | Large | Aquatic | Water only | Harness |
| Giant Seahorse | Large | Aquatic | Water only | Saddle, Harness |

Note: mules are not mountable (medium-sized, too small to ride) — they are pack animals only. All mountable animals are large or bigger.

### Room Compatibility

When mounted, room transitions check:
- **Ground mount:** can enter rooms with `max_height == 0` and `max_depth == 0`. Cannot enter indoor rooms (tagged `no_mount`), ladders, narrow passages.
- **Flying mount:** can enter rooms with height > 0. Character gains effective FLY. Cannot enter indoor rooms.
- **Aquatic mount:** can enter rooms with depth < 0. Character gains effective WATER_BREATHING. Cannot enter land rooms while mounted.

Auto-dismount triggers when entering an incompatible room:
- "You dismount your horse before entering the tavern."
- Flying mount + no-fly room: must land first (height 0) then dismount

### Equipment (Wearslots)

Each mount type has its own wearslot enum, following the `DogWearSlot` pattern:

```python
class HorseWearSlot(Enum):
    SADDLE = "saddle"           # required to mount
    BRIDLE = "bridle"           # required to lead/control
    BARDING = "barding"         # mount armor (AC bonus)
    SADDLEBAGS = "saddlebags"   # storage container
```

Saddle and bridle are **required** to mount. Without them, the mount follows as a pet but cannot be ridden. Barding provides AC to the mount. Saddlebags extend the rider's carrying capacity.

### Stabling

Stabling uses the same bank/unbank pattern as item banking:
- `RoomStable` typeclass with `CmdSetStable` commands. Tagged `stable` for size exceptions (large animals allowed).
- `stable <pet>`: pays dynamic fee (1g base + feed + heal), calls `pet.move_to(account_bank)`. The `at_post_move` hook fires ROOM→ACCOUNT → `NFTService.bank()`. Pet Evennia object moves into the bank.
- `retrieve <pet>`: finds pet in `bank.contents`, calls `pet.move_to(room)`. The `at_post_move` hook fires ACCOUNT→ROOM → `NFTService.unbank()`. Pet reappears in the stable room, starts following, hunger reset.
- `stabled`: lists pets in `bank.contents` where `is_pet == True`.
- Stabled pets are safe from combat, hunger, and world rebuilds.
- Millholm Stables uses `RoomStable` typeclass, placed adjacent to the inn for the rent loop.

---

## Combat

### All Pets in Combat

Every pet with HP can fight. When the owner enters combat:
1. Active pets are already in the owner's follow group
2. `enter_combat()` pulls them in on the owner's side (existing mechanic)
3. Pets auto-attack using `auto_attack_first_enemy()` (existing mechanic)
4. Pets can be targeted by enemies
5. Pet death triggers NFT destruction

### Combat Companions (CombatCompanionMixin)

Pets with this mixin have enhanced combat abilities:
- Custom `damage_dice`, `attack_message`, `attacks_per_round`
- May have weapon mastery (via `WeaponMasteryMixin`)
- Can use mob combat abilities (dodge, bash, etc.)
- AI state machine for combat decisions (via `StateMachineAIMixin`)

### Mounted Combat

When a character is mounted in combat:
- Mount can be targeted separately (enemies choose: rider or mount)
- Rider cannot be targeted by ground-level melee if mount gives height advantage
- Lance weapons get mounted charge bonus (future)
- Dismounting mid-combat is a free action but costs a round

---

## Persistence & Hunger

### Always Active

Pets are always active and following their owner — there is no summon/dismiss toggle. You buy or tame a pet, it follows you everywhere. To safely store a pet, you must stable it.

### Leaving Pets Behind

Players can leave pets anywhere. The pet stays in the room and waits for the owner to return. This is the expected pattern for dungeons:

1. Ride your warhorse to the dungeon entrance
2. Dismount and enter — the horse waits outside
3. Clear the dungeon, come back, collect your horse
4. The starvation timer ensures you don't forget about it permanently

No mechanic prevents separation — starvation is the natural leash.

### Size-Based Room Restrictions

Pet size determines where it can follow:

| Pet Size | Room Access | Examples |
|----------|------------|----------|
| Tiny / Small / Medium | Goes anywhere the owner goes | Cat, dog, hawk, mule |
| Large (most mounts) | Cannot enter `max_height == 0` rooms (indoor/enclosed) | Horse, warhorse, pony |
| Huge | Cannot enter indoor rooms OR narrow passages | Wyvern, giant eagle |

Note: mules are medium-sized — they can follow owners indoors, which is their key advantage over horses as pack animals. Most mountable animals are large or bigger because you need to sit on them.

When a pet or mount can't enter a room, **the owner is blocked** — no auto-dismount or auto-stay:
- Mounted on a horse trying to enter indoors: "You cannot enter here while mounted."
- Following wyvern too large: "Your wyvern is too large to go there."
- The player must explicitly dismount (`pet.horse dismount`) or tell the pet to stay (`pet.wyvern stay`) before proceeding on foot

Stable rooms are explicitly tagged to accept large animals regardless of room size.

### Pet Hunger

Pets have a simplified hunger cycle — much slower than characters:

| State | Duration | Effect |
|-------|----------|--------|
| **Fed** | 0–8 hours | No effect. Pet is healthy. |
| **Hungry** | 8–16 hours | Warning messages to owner. No stat penalty. |
| **Starving** | 16–24 hours | Reduced stats (carry capacity halved, movement slowed). |
| **Death** | 24+ hours | Pet dies. NFT destroyed. |

- **Feeding:** Buy hay/feed from a stable NPC. Use `feed <pet>` to reset timer.
- **Stabled pets do not consume food.** The stable fee covers feeding.
- **A normal play session never requires feeding** — pets arrive fed and last 8 hours.
- **Abandonment is not viable** — a pet left anywhere for 24 hours without food dies.

### Stabling & the Rent Loop

Stables are placed adjacent to inns in every town. The intended end-of-session flow:

1. Walk to town
2. Stable your pet (safe from hunger, combat, world events)
3. Walk next door to the inn
4. `rent` to safely log out

Characters who disconnect mid-dungeon without renting leave both themselves and their pets vulnerable. The rent mechanic and stable mechanic reinforce each other.

### Server Restart

- Active pets remain at their current location
- `at_server_start()` hook re-activates following for online owners' active pets
- Stabled pets are unaffected by restarts

---

## Taming & the Pet Economy

### Wild Animals as Source Material

Wild animals spawn in the world as untameable mobs (horses in plains, hawks in mountains, wolves in forests). Characters with **ANIMAL_HANDLING** skill can tame them, converting the wild mob into a pet NFT.

Taming works like crafting:
- Wild animal is the "ingredient" (consumed on success)
- Character's ANIMAL_HANDLING mastery determines what they can tame
- Higher mastery = more exotic/powerful animals available
- Tamed pet is minted as a new NFT, owned by the tamer
- Tamer can sell the pet NFT to other players

### Mastery Tiers

| Mastery | Tameable Animals | Examples |
|---------|-----------------|----------|
| BASIC | Common domestic animals | Dog, cat, mule, pony |
| SKILLED | Common wild animals | Horse, hawk, wolf |
| EXPERT | Uncommon wild animals | Warhorse, bear, giant eagle |
| MASTER | Rare/exotic animals | Wyvern, dire wolf, giant seahorse |
| GRANDMASTER | Legendary beasts | Phoenix, nightmare, kraken hatchling |

### Taming Process

1. Character finds a wild animal (spawned mob with `tameable` tag)
2. `tame <animal>` — contested check: d20 + CHA mod + ANIMAL_HANDLING bonus vs animal's tame DC
3. Success: wild mob is consumed, pet NFT is created in tamer's inventory
4. Failure: animal may flee or become aggressive
5. Pet inherits stats from the wild animal template (HP, damage, size, movement type)

### Economy

Animal trainers are a **crafting profession** — they don't fight, they produce pets for the market:
- Tame wild horses → sell trained riding horses to players
- Tame hawks → sell scout companions to rangers
- Tame war dogs → sell combat companions to warriors
- Rare tames (wyvern, dire wolf) command premium prices

This creates a natural supply chain: wild animal spawns → trainers → player market.

---

## Infrastructure Status

| System | Status | File(s) |
|--------|--------|---------|
| `NFTPetMirrorMixin` | **Built** | `typeclasses/mixins/nft_pet_mirror.py` |
| `NFTMirrorMixin` | **Built** | `typeclasses/mixins/nft_mirror.py` (parent, inherited) |
| `BasePet` | **Built** | `typeclasses/actors/pets/base_pet.py` |
| `Mule` | **Built** | `typeclasses/actors/pets/mule.py` |
| `WarDog` | **Built** | `typeclasses/actors/pets/war_dog.py` |
| `Horse` | **Built** | `typeclasses/actors/pets/horse.py` |
| `CombatCompanionMixin` | **Built** | `typeclasses/mixins/combat_companion.py` |
| `MountMixin` | **Built** | `typeclasses/mixins/mount_mixin.py` |
| `CmdPet` | **Built** | `commands/all_char_cmds/cmd_pet.py` (multi-pet dot syntax) |
| `RoomStable` | **Built** | `typeclasses/terrain/rooms/room_stable.py` |
| Stable commands | **Built** | `commands/room_specific_cmds/stable/cmd_stable.py` |
| Pet regen/starvation | **Built** | `typeclasses/scripts/regeneration_service.py` |
| Pet auto-flee | **Built** | `commands/all_char_cmds/cmd_flee.py` |
| Pet disband protection | **Built** | `commands/all_char_cmds/cmd_follow.py` |
| Pet size restrictions | **Built** | `typeclasses/actors/character.py` |
| Pet tests | **Built** | `tests/typeclass_tests/test_pets.py` (27 tests) |
| `FollowableMixin` | Built | `typeclasses/mixins/followable.py` |
| `FamiliarMixin` | **Built** | `typeclasses/mixins/familiar_mixin.py` — remote control, scouting |
| Familiar types (5) | **Built** | `typeclasses/actors/pets/familiars/` — rat, cat, owl, hawk, imp |
| Find Familiar spell | **Built** | `world/spells/conjuration/find_familiar.py` |
| Combat side system | Built | `combat/combat_utils.py` |
| `DogWearSlot` | Built | Example of non-humanoid wearslots |
| `cmd_tame` | **Built** | Tame wild animal into pet — contested d20 + CHA + mastery vs tame DC |
| `ANIMAL_HANDLING` skill | Defined | General skill, all classes |
| `WildMule` | **Built** | Tameable mob POC (Eeyore's Gloomy Place, tame_dc=10, BASIC) |
| Pet NFTItemTypes | **Built** | Mule, War Dog, Horse seeded in migration |

---

## Implementation Phases

### Phase 1: Core Pet System ✅
- `BasePet(NFTPetMirrorMixin, FollowableMixin, BaseNPC)` — actor in room with NFT tracking
- `Mule` — medium pack animal POC
- `CmdPet` routing command — follow, stay, feed, status
- Pet hunger timer (8h fed → 8h hungry → 8h starving → death)
- Pet size restrictions (large+ blocked from indoor rooms)
- Pet death → NFT destruction via `at_object_delete`

### Phase 2: Combat Companions ✅
- `CombatCompanionMixin` — composable combat for any pet
- `WarDog` — medium combat pet (1d6 bite, 25 HP)
- `pet attack <target>` command
- Auto-enter combat with owner via FollowableMixin group mechanics
- Pet auto-flee with owner
- Pet disband protection (re-attach to owner on group disband)

### Phase 3: Mounts ✅
- `MountMixin` — mount/dismount, movement bonus, force dismount on death
- `Horse` — large mount (3x move efficiency, 1d4 kick, 30 HP)
- `pet mount` / `pet dismount` commands
- Mounted movement cost reduction (1 point per N moves)
- Block indoor entry while mounted on large+ pet
- Room display: "Bob rides a black stallion." (combined, mount hidden)

### Phase 4: Stabling ✅
- `RoomStable` typeclass with stable tag (large animals allowed)
- Stable = `pet.move_to(account_bank)` → at_post_move → bank()
- Retrieve = `pet.move_to(room)` from bank.contents → at_post_move → unbank()
- Dynamic pricing: 1g base + feed + heal
- Millholm Stables updated to RoomStable
- Stabled pets listed from bank.contents

### Phase 5: Partial ✅
- Multi-pet dot syntax (`pet.horse mount`, `pet.dog attack goblin`)
- Pet naming (`pet name Rover`)
- Pet HP regen via regeneration service
- Pet starvation death via regen service

### Phase 6: Familiars ✅
- `FamiliarMixin` — remote control (see through eyes, move room-to-room, return, scouting state)
- 5 familiar types by conjuration mastery tier:
  - BASIC: Rat (small, goes anywhere)
  - SKILLED: Cat (stealth — doesn't trigger mob aggro)
  - EXPERT: Owl (FlyingMixin — aerial scouting)
  - MASTER: Hawk (FlyingMixin + CombatCompanionMixin — fly + fight)
  - GM: Imp (FlyingMixin + CombatCompanionMixin + LIGHT_SPELL — fly + fight + illuminate)
- Find Familiar spell (conjuration) — one per caster (global via `creator_key`)
- `pet look` — see through familiar's eyes from any distance
- `pet <direction>` — move familiar remotely, auto-look
- `pet return` — teleport familiar back to caster
- `pet dismiss` — destroy summoned pet (magical pets only, owner not creator)
- `_find_my_pets()` extended to find scouting familiars globally
- Creator limit: can't summon new familiar while old one exists (even if transferred)
- Dismiss gating: only current owner can dismiss, not creator

### Phase 7: Taming ✅
- `CmdTame` — contested d20 + CHA modifier + mastery bonus vs tame DC
- `WildMule` tameable mob typeclass (passive, tame_dc=10, BASIC mastery)
- Taming flow: validate tameable → check mastery → roll → assign_item_type → spawn_pet → auto-follow
- Pet NFTItemType seed data: Mule, War Dog, Horse in migration
- POC: WildMule in Eeyore's Gloomy Place (Hundred Acre Wood book zone)
- Tested end-to-end: tame → follow → book zone recall → library transitions

### Remaining (Future)
- `CarryMixin` — mule/horse saddlebags extend carry capacity
- Flying mounts (wyvern — grants FLY condition)
- Aquatic mounts (dolphin — grants WATER_BREATHING)
- Mount wearslots (HorseWearSlot: saddle, bridle, barding)
- Mounted combat bonuses (lance charge, height advantage)
- More tameable animals (wild horse, wolf, hawk — higher mastery tiers)
- Pet purchase from stable NPCs
- Pet leveling / XP sharing

---

## Future Ideas

- **Pet loyalty** — neglected pets may run away or refuse commands
- **Pet breeding** — combine two pets for offspring with mixed traits
- **Pet appearance in prompts** — show pet HP in combat prompt
- **Pet-specific quests** — fetch quests that require a dog, scouting that requires a flying pet
- **Companion dialogue** — LLM-powered pet personality for familiars
- **Pack animals** — mules in caravans for trade route gameplay
- **Mounted archery** — ranged attacks from horseback
- **Jousting** — mounted PvP with lance mechanics
