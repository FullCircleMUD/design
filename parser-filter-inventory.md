# Parser and filter inventory

The standard way to parse what a player typed, or to find and filter something. Check here before
writing one; log any new parser or filter helper here.

- **Parser** — takes a string and splits it into arguments.
- **Filter** — takes parsed arguments and locates something: in contents, exits, a registry, the
  database. Includes the predicates that narrow candidates.

One way to do each job. Use it; don't write another.

| To do this | Use | From |
|---|---|---|
| Read a yes/no answer through your own prompt | `parse_yes` / `parse_no` | `evennia_targeting` |
| Ask a yes/no question | `ask_yes_no` | `evennia.utils.evmenu` |
| Offer a choice from several options | `EvMenu`, `list_node` | `evennia.utils.evmenu` |
| Read a leading amount or `all` | `parse_quantity` | `evennia_targeting` |
| Read `#<n>` as an NFT's `uri_id` — a bare number is an amount, not an id | `NFTService.parse_uri_id` | `fcm_xrpl` |
| Split `X <keyword> Y` | `parse_split` | `evennia_targeting` |
| Separate switches | `parse_switches` | `evennia_targeting` |
| Match a typed word against a known list of names — skills, slots, switch words, enum values | `parse_match` — exact first, then word-start, then substring with `substring=True`; returns every match for the command to judge | `evennia_targeting` |
| Pick a command's mode — a sub-command | a switch: `prompt/set %h >`, read with `parse_switches`. There is no sub-command parser | `evennia_targeting` |
| Turn a switch into a language | `parse_language_switch` | `components.languages` |
| Find a direction in text | `parse_direction`, `resolve` | `components.directions` |
| Pick the Nth of several matches (`sword-2`) | `caller.search` | Evennia |
| Filter a room's or object's contents | `walk_contents`, `bucket_contents`, with `p_` / `f_` / `op_` predicates | `evennia_targeting` |
| Match a name against keys and aliases | `f_key_matches` | `evennia_targeting` |
| Resolve one word that could be a plain name **or** an object — a resource held or an item carried | `match_named(caller, text, names, candidates)` — a name in full, then the objects through `caller.search(quiet=True)`, then a start of a name. Filter both lists first; it filters nothing and messages nobody. Decide by the total it returns: one is the answer, more is a question | `evennia_targeting` |
| Every resource a holder has — details and quantity | `holder.get_held_resource_data()` → `{resource_id: (info, quantity)}`. Gold is not in it | `fcm_xrpl` `XRPLFungibleInventoryMixin` |
| A resource's details by id — name, unit, weight, code | `ResourceService.get_resource_type(resource_id)` | `fcm_xrpl` |
| Gold's details — name, unit, weight, code | `GoldService.get_gold_type()` | `fcm_xrpl` |
| A resource's id from its full name | `ResourceService.get_resource_id_by_name(name)` | `fcm_xrpl` |
| Find a room by its uuid | `find_room_by_uuid` | `evennia_scaling.mixins` |
