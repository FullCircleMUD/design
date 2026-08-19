# Unified spawn system — predecessors

> **Archived historical material, not authoritative.** Kept so that a
> reference to one of the retired names below can be traced to what replaced
> it. For how the spawn system works, see
> [unified-item-spawn-system.md](../unified-item-spawn-system.md).

Before the unified spawn system, resource spawning was handled by
`ResourceSpawnService` with its own configuration, and gold and knowledge
items had no spawn system at all. The mapping below records where each
retired piece went.

| Retired | Replaced by |
|---|---|
| `ResourceSpawnService._process_resource()` calculation | `ResourceCalculator.calculate()` |
| `ResourceSpawnService._process_resource()` room distribution | unified target pool |
| `ResourceSpawnService.schedule_mob_drip_feed()` | unified target pool |
| `ResourceSpawnService._apply_mob_drip()` | the per-tick decide-and-place path |
| `RESOURCE_SPAWN_CONFIG` | unified `SPAWN_CONFIG` |
| `MOB_RESOURCE_SPAWN_CONFIG` | removed — tags on targets control distribution |
| no gold spawn system | `GoldCalculator` + `GoldDistributor` |
| event-driven knowledge item drops (designed, never built) | `KnowledgeCalculator` + scroll/recipe distributors |

Two of these are worth knowing about beyond name-tracing:

**Distribution stopped being configured.** `MOB_RESOURCE_SPAWN_CONFIG` declared
which mobs carried which resources. That is now expressed entirely by tags and
`spawn_<category>_max` attributes on the targets themselves, so adding a
resource or a mob type never touches spawn configuration.

**Knowledge drops were never event-driven in practice.** The earlier design
rolled a drop chance at mob death; the built system pre-places scrolls on
living mobs instead. The statistical outcome is the same — undersaturated
scrolls appear on more mobs — with one fewer system to maintain.
