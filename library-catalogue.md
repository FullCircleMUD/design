# Library catalogue

Every library the game uses, ordered from fewest dependencies to most. *Dependencies* lists only
other libraries in this catalogue.

| Library | What it does | Dependencies | Docs |
|---|---|---|---|
| `evennia-logging-extension` | Evennia's per-file logging, working before the reactor is running. | — | |
| `evennia-yaml-reader` | Reads YAML files from any source for declarative-content libraries. | — | |
| `evennia-targeting` | Targeting predicates, content filters and input parsers. | logging-extension | |
| `evennia-calendar` | Game-world dates, seasons and time of day, derived from game time. | logging-extension | |
| `evennia-effects-conditions` | Ref-counted condition flags and timed, anti-stacking effects. | logging-extension | |
| `evennia-database-cascade` | Resolves a game's database aliases from the environment, with matching routers. | logging-extension | |
| `evennia-portal-multiplex` | Runs several servers behind one portal and redirects sessions between them. | logging-extension | |
| `evennia-llm-service` | The LLM provider client and prompt template loader. | logging-extension | |
| `evennia-equipment` | Wear slots, worn and wielded equipment, and carrying capacity. | logging-extension, targeting | |
| `evennia-environment` | Terrain and weather, and what they do to whoever is in a room. | logging-extension, calendar | |
| `evennia-survival` | Hunger, thirst and regeneration meters. | logging-extension, calendar | |
| `evennia-world-builder` | Declarative YAML world authoring. | logging-extension, yaml-reader | |
| `evennia-mob-spawner` | Declarative YAML mob spawning. | logging-extension, yaml-reader | |
| `evennia-archive` | Archives accounts and characters to a separate database, so the world can be rebuilt without losing players. | logging-extension, database-cascade | |
| `evennia-message-bus` | Messages between separate instances through a shared database. | logging-extension, database-cascade | |
| `fcm-xrpl` | On-chain item and currency ownership on the XRP Ledger. | logging-extension, database-cascade, targeting | |
| `fcm-subscriptions` | Subscription access and on-chain payment. | logging-extension, database-cascade, fcm-xrpl | |
| `evennia-ai-memory` | Embedding-backed memory and lore retrieval for LLM-driven NPCs. | logging-extension, database-cascade, targeting, yaml-reader | |
| `evennia-scaling` | Moves a character between independent instances, each on its own database. | logging-extension, portal-multiplex, archive, message-bus | |

## Unfinished

| Library | What it does | Dependencies | Docs |
|---|---|---|---|
| `evennia-procedural-dungeons` | Periodically rewires the exits between a fixed set of rooms. | logging-extension | |
| `evennia-mob-decision-engine` | Mob decision-making. A README and licence only so far. | — | |
