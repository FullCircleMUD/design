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
| Cowboy Boots | Leather Boots | Detect invisible |
| Title Belt | Leather Belt | +10 bludgeoning resistance |
| Rustler's Chaps | Leather Pants | +15 max move |
| Shepherd's Sling | Sling | +1 hit / +1 damage with sling |
| Warden's Leather | Leather Armor | +2 AC, +10 piercing resistance (no mages) |
| Defender's Helm | Bronze Helm | Crit immune (no mages) |
| Bracers of Deflection | Bronze Bracers | +10 slashing resistance (no mages) |
| Greaves of the Vanguard | Bronze Greaves | +1 initiative (no mages) |
| Nightseer's Ring | Copper Ring | Darkvision |
| Runeforged Chain | Copper Chain | +1 hit / +1 damage (dwarf only) |
| Spellweaver's Bangle | Copper Bangle | +5 max mana |
| Truewatch Studs | Copper Studs | +1 perception |
| Skydancer's Ring | Pewter Ring | Fly |
| Aquatic N95 | N95 Mask | Water breathing |
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

| Mastery | Spell | Role | Description |
|---------|-------|------|-------------|
| BASIC | Magic Missile | Workhorse | Auto-hit force. tier × (1d4+1) missiles. Spammable, no save, no resistance interaction. |
| BASIC | Fire Bolt | Hit-roll DPS | d20+INT+mastery vs AC. tier × d8 fire. Can miss, crit on nat 20 doubles dice. Subject to fire resistance. |
| BASIC | Frostbolt | Debuff workhorse | Flat 1d6 cold + contested INT+mastery vs CON to apply SLOWED (1–5 rounds per tier). The SLOWED is the value, damage is incidental. |
| SKILLED | Flame Burst | Safe AoE | 3–6d6 fire (per tier). Diminishing accuracy per target (100/80/60/40/20%). |
| EXPERT | Fireball | **Fireball eq.** | 8–14d6 fire (per tier), unsafe AoE, DEX save (DC = caster d20+INT+mastery) for half. Hits caster + allies. |
| MASTER | Cone of Cold | Safe AoE + CC | 10–13d6 cold (per tier). Safe AoE diminishing accuracy. Auto-applies SLOWED (1 round at MASTER, 2 at GM — see Known Discrepancies). |
| GM | Power Word: Death | **Wow factor** | HP ≤20 auto-kills (unless nat 1). HP >20 contested INT+8 vs CON+HD bonus; nat 20 always kills, nat 1 always fails. 100 mana, 3 round cooldown. |

### Abjuration — Protection & Dispel

| Mastery | Spell | Role | Description |
|---------|-------|------|-------------|
| BASIC | Mage Armor | Workhorse | +3/+3/+4/+4/+5 AC, 1–3 hours per tier. Shares ARMORED effect with Divine Armor. Anti-stack with mana refund on recast. |
| BASIC | Shield | Reactive | **Reactive only — cannot be cast manually.** Auto-triggers when hit via weapon hook. +4/+4/+5/+5/+6 AC for 1/2/2/3/3 rounds. Mana cost 3/5/7/9/12 per trigger. Toggle via `toggle shield`. |
| BASIC | Feather Fall | Utility | Negates fall damage. 10 min–4 hours per tier. Checked in `_check_fall()`. Mana refund on recast while active. |
| SKILLED | Resist Elements | Utility buff | 20–60% resistance to one element (fire/cold/lightning/acid/poison) for 30s. Uses `has_spell_arg` for element choice (`cast resist fire`). Stacks with racial/gear up to 75% cap. |
| SKILLED | Shadowcloak | Group stealth | +4 to +10 stealth on caster and same-room group members for 4–10 minutes per tier. Mana refund if all targets already affected. |
| EXPERT | Antimagic Field | **Fireball eq.** *(stub)* | Unsafe AoE — dispels all spell/potion effects in room, suppresses casting 1–3 rounds. Permanent item enchantments unaffected. |
| MASTER | Group Resist | Party buff *(stub)* | Resist Elements applied to all party members. Uses spell argument for element. |
| GM | Invulnerability | **Wow factor** *(stub)* | All damage reduced to 0 for 1 combat round. Antimagic Field can dispel. 100 mana. |

### Necromancy — Life Manipulation

