# Component catalogue

Every component in the game, ordered from fewest dependencies to most. *Dependencies* is the
component's `DEPENDS_ON`, declared in its `__init__.py`.

| Component | What it does | Dependencies | Docs |
|---|---|---|---|
| `custom_properties` | The `AttributeProperty` subclasses the game declares its attributes with. | — | |
| `damage_type` | The kinds of damage a blow can deal, and the verbs each is described with. | — | |
| `dice` | Rolls dice from a standard RPG roll string. | — | |
| `mastery` | How good someone is at something, on a six-step scale. | — | |
| `signals` | The game's signal declarations, in one place. | — | |
| `static_registry` | The base every registry in the game is built on. | — | |
| `preferences` | The switches a player sets about their own play, and the `toggle` command. | custom_properties | |
| `water_container` | What holds drinkable water, what fills it, and the commands between them. | custom_properties | |
| `queued_cmds` | Commands that wait their turn in a fight. | combat | |
| `telemetry_spawn` | Measures the game economy each hour, and spawns into it what the measurements call for. | fcm_xrpl | |
| `alignment` | A moral standing, as one number from pure evil to pure good. | custom_properties, signals | |
| `busy` | An action that takes time, holding the character while it runs. | messaging, signals | |
| `damage_resistance` | How much a blow is reduced or amplified by what it lands on. | damage_type, signals | |
| `directions` | The ways out of a room, and what they are called. | custom_properties, evennia_world_builder | |
| `durability` | How much wear a thing has left, and what happens when it runs out. | custom_properties, signals | |
| `groups` | Characters who play together and share what they earn. | perception, preferences | |
| `kit_classes` | The classes an actor can take, and what each grants per level. | experience, signals | |
| `light_source` | Who carries light, where it reaches, and what keeping it burning costs. | custom_properties, signals | |
| `perception` | By which senses one perceiver reaches one subject. | custom_properties, position | |
| `remort` | Going round again: how many times, what was taken for it, and what that is worth. | custom_properties, static_registry | |
| `xaman_login` | Signing in with a Xaman wallet. | fcm_xrpl, evennia_archive | |
| `experience` | Experience points, levels, and what killing an actor pays. | custom_properties, groups, signals | |
| `messaging` | What each perceiver in a room is told, given what their senses reach. | directions, perception, preferences | |
| `position` | An actor's posture, what it is worth, and what it lets them do. | custom_properties, messaging, perception | |
| `size` | How big something is, and comparing one thing to another. | custom_properties, exit_types.traversal, signals | |
| `skills` | What an actor has learned, the points to learn more, and the trainers who teach it. | custom_properties, mastery, signals | |
| `languages` | What is spoken, who follows it, and what it sounds like to whoever does not. | custom_properties, perception, signals, static_registry | |
| `core_actor_stats` | The numbers every actor has, and the rebuild that keeps them true. | custom_properties, mastery, signals, skills, weapons | |
| `races` | The races an actor can be, and what each grants. | core_actor_stats, custom_properties, languages, signals, size, skills | |
| `combat` | Who is fighting whom, and the Circle-style round that resolves it. | dice, directions, exit_types, position, preferences, signals, weapons | |
| `item_restrictions` | Whether an actor is permitted to use an item. | alignment, core_actor_stats, custom_properties, kit_classes, races, size, skills | |
| `weapons` | What a weapon is, what mastery of it buys, and what it does in an attack. | custom_properties, damage_type, dice, kit_classes, mastery, signals, size, skills | |
| `death` | What follows an actor dying: the sequence, the corpse, looting it, and purgatory. | busy, custom_properties, experience, messaging, perception, position, room_types, signals | |
| `spells` | What a spell is, which exist, what an actor knows and has memorised, and casting. | busy, combat, custom_properties, dice, kit_classes, mastery, messaging, perception, position, signals, skills, static_registry, weapons | |
| `room_types` | Room mixins with no feature component of their own. A mixin tied to a component's feature lives in that component. | not declared | |
| `exit_types` | Exit mixins with no feature component of their own. A mixin tied to a component's feature lives in that component. | not declared | |

## Unfinished

| Component | What it does | Dependencies | Docs |
|---|---|---|---|
| `consumable` | A thing used up in the using. No README yet. | custom_properties | |
| `crafting` | What can be made, who knows how, and where it is made. No README yet. | custom_properties, mastery, signals, skills, static_registry | |
| `enchanting` | No description yet; the README is empty. | alignment, kit_classes, mastery, races, static_registry | |
| `quest` | The quest engine: state, step dispatch and the consumer hooks. The README is empty. | — | |
| `prompt` | The status line a player sees, built from tokens they choose. The README is a placeholder. | — | |
| `parsers` | Turning what a player typed into the thing they meant. No code yet. | — | |
| `economy` | An empty folder. | — | |
