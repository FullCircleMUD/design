# New Player Experience

## Design Intent

FullCircleMUD is an economy-first game. The on-chain economy means inflation has real consequences — every item, every coin, every resource is drawn from a finite, managed supply. The new player experience exists to bootstrap players into that economy as quickly as possible, without overwhelming them, and without letting them extract from the system without playing in the spirit of the game.

The design philosophy is **teach the loop, not the commands**. Players who understand why they need bread, where it comes from, and how to produce it themselves are players who will engage with the economy long-term. Players who learn a sequence of commands without understanding the chain behind them will disengage as soon as the hand-holding stops.

---

## The Economic Foundation: Hunger

Hunger is not a tutorial mechanic. It is a **permanent, constant economic drain** that exists at every level of play.

Bread costs approximately 5–6 gold. Every character needs to eat. This is the floor that the entire economy stands on — it creates constant, inescapable demand for gold and for the resources that produce bread. Bread's price anchors the value of everything else: if a potion is worth acquiring, it must be worth meaningfully more than bread.

New players feel hunger pressure most acutely because they have the least gold. This is intentional and correct behaviour, not a flaw to be designed away. The starter quests are calibrated so that an engaged new player stays fed through their first session — but only just. After the onboarding period, players are expected to sustain themselves.

**Quest reward: free bread.** Five of the six starter quests give one bread as part of the completion reward. This serves two purposes: it keeps new players fed through onboarding, and it demonstrates the value of bread directly. A player who receives bread as a reward immediately understands what they're working to maintain. Hendricks' Bronze is the exception — the higher gold reward is the signal there, and by the time a player reaches the mine they've already learned what bread is worth.

---

## Rowan — The Handhold

Rowan is the first character a new player meets. He tends bar at The Harvest Moon Inn — the room where new characters arrive — and is implemented as a state machine that responds to the player's current progress.

His role changes as the player advances:

Rowan's `BartenderNPC._build_quest_context()` selects one of several context blocks based on player state, which is then injected into his LLM prompt:

| Context | When |
|---|---|
| `NEW_PLAYER_CONTEXT` | Tutorial not yet completed, no active rat cellar quest — points the player at the tutorial |
| `QUEST_PITCH_CONTEXT` | Tutorial done, no active rat cellar quest — pitches the rat cellar |
| `QUEST_ACTIVE_CONTEXT` | Player has the rat cellar quest, not yet completed |
| `TUTORIAL_SUGGEST_CONTEXT` | Player completed the rat cellar but skipped the tutorial — suggests the tutorial |
| `GENERIC_CONTEXT` | Fallthrough — over the starter level cap, fully onboarded, or no `quest_key` configured |

There's also a **warrior guild referral bypass**: if a player above the starter level cap is on the warrior_initiation guild quest, Rowan re-enters the rat cellar pitch/active flow even though the level gate would otherwise have flipped him to generic. This is the only way a high-level player can pick up the rat cellar quest.

**Starter level cap:** `BartenderNPC.STARTER_LEVEL_CAP = 3`. Above this level, Rowan defaults to `GENERIC_CONTEXT` and behaves as a background bartender — the assumption being that any player past level 3 has already onboarded.

**Tutorial delivery:** Rowan suggests the tutorial but does not run it himself. The tutorial chunks are spawned by `ExitTutorialStart` exits in the Tutorial Hub via `TutorialInstanceScript` — the player physically leaves Rowan to do the tutorial. Rowan has no "tutorial in progress" state.

The transition from Rowan to the **noticeboard** is the graduation point. Rowan is the hand-hold; the noticeboard is where autonomous play begins. Once a player is reading the noticeboard to find NPCs with quests, they've crossed from onboarding into the game proper.

Rowan also sells beer and ale — this is handled by the room's AMM mechanic, but from the player's perspective Rowan is the source. Players who return to the gate area after levelling up still have a reason to interact with him.

---

## The Tutorial — Three Chunks

The tutorial is delivered progressively by Rowan, not as a wall of text at login.

| Chunk | Topic |
|---|---|
| 1 — Survival | Basic commands, hunger, eating bread |
| 2 — Economic Loop | Earning money, spending it, why resources matter |
| 3 — Growth & Social | Skills, guilds, other players |

Players who skip the tutorial miss context but are never locked out of content. Experienced MUD players and independent personalities may skip directly to questing. The tutorial exists for players who want it, not as a mandatory gate.

---

## The Starter Quests

These quests are available from the **noticeboard** in Millholm. They are **suggested, not enforced** — players can do them in any order, skip ones they find uninteresting, or ignore them entirely. The scaffolding is intentional but not mandatory.

### 1. Rat Cellar (`rat_cellar`) — Rowan, The Harvest Moon

**Teaching goal:** Combat basics.

The simplest possible fight. A player who completes this knows they can survive a fight and understands the basics of the combat system. The only purely combat-focused starter quest.

**Reward:** 10 gold + 1 bread.

---

### 2. Baker's Flour (`bakers_flour`) — Bron, Goldencrust Bakery

**Teaching goal:** The resource production loop.

The full chain: find wheat → mill to flour → deliver to baker. By the end, a player knows where wheat grows, where the mill is, and how processing works. The most important quest in the set — it teaches the economic loop that underlies the entire game.