| Mastery | Spell | Role | Description |
|---------|-------|------|-------------|
| BASIC | Drain Life | Workhorse | 2–6d6 cold (per tier). Caster heals 100% of actual damage dealt (after resistance), capped at max HP. Undead are immune (no life force). |
| BASIC | Fear | CC | Contested INT+mastery vs WIS. FRIGHTENED for 1–5 rounds — target flees a random exit each round (or cowers if no exits). Save-each-round WIS to break early. HUGE+ immune. |
| BASIC | Raise Skeleton | Summon *(stub)* | Raise 1–3 weak skeleton minions from a corpse. 2–10 minute duration per tier. Consumes the corpse. |
| SKILLED | Vampiric Touch | Melee drain | Touch attack (d20+INT+mastery vs AC). 1–4d6 necrotic per tier. Caster heals **above** max HP. Mana cost scales dynamically (3%–95% of max) with the bonus HP bracket. 10-minute duration. |
| SKILLED | Raise Dead | Summon *(stub)* | Raise 1–4 corpses as undead minions. Minion count and duration scale with tier. Player corpses with equipment are protected. |
| EXPERT | Soul Harvest | **Fireball eq.** | 8–14d6 cold (per tier) unsafe AoE — drains all living entities except the caster (excludes undead). Caster heals for the **total** damage dealt across all targets. |
| MASTER | Raise Lich | Elite summon *(stub)* | Transforms a single corpse into an intelligent lich (NPC spellcaster). 10–30 minute duration per tier. One active at a time. Player corpses protected. |
| GM | Death Mark | **Wow factor** *(stub)* | Marks target for 1 combat round. All damage dealt to the marked target heals the attacker for the same amount (capped at attacker's max HP). |

### Conjuration — Summoning & Dimensional Control

| Mastery | Spell | Role | Description |
|---------|-------|------|-------------|
| BASIC | Acid Arrow | Workhorse | DoT: 1d4+1 acid per round for 1–5 rounds (scales with tier). New cast replaces existing DoT. Spammable. |
| BASIC | Light | Utility | Conjures a magical light orb that illuminates the room for everyone. Follows caster. 30 min–4 hours per tier. Shares LIGHT_SPELL effect with Divine Light. Mana refund on recast. |
| BASIC | Find Familiar | Summon | Summons a familiar with remote control (see through eyes, move room-to-room, return on command). Tier determines creature type: rat (BASIC) → cat/stealth (SKILLED) → owl/flies (EXPERT) → hawk/flies+fights (MASTER) → imp/flies+fights+light (GM). One per caster. Persistent until dismissed or killed. |
| SKILLED | Teleport | Utility *(stub)* | Self-teleport within range (district → zone → world by tier). Blocked by `no_teleport_out`/`no_teleport_to` room flags and DIMENSION_LOCKED. 60s cooldown. |
| EXPERT | Dimensional Lock | **Fireball eq.** *(stub)* | Unsafe AoE — applies DIMENSION_LOCKED to all in room (blocks flee/teleport/summon) for 1–3 rounds. Contested WIS save (Expert normal, Master disadvantage, GM no save). |
| MASTER | Conjure Elemental | Combat summon *(stub)* | Summons a lesser/greater elemental (fire/ice/earth/air) for 10–30 minutes. One active at a time. Recasting replaces existing. Blocked by DIMENSION_LOCKED. |
| GM | Gate | **Wow factor** *(stub)* | Opens a portal to any waygate the caster has **personally discovered** (gating doesn't count as discovery). Party can walk through. 30 second portal duration, 5 minute cooldown. |

### Divination — Knowledge & Perception

| Mastery | Spell | Role | Description |
|---------|-------|------|-------------|
| BASIC | Identify | Workhorse | Reveals item or creature properties via dynamic templates. Actor detail level-gated: 1–5 BASIC, 6–15 SKILLED, 16–25 EXPERT, 26–35 MASTER, 36+ GM. PvP room check applies to actor identify. |
| BASIC | Darkvision (Infravision) | Utility | Grants DARKVISION condition for 30 min–4 hours per tier. Shared effect with Divine Sight. Mana refund on recast. (File is `infravision.py`, registered as key `darkvision`.) |
| BASIC | Locate Object | Utility *(stub)* | Find named object within range (room → district → zone → any zone by tier with increasing detail). Uses spell argument for object name. |
| BASIC | Detect Traps | Utility *(stub)* | Reveals traps in current room. Passive detection on move (SKILLED+), disarm bonus (EXPERT+), hidden door reveal (MASTER+). 5 min–1 hour duration. |
| SKILLED | True Sight | Self-buff | SKILLED: see HIDDEN. EXPERT: auto-detect traps. MASTER: see INVISIBLE. 5–60 min duration per tier. Personal buff only — doesn't reveal targets to others. Anti-stack (refresh-only). |
| SKILLED | Scry | Remote intel *(stub)* | Remote target intel: tier 2 status+zone, tier 3 +room+HP, tier 4 +AC/level/resistances, tier 5 +equipment. 30s cooldown. |
| EXPERT | Mass Revelation | **Fireball eq.** *(stub)* | Unsafe AoE — strips HIDDEN/INVISIBLE from all in room. MASTER+ also strips Greater Invisibility. GM tier reveals traps and secret exits. |

### Illusion — Deception & Concealment

| Mastery | Spell | Role | Description |
|---------|-------|------|-------------|
| BASIC | Blur | Workhorse | Self-buff, combat only. Enemies have disadvantage on 1 attack per round for 3–7 rounds per tier. Per-round BlurScript. Anti-stack (replace existing). |
| BASIC | Distract | Utility | Dual-mode. In combat: grants all allies advantage vs target for `tier` rounds. Out of combat: grants caster non-combat advantage. |
| BASIC | Mirror Image | Evasion *(stub)* | Creates 1–5 illusory duplicates. Incoming attacks may hit an image instead; images are destroyed on hit. |
| BASIC | Disguise Self | Utility *(stub)* | Change visible name (SKILLED+ also adds description and race). Breaks on combat. True Sight reveals at lower tiers. Uses spell argument for disguise name. |
| SKILLED | Invisibility | Stealth | INVISIBLE condition, 5–60 minutes per tier. Breaks on attack or cast. Grants 1 round of advantage on the first attack out of invisibility. Refresh-only (no downgrade). |
| EXPERT | Mass Confusion | **Fireball eq.** *(stub)* | Unsafe AoE — applies CONFUSED for 1–3 rounds, causing random target selection each round. Foresight charges (when implemented) auto-save. |
| MASTER | Greater Invisibility | Persistent stealth *(stub)* | INVISIBLE that does **not** break on attack or cast. 5–10 minutes per tier. Mass Revelation strips at MASTER+. |
| GM | Phantasmal Killer | **Wow factor** *(stub)* | Contested WIS save. Fail: 20d6 psychic damage (can kill from fright with special death message). Success: 10d6 half. |

---

## Divine Spell Schools

### Divine Healing — Restoration (Cleric/Paladin)

| Mastery | Spell | Role | Description |
|---------|-------|------|-------------|
| BASIC | Cure Wounds | Workhorse | Heals tier × d6 + WIS mod, capped at target's max HP. Single target, melee range. Mana refund if target at full HP. |
| BASIC | Vigorise | Workhorse | Restores (tier+4) × d6 + WIS mod movement points, capped at target's max move. Single target, melee range. Mana refund if target at full movement. |
| BASIC | Cure Blindness | Condition removal | Removes BLINDED. EXPERT+: also removes DEAF. Mana refund if no effect. |
| BASIC | Cure Poison | Condition removal | Removes POISONED and PoisonDoTScript. EXPERT+: also grants 25–75% poison resistance for 5–10 minutes per tier. Mana refund if no poison and no resistance applies. |
| SKILLED | Purify | Condition removal *(stub)* | Removes a single harmful condition (poison, disease, etc.). |
| EXPERT | Mass Heal | **Fireball eq.** *(stub)* | Heals all allies in room. Scales with tier and WIS modifier. |
| GM | Death Ward | **Wow factor** *(stub)* | Pre-emptive buff — when target would die, they survive at 1 HP instead. Ward is consumed on trigger. 100 mana, 3 round cooldown. |

### Divine Protection — Warding (Cleric/Paladin)

| Mastery | Spell | Role | Description |
|---------|-------|------|-------------|
| BASIC | Sanctuary | Workhorse | Self-buff — enemies cannot target caster. Breaks on offensive action. 1–5 minutes per tier. Refresh-only (won't downgrade); mana refund on no-op. |
| BASIC | Divine Armor | Workhorse | +2/+2/+3/+3/+4 AC, 1–3 hours per tier. Shares ARMORED effect with Mage Armor (weaker variant). Stacks with Shield. |
| BASIC | Bless | Support | +1/+1/+2/+2/+3 to hit rolls and save-each-round rolls on friendly target. 1–3 minutes per tier. Anti-stack with mana refund on recast. |
| EXPERT | Holy Aura | **Fireball eq.** *(stub)* | Group buff — AC and resistance bonus to all allies in room. Scales with tier. |
| GM | Divine Aegis | **Wow factor** *(stub)* | Total damage immunity on target (self or ally) for short duration. Parallel to Invulnerability but for allies. |

### Divine Judgement — Holy Wrath (Paladin only)

| Mastery | Spell | Role | Description |
|---------|-------|------|-------------|
| BASIC | Bolt of Judgement | Workhorse | Auto-hit radiant: tier × (1d4+1). **Alignment-based scaling**: damage multiplied by `max(1, ceil(-target_alignment/250))` — good/neutral 1×, evil 2×, very evil 3×, pure evil 4×. Deterministic, based on a public game-state variable. |
| BASIC | Smite | Reactive | **Reactive only — cannot be cast manually.** Auto-triggers on weapon hits in combat. Adds tier × 1d6 bonus radiant damage per triggering hit. Mana cost 3/5/7/9/12 per trigger. Toggle via `smite` command. |
| BASIC | Bravery | Self-buff | +1/+1/+2/+2/+3 AC and +5/+10/+15/+20/+25 max HP for 5–15 minutes per tier. Heals by the bonus HP amount on cast. HP clamped on expiry. Anti-stack. |
| EXPERT | Holy Fire | **Fireball eq.** *(stub)* | Safe AoE radiant (8–14d6 per tier) with diminishing hit accuracy (100/80/60/40/20%). Enemies only. |
| GM | Wrath of God | **Wow factor** *(stub)* | Massive unsafe AoE radiant damage + BLINDED/STUNNED condition to all in room. Hits caster and allies. 100 mana, 3 round cooldown. |

### Divine Revelation — Divine Knowledge (Cleric/Paladin)

All spells in this school are fully implemented.

| Mastery | Spell | Role | Description |
|---------|-------|------|-------------|
| BASIC | Holy Insight | Workhorse | Divine version of Identify. Includes alignment detection and undead detection on actors. Level-gated identification same as Identify. |
| BASIC | Divine Light | Utility | Conjures a sphere of holy radiance that illuminates the room for everyone. 30 min–4 hours per tier. Shares LIGHT_SPELL effect with Conjuration's Light. Mana refund on recast. |
| BASIC | Divine Sight | Utility | Grants DARKVISION through divine blessing. 30 min–4 hours per tier. Shares effect with Divination's Darkvision. |
| BASIC | Detect Alignment | Utility | Grants alignment aura perception on creatures in room: (Evil) red, (Good) gold, (Neutral) white. 30 min–4 hours per tier. |
| SKILLED | Holy Sight | Divine perception | SKILLED: auto-detect traps. EXPERT: see INVISIBLE. MASTER: see HIDDEN. 5–60 minutes per tier. Note: unlock order differs from True Sight. Personal buff only. Anti-stack. |

### Divine Dominion — Control & Compulsion (Cleric/Paladin)

| Mastery | Spell | Role | Description |
|---------|-------|------|-------------|
| BASIC | Command | Workhorse | Combat-only. Contested WIS (caster d20+WIS+mastery vs target d20+WIS). Issues one of four words via spell argument: **halt** (STUNNED), **grovel** (PRONE), **drop** (force disarm), **flee** (force flee). HUGE+ immune. |
| BASIC | Blindness | CC | Inflicts BLINDED for 3–8 rounds per tier. Contested WIS (caster) vs CON (target). Save-each-round to break early. Grants advantage to enemies. HUGE+ immune. |
| BASIC | Calm | CC *(stub)* | Ends all combat in room. Applies CALM effect preventing re-engagement for 10–30 seconds per tier. Contested WIS vs WIS for each target. HUGE+ immune. Caster must be in combat to cast. |
| EXPERT | Hold | Single-target CC | Binds target in divine chains (PARALYSED) for 3–5 rounds. Contested WIS+mastery sets DC; target rolls d20+WIS each round to escape. Size-gated: EXPERT can hold up to MEDIUM, MASTER up to LARGE, GM up to HUGE. GARGANTUAN always immune. |
| GM | Word of God | **Wow factor** *(stub)* | Mass STUNNED on all enemies in room. No save first round; contested WIS each subsequent round to break. 100 mana, 3 round cooldown. |

---

## Nature Spell School

### Nature Magic — Primal Forces (Druid/Ranger)

| Mastery | Spell | Role | Description |
|---------|-------|------|-------------|
| BASIC | Entangle | Workhorse | Single-target vines. Contested WIS+mastery vs STR. ENTANGLED for 1–5 rounds — denies action and grants enemies advantage. Spammable druid workhorse. |
| BASIC | Thorn Whip | Damage + CC | Deals tier × d6 piercing (always, regardless of save). Contested WIS+mastery vs STR to pull target to caster's height (uses height system). Held for `tier` rounds. HUGE+ immune to pull only. Pulled-and-released targets fall or drown depending on terrain. |
| BASIC | Water Breathing | Utility | Grants WATER_BREATHING condition for 10 min–4 hours per tier. Cancels active breath timer. Can target self or ally. Mana refund on recast. |
| BASIC | Speak with Animals | Utility *(stub)* | Communicate with animal mobs. Calm aggression at SKILLED+, animals share info at EXPERT+, brief allies at MASTER+. |
| EXPERT | Call Lightning | **Fireball eq.** | 6–12d6 lightning (per tier) unsafe AoE. DEX save (DC = caster d20+WIS+mastery) for half. Uses WIS modifier (not INT). Lower mana than Fireball — nature design choice. |
| GM | Earthquake | **Wow factor** *(stub)* | Massive unsafe AoE bludgeoning damage + STUNNED/knockdown to all in room. Hits caster and allies. 100 mana, 3 round cooldown. |

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

### Spell Arguments (`has_spell_arg`)

Some spells take a free-text argument that selects a sub-mode or chooses among several effects. Set `has_spell_arg = True` on the Spell subclass and parse the argument inside `_execute()`.

Examples:
- **Resist Elements** — `cast resist fire` selects which element to resist
- **Command** — `cast command halt` / `grovel` / `drop` / `flee` selects which compulsion word
- **Locate Object** — `cast locate <object name>` selects what to find
- **Disguise Self** — `cast disguise self <name>` selects the disguise identity

The base `Spell.cast()` flow strips the argument before doing target resolution and passes it through to `_execute()` for the spell to interpret.

### Mana Refund on No-Op

A common pattern across self-buffs and condition-removal spells: if the cast would have no effect (target already at full HP, target already has the buff at equal or higher tier, target has no condition to cure), the spell **refunds the mana** and returns `(False, ...)` instead of consuming it. This prevents the player from accidentally burning mana on a cast that does nothing.

Spells using this pattern:
- **Cure Wounds** — refund if target is at full HP
- **Vigorise** — refund if target is at full movement
- **Cure Blindness** — refund if no BLINDED/DEAF to remove
- **Cure Poison** — refund if no poison and no resistance applies
- **Mage Armor** — refund if already armored at equal or higher tier
- **Divine Armor** — refund if already armored at equal or higher tier
- **Bless** — refund if already blessed
- **Sanctuary** — refund if existing sanctuary is stronger or has more time remaining
- **Shadowcloak** — refund if **all** group members in room are already affected
- **Light** / **Divine Light** / **Darkvision** / **Divine Sight** / **Detect Alignment** / **Water Breathing** / **Feather Fall** — refund on recast while existing effect has more remaining time
- **True Sight** / **Holy Sight** / **Invisibility** — refresh-only (won't downgrade), refund if existing is stronger

Pair this with the **recast refresh pattern** documented above: `get_effect_remaining_seconds()` is the canonical way to compare existing vs new duration before deciding to refresh, refund, or replace.

### Alignment-Based Scaling

A pattern for divine spells that punish evil targets: damage is multiplied by a function of the target's `alignment_score`, which is a public deterministic game-state variable (the same one used by the alignment system documented in [ALIGNMENT_SYSTEM.md](ALIGNMENT_SYSTEM.md)).

**Bolt of Judgement** is the reference implementation:

```python
multiplier = max(1, math.ceil(-target.alignment_score / 250))
```

This gives:

| Target alignment | Score range | Multiplier |
|---|---|---|
| Good or Neutral | ≥ 0 | 1× |
| Evil | -1 to -500 | 2× |
| Very Evil | -501 to -750 | 3× |
| Pure Evil | -751 to -1000 | 4× |

**Compliance note:** this mechanic uses **no random number generation**. The damage multiplier is a deterministic function of a publicly visible game-state variable (the target's alignment, which players can read via Detect Alignment and Holy Insight). It does not violate the gambling-law constraint that variable outcomes must be disclosed before payment, because the player can compute the exact damage multiplier before casting. See [COMPLIANCE.md § Gambling Law](COMPLIANCE.md#gambling-law--deterministic-distribution--pre-disclosure) for the broader principle.

This pattern is reusable for any divine spell that's meant to punish evil targets without introducing chance.

### Reactive Spells

Two spells trigger automatically during combat without consuming the player's action:

**Shield (Abjuration)** and **Smite (Divine Judgement — Paladin only)** are flagged as reactive — they cannot be cast manually with `cast`. Instead, the player toggles them on (`toggle shield`, `smite`) and the combat handler calls into `combat/reactive_spells.py` from inside the 14-hook attack pipeline:

- **Shield** triggers via `at_wielder_about_to_be_hit` (defender hook). Gates: toggle on, memorised, mana available, not already shielded. Effect: AC bonus for a few rounds. Mana cost is per trigger, not per round.
- **Smite** triggers via `at_post_attack` on a successful hit (attacker hook). Gates: toggle on, memorised, mana. Effect: bonus radiant damage on the triggering hit. Mana cost is per trigger.

The trigger happens inside the combat tick, so the player's normal action is unaffected. They get the benefit on top of whatever they were doing.

**Why this design:** lets reactive defensive spells coexist with active casting in a single round. A mage can swing a staff, get hit, have Shield trigger, and cast a Magic Missile all in the same combat tick. See [COMBAT_SYSTEM.md § Reactive Spells](COMBAT_SYSTEM.md#reactive-spells) for the combat-side of the integration and the full hook ordering.

### Familiar System (Find Familiar)

`Find Familiar` (Conjuration BASIC) is a major system in its own right rather than just a buff or summon. Brief overview:

- **Tier-based creature types:** rat (BASIC) → cat (SKILLED, stealth) → owl (EXPERT, flies) → hawk (MASTER, flies + fights) → imp (GM, flies + fights + light source)
- **One per caster** — dismiss or let die before recasting
- **Persistent** — survives logout and reload, not duration-limited
- **Remote control commands** — the caster can `look` through the familiar's eyes, send it to adjacent rooms, and recall it. Higher-tier familiars (hawk, imp) can also engage in combat on the caster's behalf.

The familiar system overlaps with the broader pet system. Full mechanics live in [PETS_AND_MOUNTS.md](PETS_AND_MOUNTS.md). The Conjuration spell file is just the entry point that creates the persistent familiar object and links it to the caster.

### Implementation Status

The school tables above are the source of truth for what's implemented vs scaffolded. Spells marked `*(stub)*` in those tables exist as files but their `_execute()` either raises `NotImplementedError` or returns `(False, ...)` without producing an effect.

**Current totals:** 76 spell files across 12 schools. **35 are real implementations**, **41 are stubs** awaiting either a blocking dependency or a design decision. The stubs are not ordered or prioritised here — that's tracked in `ops/PLANNING/0_BACKLOG`.

Common blocking dependencies for the stubs:

- **Damage immunity / death interception hooks** — needed for Invulnerability, Divine Aegis, Death Ward, Death Mark
- **Pet / retainer / minion system** — needed for Raise Dead, Raise Lich, Conjure Elemental, Greater Familiar control extensions
- **Room flag and portal system** — needed for Teleport, Dimensional Lock, Gate
- **CONFUSED condition** — needed for Mass Confusion (combat tick must override target selection randomly while still being deterministic from a player-action standpoint)
- **Trap and hidden-object exposure** — needed for Mass Revelation, Detect Traps full feature set
- **NPC spellcasting AI** — needed for Raise Lich (the lich casts Drain Life)
- **Group/party detection** — needed for Mass Heal, Holy Aura, Group Resist (the Shadowcloak pattern works but needs to be lifted out)

### Known Discrepancies

These are doc-vs-code mismatches that need a **design decision** rather than just a doc update — pick one and either fix the code or fix the doc.

| Item | Doc says | Code says | Notes |
|---|---|---|---|
| Cone of Cold SLOWED duration | 2 rounds at MASTER, 3 at GM | 1 round at MASTER, 2 at GM | The doc historically claimed 2/3; the code implements 1/2 via `_SLOW_ROUNDS = {4: 1, 5: 2}`. Pick one — update the table above and the code. Either is defensible from a balance standpoint. |

