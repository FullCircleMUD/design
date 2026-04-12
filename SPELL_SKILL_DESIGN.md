# Spell & Skill Design Reference

Central source of truth for all crafting recipes, spell schools, their
progression, and spell system implementation architecture.

---

## Spell Design Philosophy

The starting point for designing any spell school is three anchor spells:

1. **Workhorse (BASIC).** Cheap, no cooldown, spammable. Defines the school's
   identity and gives a caster something useful from day one. A mage who only
   knows their workhorse spell should still feel functional.

2. **"Fireball Equivalent" (EXPERT).** The double-edged sword — powerful but
   dangerous. An **unsafe AoE** that hits everything in the room including the
   caster and allies. The price of power is risk. DEX save for half damage, DC
   set by caster's roll. Safe to use at range (flying vs ground, or
   cross-room).

3. **"Wow Factor" (GRANDMASTER).** 100 mana, 3-round cooldown, massive single
   effect. The trophy spell that makes the school feel epic. Instant kills,
   total immunity, party portals, death interception — these are the spells
   people talk about.

Design the three anchors first, then fill in other spells around them at
SKILLED and MASTER tiers to round out the school's toolkit.

**Rules of thumb** (guidelines, not hard rules — adjusted for balance):

- **Mana cost:** ~1 mana per average point of damage. Conditions, utility, and
  AoE do not add extra cost. Individual spells may deviate for balance.
- **Cooldowns:** BASIC/SKILLED = 0, EXPERT = 1 round, MASTER = 2 rounds,
  GM = 3 rounds. Override via `cooldown` class attribute on individual spells.

**Example damage scaling** (not all spells scale this way):
- BASIC spells: +1d6 per mastery tier (e.g. Magic Missile adds a missile)
- SKILLED spells: +2d6 per mastery tier
- EXPERT+ spells: +3d6 per mastery tier

**AoE types:**
- **Unsafe AoE** — hits everything (caster, allies, enemies). DEX save for
  half.
- **Safe AoE** — hits enemies only with diminishing accuracy (100% / 80% / 60%
  / 40% / 20%).

---

## Skill Training System

### Overview

Skill mastery advances through **trainer NPCs** scattered across the game
world. Players spend gold and skill points at a trainer to advance a skill,
class skill, or weapon mastery one tier at a time. Training is the only way
to gain new mastery levels — skill points alone are not enough; the player
must find a qualified trainer.

### Trainer NPCs

A `TrainerNPC` is configured per instance with:

- **`trainable_skills`** — list of general/class skill keys this trainer teaches
- **`trainable_weapons`** — list of weapon type keys this trainer teaches
- **`trainer_class`** — which character class this trainer serves (e.g.
  `"warrior"`). Determines which class skill point pool is used for class
  skills.
- **`trainer_masteries`** — dict mapping skill/weapon key to the trainer's
  own mastery level (1-5). A trainer can teach any student whose current
  mastery is **strictly below** the trainer's. So a SKILLED trainer (2) can
  train a BASIC student (1) up to SKILLED, and a GRANDMASTER trainer (5)
  can train any student all the way up to GRANDMASTER. A trainer cannot
  teach a student who is already at the trainer's level or above.
- **`recipes_for_sale`** — dict of `{recipe_key: gold_cost}` for crafting
  recipes the trainer also sells.

### Three Resource Pools

Players have three separate skill point pools that can be spent on training:

| Pool | Source | Spent on |
|---|---|---|
| **General skill points** | Earned with character level | General skills (cartography, perception, stealth, etc.) |
| **Class skill points** | Earned per class with class level | Class-specific skills (bash, sneak attack, lay on hands) |
| **Weapon skill points** | Earned with character level | Weapon mastery (long sword, dagger, bow, etc.) |

The cost in points scales with the **target** mastery level — advancing to
higher mastery costs more points than advancing to lower mastery.

### Gold Cost

Training also consumes gold. The base cost scales with target mastery:

| Target Mastery | Base Gold Cost |
|---|---|
| BASIC | 10 |
| SKILLED | 25 |
| EXPERT | 50 |
| MASTER | 100 |
| GRANDMASTER | 200 |

**Charisma modifier:** the player's CHA score adjusts the gold cost. A high
Charisma character gets a discount (5% per modifier point); a low Charisma
character pays a surcharge. Minimum cost is always 1 gold.

Spent gold flows to the SINK location for reward recirculation.

### Training Flow

1. Player walks into a guild room containing a `TrainerNPC`.
2. Player types `train` to see the trainer's offerings — trainable skills,
   trainable weapons, costs, and current eligibility status.
3. Player types `train <skill>` or `train weapon <name>` to begin training
   a specific skill or weapon.
4. The system validates: trainer can teach (mastery gap > 0), player has
   enough gold, player has enough skill points, player is not already at
   GRANDMASTER, player is not already busy.
5. If validation passes, the player is shown a Y/N confirmation prompt with
   the full cost and outcome.
6. On Y: gold is deducted, a progress bar runs for the training duration
   (10-30 seconds depending on tier), and on completion the mastery is
   advanced and skill points are deducted.
7. On N: nothing happens. No gold spent, no points deducted.

### Compliance: Deterministic, No Random Failure

**Training always succeeds.** There is no roll, no chance of failure, no
"you wasted your gold" outcome. Once a player commits to training, they
receive exactly what the system promised them in the Y/N prompt.

This is a deliberate compliance design choice. An earlier version of the
training system rolled `d100` against a success chance derived from the
trainer-trainee mastery gap. On failure, gold was deducted, no
advancement occurred, and a one-hour cooldown was set on that trainer.
This pattern matched the textbook gambling triad:

1. **Consideration** — gold paid (and not refunded)
2. **Chance** — `randint(1, 100)` against success chance
3. **Prize** — mastery advancement (with downstream tradeable value via
   crafting and equipment)

