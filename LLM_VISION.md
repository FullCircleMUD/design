# LLM Vision — Making the World Come Alive

> This document is the narrative entry point for how FullCircleMUD uses LLMs to create a living, reactive world. It describes the **why** and stitches the three memory systems into a single picture. For the authoritative specs of each system, follow the cross-references — this doc does not restate them.
>
> **Sources of truth:**
> - [LORE_MEMORY.md](LORE_MEMORY.md) — world knowledge, scope tags, retrieval, authoring workflow
> - [COMBAT_AI_MEMORY.md](COMBAT_AI_MEMORY.md) — combat memory, strategy bot, tactical plans, post-combat logging
> - [NPC_MOB_ARCHITECTURE.md](NPC_MOB_ARCHITECTURE.md) — composition hierarchy, AI tiers, Three Memory Systems section
> - [DATABASE.md](DATABASE.md) § pgvector for AI Memory — `NpcMemory` (dialogue) schema and dual-backend embedding storage
> - [WORLD.md](WORLD.md) — the world lore the memory systems surface at runtime
> - `src/game/CLAUDE.md` § LLM NPC system — implementation pointers

---

## Table of Contents

- [The Vision](#the-vision)
- [Design Principles](#design-principles)
- [The Three Memory Systems](#the-three-memory-systems)
- [How Emergence Happens](#how-emergence-happens)
- [What This Replaces](#what-this-replaces)
- [Tier Gating — Who Gets What](#tier-gating--who-gets-what)
- [Latency and Cost Shape the Design](#latency-and-cost-shape-the-design)
- [Authoring Model](#authoring-model)
- [Testing and Validation](#testing-and-validation)
- [Current Status](#current-status)
- [Where Each Piece Is Specified](#where-each-piece-is-specified)

---

## The Vision

A traditional MUD is a static text world: rooms describe themselves with a paragraph, NPCs read canned responses from a dialogue tree, mobs pick from a handful of scripted tactics. Players learn the seams quickly — every bartender gives the same answer, every boss fights the same way, every quest unfolds on rails.

FCM's LLM layer is a deliberate attempt to push past that. The goal is a world where:

- **NPCs know what they plausibly would know.** A Millholm bartender can gossip about continental history and local rumours without having those topics hand-authored for *that specific NPC*. A Mages Guild archmage knows more, because his scope tags let him reach lore a bartender never could. Same lore, different access.
- **NPCs remember you.** Returning to Bron the smith after a week feels different than meeting him for the first time. He remembers what you bought, what you asked about, whether you were rude. The continuity is not scripted — it is retrieved from his interaction memory and injected into his prompt.
- **Bosses adapt.** A gnoll warlord that has lost three times to a warrior+cleric party does not repeat the same opening. He pre-buffs, silences the cleric, focuses damage on the healer. The party that always attacks gets a harder fight every time; the party that tries diplomacy first opens a different branch entirely.
- **Gameplay emerges from history, not from scripts.** A boss's disposition toward a specific party is not a flag on the mob — it is the balance of their interaction and combat records. Players change the world by how they act in it.

This is not an AI gimmick layered on top of the game. It is a first-class design pillar: the world is built to *learn* about the people inside it.

---

## Design Principles

1. **Semantic retrieval, not hard-coded dialogue.** NPCs are not given a list of topics they can discuss. They are given a personality, a scope, and access to a shared knowledge base. The LLM retrieves what is relevant and the personality shapes delivery.
2. **Shared world, individual relationships.** Lore is authored once and reaches every NPC whose scope qualifies. Interaction memory is per-NPC — *your* history with Rowan lives on Rowan. Combat memory is per *mob type* by default (all gnolls learn together) and per *named instance* for bosses (Varkoth remembers his own fights).
3. **Emergence from composition, not branching.** No dialogue trees, no quest flowcharts. A boss's reaction to an approaching party is the output of combining three memory systems against the current context. The same boss is warm to one party, hostile to another, and wary of a third — not because each path was scripted, but because each party has a different history with him.
4. **Latency never blocks gameplay.** Expensive LLM reasoning happens during pre-fight approach time (strategy bot) or out-of-band after combat ends (post-combat summariser). In-combat ticks are pure state machine — the tactical plan is already sitting on `ndb` by the time swords come out. See [COMBAT_AI_MEMORY.md § Latency Budget](COMBAT_AI_MEMORY.md#latency-budget).
5. **Memory survives game DB wipes.** All three memory systems live in the `ai_memory` database behind a dedicated router, independent of the main game DB. Wiping and rebuilding the game does not erase what NPCs have learned.
6. **Structured data for future ML.** Every combat memory row is a labelled training example (party composition, tactics, outcome, HP margin). After enough encounters accumulate, a lightweight classifier or fine-tuned model can replace the general-purpose strategy LLM. See [COMBAT_AI_MEMORY.md § Future: ML Training Data](COMBAT_AI_MEMORY.md#future-ml-training-data).

---

## The Three Memory Systems

FCM composes three independent memory systems at prompt-build time. Each answers a different question, and each is specified in its own design document.

| System | Question it answers | Scope | Primary doc |
|---|---|---|---|
| **Lore memory** | *"What do I know about the world?"* | Shared, filtered per NPC by scope tags | [LORE_MEMORY.md](LORE_MEMORY.md) |
| **Interaction memory** (`NpcMemory`) | *"What do I know about **you**?"* | Per-NPC, per-player | [DATABASE.md § pgvector](DATABASE.md) |
| **Combat memory** (`CombatMemory`) | *"What do I know about **fighting** you?"* | Per mob type or per named boss | [COMBAT_AI_MEMORY.md](COMBAT_AI_MEMORY.md) |

All three use the same `ai_memory` Django app, the same dual-backend embedding storage (pgvector on Postgres, numpy on SQLite), and the same router. They are queried independently — combat memory never pollutes dialogue search and vice versa. For the composition diagram showing how all three are layered on a single NPC, see [NPC_MOB_ARCHITECTURE.md § Three Memory Systems](NPC_MOB_ARCHITECTURE.md#three-memory-systems).

### Why three, not one

Mixing the three into a single memory table would destroy semantic retrieval. A player asking a bartender about "the war" should retrieve lore about historical conflicts, not the time someone brawled in the tavern. A strategy bot planning against a gnoll warlord should retrieve past fights with similar parties, not small-talk from a previous visit. The three are semantically distinct, scoped differently, and updated at different points in the game loop — so they live in three tables and are queried independently.

They only unify at prompt-build time, when the LLM mixin assembles the context for a specific moment.

---

## How Emergence Happens

Emergence is not a feature; it is what falls out of the composition. Two concrete examples (drawn from the source docs — see the cross-references for implementation detail).

### Example 1 — Consistent world knowledge, different delivery

A player asks two different NPCs how Millholm was founded.

- **Rowan (bartender, `millholm_town`)** — lore retrieval returns the continental founding entry. His personality template delivers it as tavern gossip: *"Oh, Millholm's been here nigh on four hundred years! My gran used to say..."*
- **Archmage Tindel (`millholm_town`, `mages_guild`)** — lore retrieval returns the same founding entry **plus** a faction-scoped entry about the Mages Guild chapter's arrival 200 years later. His personality template delivers it as a lecture: *"The settlement was established approximately four centuries ago by eastern colonists. The Mages Guild established its Millholm chapter two centuries later..."*

Same lore. Different scope. Different personality. Different delivery. Zero per-NPC knowledge authoring. (Full walkthrough: [LORE_MEMORY.md § Examples](LORE_MEMORY.md#examples).)

### Example 2 — A boss's behaviour shaped by history

A party approaches the gnoll warlord. The strategy bot fires during their approach and produces a **unified briefing** from both memory systems:

- **Interaction memory** — "This party tried to negotiate last time, offered food."
- **Combat memory** — "Lost to warrior+cleric composition × 3; cleric healing was the main problem."
- **Lore memory** — "Gnoll tribes have held the southern grasslands for centuries..."

The briefing produced: *"This party has been both peaceful and hostile. Speak first — they may negotiate. If combat: silence the cleric immediately, focus melee."*

The same warlord, facing a different party with a pure-hostile history, produces a different briefing: *"Pre-cast shield, silence cleric on sight, bash warrior once cleric drops."*

The disposition is not a flag. It is the output of reading the history. A party that always attacks gets a harder, more optimised fight each time. A party that tries diplomacy first may eventually reach a wholly different outcome — trade, information, quest hooks — without any branching dialogue tree existing in code. (Full walkthrough: [COMBAT_AI_MEMORY.md § Unified Briefing](COMBAT_AI_MEMORY.md#unified-briefing) and [§ Behavioral Spectrum](COMBAT_AI_MEMORY.md#behavioral-spectrum).)

---

## What This Replaces

| Traditional approach | FCM's approach |
|---|---|
| Per-NPC `knowledge` string hand-authored for every NPC | One shared lore base; NPCs filter by scope tags |
| Dialogue trees branching on flags | LLM prompt with retrieved lore + memories + personality |
| Mob tactics as a priority list in code | Strategy bot synthesises a tactical plan from past encounters |
| Reputation as a single integer | Disposition emerges from the balance of interaction + combat records |
| "Boss mechanics" hand-designed per boss | Tactics learned per fight and applied to the next one |

Where traditional systems still make sense, FCM keeps them. Tier 1 commodity mobs (rats, wolves) are pure state machines with no memory. Shopkeepers are scripted where scripting is the right tool. The LLM layer is applied where it earns its cost — see [Tier Gating](#tier-gating--who-gets-what).

---

## Tier Gating — Who Gets What

Not every mob needs memory. Not every NPC needs an LLM prompt. FCM defines four intelligence tiers (specified in full at [NPC_MOB_ARCHITECTURE.md § NPC Intelligence Tiers](NPC_MOB_ARCHITECTURE.md#npc-intelligence-tiers)):

| Tier | Example | Lore memory | Interaction memory | Combat memory | Strategy bot |
|---|---|---|---|---|---|
| **1. Commodity** | rat, wolf, kobold | — | — | — | — |
| **2. Generic named-role** | townsperson, guard | Yes | Short-term (rolling) | — | — |
| **3. Named individual** | Rowan, Bron | Yes | Long-term (`NpcMemory`) | — | — |
| **4. Named combatant** | gnoll warlord, dragon | Yes | Long-term | Yes | Yes |

The gating keeps cost and latency in check. Commodity mobs run at full state-machine speed with no LLM in the loop. Tier 4 bosses get the full stack — three memory systems, pre-fight strategy synthesis, unified briefing — because the encounter is rare, high-stakes, and worth the tokens.

---

## Latency and Cost Shape the Design

Two constraints drive the architectural choices:

**Latency:** A player waiting 3 seconds for a mob to pick its next attack breaks the game. The design resolves this by moving LLM calls off the critical path:

- **Dialogue** — the player is already waiting for a response, so a 1-2 second LLM call is acceptable.
- **Pre-fight strategy** — fires when the player enters an approach room (1-2 rooms from the boss). Traversal time absorbs the LLM latency; by the time combat starts, `ndb.tactical_plan` is already populated.
- **Post-combat summarisation** — async, non-blocking, after `stop_combat()` fires.
- **In-combat ticks** — zero LLM calls. State machine reads the pre-computed tactical plan and executes.

Full detail: [COMBAT_AI_MEMORY.md § Early Warning Pattern](COMBAT_AI_MEMORY.md#early-warning-pattern) and [§ Latency Budget](COMBAT_AI_MEMORY.md#latency-budget).

**Cost:** One strategy LLM call per fight is affordable. One call per combat tick would not be. The design makes this trade explicit — reasoning happens once, execution is cheap and repeatable.

---

## Authoring Model

Lore is not written into code. It lives in a separate repository (`FCM/lore/`) as YAML files organised by scope — continental, regional, local, faction. A standalone importer runs in a Railway service with access to the game's `ai_memory` database, embedding new entries on push. Adding new lore never requires a game restart or code change.

Full authoring workflow: [LORE_MEMORY.md § Lore Authoring Workflow](LORE_MEMORY.md#lore-authoring-workflow).

NPC personality templates live alongside code in `src/game/llm/prompts/`. They accept `{lore_context}`, `{memories}`, `{quest_context}`, and other variables — they are *framing*, not content. The same template works for any NPC; the scope tags and memory queries fill in what that specific NPC knows.

---

## Testing and Validation

LLM behaviour is non-deterministic, so FCM has a dedicated testing harness (`FCM/llm-test-harness/`) that drives virtual clients against a running server, scripts conversations, detects loops, and compares outputs across model versions. See the harness README for bot configuration format and usage.

Validation goals:

- Does a scope-gated question (ley lines to a bartender) correctly fail over with a deflection rather than leaking guild lore?
- Does a returning player get recognised by name and topic from their last visit?
- Does the strategy bot produce meaningfully different plans for different party compositions against the same boss?

The harness exists because these are gameplay concerns, not code-correctness concerns. Unit tests cannot catch "the archmage feels wrong."

---

## Current Status

As of 2026-04-23, the picture across the three systems is:

| System | Model & storage | Retrieval & integration | Emergent behaviour |
|---|---|---|---|
| **Lore memory** | Built | Built | Live — NPCs retrieve scope-filtered lore at prompt time |
| **Interaction memory** | Built | Built | Live — NPCs recall past conversations with returning players |
| **Combat memory** | Model built, migrated | **Strategy bot not yet built; post-combat summariser not yet built; early-warning triggers not yet built** | Not yet wired end-to-end |

The infrastructure (tables, embeddings, routers, dual-backend storage) is done. The remaining work is the combat-side agent scaffolding — the strategy bot service, the post-combat hook, the approach-room triggers, and the state-machine plumbing that reads `ndb.tactical_plan`. These are tracked in [ops/PLANNING/2_IN_PROGRESS](../ops/PLANNING/2_IN_PROGRESS) under the "AI Mob System — Memory, Learning, and Emergent Behaviour" epic.

For per-component implementation status tables, see:
- [LORE_MEMORY.md § Implementation Status](LORE_MEMORY.md#implementation-status)
- [COMBAT_AI_MEMORY.md § Implementation Dependencies](COMBAT_AI_MEMORY.md#implementation-dependencies)

---

## Where Each Piece Is Specified

This doc is the narrative summary. The authoritative specifications live in:

| Topic | Document | Section |
|---|---|---|
| Lore memory schema, scope tags, AND semantics | [LORE_MEMORY.md](LORE_MEMORY.md) | §§ Lore Scoping, LoreMemory Schema |
| Lore retrieval and prompt integration | [LORE_MEMORY.md](LORE_MEMORY.md) | § How Lore Reaches the NPC |
| Lore authoring (YAML repo, importer, Railway deploy) | [LORE_MEMORY.md](LORE_MEMORY.md) | § Lore Authoring Workflow |
| NPC scope resolution from Evennia tags | [LORE_MEMORY.md](LORE_MEMORY.md) | § NPC Scope Resolution |
| Interaction memory (`NpcMemory`) schema | [DATABASE.md](DATABASE.md) | § pgvector for AI Memory |
| Interaction memory prompt integration | `src/game/CLAUDE.md` | § LLM NPC system |
| Combat memory schema and summary format | [COMBAT_AI_MEMORY.md](COMBAT_AI_MEMORY.md) | §§ Combat Memory Schema |
| Three-layer architecture (persistence → strategy → execution) | [COMBAT_AI_MEMORY.md](COMBAT_AI_MEMORY.md) | § Three-Layer Architecture |
| Pre-fight timing (approach room triggers, async LLM call) | [COMBAT_AI_MEMORY.md](COMBAT_AI_MEMORY.md) | § Early Warning Pattern |
| Unified briefing (interaction + combat memory merged) | [COMBAT_AI_MEMORY.md](COMBAT_AI_MEMORY.md) | § Unified Briefing |
| Per-type vs per-instance memory scope | [COMBAT_AI_MEMORY.md](COMBAT_AI_MEMORY.md) | § Memory Scope |
| AI tier taxonomy (which mobs get which memories) | [NPC_MOB_ARCHITECTURE.md](NPC_MOB_ARCHITECTURE.md) | § NPC Intelligence Tiers |
| Three memory systems composed on one NPC | [NPC_MOB_ARCHITECTURE.md](NPC_MOB_ARCHITECTURE.md) | § Three Memory Systems |
| Hybrid combat+dialogue mobs (`LLMCombatMob`) | [NPC_MOB_ARCHITECTURE.md](NPC_MOB_ARCHITECTURE.md) | § Hybrid combat+dialogue pattern |
| World lore source material (the content that gets embedded) | [WORLD.md](WORLD.md) | (entire document) |
| LLM testing harness | `llm-test-harness/README.md` | (entire document) |
| In-progress combat agent work | [ops/PLANNING/2_IN_PROGRESS](../ops/PLANNING/2_IN_PROGRESS) | AI Mob System epic |