**Reward:** 4 gold + 1 bread. The player now knows how to produce bread themselves.

---

### 3. Oakwright's Timber (`oakwright_timber`) — Master Oakwright, Woodshop

**Teaching goal:** Skilled resource gathering and zone geography.

A slightly longer loop introducing the wider geography of Millholm and how resource nodes work for non-food materials. Players learn that resources come from different places and require different approaches.

**Reward:** 5 gold + 1 bread.

---

### 4. Mara's Moonpetal (`mara_moonpetal`) — Mara Brightwater, The Mortar and Pestle

**Teaching goal:** Alchemy ingredients and the apothecary ecosystem.

Introduces Mara and the alchemy ingredient supply chain. Players learn that potions require ingredients, that ingredients can be harvested or bought, and that the apothecary is an ongoing destination.

**Reward:** 5 gold + 1 bread. Players now know where alchemy ingredients come from.

---

### 5. Elena's Cloth (`elena_cloth`) — Elena Copperkettle, her cottage

**Teaching goal:** The textile production loop.

Cotton → loom → cloth. Introduces Millholm Textiles and the second farming resource chain alongside wheat. A short loop that teaches a parallel production path.

**Reward:** 5 gold + 1 bread.

---

### 6. Hendricks' Bronze (`hendricks_ore`) — Old Hendricks, the Smithy

**Teaching goal:** Mining, smelting, and zone exploration.

The hardest starter quest. Requires traversing the deep woods to the abandoned mine, mining copper and tin ore, smelting them into bronze ingots at the smelter, then delivering to Hendricks. Multi-step, genuinely dangerous. Teaches the full metals production chain and introduces the mine as a zone.

**Reward:** 10 gold (no bread — the gold reflects the difficulty).

---

## The Expected Economic Loop After Onboarding

By the time a player finishes the starter quests (in whatever order), they know:

1. How to fight (combat income)
2. Where wheat grows and how to turn it into bread (self-sufficiency)
3. How to gather and sell timber and cloth (resource income)
4. Where alchemy ingredients come from (upmarket economy)
5. How to mine and smelt metals (heavy industry, harder path)

They are equipped to be self-sustaining. The noticeboard will direct them to further quests and deeper content. Hunger will continue to apply pressure. The game is now theirs to navigate.

---

## Account-Level Quest Caps

### The Problem

Starter quests give gold and a free bread on completion. This creates an exploit: a player creates a character, completes the easy quests, extracts the rewards, deletes the character, and repeats. This is a direct attack on the on-chain economy.

At current pricing, one full cycle (all 6 quests, one character) extracts approximately $0.18 of net value — gross quest rewards minus the AMM opportunity cost of the resources consumed. At 10 completions the maximum account-level exposure is approximately $2. The cap exists to put a hard ceiling on how many times this cycle can run before the account is cut off.

### The Cap

**10 completions per account per quest.**

This limit is per-account, not per-character. Deleting and recreating a character does not reset it.

**Why 10?**

- 4 character slots × legitimate first playthroughs = 4 uses before remort
- ~6 additional remort cycles before a character is powerful enough that starter quests are irrelevant
- Total legitimate uses ≈ 10
- 6 quests available means a player legitimately completing them all on a remort uses 6 of those 10 slots in one pass — the cap is generous enough that normal play never hits it

A player who legitimately remorts multiple times will never hit the cap through normal play. A player farming the exploit hits it quickly and is cut off with a maximum loss to the economy of approximately $1 at current pricing.

### Behaviour at Cap

- **Quest offer suppressed**: the NPC does not offer the quest once the account cap is reached
- **No error message visible to the player**: the NPC simply doesn't raise the topic (avoids gaming the cap by making it invisible)
- Quest rewards are only paid out on completion, so partial progress before hitting the cap is harmless

### Implementation Status

**Account-level cap: implemented.** Each starter quest sets `account_cap = 10` on its `BaseQuest` subclass. `BaseQuest.can_offer()` checks `account.db.quest_completion_counts[quest_key]` before offering the quest, and `BaseQuest._on_complete()` increments the counter on successful completion. The counter persists across character deletion and recreation because it lives on the account, not the character.

**Remort quest wipe: not yet implemented.** `cmd_remort.py` currently does not touch quest state. This means a remorted character retains its completed-quests list from the previous remort and cannot replay the starter quests. The "Why 10?" math above assumes remort wipe is in place — until it lands, a single legitimate player only consumes *one* slot per quest unless they create multiple characters. The cap is therefore safe (it can never block a legitimate player today), but the headroom calculation is theoretical until remort wipe ships. See `ops/PLANNING/0_BACKLOG` → "Remort Quest Wipe" for the remaining work.

---

## Relationship to the On-Chain Economy

The new player experience is designed with the on-chain economy in mind at every step. The hunger system, the bread economy, the quest rewards — all of it is calibrated around the principle that **inflation has real consequences**.

Starter quests are slightly net-positive for new players by design: they should earn enough to stay fed and have a little left over. That surplus is the carrot that motivates engagement. The quest cap ensures that surplus cannot be exploited at scale.

The goal is a game where every gold coin that enters the economy was earned by doing something in the spirit of the game.
