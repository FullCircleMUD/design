# Parser, filter and helper inventory

The standard way to do a common job — parse what a player typed, find or filter something, word what
happened, move an asset. Check here before writing any helper; log any new one here.

- **Parser** — takes a string and splits it into arguments.
- **Filter** — takes parsed arguments and locates something: in contents, exits, a registry, the
  database. Includes the predicates that narrow candidates.
- **Helper** — any other job a second piece of code would otherwise write again.

One way to do each job. Use it; don't write another.

| To do this | Use | From |
|---|---|---|
| Read a yes/no answer through your own prompt | `parse_yes` / `parse_no` | `evennia_targeting` |
| Ask a yes/no question | `ask_yes_no` | `evennia.utils.evmenu` |
| Offer a choice from several options | `EvMenu`, `list_node` | `evennia.utils.evmenu` |
| Read a leading amount or `all` | `parse_quantity` | `evennia_targeting` |
| Read which entry of a numbered list was typed — a whole number from `low` to `high`, or `None` to try the text another way, such as a name with `parse_match` | `parse_int_in_range(text, low, high)` | `evennia_targeting` |
| Read `#<n>` as an NFT's `uri_id` — a bare number is an amount, not an id | `NFTService.parse_uri_id` | `fcm_xrpl` |
| Split `X <keyword> Y` | `parse_split` | `evennia_targeting` |
| Separate switches | `parse_switches` | `evennia_targeting` |
| Split a typed line into the commands it holds, on `;` — `py`, `nick` and interactive-py lines kept whole | `parse_stacked(line, in_prompt=False)` | `evennia_targeting` |
| Match a typed word against a known list of names — skills, switch words, enum values | `parse_match` — exact first, then word-start, then substring with `substring=True`; returns every match for the command to judge | `evennia_targeting` |
| Find the spell a player names among a set of spells — known, memorised | `match_spell(caller, typed, spells, not_found)` — `parse_match` over the spells' names; the spell, or `None` once the caller has been told why not. The spells commands' own | `components/spells/commands/matching.py` |
| Turn a typed slot name into one of the caller's wear slots | `match_slot(caller, text)` — `parse_match` over the caller's own slots; the enum member, or a refusal | `evennia_equipment.finders` |
| Find the item a player names among what they carry and are not wearing | `find_carried(caller, text)` — Evennia's search over the carried items, so aliases and `sword-2` work; the item, or a refusal naming a worn match | `evennia_equipment.finders` |
| Find the item a player names among what they are wearing | `find_worn(caller, text)` — the same, over the worn items | `evennia_equipment.finders` |
| Where an item would go on a wearer | `wearer.slots_for(item, slot=None)` — the group of slots, or `None`; changes nothing | `evennia_equipment` `EquipmentWearslotsMixin` |
| Parse a `wear` or `remove` argument — `<item> [on <slot>]`, `<item> from <slot>`, `from <slot>` | `resolve_wear(caller, text)`, `resolve_remove(caller, text)` — the item, and for wearing the slot, or a refusal | `evennia_equipment.contrib.utils` |
| Pick a command's mode — a sub-command | a switch: `prompt/set %h >`, read with `parse_switches`. There is no sub-command parser | `evennia_targeting` |
| Turn a switch into a language | `parse_language_switch` | `components.languages` |
| Find a direction in text | `parse_direction`, `resolve` | `components.directions` |
| Pick the Nth of several matches (`sword-2`) | `caller.search` | Evennia |
| Filter a room's or object's contents | `walk_contents`, `bucket_contents`, with `p_` / `f_` / `op_` predicates | `evennia_targeting` |
| Match a name against keys and aliases | `f_key_matches` | `evennia_targeting` |
| Resolve one word that could be a plain name **or** an object — a resource held or an item carried | `match_named(caller, text, names, candidates)` — a name in full, then the objects through `caller.search(quiet=True)`, then a start of a name. Filter both lists first; it filters nothing and messages nobody. Decide by the total it returns: one is the answer, more is a question | `evennia_targeting` |
| Count what a player named against the number they asked for — `get 2 sword` | `match_count(caller, text, candidates, quantity=1)` → `(found, CountOutcome)`: `EXACT`, `TOO_MANY`, `TOO_FEW` or `NONE`. One quiet search; filters nothing and messages nobody | `evennia_targeting` |
| Tell the caller why a count found no answer | `msg_count_outcome(caller, name, found, outcome, nofound, too_few=None)` — `nofound`, `too_few`, or Evennia's numbered list. Raises on `EXACT` | `components.parsers_filters` |
| Find what a player named in a holder — gold, a resource, or objects, counted | `find_in_holder(caller, holder, name, candidates, nofound, too_few, quantity=None)` → `(AssetKind, subject)`, or `None` once the caller is told why. The command narrows `candidates` and supplies the wording | `components.parsers_filters` |
| Find the container a player named — carried, or seen in the room — refusing a closed one or anything that is not a container | `find_container(name, no_container, not_a_container, closed, corpse=None)` on `ItemActingMixin`. A container is anything composing `EquipmentContainerMixin`; `corpse`, when given, answers a corpse. The item commands' own | `commands/item_interaction/acting.py` |
| Move gold or a resource for a typed amount — judge it against the holding, then transfer | `ItemActingMixin.move_fungible(source, destination, kind, resource_id, quantity, moved)`. The item commands' own | `commands/item_interaction/acting.py` |
| Tell the caller what moved in one line, and the room once | `ItemActingMixin.report(moved, line, others=(), mapping=None, **fields)` — answers the wording; `others` are left out of the room's telling. The item commands' own | `commands/item_interaction/acting.py` |
| Cost a turn in a fight, and nothing out of one | `ItemActingMixin.spend_turn()`. The item commands' own | `commands/item_interaction/acting.py` |
| Word a list — `a`, `a and b`, `a, b and c` | `joined(parts)` | `components.messaging` |
| Word a number as a player reads it — `12`, not `12.000000` | `plain_amount(value)` | `components.messaging` |
| Word gold and resources — `["12 gold", "5 wheat"]` | `fungible_parts(gold, resources)` — gold first, then each resource by name, leaving out a zero | `components.messaging` |
| How much of a holding a typed amount moves — `get 50 gold`, `loot all wheat` | `judge_amount(quantity, held, kind)` → the amount, or `0`. `None` and `all` are everything held; more than held is what is held; zero, a negative or a fraction of gold is nothing. Arithmetic only | `components.parsers_filters` |
| The objects in a holder the caller can see | `seen_items(caller, holder)` — sight is the test; every object in the holder is considered | `components.parsers_filters` |
| Find the target of anything violent, or of a spell at an actor — an attack, a bash, a backstab, a spell | `find_combat_target(caller, name, messages=None, include_self=False, hostile_act=True)` — a combat actor the caller can see in the room, or `None` once the caller has been told why not. `messages` rewords the refusals for the command; `include_self` lets the caller be the target; `hostile_act` applies the peaceful-room and PvP gates, and is off for a friendly act | `components.combat` |
| Every resource a holder has — details and quantity | `holder.get_held_resource_data()` → `{resource_id: (info, quantity)}`. Gold is not in it | `fcm_xrpl` `XRPLFungibleInventoryMixin` |
| Gold and each resource a holder has, by name, and which each is | `holder.get_held_fungible_names()` → `{name: (AssetKind, resource_id)}`; `resource_id` is `None` for gold. Only what is held | `fcm_xrpl` `XRPLFungibleInventoryMixin` |
| Tell gold, a resource and an NFT item apart | `AssetKind` — `GOLD`, `RESOURCE`, `NFT_ITEM` | `fcm_xrpl` |
| A resource's details by id — name, unit, weight, code | `ResourceService.get_resource_type(resource_id)` | `fcm_xrpl` |
| Gold's details — name, unit, weight, code | `GoldService.get_gold_type()` | `fcm_xrpl` |
| A resource's id from its full name | `ResourceService.get_resource_id_by_name(name)` | `fcm_xrpl` |
| Find a room by its uuid | `find_room_by_uuid` | `evennia_scaling.mixins` |