Disclosing the success percentage in the prompt did **not** save the
mechanic. Showing the *odds* before payment is exactly what slot machines
do. Compliance requires disclosing the **outcome**, not just the
probability of an outcome.

The redesigned system removes the chance element entirely:

- The Y/N prompt shows: skill points cost, gold cost, training time,
  and the mastery advancement to be granted
- On confirmation, the player receives exactly that — no roll, no
  failure branch, no cooldown
- The trainer mastery gap is now a binary gate (can teach or can't teach),
  not a probability modifier
- Skill points and gold remain the rate-limiters of progression — they're
  scarce, and that's what creates the gameplay pacing

This aligns with FCM's overall compliance position:

> **No player will ever pay consideration for an outcome that is not
> fully disclosed to them before payment.**

See `design/COMPLIANCE.md` and `ops/COMPLIANCE_LEGAL.md` §9.5 for the
full gambling law analysis. Other game systems that have been (or will
be) redesigned around the same principle:

- **Loot spawning** — fully deterministic budget-driven push, no per-kill
  rolls (see `design/ECONOMY.md`)
- **Gem enchanting** — pre-disclosed slot consumption model, redesign in
  progress (see `design/ECONOMY.md` § Weapons — Gem Insets)

### Recipe Sales

Trainer NPCs can also sell crafting recipes via the `buy recipe` command.
This is a simple gold-for-knowledge transaction — no rolls, no chance,
no failure. The player pays the listed gold cost and learns the recipe
immediately.

---

## Crafting Skills

### Carpentry (Woodshop) — 14 recipes

| Mastery | Recipe | Description |
|---------|--------|-------------|
| BASIC | Training Dagger | Wooden practice dagger |
| BASIC | Training Shortsword | Wooden practice shortsword |
| BASIC | Training Longsword | Wooden practice longsword |
| BASIC | Training Greatsword | Wooden practice greatsword |
| BASIC | Training Bow | Wooden practice bow |
| BASIC | Training Lance | Wooden practice lance |
| BASIC | Club | Simple wooden bludgeon |
| BASIC | Wooden Shield | Basic wooden shield |
| BASIC | Shaft | Component for spear assembly (blacksmithing) |
| BASIC | Haft | Component for axe/hammer assembly |
| BASIC | Stock | Component for crossbow assembly (blacksmithing) |
| BASIC | Wooden Torch | Light source |
| SKILLED | Shortbow | Functional ranged weapon |
| SKILLED | Quarterstaff | Two-handed staff weapon |

### Blacksmithing (Smithy) — 25 recipes

| Mastery | Recipe | Description |
|---------|--------|-------------|
| BASIC | Bronze Dagger | Light melee weapon |
| BASIC | Bronze Shortsword | One-handed melee weapon |
| BASIC | Bronze Longsword | Versatile melee weapon |
| BASIC | Bronze Hand Axe | One-handed axe |
| BASIC | Bronze Spear | Reach melee weapon |
| BASIC | Bronze Mace | One-handed bludgeon |
| BASIC | Bronze Hammer | Heavy bludgeon |
| BASIC | Bronze Greatsword | Two-handed melee weapon |
| BASIC | Bronze Battleaxe | Two-handed axe |
| BASIC | Bronze Rapier | Finesse melee weapon |
| BASIC | Bronze Helm | Head armor |
| BASIC | Bronze Bracers | Wrist armor |
| BASIC | Bronze Greaves | Leg armor |
| BASIC | Bronze Lantern | Light source (metal) |
| BASIC | Spear | Iron tip + wooden shaft (cross-skill) |
| BASIC | Crossbow | Iron mechanism + wooden stock (cross-skill) |
| BASIC | Studded Leather Armor | Iron studs + leather (cross-skill) |
| SKILLED | Iron Dagger | Upgraded light melee |
| SKILLED | Iron Shortsword | Upgraded one-handed melee |
| SKILLED | Iron Longsword | Upgraded versatile melee |
| SKILLED | Iron Hand Axe | Upgraded one-handed axe |
| SKILLED | Iron Mace | Upgraded bludgeon |
| SKILLED | Iron Hammer | Upgraded heavy bludgeon |
| SKILLED | Iron Spiked Club | Upgraded club variant |
| SKILLED | Ironbound Shield | Upgraded shield |

### Leatherworking (Leathershop) — 11 recipes

| Mastery | Recipe | Description |
|---------|--------|-------------|
| BASIC | Leather Armor | Body armor (requires gambeson + straps) |
| BASIC | Leather Boots | Foot armor |
| BASIC | Leather Gloves | Hand armor |
| BASIC | Leather Belt | Waist slot |
| BASIC | Leather Cap | Head armor |
| BASIC | Leather Pants | Leg armor |
| BASIC | Leather Straps | Component for leather armor assembly |
| BASIC | Backpack | Container — increases carry capacity |
| BASIC | Panniers | Container — mount storage |
| BASIC | Bridle | Mount equipment |
| BASIC | Sling | Ranged weapon |

### Tailoring (Tailor) — 11 recipes

| Mastery | Recipe | Description |
|---------|--------|-------------|
| BASIC | Gambeson | Under-armor, component for leather armor |
| BASIC | Coarse Robe | Caster body armor |
| BASIC | Bandana | Head slot (cosmetic / enchant base) |
| BASIC | Kippah | Head slot (cosmetic / enchant base) |
| BASIC | Cloak | Back slot |
| BASIC | Veil | Face slot |
| BASIC | Scarf | Neck slot |
| BASIC | Sash | Waist slot |
| BASIC | Brown Corduroy Pants | Leg armor |
| BASIC | Warrior's Wraps | Hand slot |
| BASIC | N95 Mask | Face slot |

### Alchemy (Apothecary) — 9 recipes

All potions are BASIC entry. Effect **scales with crafter mastery** at brew time:

| Stat Buff | BASIC | SKILLED | EXPERT | MASTER | GM |
|-----------|-------|---------|--------|--------|-----|
| Bonus | +1 | +2 | +3 | +4 | +5 |
| Duration | 60s | 120s | 180s | 240s | 300s |

| Stat Restore | BASIC | SKILLED | EXPERT | MASTER | GM |
|--------------|-------|---------|--------|--------|-----|
| Heal | 2d4+1 | 4d4+2 | 6d4+3 | 8d4+4 | 10d4+5 |

| Recipe | Effect | Scaling |
|--------|--------|---------|
| Potion of Life's Essence | HP restore | Heal formula above |
| Potion of the Wellspring | Mana restore | Heal formula above |
| Potion of the Bull | +STR | Buff table above |
| Potion of the Zephyr | +DEX | Buff table above |
| Potion of Cat's Grace | +DEX (variant) | Buff table above |
| Potion of the Bear | +CON | Buff table above |
| Potion of Fox's Cunning | +INT | Buff table above |
| Potion of Owl's Insight | +WIS | Buff table above |
| Potion of Silver Tongue | +CHA | Buff table above |

Anti-stacking: named effect keys prevent doubling (e.g. can't stack two STR
potions).

### Jewellery (Jeweller) — 8 recipes

| Mastery | Recipe | Description |
|---------|--------|-------------|
| BASIC | Copper Ring | Finger slot (enchant base) |
| BASIC | Copper Bangle | Wrist slot (enchant base) |
| BASIC | Copper Studs | Ear slot (enchant base) |
| BASIC | Copper Chain | Neck slot (enchant base) |
| BASIC | Pewter Ring | Finger slot (enchant base) |
| BASIC | Pewter Bracelet | Wrist slot (enchant base) |
| BASIC | Pewter Hoops | Ear slot (enchant base) |
| BASIC | Pewter Chain | Neck slot (enchant base) |

### Enchanting (Wizard's Workshop) — 23 recipes

Mage-only. Transforms vanilla items into enchanted variants using Arcane Dust.
Recipes auto-granted at mastery level-up (no scrolls needed).

| Recipe | Base Item | Effect |
|--------|-----------|--------|
| Rogue's Bandana | Bandana | +1 DEX |
| Sage's Kippah | Kippah | +1 WIS |
| Titan's Cloak | Cloak | +1 STR |
| Veil of Grace | Veil | +1 CHA |
| Professor's Scarf | Scarf | +1 INT |
| Sun Bleached Sash | Sash | +1 CON |
| Scout's Cap | Leather Cap | +1 Initiative |
| Pugilist's Gloves | Leather Gloves | +1 hit/dam unarmed |
| Cowboy Boots | Leather Boots | TBD |
| Title Belt | Leather Belt | TBD |
| Rustler's Chaps | Leather Pants | TBD |
| Shepherd's Sling | Sling | TBD |
| Warden's Leather | Leather Armor | TBD |
| Defender's Helm | Bronze Helm | TBD |
| Bracers of Deflection | Bronze Bracers | TBD |
| Greaves of the Vanguard | Bronze Greaves | TBD |
| Nightseer's Ring | Copper Ring | TBD |
| Runeforged Chain | Copper Chain | TBD |
| Spellweaver's Bangle | Copper Bangle | TBD |
| Truewatch Studs | Copper Studs | TBD |
| Skydancer's Ring | Pewter Ring | TBD |
| Aquatic N95 | N95 Mask | TBD |
| Enchanted Ruby | Ruby | Random effect via gem table |

All enchanting recipes are BASIC entry. Gem enchanting uses output tables with
probabilistic effects that scale by crafter mastery.

### Shipwright (Shipyard) — 5 recipes

| Mastery | Recipe | Description |
|---------|--------|-------------|
| BASIC | Cog | Small single-masted trading vessel. Tier 1 ship. |
| SKILLED | Caravel | Medium multi-masted exploration vessel. Tier 2 ship. |
| EXPERT | Brigantine | Fast two-masted vessel with square and lateen rigging. Tier 3 ship. |
| MASTER | Carrack | Large three-masted merchant vessel built for long voyages. Tier 4 ship. | Pending |
| GM | Galleon | Massive multi-decked ship, pinnacle of naval architecture. Tier 5 ship. | Pending |

Ships are NFT items (ShipNFTItem). Higher-tier ships unlock higher boat_level
sea routes.

---

## Mage Spell Schools

### Evocation — Direct Damage

| Mastery | Spell | Role | Description | Status |
|---------|-------|------|-------------|--------|
| BASIC | Magic Missile | Workhorse | Auto-hit force. 1–5 missiles (1d4+1 each), scales with tier. | Done |
| BASIC | Frostbolt | Workhorse | 1d6 cold + contested SLOWED (1–5 rounds). | Done |
| BASIC | Fire Bolt | Hit-roll DPS | d20 + INT + mastery vs AC. (tier)d8 fire. Can miss, can crit (2x dice on nat 20). | Done |
| SKILLED | Flame Burst | Safe AoE | 3d6 fire, diminishing accuracy per target. | Done |
| SKILLED | Lightning Bolt | Single-target | High single-target lightning damage. | Planned |
| EXPERT | Fireball | **Fireball eq.** | 8d6 fire, unsafe AoE, DEX save half. Hits caster + allies. | Done |
| EXPERT | Chain Lightning | Multi-target | Bouncing lightning between targets. | Planned |
| MASTER | Cone of Cold | Safe AoE + CC | 10d6 cold, safe AoE, auto-applies SLOWED (2 rounds). | Done |
| GM | Power Word: Death | **Wow factor** | Instant death (HP ≤ 20) or contested save. | Done |

### Abjuration — Protection & Dispel

| Mastery | Spell | Role | Description | Status |
|---------|-------|------|-------------|--------|
| BASIC | Mage Armor | Workhorse | +3/+3/+4/+4/+5 AC, 1–3 hours. Shares ARMORED effect with Divine Armor. | Done |
| BASIC | Shield | Reactive | Auto-triggers when hit. +4/+4/+5/+5/+6 AC for 1/2/2/3/3 rounds, 3/5/7/9/12 mana per trigger (scales with tier). Toggle. | Done |
| BASIC | Feather Fall | Utility | Negates fall damage. 10 min–4 hours. Checked in _check_fall(). | Done |
| SKILLED | Resist Elements | Utility buff | 20% resistance to one element (fire/cold/lightning/acid/poison), 30s. | Done |
| SKILLED | Shadowcloak | Group stealth | +4 stealth to caster + group, 4 minutes. | Done |
| EXPERT | Antimagic Field | **Fireball eq.** | Unsafe AoE — dispels all spell/potion effects, suppresses casting 1 round. | Scaffolded |
| MASTER | Group Resist | Party buff | Resist Elements on all party members. | Scaffolded |
| GM | Invulnerability | **Wow factor** | All damage reduced to 0 for 1 combat round. | Scaffolded |

### Necromancy — Life Manipulation

| Mastery | Spell | Role | Description | Status |
|---------|-------|------|-------------|--------|
| BASIC | Drain Life | Workhorse | 2d6 cold, heals caster 100% of damage dealt. | Done |
| BASIC | Raise Skeleton | Summon | Raise weak skeleton minion from corpse. 1–3 skeletons, 2–10 min. | Scaffolded |
| BASIC | Fear | CC | Contested INT vs WIS. FRIGHTENED = forced flee each round. Save-each-round WIS. HUGE+ immune. 1–5 rounds. | Done |
| SKILLED | Vampiric Touch | Melee drain | Touch attack 1d6 necrotic, heals past max HP. Escalating mana cost. | Done |
| SKILLED | Raise Dead | Summon | Raise 1 corpse as undead minion for 2 minutes. | Scaffolded |
| EXPERT | Soul Harvest | **Fireball eq.** | 8d6 cold unsafe AoE (everyone except caster). Caster heals total dealt. | Done |
| MASTER | Raise Lich | Elite summon | Raise 1 lich (intelligent, casts Drain Life), 10 minutes. One at a time. | Scaffolded |
| GM | Death Mark | **Wow factor** | Mark target 1 round — all damage to marked target heals the attacker. | Scaffolded |

### Conjuration — Summoning & Dimensional Control

| Mastery | Spell | Role | Description | Status |
|---------|-------|------|-------------|--------|
| BASIC | Acid Arrow | Workhorse | 1d4+1 acid DoT per round, 1–5 rounds (scales with tier). | Done |
| BASIC | Light | Utility | Conjures magical light that illuminates room for everyone. Follows caster. 30 min–4 hours. Shares effect with Divine Light. | Done |
| BASIC | Find Familiar | Summon | Summons a familiar with remote control (see through eyes, move room-to-room). Tier determines type: rat (BASIC), cat/stealth (SKILLED), owl/flies (EXPERT), hawk/flies+fights (MASTER), imp/flies+fights+light (GM). One per caster. Permanent until dismissed/killed. | Done |
| SKILLED | Teleport | Utility | Self-teleport within range (district → zone → world by tier). | Scaffolded |
| EXPERT | Dimensional Lock | **Fireball eq.** | Unsafe AoE — applies DIMENSION_LOCKED (blocks flee/teleport/summon). | Scaffolded |
| MASTER | Conjure Elemental | Combat summon | Summon elemental (fire/ice/earth/air) for 10 min. One at a time. | Scaffolded |
| GM | Gate | **Wow factor** | Party-wide portal to any waygate the caster has personally discovered. | Scaffolded |

### Divination — Knowledge & Perception

| Mastery | Spell | Role | Description | Status |
|---------|-------|------|-------------|--------|
| BASIC | Identify | Workhorse | Reveals item/creature properties. Actor detail scales by tier. | Done |
| BASIC | Darkvision | Utility | Grants DARKVISION condition. 30 min–4 hours. Shares effect with Divine Sight. | Done |
| BASIC | Locate Object | Utility | Find named object by range (room→district→zone→world by tier). | Scaffolded |
| BASIC | Detect Traps | Utility | Reveals traps in current room. Passive detection on move at SKILLED+. | Scaffolded |
| SKILLED | True Sight | Self-buff | SKILLED: see HIDDEN. EXPERT: detect traps. MASTER: see INVISIBLE. 5–60 min. | Done |
| SKILLED | Scry | Remote intel | Query creature status from anywhere. Info scales by tier. | Scaffolded |
| EXPERT | Mass Revelation | **Fireball eq.** | Unsafe AoE — strips HIDDEN/INVISIBLE from all. Reveals traps at GM. | Scaffolded |

### Illusion — Deception & Concealment

| Mastery | Spell | Role | Description | Status |
|---------|-------|------|-------------|--------|
| BASIC | Blur | Workhorse | Self-buff, combat only. Enemies have disadvantage on 1 attack/round. 3–7 rounds. | Done |
| BASIC | Mirror Image | Evasion | Creates 1–5 illusory duplicates that absorb attacks. | Scaffolded |
| BASIC | Disguise Self | Utility | Change visible name/desc/race. Breaks on combat. spell_arg for name. | Scaffolded |
| BASIC | Distract | Utility | Grants caster advantage for 1–3 rounds (combat) or non-combat advantage. Auto-flee. | Scaffolded |
| SKILLED | Invisibility | Stealth | INVISIBLE condition, 5–60 min. Breaks on attack/cast. | Done |
| EXPERT | Mass Confusion | **Fireball eq.** | Unsafe AoE — applies CONFUSED (random target selection each round). | Scaffolded |
| MASTER | Greater Invisibility | Persistent stealth | INVISIBLE that doesn't break on attack/cast. 5–10 min. | Scaffolded |
| GM | Phantasmal Killer | **Wow factor** | Contested WIS save. Fail: 20d6 psychic (or death). Save: 10d6. | Scaffolded |

---

## Divine Spell Schools

### Divine Healing — Restoration (Cleric/Paladin)

| Mastery | Spell | Role | Description | Status |
|---------|-------|------|-------------|--------|
| BASIC | Cure Wounds | Workhorse | Heals (tier)d6 + WIS mod. Single target. | Done |
| BASIC | Vigorise | Workhorse | Restores (tier+4)d6 + WIS mod movement. Single target. | Done |
| BASIC | Cure Blindness | Condition removal | Removes BLINDED. EXPERT+: also removes DEAF. | Done |
| BASIC | Cure Poison | Condition removal | Removes POISONED + PoisonDoTScript. EXPERT+: grants poison resistance. | Done |
| SKILLED | Purify | Condition removal | Removes one harmful condition (poison, disease, etc.). | Scaffolded |
| EXPERT | Mass Heal | **Fireball eq.** | Heals all allies in room. Scales with WIS mod. | Scaffolded |
| GM | Death Ward | **Wow factor** | Pre-emptive buff — intercepts death, target survives at 1 HP. Consumed on trigger. | Scaffolded |

### Divine Protection — Warding (Cleric/Paladin)

| Mastery | Spell | Role | Description | Status |
|---------|-------|------|-------------|--------|
| BASIC | Sanctuary | Workhorse | Self-buff — enemies can't target caster. Breaks on offensive action. 1–5 min. | Done |
| BASIC | Divine Armor | Workhorse | +2/+2/+3/+3/+4 AC, 1–3 hours. Shares ARMORED effect with Mage Armor. | Done |
| BASIC | Bless | Support | +1/+1/+2/+2/+3 hit + save bonus on friendly target. 1–3 min. | Done |
| EXPERT | Holy Aura | **Fireball eq.** | AC + resistance bonus to all allies in room. | Scaffolded |
| GM | Divine Aegis | **Wow factor** | Total damage immunity on target (self or ally) for short duration. | Scaffolded |

### Divine Judgement — Holy Wrath (Paladin only)

| Mastery | Spell | Role | Description | Status |
|---------|-------|------|-------------|--------|
| BASIC | Smite | Reactive | Auto-triggers on weapon hit. Bonus radiant 1d6–5d6 per hit. Toggle. 3–12 mana/hit. | Done |
| BASIC | Bravery | Self-buff | +1/+1/+2/+2/+3 AC + 5–25 bonus HP. 5–15 min. HP clamped on expiry. | Done |
| BASIC | Bolt of Judgement | Workhorse | Auto-hit radiant (tier x 1d4+1). Evil multiplier: max(1, ceil(-alignment/250)). 1x–4x damage. | Done |
| EXPERT | Holy Fire | **Fireball eq.** | Radiant damage 8d6, diminishing accuracy, enemies only. | Scaffolded |
| GM | Wrath of God | **Wow factor** | Massive unsafe AoE radiant + BLINDED/STUNNED to all. | Scaffolded |

### Divine Revelation — Divine Knowledge (Cleric/Paladin)

| Mastery | Spell | Role | Description | Status |
|---------|-------|------|-------------|--------|
| BASIC | Holy Insight | Workhorse | Identify + divine sight (alignment, undead, evil aura detection). | Done |
| BASIC | Divine Light | Utility | Holy radiance illuminates room for everyone. Follows caster. 30 min–4 hours. Shares effect with Light. | Done |
| BASIC | Divine Sight | Utility | Grants DARKVISION condition. 30 min–4 hours. Shares effect with Darkvision. | Done |
| BASIC | Detect Alignment | Utility | See coloured alignment tags on creatures: (Evil) red, (Good) gold, (Neutral) white. 30 min–4 hours. | Done |
| SKILLED | Holy Sight | Divine perception | SKILLED: detect traps. EXPERT: see INVISIBLE. MASTER: see HIDDEN. 5–60 min. | Done |

### Divine Dominion — Control & Compulsion (Cleric/Paladin)

| Mastery | Spell | Role | Description | Status |
|---------|-------|------|-------------|--------|
| BASIC | Command | Workhorse | Contested WIS. Four words: halt (STUNNED), grovel (PRONE), drop (disarm), flee. HUGE+ immune. | Done |
| BASIC | Calm | CC | Stops all combat in room. CALM effect prevents re-engagement. Contested WIS vs WIS. HUGE+ immune. 10–30s. | Scaffolded |
| BASIC | Blindness | CC | Inflicts BLINDED. Contested WIS vs CON. Save-each-round CON. Grants advantage to enemies. HUGE+ immune. 3–8 rounds. | Done |
| EXPERT | Hold | Single-target CC | PARALYSED with per-round WIS save to break. Size-gated: EXPERT=medium, MASTER=large, GM=huge. | Done |
| GM | Word of God | **Wow factor** | Mass STUNNED all enemies. No save first round, contested WIS saves after. | Scaffolded |

---

## Nature Spell School

### Nature Magic — Primal Forces (Druid/Ranger)

| Mastery | Spell | Role | Description | Status |
|---------|-------|------|-------------|--------|
| BASIC | Entangle | Workhorse | Single-target vines. Contested WIS vs STR. ENTANGLED = can't act, attackers gain advantage. 1–5 rounds. | Done |
| BASIC | Speak with Animals | Utility | Communicate with animal mobs. Calm aggression at SKILLED+. 5 min–1 hour. | Scaffolded |
| BASIC | Thorn Whip | Damage + CC | (tier)d6 piercing. If target flying: contested WIS vs STR pulls to ground. HUGE+ immune to pull. | Scaffolded |
| BASIC | Water Breathing | Utility | Grants WATER_BREATHING condition. Cancels breath timer. 10 min–4 hours. Can target allies. | Done |
| EXPERT | Call Lightning | **Fireball eq.** | 6d6–12d6 lightning unsafe AoE. DEX save for half. Lower damage than Fireball (nature design choice). | Done |
| GM | Earthquake | **Wow factor** | Massive unsafe AoE bludgeoning + STUNNED/knockdown to all. | Scaffolded |

---

## Spell System Architecture

Spells are **class-based** (one Python class per spell) with a **registry** for discovery. This differs from recipes which are data-driven dicts — spells need per-tier execution logic that is genuinely different code, not just parametric scaling.

### Why Class-Based, Not Data-Driven

Some spells scale parametrically (magic missile: 1/2/3/4/5 missiles) but many have qualitatively different behavior per mastery tier:
- **Teleport**: basic=within area, skilled=within zone, expert=within continent, master=within world, GM=across worlds
- **Summon**: basic=rat, GM=dragon (different creatures, AI, duration)
- **Invisibility**: basic=breaks on attack, expert=breaks on cast only, GM=doesn't break

A unified class-per-spell system avoids the confusion of maintaining two systems (parametric vs custom) and lets any spell evolve from simple to complex without migration.

### Registry Pattern

```python
# world/spells/registry.py
SPELL_REGISTRY = {}

def register_spell(cls):
    SPELL_REGISTRY[cls.key] = cls()
    return cls

# world/spells/evocation/magic_missile.py
@register_spell
class MagicMissile(Spell):
    key = "magic_missile"
    aliases = ["mm"]
    name = "Magic Missile"
    school = skills.EVOCATION          # skills enum member
    min_mastery = MasteryLevel.BASIC
    mana_cost = {1: 5, 2: 8, 3: 10, 4: 14, 5: 16}
    target_type = "hostile"

    def _execute(self, caster, target):
        tier = self.get_caster_tier(caster)
        missiles = tier
        total_damage = sum(dice.roll("1d4+1") for _ in range(missiles))
        target.hp = max(0, target.hp - total_damage)
        s = "s" if missiles > 1 else ""
        return (True, {
            "first":  f"You fire {missiles} glowing missile{s} at {target.key}...",
            "second": f"{caster.key} fires {missiles} glowing missile{s} at you...",
            "third":  f"{caster.key} fires {missiles} glowing missile{s} at {target.key}...",
        })
```

**Base `Spell` class** (`world/spells/base_spell.py`) handles: mastery check, cooldown check, mana deduction, dispatch to `_execute()`. Subclass per spell implements `_execute(caster, target)` which returns `(bool, dict)` with first/second/third person messages. Validation failures return `(False, str)`. Each spell also has `description` (short flavour text) and `mechanics` (multi-line rules/scaling text) for the future `spellinfo` command.

**Class attributes:**
- `key` — unique registry key (e.g. `"magic_missile"`)
- `aliases` — shorthand names (e.g. `["mm"]`), default `[]`
- `name` — display name (e.g. `"Magic Missile"`)
- `school` — `skills` enum member (e.g. `skills.EVOCATION`). Use `spell.school_key` property for string lookups against `class_skill_mastery_levels` dict.
- `min_mastery` — `MasteryLevel` enum (BASIC=1 through GRANDMASTER=5)
- `mana_cost` — dict `{tier: cost}` (e.g. `{1: 5, 2: 8, ...}`)
- `target_type` — `"hostile"`, `"friendly"`, `"self"`, or `"none"`

**File structure**: `world/spells/<school>/<spell_name>.py` — one file per spell, organised by school/domain.

**Discovery**: `@register_spell` decorator populates `SPELL_REGISTRY`. `__init__.py` per school folder imports all spell modules so decorators fire at import time.

**Registry helpers**: `get_spell(key)`, `get_spells_for_school(school)`, `list_spell_keys()`

### SpellbookMixin (typeclasses/mixins/spellbook.py)

Mixed into `FCMCharacter`. Provides `learn_spell()`, `knows_spell()`, `memorise_spell()`, `forget_spell()`, `is_memorised()`, `get_memorisation_cap()`, `get_known_spells()`, `get_memorised_spells()`. Storage: `db.spellbook` and `db.memorised_spells` (both `{spell_key: True}` dicts).

**Commands**: `cast`, `transcribe`, `memorise`/`memorize`, `forget`, `spells` — all in `commands/all_char_cmds/`.

### Mage vs Cleric Differences

| Aspect | Mage | Cleric |
|---|---|---|
| Schools | evocation, conjuration, divination, abjuration, necromancy, illusion (6 casting schools — enchanting is a crafting skill, not casting) | divine_healing, divine_protection, divine_revelation, divine_dominion (cleric/paladin); divine_judgement (**paladin only**); nature_magic (druid/ranger) |
| Learning | `transcribe <scroll>` — consumes spell scroll NFT | Auto-learn all spells at new skill tier when gaining a domain skill level (deferred) |
| Spellbook | Learned via transcribe | Populated automatically on skill-up |
| Memorise/Forget | Yes — memorise has delay, forget is instant | Same |
| Cast | From memory, costs mana | Same |
| Memorise cap | floor(mage_class_level / 4) + get_attribute_bonus(intelligence) + extra_memory_slots | floor(cleric_class_level / 4) + get_attribute_bonus(wisdom) + extra_memory_slots |

### Memory Slot System

- `extra_memory_slots` — cacheable stat from equipment (via `stat_bonus` effect type), follows standard pattern
- Cap checked at **memorise time only** — buff INT/WIS to memorise extra spells, they stay memorised when buff drops
- If spells are forgotten while at lower ability, re-memorising uses the lower cap
- Same universal pattern: `effective_cap = floor(class_level / 4) + get_attribute_bonus(ability) + extra_memory_slots`

### Spell Scroll NFTs

- `SpellScrollNFTItem` — `ConsumableNFTItem` subclass with `spell_key` AttributeProperty
- Mages consume via `transcribe` command — Y/N confirmation, then spell added to spellbook, scroll consumed
- Scrolls can also be cast directly (one-time use, no transcription, lower/no level requirement) — deferred
- Every mage spell has a corresponding scroll prototype in `world/prototypes/consumables/scrolls/`. Cleric spells are auto-learned on skill-up (no scrolls needed).

### Spell Implementation Patterns

**Cooldown tracking:** Spell-specific cooldowns tracked in `caster.db.spell_cooldowns`. Override default via `cooldown` class attribute on individual spells.

**Duration storage convention:** `_DURATION` dicts store values in their **natural human-readable unit**, then convert to the effect system's unit in `_execute()`:
- **Seconds-based effects** (`duration_type="seconds"`): store in **minutes** (e.g. `{1: 1, 2: 2, 3: 5}`), convert `* 60` before passing to `apply_named_effect()`. Examples: Invisibility, Sanctuary, Shadowcloak, True Sight.
- **Hours-based effects**: store in **hours** in `_SCALING` tuples, convert `* 3600`. Example: Mage Armor.
- **Combat-round effects** (`duration_type="combat_rounds"`): store in **rounds** directly — no conversion needed. Use `_ROUNDS` or `_SCALING` dict. Examples: Shield, Blur.
- **Display**: compute display string from the stored unit directly (e.g. `f"({duration_minutes} minutes)"`), not by reverse-converting from seconds.

**Recast refresh pattern** (for self-buffs like Invisibility, Sanctuary): if the effect is already active, compare new duration vs remaining time via `get_effect_remaining_seconds()`. Only refresh (remove + reapply) if gaining time. If existing is stronger, refund mana and return `(False, {...})`. This prevents downgrading a MASTER-tier cast with a BASIC recast.

**Range rules:** Spells use a `spell_range` attribute on the Spell base class (separate from the weapon `can_reach_target()` system):
- `"self"` — no height check (self-targeted spells like Shield, Mage Armor)
- `"melee"` — same `room_vertical_position` required (Vampiric Touch, Cure Wounds). Blocked across heights; mana not deducted on failure.
- `"ranged"` — any height within the room (default — Magic Missile, Fireball, Drain Life, etc.)
- Height validation runs in `Spell.cast()` before mana deduction. Spells that override `cast()` must replicate the check.
- Future: adjacent-room spells, ranged-in-melee penalty (Crossbow Expert feat analogue).

**Creature size tiers** (scaling mechanic for summons, knockback, reanimation): BASIC=Small, SKILLED=Medium, EXPERT=Large, MASTER=Huge, GM=Gargantuan.

### Spell Utility Helpers (world/spells/spell_utils.py)

- `apply_spell_damage(target, raw_damage, damage_type)` — applies damage with resistance check, triggers death.
- `get_room_enemies(caster)` — gets enemies via combat sides or NPC detection fallback. All heights.
- `get_room_all(caster)` — all living entities including caster (for unsafe AoE). All heights.
- `get_room_enemies_at_height(caster)` — enemies at the caster's `room_vertical_position` only.
- `get_room_all_at_height(caster)` — all living entities at the caster's height only.

### Implementation Status by School

> Spell balance numbers (damage dice, mana costs, cooldowns, scaling) are in the school tables above. Below lists implementation status and key technical notes only.

**Evocation (6/8):** MagicMissile (BASIC, auto-hit force), Frostbolt (BASIC, cold + contested SLOWED), FlameBurst (SKILLED, safe AoE, fire), Fireball (EXPERT, unsafe AoE, DEX save half), ConeOfCold (MASTER, safe AoE, diminishing accuracy + SLOWED), PowerWordDeath (GM, contested save). Planned: Lightning Bolt (SKILLED), Chain Lightning (EXPERT).

**Abjuration (4/7):** Shield (BASIC, reactive — auto-triggers via `check_reactive_shield()` in `combat/reactive_spells.py`, toggle via `toggle shield`, anti-stacking via `has_effect("shield")`), MageArmor (BASIC, seconds-based, stacks with Shield), Resist (SKILLED, `has_spell_arg` — `cast resist fire [target]`, per-element named effects coexist, uses DamageResistanceMixin via Layer 2 dispatch), Shadowcloak (SKILLED, group targeting via `get_group_leader()` + `get_followers(same_room=True)`).

**Necromancy (3/6):** DrainLife (BASIC, heals caster 100% capped at max HP), VampiricTouch (SKILLED, touch attack, heals PAST max HP, escalating mana cost), SoulHarvest (EXPERT, unsafe AoE, drains all living except caster).

**Conjuration (1/5):** AcidArrow (BASIC, DoT via AcidDoTScript with combat-round ticks, anti-stacking: new cast replaces old).

**Divination (2/4):** Identify (BASIC, target_type="any", actor + item branches, actors level-gated), TrueSight (SKILLED, DETECT_INVIS at MASTER+, `db.true_sight_tier` for trap detection).

**Illusion (2/5):** Blur (BASIC, BlurScript sets 1 disadvantage on all enemies per round), Invisibility (SKILLED, `break_invisibility()` zeros all refs + stops timer, recast refresh pattern).

**Divine Healing (1/4):** CureWounds (BASIC, WIS modifier scaling).

**Divine Protection (1/3):** Sanctuary (BASIC, `break_sanctuary()` zeros all refs + stops timer, recast refresh pattern, combat hooks).

**Divine Judgement (1/3):** Smite (BASIC, reactive — auto-triggers on weapon hit, toggle, respects radiant resistance). Paladin only.

**Divine Revelation (2/2):** HolyInsight (BASIC, extends Identify, evil/undead detection at tier 2+), HolySight (SKILLED, `_can_see_hidden()` helper in room_base.py, 23 tests).

**Divine Dominion (2/3):** Command (BASIC, `has_spell_arg`, 4 words, HUGE+ immune, 33 tests), Hold (EXPERT, PARALYSED with per-round WIS save, size gate scales, 24 tests).

**Nature Magic (2/3):** Entangle (BASIC, save_dc/save_stat, on-apply callback grants advantage), CallLightning (EXPERT, unsafe AoE, WIS-based save DC, 14 tests).

**Total: 29 of 50 spells implemented. 318+ spell tests.**

---

## Spell Backlog

All scaffolded spells grouped by blocking dependency. Implement the dependency
first, then the spells unlock.

### Ready to implement — no blockers

| School | Spell | Tier | Notes |
|--------|-------|------|-------|
| Evocation | Lightning Bolt | SKILLED | High single-target. Design needed. |
| Evocation | Chain Lightning | EXPERT | Bouncing multi-target. Design needed. |
| Divine Healing | Purify | SKILLED | Condition removal — pick purgeable list |
| Divine Judgement | Holy Fire | EXPERT | Safe AoE pattern (copy Flame Burst) |
| Abjuration | Group Resist | MASTER | Copy Shadowcloak group-targeting pattern + Resist logic |
| Divine Healing | Mass Heal | EXPERT | Copy Shadowcloak group-targeting pattern + Cure Wounds logic |
| Divine Protection | Holy Aura | EXPERT | Copy Shadowcloak group-targeting pattern + AC/resist buff |
| Divine Dominion | Word of God | GM | Mass STUNNED — named effect system handles this already |
| Nature Magic | Earthquake | GM | Mass damage + STUNNED — same infrastructure as Word of God |
| Divine Judgement | Wrath of God | GM | Mass radiant + BLINDED/STUNNED — same pattern |
| Divination | Mass Revelation | EXPERT | Trap system + hidden object system both fully implemented |
| Illusion | Phantasmal Killer | GM | WIS save system exists (Hold uses it), DamageType.PSYCHIC exists |
| Illusion | Greater Invisibility | MASTER | Small change — add flag check in break_invisibility() |
| Divination | Scry | SKILLED | ObjectDB.objects.filter() available for global mob search |

### Blocked on: damage immunity / death interception hooks

Need new hooks in `take_damage()` and/or `die()` for damage immunity and
death prevention. The combat round system and damage pipeline exist, but these
specific interception points don't yet.

| School | Spell | Tier | Notes |
|--------|-------|------|-------|
| Abjuration | Antimagic Field | EXPERT | Dispel iteration ready (break_effect exists), but room-level casting suppression needed |
| Abjuration | Invulnerability | GM | Needs damage immunity flag check in take_damage() |
| Divine Protection | Divine Aegis | GM | Same as Invulnerability — damage immunity on target |
| Necromancy | Death Mark | GM | Needs heal-attacker hook in damage pipeline |
| Divine Healing | Death Ward | GM | Needs death interception hook in die() |

### Blocked on: pet/retainer system

No owner tracking on NPCs, no minion AI, no minion duration timers.

| School | Spell | Tier | Notes |
|--------|-------|------|-------|
| Necromancy | Raise Dead | SKILLED | Raise 1 corpse for 2 min |
| Necromancy | Raise Lich | MASTER | Intelligent undead, casts Drain Life |
| Conjuration | Conjure Elemental | MASTER | Elemental pet for 10 min |

### Blocked on: room/world systems

| School | Spell | Tier | Notes |
|--------|-------|------|-------|
| Conjuration | Teleport | SKILLED | Room teleport flags not yet on RoomBase |
| Conjuration | Dimensional Lock | EXPERT | DIMENSION_LOCKED enum defined but checks not in flee/teleport |
| Conjuration | Gate | GM | Waygate typeclass + portal objects don't exist yet |

### Blocked on: CONFUSED condition

| School | Spell | Tier | Notes |
|--------|-------|------|-------|
| Illusion | Mass Confusion | EXPERT | CONFUSED not in conditions enum, combat tick random-target override needed |

### Enchanting effects TBD

| Recipe | Base Item | Notes |
|--------|-----------|-------|
| Cowboy Boots | Leather Boots | Effect not yet designed |
| Title Belt | Leather Belt | Effect not yet designed |
| Rustler's Chaps | Leather Pants | Effect not yet designed |
| Shepherd's Sling | Sling | Effect not yet designed |
| Warden's Leather | Leather Armor | Effect not yet designed |
| Defender's Helm | Bronze Helm | Effect not yet designed |
| Bracers of Deflection | Bronze Bracers | Effect not yet designed |
| Greaves of the Vanguard | Bronze Greaves | Effect not yet designed |
| Nightseer's Ring | Copper Ring | Effect not yet designed |
| Runeforged Chain | Copper Chain | Effect not yet designed |
| Spellweaver's Bangle | Copper Bangle | Effect not yet designed |
| Truewatch Studs | Copper Studs | Effect not yet designed |
| Skydancer's Ring | Pewter Ring | Effect not yet designed |
| Aquatic N95 | N95 Mask | Effect not yet designed |
