# Lore Memory — Embedded World Knowledge for NPCs

> For the NPC/mob AI architecture and how all three memory systems compose, see [NPC_MOB_ARCHITECTURE.md](NPC_MOB_ARCHITECTURE.md) § Three Memory Systems. For NPC interaction memory (dialogue), see [DATABASE.md](DATABASE.md) § pgvector for AI Memory. For combat AI memory and the strategy bot, see [COMBAT_AI_MEMORY.md](COMBAT_AI_MEMORY.md). For world lore source material, see [WORLD.md](WORLD.md).

---

## Table of Contents

- [Overview](#overview)
- [The Problem Lore Memory Solves](#the-problem-lore-memory-solves)
- [Lore Scoping](#lore-scoping)
- [LoreMemory Schema](#lorememory-schema)
- [How Lore Reaches the NPC](#how-lore-reaches-the-npc)
- [NPC Scope Resolution](#npc-scope-resolution)
- [Lore Authoring Workflow](#lore-authoring-workflow)
- [Three Memory Systems in Composition](#three-memory-systems-in-composition)
- [Examples](#examples)
- [Implementation Dependencies](#implementation-dependencies)

---

## Overview

Lore memory is a shared, embedded knowledge base that gives NPCs dynamic awareness of the world — its history, geography, politics, factions, and local events. Instead of manually writing world knowledge into each NPC's `llm_knowledge` attribute, lore is authored once, tagged with scope, embedded, and retrieved semantically at prompt-build time based on what the player is asking about.

Every NPC draws from the same lore base, filtered by what they would plausibly know. A bartender in Millholm knows local town gossip and continent-wide common knowledge. The Archmage knows that plus arcane history and Mages Guild secrets. A gnoll warlord knows gnoll tribal lore and regional territory. The lore is the same data — the NPC's scope tags determine which slices they can access.

This is the third memory system alongside interaction memory (`NpcMemory` — what an NPC remembers about conversations with you) and combat memory (`CombatMemory` — what a mob learned from past fights). Together they provide the full context for any NPC encounter.

---

## The Problem Lore Memory Solves

**Current state:** Each NPC's world knowledge is manually written in `llm_knowledge` (a string attribute) or baked into the prompt template. This has several problems:

- **Doesn't scale.** Every NPC needs individual knowledge authoring. Adding a new piece of lore means updating every NPC that should know it.
- **Drifts out of sync.** As the world evolves (new zones, new history, events), NPC knowledge becomes stale unless manually updated.
- **Inconsistent.** Two NPCs in the same town might give contradictory information because their knowledge strings were written at different times.
- **Shallow.** NPCs can only discuss topics explicitly written into their knowledge. A player asking about something the author didn't anticipate gets a vague non-answer.

**With lore memory:** Lore is authored once in the knowledge base with scope tags. When a player asks Rowan about the Shadow War, the system embeds the question, searches `LoreMemory` filtered by Rowan's scope tags (`millholm`, `continental`), and injects the relevant lore entries into the prompt. Rowan's personality template shapes the delivery (tavern gossip, not a history lecture), but the knowledge is dynamically retrieved.

Adding new lore (a new zone's history, a new faction) automatically becomes available to every NPC whose scope tags match. No per-NPC updates needed.

---

## Lore Scoping

Lore entries are tagged with one or more scope levels. An NPC's access is determined by their own scope tags — they can retrieve lore at their level and all broader levels above it.

| Scope Level | What it covers | Who knows it | Example |
|---|---|---|---|
| **Continental** | Major history, geography, creation myths, world-shaping events | Every NPC | The Shadow War, the five nations, the Sundering |
| **Regional** | Zone-level history, political structure, trade routes, regional threats | NPCs in that region/zone | Millholm's founding, the Deep Woods curse, the mine collapse |
| **Local** | District-level knowledge, notable residents, local customs, recent events | NPCs in that district | The Harvest Moon's best ale, the sewer entrance rumor, who runs the black market |
| **Faction** | Guild secrets, religious doctrine, arcane theory, underworld dealings | NPCs with matching faction tags | Mages Guild wards, Thieves Guild safe houses, Temple prophecies |

**Scope inheritance:** Broader scopes are always accessible. A Millholm bartender tagged `district:millholm_town` automatically gets `regional:millholm` and `continental` lore. A Mages Guild NPC tagged `faction:mages_guild` gets faction lore plus whatever geographic scopes they're in.

---

## LoreMemory Schema

```python
class LoreMemory(models.Model):
    """A single piece of world knowledge, embedded for semantic retrieval."""

    # ── Identity ──
    title = models.CharField(max_length=200)         # human-readable label
    content = models.TextField()                      # the actual lore text

    # ── Scope (what determines who can access this) ──
    scope_level = models.CharField(max_length=20)     # "continental", "regional", "local", "faction"
    scope_tags = models.JSONField(default=list)
    # Examples:
    #   continental: []  (no further scoping needed)
    #   regional:    ["millholm"]
    #   local:       ["millholm_town"]
    #   faction:     ["mages_guild"]
    #   multi-tag:   ["millholm", "mages_guild"]
    # Multi-tag uses AND semantics — see "Multi-tag scope semantics" below.

    # ── Embedding (dual-backend, same as NpcMemory / CombatMemory) ──
    embedding = models.BinaryField(null=True, blank=True)           # SQLite path
    embedding_vector = VectorField(dimensions=1536, null=True, blank=True)  # Postgres path

    # ── Metadata ──
    source = models.CharField(max_length=200, blank=True, default="")
    # Where this lore originated: "WORLD.md", "millholm/town.py", "manual", etc.
    # Useful for tracking what needs re-embedding when source docs change.

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        app_label = "ai_memory"
        indexes = [
            models.Index(fields=["scope_level"]),
            models.Index(fields=["scope_level", "created_at"]),
        ]
```

**Design decisions:**

- **Same `ai_memory` app and database** — shares the pgvector infrastructure, router, and dual-backend pattern. One migration adds the table.
- **`scope_tags` as JSON list** — flexible multi-tagging without a join table. Access uses AND semantics (see below).
- **`source` field** — tracks where the lore came from. When `WORLD.md` is updated, you can find and re-embed all lore entries sourced from it.
- **`title`** — human-readable label for admin/debugging. Not embedded or searched.
- **No NPC ID or speaker ID** — lore is shared, not per-NPC. The NPC's scope tags determine access at query time.

### Multi-tag scope semantics (AND, not OR)

A lore entry with multiple `scope_tags` requires the NPC to have **all** of those tags before they can access the entry. This is the intersection, not the union.

Example: an entry tagged `["millholm", "mages_guild"]` is accessible **only** to NPCs that are *both* in Millholm *and* members of the Mages Guild. A Millholm bartender (no guild tag) cannot see it. A mage in another city (no Millholm tag) cannot see it. Only an NPC carrying both tags qualifies.

This is intentional — it prevents a thief in another city from knowing Millholm-specific guild secrets, and prevents a Millholm townsperson from knowing guild secrets. If you want a piece of lore to reach two distinct audiences, author two entries (one per scope) rather than combining tags.

Implementation: `_npc_can_access_lore()` in `ai_memory/services.py` performs the AND check after the database pre-filter.

---

## How Lore Reaches the NPC

At prompt-build time, when `_get_context_variables()` runs in `LLMMixin`:

```
1. Player says something to the NPC
2. LLMMixin builds the prompt:
   a. Embed the player's message
   b. Determine the NPC's lore scope tags (from attributes/tags)
   c. Query LoreMemory:
      - Filter by scope: continental entries + entries matching NPC's tags
      - Semantic search: embed player's message, find top N relevant entries
   d. Format results as {lore_context} for the prompt template
3. Prompt sent to LLM with lore context alongside personality, memories, quest state
```

**Query logic:**

```python
def get_relevant_lore(npc, query_text, top_k=3):
    """
    Retrieve lore entries relevant to the query, filtered by NPC scope.
    """
    # 1. Determine NPC's accessible scopes
    npc_tags = npc.lore_scope_tags  # e.g. ["millholm_town", "millholm", "mages_guild"]

    # 2. Build pre-filter: continental (everyone) + entries that contain
    #    at least one of the NPC's tags. This is a permissive pre-filter —
    #    AND semantics are enforced afterwards by _npc_can_access_lore().
    q = Q(scope_level="continental")
    for tag in npc_tags:
        q |= Q(scope_tags__contains=[tag])  # Postgres JSON contains (per tag)
    qs = LoreMemory.objects.using("ai_memory").filter(q)

    # 3. Semantic search within filtered set
    # (same pgvector / numpy branching as NpcMemory search)
    # 4. Post-filter results through _npc_can_access_lore() to enforce
    #    AND semantics on multi-tag entries
    ...
```

**Prompt template integration:**

```markdown
You are {name}, {personality}.

## What you know about the world
{lore_context}

## Your history with this character
{memories}

## Current situation
You are in: {location}
{quest_context}
```

The `{lore_context}` replaces or supplements the existing `{knowledge}` attribute. For NPCs that have both, `{knowledge}` can provide personality-specific framing ("you learned this from your grandmother") while `{lore_context}` provides the factual content.

---

## NPC Scope Resolution

An NPC's lore access is determined by its tags. This aligns with the existing Evennia tag system:

| NPC Tag | Lore Access |
|---|---|
| Zone tag (`category="zone"`, e.g. `millholm`) | Regional lore for that zone |
| District tag (`category="district"`, e.g. `millholm_town`) | Local lore for that district |
| Faction tag (`category="faction"`, e.g. `mages_guild`) | Faction-specific lore |
| Continental (implicit) | Everyone gets continental lore |

**Example NPC scope tags:**

| NPC | Tags | Lore Access |
|---|---|---|
| Rowan (bartender) | `millholm`, `millholm_town` | Continental + Millholm regional + Millholm Town local |
| Archmage Tindel | `millholm`, `millholm_town`, `mages_guild` | All of the above + Mages Guild faction |
| Shadow Mistress Vex | `millholm`, `millholm_sewers`, `thieves_guild` | Continental + Millholm regional + Sewers local + Thieves Guild faction |
| Gnoll Warlord (southern) | `millholm`, `millholm_southern`, `gnoll` | Continental + Millholm regional + Southern local + Gnoll tribal lore |
| Farmer Bramble | `millholm`, `millholm_farms` | Continental + Millholm regional + Farms local |

The zone and district tags already exist on rooms. NPCs inherit their room's tags or have their own. Faction tags are a new category but follow the same Evennia tag pattern.

---

## Lore Authoring Workflow

### Lore repository

Lore YAML files live in a dedicated repository: `FCM/lore/`. This repo is separate from the game code and can be edited collaboratively. The directory structure mirrors the scope hierarchy:

```
FCM/lore/
    continental.yaml              # world-wide common knowledge
    millholm/
        regional.yaml             # Millholm zone-level lore
        millholm_town.yaml        # town district local lore (future)
        millholm_farms.yaml       # farms district local lore (future)
        ...
    factions/
        mages_guild.yaml          # Mages Guild faction lore
        thieves_guild.yaml        # Thieves Guild faction lore (future)
        ...
```

### YAML file format

Each file contains a `source` identifier and a list of `entries`:

```yaml
source: "millholm/regional.yaml"
entries:
  - title: "Founding of Millholm"
    scope_level: regional
    scope_tags: ["millholm"]
    content: |
      Millholm was founded roughly four hundred years ago by four
      families from the eastern coast: the Stonefields, the
      Brightwaters, the Goldwheats, and the Ironhands...
```

**Required fields:** `title`, `content`, `scope_level`, `scope_tags`

### Importing lore

The importer is a **standalone Python tool** that lives in the lore repo at [`FCM/lore/tools/import_lore.py`](../lore/tools/import_lore.py). It connects to the game's `ai_memory` database directly via psycopg (Postgres) or sqlite3 (local SQLite) — no Django, no Evennia dependency. Same script runs locally for development and on Railway in production.

**Why standalone:** the Railway game container does not include the lore repo files, so an in-game management command can't see them. The standalone tool deploys as a separate Railway service in the same project as the game, sharing `DATABASE_URL` and `OPENAI_API_KEY` via service references. Each push to the lore repo auto-deploys the service, runs the import, and exits.

```bash
# Local
cd FCM/lore/tools
pip install -r requirements.txt
cp .env.example .env   # set OPENAI_API_KEY
set -a; source .env; set +a
python import_lore.py
python import_lore.py --dry-run    # preview, no DB writes, no embedding calls
python import_lore.py --prune      # delete DB entries no longer in YAML (off by default)
```

The default lore directory is the parent of `tools/` (the lore repo root). The game server does not need to be running. See [`FCM/lore/tools/README.md`](../lore/tools/README.md) for full usage including Railway deployment.

**Idempotent:** Entries are matched on `(source, title)`. A unique constraint on those columns (added by `ai_memory/migrations/0005_lorememory_unique_source_title.py`) makes the upsert atomic via Postgres `INSERT ... ON CONFLICT`. Re-running the import:
- **New entry** → created and embedded
- **Content changed** → updated and re-embedded
- **Content unchanged** → skipped (no wasted embedding calls)

**Schema dependency:** the standalone tool writes directly to the `ai_memory_lorememory` table by column name. If you change the `LoreMemory` Django model, update the column list in `import_lore.py` to match. The script's top-of-file comment calls this out.

### Adding new lore

1. Add or edit entries in the YAML files in the lore repo
2. Push to the lore repo — Railway auto-deploys the importer service, which runs and exits
3. All NPCs with `llm_use_lore=True` immediately have access to the new knowledge

For local dev: run `python tools/import_lore.py` from the lore repo after editing.

No code changes, no game-side migrations, no server restart required.

### Chunk sizing

Lore entries should be **focused and self-contained** — one topic per entry, roughly paragraph-length. This is important for embedding quality:
- Too short ("Millholm is a town") → weak embedding, matches too broadly
- Too long (entire history in one entry) → diluted embedding, relevant details buried
- Right size: a 3-6 sentence paragraph covering one topic, person, place, or event

---

## Three Memory Systems in Composition

> Which NPCs use lore memory depends on their intelligence tier. Tier 2+ NPCs (generic named-roles and above) access lore; Tier 1 commodity mobs do not. See [NPC_MOB_ARCHITECTURE.md](NPC_MOB_ARCHITECTURE.md) § NPC Intelligence Tiers for the full taxonomy.

The three memory systems are independent but compose at prompt time to create rich, contextual NPC behavior:

```
Player asks Rowan about the Shadow War:

  LoreMemory (continental)  →  "The Shadow War consumed the continent 2000 years ago..."
  NpcMemory (interaction)   →  "Last time Bob visited, we talked about the old ruins north of town"
  CombatMemory              →  (not applicable — Rowan isn't a combat NPC)

  Rowan's prompt:
    personality: warm, gossipy bartender
    lore: Shadow War common knowledge (from LoreMemory)
    memories: Bob's interest in ruins (from NpcMemory)
    → Rowan tells the Shadow War story as tavern gossip, connecting it to the ruins Bob asked about last time


Player's party approaches the Gnoll Warlord:

  LoreMemory (regional + gnoll) →  "The gnoll tribes have held the southern grasslands for centuries..."
  NpcMemory (interaction)       →  "This player tried to negotiate last time, offered food"
  CombatMemory (tactical)       →  "Lost to warrior+cleric party x3, cleric healing was the problem"

  Strategy bot's unified briefing:
    lore: gnoll territorial context
    relationship: previous peaceful attempt
    combat history: cleric is the threat
    → "This party has been both peaceful and hostile. Speak first — they may negotiate.
       If combat: silence the cleric immediately, focus melee."
```

Each system answers a different question:
- **LoreMemory:** "What do I know about the world?"
- **NpcMemory:** "What do I know about *you*?"
- **CombatMemory:** "What do I know about *fighting* you?"

---

## Examples

### Consistent world knowledge, different delivery

**Player asks two NPCs about the founding of Millholm:**

*Rowan (bartender):*
> LoreMemory returns: "Millholm was founded 400 years ago by settlers from the eastern coast..."
> Rowan's personality: warm, gossipy
> Rowan says: "Oh, Millholm's been here nigh on four hundred years! My gran used to say the first settlers came from the east coast — followed the Trade Way till they found this crossroads and decided it was a good spot for an inn. Can't say I blame 'em!"

*Archmage Tindel (mages guild):*
> LoreMemory returns: same founding entry + faction entry about the mages guild's arrival 200 years later
> Tindel's personality: scholarly, precise
> Tindel says: "The settlement was established approximately four centuries ago by eastern colonists. The Mages Guild established its Millholm chapter two centuries later, drawn by the unusual ley line convergence beneath the town — which, I might add, remains poorly understood to this day."

Same lore. Different scope (Tindel gets the guild-specific entry). Different personality. Different delivery. No manual per-NPC authoring.

### Lore-aware quest context

A player asks Brother Aldric at the Temple about rumours of undead in the south:

> LoreMemory (regional + local) returns: entries about the barrow mounds south of town, the ancient catacombs beneath
> LoreMemory (faction: temple) returns: temple doctrine about undead, the clerical duty to consecrate burial sites
> NpcMemory returns: this player is a cleric initiate who recently joined the temple
> → Aldric connects the lore to the player's situation: "The barrows south of town have been sealed for generations, but there are... concerning reports. As a new initiate of the order, this would be an excellent opportunity to prove your devotion. The dead should rest, child."

The quest hook emerges naturally from the intersection of lore, faction knowledge, and personal memory — not from a scripted quest dialogue tree.

### Specialist knowledge gating

A player asks Rowan (bartender) about ley lines:

> LoreMemory search for "ley lines" filtered by Rowan's tags (millholm, millholm_town) → no results (ley line lore is tagged `faction:mages_guild`)
> Rowan says: "Ley lines? That's mage talk, love. You'd want to ask up at the Guild about that sort of thing."

The same question to Tindel:
> LoreMemory search filtered by Tindel's tags (includes `mages_guild`) → returns ley line entries
> Tindel gives a detailed answer.

Knowledge gating happens naturally through scope tags. Rowan doesn't awkwardly know arcane theory, and she naturally redirects to someone who does.

---

## Implementation Status

| Component | Status | Location |
|---|---|---|
| `LoreMemory` model | Built | `ai_memory/models.py` |
| pgvector dual-backend | Built | `ai_memory/services.py` — `_is_postgres()` branching |
| `store_lore()` | Built | `ai_memory/services.py` — idempotent upsert with embedding |
| `search_lore()` | Built | `ai_memory/services.py` — scope-filtered semantic search |
| `get_recent_lore()` | Built | `ai_memory/services.py` — fallback without embeddings |
| `_build_lore_scope_filter()` | Built | `ai_memory/services.py` — `scope_tags__contains` (both backends) |
| `llm_use_lore` attribute | Built | `typeclasses/mixins/llm_mixin.py` — per-NPC toggle |
| `_get_lore_scope_tags()` | Built | `typeclasses/mixins/llm_mixin.py` — room tags + faction tags |
| `{lore_context}` prompt variable | Built | `typeclasses/mixins/llm_mixin.py` — `_get_context_variables()` |
| Prompt templates | Built | NPC personality templates in `llm/prompts/` accept `{lore_context}` |
| Standalone `import_lore.py` | Built | `FCM/lore/tools/import_lore.py` (in the lore repo, runs on Railway as its own service) |
| Lore YAML repo | Built | `FCM/lore/` — separate git repo, organised by scope (continental, regional, factions) |
| Faction tags on NPCs | Built | Faction tags applied to NPCs across multiple guilds (mages, temple, thieves, warriors) |
| Evennia tag system | Built | Zone + district tags on all rooms |
