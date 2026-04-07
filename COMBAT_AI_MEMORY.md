# Combat AI Memory & Strategy System

> For NPC/mob class hierarchy and AI tiers, see [NPC_MOB_ARCHITECTURE.md](NPC_MOB_ARCHITECTURE.md). For the interaction memory system (NPC dialogue), see [DATABASE.md](DATABASE.md) § pgvector for AI Memory. For embedded world knowledge, see [LORE_MEMORY.md](LORE_MEMORY.md). For how all three memory systems compose, see [NPC_MOB_ARCHITECTURE.md](NPC_MOB_ARCHITECTURE.md) § Three Memory Systems. For combat mechanics and attack resolution, see [COMBAT_SYSTEM.md](COMBAT_SYSTEM.md). For the existing `ai_memory` app and dual-backend embedding storage, see `src/game/CLAUDE.md` § LLM NPC system.

---

## Table of Contents

- [Overview](#overview)
- [Two Memory Systems, Two Purposes](#two-memory-systems-two-purposes)
- [Combat Memory Schema](#combat-memory-schema)
- [Three-Layer Architecture](#three-layer-architecture)
  - [Layer 1: Combat Memory (Persistence)](#layer-1-combat-memory-persistence)
  - [Layer 2: Strategy Bot (Pre-Fight Planning)](#layer-2-strategy-bot-pre-fight-planning)
  - [Layer 3: Execution (In-Fight)](#layer-3-execution-in-fight)
- [Early Warning Pattern](#early-warning-pattern)
- [Unified Briefing](#unified-briefing)
- [Memory Scope: Per-Type vs Per-Instance](#memory-scope-per-type-vs-per-instance)
- [AI Tier Mapping](#ai-tier-mapping)
- [Behavioral Spectrum](#behavioral-spectrum)
- [Latency Budget](#latency-budget)
- [Future: ML Training Data](#future-ml-training-data)

---

## Overview

Mobs with persistent combat memory learn from past encounters. After every fight, the mob records a structured summary: who they fought, what tactics they used, what the enemy did, and the outcome. Before a new fight, a **strategy bot** searches this memory for similar encounters and synthesises a tactical plan. The mob then executes the plan via its existing AI layer (state machine or lightweight LLM).

This creates emergent gameplay — mobs that fought a warrior+cleric party three times and lost to sustained healing will prioritise silencing the cleric next time. Bosses that remember a player's peaceful visits will greet them differently than a player who attacks on sight.

The system is built on the same `ai_memory` infrastructure as NPC dialogue memory (pgvector on PostgreSQL, numpy on SQLite), but uses a separate table with a schema tuned for tactical data.

---

## Two Memory Systems, Two Purposes

| | Interaction Memory (`NpcMemory`) | Combat Memory (`CombatMemory`) |
|---|---|---|
| **What it stores** | Conversation exchanges (player said X, NPC replied Y) | Encounter summaries (party comp, tactics, outcome) |
| **Embedding source** | Natural language conversation summary | Structured tactical narrative |
| **Query pattern** | "What have we talked about?" | "What worked against this party composition?" |
| **Who queries it** | LLMMixin (dialogue prompt building) | Strategy bot (pre-fight planning) |
| **Unit of record** | One conversational exchange | One completed encounter |
| **Volume per mob** | Low (a few exchanges per player visit) | Moderate (one per fight, shared across instances) |
| **Latency tolerance** | Seconds (player waits for speech response) | Seconds (pre-fight, absorbed by approach time) |

Both tables live in the `ai_memory` database (same router, same pgvector infrastructure, survives game DB wipes). They are queried independently — combat memory never pollutes dialogue search and vice versa.

---

## Combat Memory Schema

```python
class CombatMemory(models.Model):
    """A single completed combat encounter, recorded for tactical learning."""

    # ── Who fought ──
    mob_type = models.CharField(max_length=80, db_index=True)   # e.g. "gnoll_warlord"
    mob_level = models.IntegerField()
    mob_name = models.CharField(max_length=80)                   # display name (fallback)

    # ── Party composition (structured for filtering + ML) ──
    party_composition = models.JSONField()
    # Example: [{"name": "Bob", "class": "warrior", "level": 10},
    #           {"name": "Alice", "class": "cleric", "level": 9}]
    party_size = models.IntegerField(db_index=True)              # quick pre-filter

    # ── What happened ──
    mob_tactics = models.TextField()        # what the mob did
    enemy_tactics = models.TextField()      # what the party did
    rounds_survived = models.IntegerField()

    # ── Outcome ──
    outcome = models.CharField(max_length=20)  # "win", "loss", "fled", "draw"
    hp_remaining_pct = models.FloatField()     # 0.0 = dead, 0.95 = easy win
    # For wins: how decisive. For losses: how close was it.

    # ── Embedding (dual-backend, same as NpcMemory) ──
    summary = models.TextField()                                    # embedding source
    embedding = models.BinaryField(null=True, blank=True)           # SQLite path
    embedding_vector = VectorField(dimensions=1536, null=True, blank=True)  # Postgres path

    # ── Metadata ──
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        app_label = "ai_memory"
        indexes = [
            models.Index(fields=["mob_type", "created_at"]),
            models.Index(fields=["mob_type", "party_size"]),
            models.Index(fields=["mob_type", "outcome"]),
        ]
```

**Design decisions:**

- **`mob_type` not `mob_id`** — mobs are deleted on death. The type string is the durable identity that lets all gnoll warlords share learnings (see [Memory Scope](#memory-scope-per-type-vs-per-instance)).
- **`party_composition` as JSON** — structured data for future ML training. The strategy bot can also use this for pre-filtering before vector search ("find fights against 2-3 player parties").
- **`hp_remaining_pct`** — captures fight margin. A win at 95% HP means the tactic was dominant. A win at 5% HP means it barely worked. The strategy bot can weight recommendations by decisiveness.
- **Separate `mob_tactics` and `enemy_tactics`** — the strategy bot needs both: what *we* did (to recommend or avoid repeating) and what *they* did (to anticipate and counter).
- **`summary`** — the full narrative embedding source, combining all fields into a natural language paragraph that embeds well for semantic search.

**Summary format** (what gets embedded):

> "Gnoll Warlord (L8) fought a party of 3: L10 warrior, L9 cleric, L7 mage. 8 rounds. Mob tactics: focused cleric with melee attacks rounds 1-4, switched to mage round 5 after cleric died. Enemy tactics: warrior used bash x3, cleric cast cure wounds x4, mage cast fireball x2. Result: win at 35% HP. Learning: focusing the cleric worked — healing output was the main threat. Fireball from the mage was painful but survivable."

---

## Three-Layer Architecture

### Layer 1: Combat Memory (Persistence)

The `CombatMemory` table. Written once at the end of every encounter. Read by the strategy bot at the start of new encounters.

**When it's written:** After `stop_combat()` fires on the mob's combat handler. A post-combat hook collects:
- Party composition from `get_sides()` (enemy list with classes and levels)
- Combat log data (actions taken per round, damage dealt/received, spells cast)
- Outcome (mob HP at end, did mob die, did enemies flee)

**What generates the summary:** A dedicated summariser — could be a simple template-based formatter initially, upgraded to an LLM summariser later for richer "learning" observations.

**Embedding:** Same dual-backend as `NpcMemory` — pgvector `VectorField` on Postgres, numpy `BinaryField` on SQLite. Same `_is_postgres()` detection. Same HNSW index for cosine search.

### Layer 2: Strategy Bot (Pre-Fight Planning)

A single LLM call that synthesises a tactical plan from memory. Fires **before combat starts** (see [Early Warning Pattern](#early-warning-pattern)).

**Input:**
1. Current party composition (who's approaching, classes, levels, visible equipment)
2. Top N similar encounters from `CombatMemory` (via vector search on party comp summary)
3. Interaction history from `NpcMemory` (if the mob is also an LLM dialogue NPC — e.g. a boss)
4. The mob's own capabilities (available commands, spells, abilities)

**Output:** A short tactical briefing stored on `ndb` (in-memory, per-fight):

```
DISPOSITION: hostile (attacked 4 previous times, never peaceful)
PRIMARY TARGET: cleric (healing sustained warrior in last 3 fights)
OPENING MOVE: silence cleric, then melee focus
SECONDARY TARGET: mage (high damage, low HP)
FLEE THRESHOLD: 15% HP
PRE-BUFF: cast shield before engagement
```

**Model choice:** Can use a more capable model (Sonnet, GPT-4o) since this is one call per fight, not per tick. The cost is justified by the quality of tactical reasoning.

### Layer 3: Execution (In-Fight)

The mob AI that runs every combat tick. Reads `ndb.tactical_plan` and executes accordingly.

**For most mobs: state machine.** The tactical plan maps directly to state machine parameters:
- Target priority list → mob targets highest-priority living enemy
- Action preferences → weighted action selection (bash, cast, attack)
- Flee threshold → simple HP percentage check
- Conditional triggers → "if cleric is casting, interrupt" as event checks

The existing `StateMachineAIMixin` already has the `ai_fight()` tick. It would be extended to consult `ndb.tactical_plan` when present, falling back to default behavior when absent.

**For elite bosses (optional): lightweight LLM.** A fast, cheap model (Haiku, GPT-4o-mini) with minimal context — just the tactical plan + current fight state. Returns a single command string. Small context = fast response. But the state machine approach is preferred for predictability and zero latency.

**Key principle:** The execution layer doesn't *think* about strategy. It follows instructions. All the reasoning happens once in the strategy bot.

---

## Early Warning Pattern

The strategy bot fires **before the player reaches the boss**, not when combat starts. This eliminates LLM latency entirely.

```
Player enters approach room (1-2 rooms from boss)
  → Boss room detects incoming party
  → Strategy bot fires immediately (async, non-blocking)
  → 5-15 seconds of real traversal time absorbs LLM latency

Player enters boss room
  → ndb.tactical_plan already populated
  → Boss reacts INSTANTLY
  → No visible delay, no "thinking" emote needed
```

**Trigger mechanisms:**

- **Dungeon bosses:** The dungeon template knows its layout. The boss room can tag approach rooms at build time. Player entering a tagged room fires `at_object_receive()` which notifies the boss.
- **Open-world bosses:** The boss registers interest in adjacent rooms (1-2 rooms away). `AggressiveMixin` already scans nearby rooms via `ai_wander()` — same concept, triggering strategy prep instead of (or before) aggro.
- **Shared dungeons:** For `"shared"` instance mode, the boss may need to re-evaluate when new players join the instance mid-fight. This would be a re-query of the strategy bot, but only on significant party composition changes.

**Fallback:** If the strategy bot hasn't returned by the time combat starts (player sprinted through, LLM was slow), the mob uses default state machine behavior for the first few ticks, then switches to the tactical plan when it arrives.

---

## Unified Briefing

For bosses that are also dialogue NPCs (LLMRoleplayNPC + CombatMixin), the strategy bot queries **both** memory systems and produces a unified briefing:

| Memory Source | What it provides |
|---|---|
| **NpcMemory** (interaction) | Past conversations, relationship history, disposition, topics of interest |
| **CombatMemory** (combat) | Past fights, effective tactics, party composition patterns |
| **Current context** | Party composition now, levels, equipment, HP state, active buffs |

This enables bosses that behave differently based on the full history of the relationship, not just combat history.

**Example briefings from the same boss:**

*First visit, unknown player:*
> Unknown party approaching. L8 warrior, L6 mage. No prior encounters. No conversation history. Recommend: observe, speak first, gauge intent. If combat: standard aggression, target mage.

*Returning friend:*
> Bob has visited 3 times, always peaceful. Last conversation: interested in the history of the ruins. Has never attacked. Recommend: greet warmly, continue ruins discussion. If provoked: warn twice before engaging.

*Known hostile:*
> Bob's party has attacked 4 times. Composition matches previous (warrior+cleric). Lost 3 of 4 fights. Cleric healing sustained the warrior in every loss. Most effective tactic from the 1 win: silenced cleric round 1, focused with melee. Recommend: pre-cast shield, silence cleric on sight, bash warrior once cleric drops.

---

## Memory Scope: Per-Type vs Per-Instance

| Scope | How it works | Use case |
|---|---|---|
| **Per-type** (default) | All instances of `mob_type="gnoll_warlord"` share memories. New spawns inherit collective knowledge. | Commodity and mid-tier mobs. Gnolls get smarter as a species. |
| **Per-instance** | A unique `mob_name` or tag identifies a specific boss that persists across respawns. Queries filter by name. | Named bosses, story NPCs. "Varkoth the Black Dragon" remembers *its own* fights. |
| **Hybrid** | Instance memories first, fall back to type memories if insufficient data. | New named boss with no personal history yet draws on what other bosses of the same type learned. |

The schema supports all three via the `mob_type` and `mob_name` fields. Query strategy is a configuration choice per mob, not a schema difference.

---

## AI Tier Mapping

Combat memory and the strategy bot are exclusively **Tier 4 (Named Combatant)** features. See [NPC_MOB_ARCHITECTURE.md](NPC_MOB_ARCHITECTURE.md) § NPC Intelligence Tiers for the full four-tier taxonomy covering all NPC types.

| Tier | Combat Memory | Strategy Bot | Notes |
|---|---|---|---|
| **1. Commodity** (rat, wolf, kobold) | None | None | Pure state machine. No learning. |
| **2. Generic Named-Role** (townsperson, guard) | None | None | State machine if fightable. No tactical memory. |
| **3. Named Individual** (Rowan, Bron) | None | None | Non-combat. Uses lore + interaction memory only. |
| **4. Named Combatant** (gnoll warlord, dragon) | Full | Pre-fight + post-fight agents | All three memory systems + strategy bot. |

Tiers 1–3 do not record or learn from combat. A town guard who gets attacked fights via state machine and forgets the encounter. A named boss learns from every fight and adapts.

---

## Behavioral Spectrum

The unified briefing enables a continuous spectrum from peaceful to hostile, determined by history:

```
Always peaceful     → Greet, offer dialogue, share knowledge
Mostly peaceful     → Greet warmly, but prepared (no pre-buffs)
Mixed history       → Cautious, speak first but ready to fight
Mostly hostile      → Terse warning, pre-buffed, attack if provoked
Always hostile      → Pre-buffed, attack on sight, optimised tactics
```

The disposition isn't a single flag — it emerges naturally from the balance of interaction and combat memories. A boss that's been fought 10 times but talked to 20 times has a "mostly peaceful" disposition. The strategy bot reads the full history and makes a judgement call.

This means players can **change a boss's behavior** through repeated peaceful interaction. A party that always attacks the gnoll warlord gets a harder, more optimised fight each time. A party that tries diplomacy first might eventually earn a different outcome entirely — trade, information, quest hooks. Emergent gameplay from memory, not scripted branching.

---

## Latency Budget

| Phase | When | Duration | What happens |
|---|---|---|---|
| **Early warning** | Player enters approach room | 5-15 seconds (real traversal time) | Strategy bot fires, queries memory, calls LLM |
| **Strategy synthesis** | Async during approach | 1-3 seconds (LLM call) | Tactical plan generated, stored on `ndb` |
| **Combat start** | Player enters boss room / attacks | 0 seconds | Plan already on `ndb`, mob reacts instantly |
| **Each combat tick** | Every few seconds | 0 seconds (state machine) | Reads plan, picks action, executes command |
| **Post-combat** | After `stop_combat()` | 1-2 seconds (async) | Summarise encounter, embed, store to `CombatMemory` |

Total LLM calls per fight: **1** (strategy synthesis). Zero per tick. Post-combat embedding is async and non-blocking.

If the strategy bot call is still in-flight when combat starts, the mob falls back to default state machine behavior and switches to the tactical plan when it arrives. The player never sees a delay.

---

## Future: ML Training Data

Every `CombatMemory` row is a labelled training example:

- **Features:** party composition, party size, mob type, mob level, mob tactics, enemy tactics
- **Label:** outcome (win/loss/fled), rounds survived, HP remaining percentage

After the game accumulates thousands of encounters, this dataset enables:
- Training a lightweight classifier: "given this party composition, what's the win probability for each tactic?"
- Fine-tuning a small model specifically for combat strategy (replaces the general-purpose LLM strategy bot)
- Balancing insights: which mob types are too easy/hard, which tactics are dominant

The structured JSON fields (`party_composition`, `outcome`, `hp_remaining_pct`, `rounds_survived`) are specifically designed to be ML-ready without needing to parse natural language. The `summary` text field and embedding are for the semantic search path; the structured fields are for the future ML path. Both coexist without compromise.

---

## Implementation Dependencies

This system builds on existing infrastructure:

| Dependency | Status | Notes |
|---|---|---|
| `ai_memory` Django app | Built | Database, router, migrations |
| pgvector dual-backend | Built | `VectorField` + `BinaryField`, `_is_postgres()` branching |
| `CombatMixin` | Built | `enter_combat()`, `stop_combat()`, combat handler lifecycle |
| `StateMachineAIMixin` | Built | `ai_fight()` tick method, state dispatch |
| `get_sides()` | Built | Party composition detection (allies/enemies) |
| Combat handler | Built | Round tracking, action logging |
| `LLMService` | Built | Chat completion + embedding API |
| `CombatMemory` model | Not yet built | New model in `ai_memory` app |
| Strategy bot service | Not yet built | New service — memory query + LLM synthesis |
| Post-combat summariser | Not yet built | Hook on `stop_combat()` to log encounters |
| Early warning triggers | Not yet built | Approach room detection for pre-computation |
| Tactical plan execution | Not yet built | Extend `ai_fight()` to read `ndb.tactical_plan` |
