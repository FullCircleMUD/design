# Weapon Damage Scaling

> Material tier + mastery level = damage dice. One lookup table per 5e base die, shared by all weapons of that class.

## Core Principles

1. **Iron at BASIC mastery = 5e D&D baseline.** An Iron Longsword (d8 base) in the hands of a BASIC swordsman does 1d8 — exactly what 5e says.

2. **One mastery step = one material step.** A SKILLED wielder with a Bronze weapon does the same damage as a BASIC wielder with an Iron weapon. The diagonal is symmetric.

3. **Grandmaster = 2x Basic.** At every material tier, a GM always does double the average damage of a BASIC wielder of the same weapon. Achieved by doubling the dice count.

4. **Unskilled ≈ half of Basic.** Floored to 1 minimum at the lowest material tiers.

5. **Wood and Bronze share a GM ceiling (d4 only).** For the smallest base die, the numbers are too compressed to maintain a meaningful wood→bronze step at GM. Wood GM = Bronze GM for d4 weapons only. Larger base dice have enough room for the natural progression throughout.

## Material Tiers (Power Tiers)

The `material` field on a weapon prototype is a **power tier**, not necessarily a literal description of what the weapon is made from. The tier names (wood, bronze, iron, steel, adamantine) are convenient labels borrowed from the metal progression, but any weapon can be assigned to any tier based on where it should sit on the damage curve.

For example, a quarterstaff is physically made of wood but is a proper weapon — not a training stick. It maps to **bronze** tier (tier 1). A future Ironwood Quarterstaff made from magically hardened wood might map to **steel** tier (tier 3). The physical material is flavour; the tier is mechanical.

Builders should use their judgement when assigning tiers to non-metal weapons. The question to ask is: *"Where should this weapon sit on the damage curve relative to the standard metal progression?"*

| Tier | Label | Typical Metal | Power Level | Examples |
|---|---|---|---|---|
| 0 | `"wood"` | — | Training / worst | Wooden training weapons, improvised weapons |
| 1 | `"bronze"` | Bronze (Copper + Tin) | Starter real weapons | Bronze weapons, standard quarterstaff, basic sling |
| 2 | `"iron"` | Iron | Standard military | Iron weapons, composite bow, war sling |
| 3 | `"steel"` | Steel (Iron + Coal) | Advanced | Steel weapons, ironwood quarterstaff |
| 4 | `"adamantine"` | Adamantine | Endgame legendary | Adamantine weapons, divine/mythic variants |

**Source zones for metal weapons:**

| Tier | Source Zone | Mastery Gate |
|---|---|---|
| 0 | Millholm (training weapons) | — |
| 1 | Millholm | BASIC |
| 2 | Ironback Peaks | SKILLED |
| 3 | Inner Islands | EXPERT |
| 4 | Vaathari / Olympus | GRANDMASTER |

> Mithril is intentionally absent from the standard progression. It occupies a separate role (light armor, magical resonance) rather than being a straight damage upgrade between Steel and Adamantine.

## Damage Lookup Tables

Each table is keyed by **(base_die, material_tier)**. The prototype declares these two values; the full mastery→dice mapping is resolved from the table. No per-prototype damage dicts.

### d4 Base (Dagger, Blowgun)

| Material | UNSKILLED | BASIC | SKILLED | EXPERT | MASTER | GM |
|---|---|---|---|---|---|---|
| **Wood** | 1 (1) | 1d2 (1.5) | 1d3 (2) | **1d4 (2.5)** | 1d5 (3) | 2d3 (4) |
| **Bronze** | 1 (1) | 1d3 (2) | **1d4 (2.5)** | 1d5 (3) | 1d6 (3.5) | 2d3 (4) |
| **Iron** | 1d2 (1.5) | **1d4 (2.5)** | 1d5 (3) | 1d6 (3.5) | 1d7 (4) | 2d4 (5) |
| **Steel** | 1d2 (1.5) | 1d5 (3) | 1d6 (3.5) | 1d7 (4) | 2d4 (5) | 2d5 (6) |
| **Adamantine** | 1d3 (2) | 1d6 (3.5) | 1d7 (4) | 2d4 (5) | 2d5 (6) | 2d6 (7) |

**Bold** = 5e baseline (1d4) on the diagonal.

### d6 Base (Shortsword, Handaxe, Mace, Club, Sai, Nunchaku)

| Material | UNSKILLED | BASIC | SKILLED | EXPERT | MASTER | GM |
|---|---|---|---|---|---|---|
| **Wood** | 1d2 (1.5) | 1d4 (2.5) | 1d5 (3) | **1d6 (3.5)** | 1d7 (4) | 2d4 (5) |
| **Bronze** | 1d2 (1.5) | 1d5 (3) | **1d6 (3.5)** | 1d7 (4) | 2d4 (5) | 2d5 (6) |
| **Iron** | 1d3 (2) | **1d6 (3.5)** | 1d7 (4) | 2d4 (5) | 2d5 (6) | 2d6 (7) |
| **Steel** | 1d3 (2) | 1d7 (4) | 2d4 (5) | 2d5 (6) | 2d6 (7) | 2d7 (8) |
| **Adamantine** | 1d4 (2.5) | 2d4 (5) | 2d5 (6) | 2d6 (7) | 2d7 (8) | 4d4 (10) |

**Bold** = 5e baseline (1d6) on the diagonal.

### d8 Base (Longsword, Rapier, Spear, Staff, Bow, Crossbow, Ninjato)

| Material | UNSKILLED | BASIC | SKILLED | EXPERT | MASTER | GM |
|---|---|---|---|---|---|---|
| **Wood** | 1d3 (2) | 1d6 (3.5) | 1d7 (4) | **1d8 (4.5)** | 1d10 (5.5) | 2d6 (7) |
| **Bronze** | 1d3 (2) | 1d7 (4) | **1d8 (4.5)** | 1d10 (5.5) | 2d6 (7) | 2d7 (8) |
| **Iron** | 1d4 (2.5) | **1d8 (4.5)** | 1d10 (5.5) | 2d6 (7) | 2d7 (8) | 2d8 (9) |
| **Steel** | 1d5 (3) | 1d10 (5.5) | 2d6 (7) | 2d7 (8) | 2d8 (9) | 2d10 (11) |
| **Adamantine** | 1d6 (3.5) | 2d6 (7) | 2d7 (8) | 2d8 (9) | 2d10 (11) | 4d6 (14) |

**Bold** = 5e baseline (1d8) on the diagonal.

### d10 Base (Greatclub, Battleaxe, Halberd)

| Material | UNSKILLED | BASIC | SKILLED | EXPERT | MASTER | GM |
|---|---|---|---|---|---|---|
| **Wood** | 1d3 (2) | 1d7 (4) | 1d8 (4.5) | **1d10 (5.5)** | 1d12 (6.5) | 2d7 (8) |
| **Bronze** | 1d4 (2.5) | 1d8 (4.5) | **1d10 (5.5)** | 1d12 (6.5) | 2d7 (8) | 2d8 (9) |
| **Iron** | 1d5 (3) | **1d10 (5.5)** | 1d12 (6.5) | 2d7 (8) | 2d8 (9) | 2d10 (11) |
| **Steel** | 1d6 (3.5) | 1d12 (6.5) | 2d7 (8) | 2d8 (9) | 2d10 (11) | 2d12 (13) |
| **Adamantine** | 1d7 (4) | 2d7 (8) | 2d8 (9) | 2d10 (11) | 2d12 (13) | 4d7 (16) |

**Bold** = 5e baseline (1d10) on the diagonal.

### d12 Base (Greataxe)

| Material | UNSKILLED | BASIC | SKILLED | EXPERT | MASTER | GM |
|---|---|---|---|---|---|---|
| **Wood** | 1d4 (2.5) | 1d8 (4.5) | 1d10 (5.5) | **1d12 (6.5)** | 2d7 (8) | 2d8 (9) |
| **Bronze** | 1d5 (3) | 1d10 (5.5) | **1d12 (6.5)** | 2d7 (8) | 2d8 (9) | 2d10 (11) |
| **Iron** | 1d6 (3.5) | **1d12 (6.5)** | 2d7 (8) | 2d8 (9) | 2d10 (11) | 2d12 (13) |
| **Steel** | 1d7 (4) | 2d7 (8) | 2d8 (9) | 2d10 (11) | 2d12 (13) | 4d7 (16) |
| **Adamantine** | 1d8 (4.5) | 2d8 (9) | 2d10 (11) | 2d12 (13) | 4d7 (16) | 4d8 (18) |

**Bold** = 5e baseline (1d12) on the diagonal.

### 2d6 Base (Greatsword)

| Material | UNSKILLED | BASIC | SKILLED | EXPERT | MASTER | GM |
|---|---|---|---|---|---|---|
| **Wood** | 1d4 (2.5) | 2d4 (5) | 2d5 (6) | **2d6 (7)** | 2d7 (8) | 4d4 (10) |
| **Bronze** | 1d5 (3) | 2d5 (6) | **2d6 (7)** | 2d7 (8) | 4d4 (10) | 4d5 (12) |
| **Iron** | 1d6 (3.5) | **2d6 (7)** | 2d7 (8) | 4d4 (10) | 4d5 (12) | 4d6 (14) |
| **Steel** | 1d7 (4) | 2d7 (8) | 4d4 (10) | 4d5 (12) | 4d6 (14) | 4d7 (16) |
| **Adamantine** | 2d4 (5) | 4d4 (10) | 4d5 (12) | 4d6 (14) | 4d7 (16) | 8d4 (20) |

**Bold** = 5e baseline (2d6) on the diagonal. Adamantine GM = 8d4 (avg 20) — tight bell curve, consistent peak damage for the ultimate greatsword.

### 2d7 Base (Lance — mounted cavalry weapon)

| Material | UNSKILLED | BASIC | SKILLED | EXPERT | MASTER | GM |
|---|---|---|---|---|---|---|
| **Wood** | 1d5 (3) | 2d5 (6) | 2d6 (7) | **2d7 (8)** | 2d8 (9) | 4d5 (12) |
| **Bronze** | 1d6 (3.5) | 2d6 (7) | **2d7 (8)** | 2d8 (9) | 4d5 (12) | 4d6 (14) |
| **Iron** | 1d7 (4) | **2d7 (8)** | 2d8 (9) | 4d5 (12) | 4d6 (14) | 4d7 (16) |
| **Steel** | 2d4 (5) | 2d8 (9) | 4d5 (12) | 4d6 (14) | 4d7 (16) | 4d8 (18) |
| **Adamantine** | 2d5 (6) | 4d5 (12) | 4d6 (14) | 4d7 (16) | 4d8 (18) | 8d5 (24) |

**Bold** = 5e baseline (2d7) on the diagonal. Adamantine GM = 8d5 (avg 24). The lance earns the highest per-hit damage in the game to justify being useless indoors — all mechanics are gated behind the mount requirement.

## Speed and Initiative

All weapons share the same combat tick interval (`COMBAT_TICK_INTERVAL` in settings, default 4.0s). Speed no longer affects tick rate — it affects **initiative** instead. Higher speed = faster to react = acts first in combat.

Initiative roll: `d20 + effective_initiative (DEX-based) + weapon speed`

### Weapon Speed Assignments (0-3 Scale)

| Speed | Melee | Ranged | Fantasy |
|---|---|---|---|
| **0** | Greatsword, Battleaxe, Greatclub, Lance | Crossbow | Heaviest / slowest reload |
| **1** | Longsword, Rapier, Spear, Staff, Hammer | Bow | Standard weapons |
| **2** | Shortsword, Handaxe, Club, Mace | Sling | Light weapons |
| **3** | Dagger, Unarmed | — | Fastest draw / no weapon |

A dagger gives +3 initiative over a greatsword. On a d20, weapon speed and DEX modifier have roughly equal weight — a dexterous character (DEX 16, +3 mod) with a greatsword (speed 0) has the same initiative bonus as an average character (DEX 10, +0 mod) with a dagger (speed 3). Weapon choice and natural agility are balanced against each other.

### Mob Initiative Speed

Animal mobs and unarmed NPCs have no weapon, so they use `initiative_speed` (AttributeProperty on CombatMob, default 0). This mirrors weapon speed for players — higher = faster / more agile. Scale matches the 0-3 weapon speed range.

| Speed | Mobs |
|---|---|
| **3** | Rabbit, Crow, Cellar Rat |
| **2** | Wolf, Kobold |
| **1** | Dire Wolf, Kobold Chieftain, Rat King |
| **0** | Gnoll, Gnoll Warlord, Training Dummy |

## Baseline Balance — Reference Weapons

The following five weapons form the balance baseline. All other weapons should be evaluated against this group. All examples use **Adamantine at GM mastery** with a **uniform tick interval**.

### Weapon Identities

**Dagger (d4)** — The assassin's tool. Low sustained damage, but crits on 18+ and Sneak Attack (+10d6 from stealth) provide devastating burst. Finesse (DEX). Three attacks at GM (1 base + 1 extra + 1 off-hand). No parries, no defence — pure glass cannon with stealth utility.

**Shortsword (d6)** — The working man's weapon. Common soldiers, archers, rangers, thieves. Gets its second attack at EXPERT — a full tier before the longsword — making it the strongest mid-game melee option. Finesse (DEX). Light parry (1 at SKILLED, 2 at GM). Outshined at endgame by the longsword's raw damage, but earns its keep through the levels where it matters most.

**Rapier (d8)** — The specialist's weapon. Only 1 base attack, but 3 parries with riposte at GM. Output scales with incoming attacks — the more aggressively you're attacked, the more damage it deals back. The counter-fighter that gets better when surrounded or facing multi-attack bosses. Finesse (DEX). Parry and riposte only work against armed opponents — useless against animals, unarmed mobs, and natural attacks.

**Longsword (d8)** — The reliable choice. 2 attacks at GM, 3 parries (block only, no riposte). Consistent 28 damage per tick regardless of scenario. No spike potential, no weakness. Pairs with a shield for the most survivable melee setup. STR-based. The safe pick that never lets you down.

**Greatsword (2d6)** — The battlefield beast. One massive swing per tick with 75% cascading cleave through additional targets. Mediocre 1v1 (20 damage), devastating against groups (55+ at 4 targets). Executioner at GM grants a bonus attack on kills, which can itself cleave — snowball potential in group fights. Two-handed (no shield), no parries. The warrior who wades into a mob.

### Single-Target Damage Per Tick (Adamantine GM)

| Weapon | Avg Hit | Attacks | Dmg/Tick | Parries |
|---|---|---|---|---|
| Dagger (dual) | 7 | 3 | **21** | 0 |
| Shortsword (dual) | 10 | 2 | **20** | 2 block |
| Rapier | 14 | 1 (+riposte) | **14 base** | 3 riposte |
| Longsword | 14 | 2 | **28** | 3 block |
| Greatsword | 20 | 1 (+cleave) | **20 base** | 0 |

### Output Scaling With Incoming Attacks

Rapier riposte triggers on each parried attack (armed opponents only). Greatsword cleave triggers on each additional target. All others are flat.

| Incoming Attacks/Tick | Dagger | Shortsword | Rapier | Longsword | Greatsword |
|---|---|---|---|---|---|
| 1 | 21 | 20 | 21 | **28** | 20 |
| 2 | 21 | 20 | 28 | **28** | 35 |
| 3 | 21 | 20 | **35** | 28 | 46 |
| 4+ | 21 | 20 | **35** | 28 | **55+** |

> Note: "Incoming attacks" for rapier means attacks from armed opponents (parry/riposte). For greatsword, the column represents number of available targets for cleave, not incoming attacks.

### Three Balance Tiers (Single Target)

- **Tier 1 — Consistent high output:** Longsword (28). Always reliable, strong defence.
- **Tier 2 — Baseline cluster:** Dagger (21), Shortsword (20), Rapier (21 floor), Greatsword (20 floor). Roughly equal base output, differentiated by niche mechanics.
- **Tier 3 — Scaling weapons:** Rapier scales up with incoming attacks (to 35). Greatsword scales up with target count (to 55+). These weapons have the highest ceilings but the lowest floors.

The longsword's 28 vs the cluster's ~20 is a 40% gap. This is the price the cluster weapons pay for their niches — burst (dagger), early maturity (shortsword), counter-fighting (rapier), and AoE (greatsword). The longsword pays for its consistency by having no spike potential and no special mechanic beyond reliable damage + defence.

### Mastery Progression — Second Attack Milestone

When each weapon reaches 2 attacks per tick (the biggest single power spike in the mastery path):

| Weapon | 2nd Attack At | Method |
|---|---|---|
| Dagger | SKILLED | extra attack |
| Shortsword | EXPERT | off-hand (-4 penalty) |
| Longsword | MASTER | extra attack |
| Greatsword | never | cleave instead |

The shortsword peaks a full tier before the longsword. For a mid-game character at EXPERT mastery, the shortsword outputs double the longsword. This is the shortsword's design intent — the practical weapon that delivers results through the mid-game, before the longsword's raw damage overtakes it at endgame.

### Shortsword Mastery Progression (Revised)

| Mastery | Hit | Parries | Off-hand | Penalty | Total Attacks |
|---|---|---|---|---|---|
| UNSKILLED | -2 | 0 | 0 | — | 1 |
| BASIC | 0 | 0 | 0 | — | 1 |
| SKILLED | +2 | 1 | 0 | — | 1 |
| EXPERT | +4 | 1 | 1 | -4 | 2 |
| MASTER | +6 | 1 | 1 | -2 | 2 |
| GM | +8 | 2 | 1 | 0 | 2 |

### Trade-Off Summary

| Weapon | Offence | Defence | Best Scenario | Worst Scenario | Stat |
|---|---|---|---|---|---|
| Dagger | burst from stealth | none | ambush opener | sustained fight | DEX |
| Shortsword | early 2nd attack | light (1-2 parry) | mid-game, versatile | outscaled at GM | DEX |
| Rapier | reactive (riposte) | strong (3 parry) | armed multi-attackers | unarmed/animals | DEX |
| Longsword | consistent (2 att) | strong (3 parry) | any 1v1 | no spike potential | STR |
| Greatsword | AoE (cleave) | none | groups of 2+ | single tough target | STR |

## Heavy Weapons Balance — Speed 0 Two-Handers

Four heavy two-handed weapons share speed 0 (slowest initiative) and no parries. Each has a distinct party role and mechanical identity. None is strictly better than another — the choice depends on scenario, party composition, and content type.

### Heavy Weapons Summary (Adamantine GM)

| Weapon | Base | Avg Hit | Cleave | Exclusive Mechanic | Classes |
|---|---|---|---|---|---|
| **Greatsword** | 2d6 | 20 | 75/50/25% + Executioner | Kill chains (free attack on kill, can itself cleave) | Warrior |
| **Battleaxe** | d10 | 16 | 60/40/20% | Sunder (stacking AC shred + armour durability) | Warrior |
| **Greatclub** | d10 | 16 | None | Heavy Stagger (30%, -4 hit, 2 rounds) | Warrior, Cleric, Druid |
| **Lance** | 2d7 | 24 | 75/50/25% | Crit on 18+ (-2) + Prone 25% (mounted only) | Warrior |

### Balance Rationale

**Greatsword (2d6)** — The AoE king. Highest base damage among always-available weapons. Executioner is the capstone: on any kill (primary or cleave), fire a free full attack on another living enemy that can itself cleave but cannot chain executioner. Against groups of weak mobs, the greatsword can potentially hit 6-8 targets in one tick via kill chain snowballs. Against a tough boss that doesn't die, executioner never triggers and it's just a 20 avg single hit. Works everywhere.

**Battleaxe (d10)** — The team utility bruiser. Same base damage as the greatclub but trades stagger for nerfed cleave + sunder. Sunder procs on every hit (primary AND cleave), stacking AC penalties on each target hit. After 3-4 rounds, the whole enemy group has reduced AC and damaged armour, making the entire party hit harder. Anti-tank specialist. The 20% gap from greatsword (16 vs 20) is the price for getting both AoE and sunder.

**Greatclub (d10)** — The cleric/druid heavy option. Same base damage as the battleaxe but no cleave at all — just a single big hit with heavy stagger chance. The stagger debuff (-4 hit for 2 rounds at GM) is defensive value through offence — a staggered enemy misses more attacks. The only speed-0 weapon accessible to clerics and druids. If they want cleave and executioner, they roll a warrior.

**Lance (2d7)** — The mounted specialist. Highest per-hit damage in the game (Adam GM avg 24, 31% above greatsword when crits are factored in). Same cleave rates as the greatsword but no executioner. Adds crit on 18+ and 25% prone on primary hit. The massive trade-off: **all mechanics are gated behind the mount requirement**. Unmounted, the lance imposes disadvantage on all attacks, caps at 1 attack, no crit bonus, no prone — effectively ~12 DPS. This makes the lance useless in dungeons, indoors, and any content where mounts cannot be used (~50% of the game). The 2d7 base die compensates for this restriction — the lance player dominates outdoor content but must carry a backup weapon for everything else, splitting mastery investment.

### Party Composition Example

Against a boss with adds (outdoor encounter):
- **Greatsword** clears the adds with cleave + executioner chains
- **Lance** locks down the boss with prone (advantage for everyone) + highest crit DPS
- **Battleaxe** sunders the boss's armour so the whole party hits harder
- **Greatclub cleric** staggers threats that reach the backline

The system is symmetrical — mobs use the same combat pipeline. An NPC party with these same weapons is equally dangerous to the players.

### Executioner Mechanics (Greatsword GM Only)

Executioner fires on any kill — primary attack or cleave hit. The free attack can itself cleave but cannot trigger a second executioner (capped at 1 per round via `combat_handler.executioner_used` flag). Maximum theoretical output in one tick:

1. Primary → hits target A
2. Cleave 1 (75%) → hits target B
3. Cleave 2 (50%) → hits target C
4. Cleave 3 (25%) → **kills target C** → executioner fires
5. Executioner → fresh attack on target D
6. Exec cleave 1 (75%) → hits target E
7. Exec cleave 2 (50%) → hits target F
8. Exec cleave 3 (25%) → hits target G

Up to 8 damage rolls in one tick, but requires the 25% cleave to succeed AND that hit to kill. In practice, 3-4 hits per tick against weak mobs is the common case.

## Medium Weapons Balance — d8 One-Handers

The d8 weapons are the "real" weapons — each has a distinct combat identity. All deal 14 avg per hit at Adamantine GM.

### d8 Weapon Summary (Adamantine GM)

| Weapon | Attacks | Dmg/Tick | Parries | Special | Shield | STR/DEX |
|---|---|---|---|---|---|---|
| **Longsword** | 2 | 28 | 3 block | — | yes | STR |
| **Rapier** | 1 (+riposte) | 14 base | 3 riposte | counter-fighter | yes | DEX |
| **Spear** | 1 (+counters) | 14 base | 0 | 2 reach counters, crit 18+ | no (2H) | STR |
| **Staff** | 1 (+riposte) | 14 base | 4 riposte | universal parry | no (2H) | STR |
| **Hammer** | 1 | 14 | 0 | 4x crit multiplier | yes | STR |

### d8 Weapon Identities

**Longsword** — The reliable choice. Best sustained 1v1 damage (28/tick) plus 3 parries. No spike potential, no utility mechanic — just consistent damage and solid defence against armed opponents. The weapon you can never go wrong with. Pairs with a shield for the most survivable melee setup in the game.

**Rapier** — The duelist's counter-fighter. Only 1 base attack, but 3 parries with riposte at GM. Output scales with incoming armed attacks — the more aggressively you're attacked, the more damage it deals back. Against a boss with 3+ weapon attacks per tick, the rapier outdamages the longsword. Against animals, unarmed mobs, or single slow hitters, the rapier is the weakest d8 weapon. Finesse (DEX) makes it the rogue/bard's sword. Parry and riposte only work against armed opponents.

**Spear** — The bodyguard. Two-handed reach weapon with alternating crit/counter mastery progression: crit on 19+ at SKILLED, first counter at EXPERT, crit on 18+ at MASTER, second counter at GM. When an enemy hits an ally in the same combat, the spear wielder fires a free counter-attack. Useless solo (just 14/tick), powerful in a party protecting a tank. Against a boss hitting the tank 2 times per tick, the spear deals 14 + (2 × 14) = 42 effective DPS with crits pushing higher. Two-handed means no shield, no parries — the spear wielder relies entirely on the tank for protection. If the tank goes down, the spear user is exposed with 14 DPS and zero defence. Counter-attacks cannot be parried and do not cascade.

**Staff** — The survivor. Worst offensive d8 weapon (14/tick, no extra attacks, no offensive mechanic), but the best defensive weapon in the game. 4 parries at GM that work against ALL physical attacks — armed, unarmed, animal bites, and missile attacks. Every other parry weapon only blocks armed melee. Two-handed (no shield), but combined with Mage Armor (+5 AC) and Shield spell (+6 AC reactive), an abjuration mage with a GM staff is a physical fortress. The Matt Cauthon weapon — terrible damage, unkillable. Vulnerable to magical attacks (spells bypass parries). Parries are automatic — the first 4 physical attacks each round regardless of source, no targeting needed.

**Hammer** — The gambler. Lowest sustained DPS of any d8 weapon (14/tick) but the biggest single-hit spike in the game. Devastating Blow multiplies already-doubled crit damage by mastery-scaled factor (up to 2.0x at GM = 4x total base damage). One GM crit = 14 × 4 = 56 damage in a single hit. Build-around weapon — stack crit threshold reduction gear to turn the gamble into a strategy. No parries, no extra attacks, no utility. Pure high-risk/high-reward.

## Light Weapons Balance — d6 One-Handers

The d6 weapons are the practical tier — lighter versions of heavier weapons, accessible to more classes, and effective with a shield. All deal 10 avg per hit at Adamantine GM.

### d6 Weapon Summary (Adamantine GM)

| Weapon | Attacks | Dmg/Tick | Parries | Special | Shield | STR/DEX |
|---|---|---|---|---|---|---|
| **Shortsword** | 2 (1+1 off) | 20 | 2 block | early 2nd attack (EXPERT) | no (off-hand) | DEX |
| **Handaxe** | 2 | 20 | 0 | light sunder (20%, -1 AC) | yes | STR |
| **Club** | 2 | 20 | 0 | light stagger (20%, -2 hit) | yes | STR |
| **Mace** | 2 | 20-36 | 0 | crush (2/4/6/8 vs armour) | yes | STR |

### d6 Weapon Identities

**Shortsword** — The working man's blade. Longsword-light — gets its 2nd attack at EXPERT, a full tier before the longsword's MASTER. The strongest mid-game melee option, outshined at endgame by the longsword's d8 damage. Finesse (DEX) and dual-wield make it the thief/ranger's bread and butter. 2 parries at GM give light defence. The practical choice for characters who need results now, not at endgame.

**Handaxe** — The one-handed sunder. Mini battleaxe — lighter sunder (20% chance, -1 AC per proc, always) for characters who want armour shredding but can't afford to go two-handed. Pairs with a shield for balanced offence/defence. No parries — the sunder IS the utility. Party-wide value: a sundered target is easier for everyone to hit.

**Club** — The everyman's stagger. Mini greatclub — lighter stagger (20% chance, -2 hit, 1 round) for clerics and druids who want CC with a shield. Same base damage as every other d6 weapon, differentiated purely by the stagger debuff reducing enemy accuracy. The cleric's stepping stone before the greatclub.

**Mace** — The armour cracker. d6 base means lower damage than the longsword (d8) against unarmoured targets, but crush bonus (2/4/6/8 cap scaling with mastery) adds damage based on target's equipment AC. Against plate wearers (AC 18+) the mace at GM deals 36/tick — 29% above the longsword. Against unarmoured targets it's 20/tick — 29% below. No parries. A deliberate bet: "I'm fighting armoured humanoids and I have a healer." Available to clerics (unlike longsword), making it the cleric's premium offensive weapon against armoured undead and plate-wearing enemies.

### d6 Tier Design Pattern

Each d6 weapon is a lighter version of a heavier weapon, trading base damage for shield compatibility:

| d6 Weapon | Big Brother | What you trade | What you keep |
|---|---|---|---|
| **Shortsword** | Longsword (d8) | d8 damage, 3 parries | early 2nd attack, DEX, 2 parries |
| **Handaxe** | Battleaxe (d10) | d10 damage, cleave | sunder, shield |
| **Club** | Greatclub (d10) | d10 damage | stagger, shield |
| **Mace** | — (unique) | defence (0 parries) | crush vs armour, shield |

### Club Material Progression

| Weapon | Material Tier | Craft |
|---|---|---|
| Wooden Club | wood (tier 0) | BASIC carpentry — 2 Timber |
| Bronze Spiked Club | bronze (tier 1) | BASIC blacksmithing — 1 Bronze Ingot + 1 Wooden Club |
| Iron Spiked Club | iron (tier 2) | SKILLED blacksmithing — 1 Iron Ingot + 1 Wooden Club |
| *Steel Spiked Club* | steel (tier 3) | future |
| *Adamantine Spiked Club* | adamantine (tier 4) | future |

## Prototype Format

Prototypes declare `base_damage` (which lookup table) and `material` (which power tier):

```python
BRONZE_DAGGER = {
    "prototype_key": "bronze_dagger",
    "typeclass": "typeclasses.items.weapons.dagger_nft_item.DaggerNFTItem",
    "key": "Bronze Dagger",
    "base_damage": "d4",
    "material": "bronze",       # tier 1 — literally bronze
    "damage_type": DamageType.PIERCING,
    "speed": 4,
    "weight": 0.5,
    "max_durability": 3600,
}

QUARTERSTAFF = {
    "prototype_key": "quarterstaff",
    "typeclass": "typeclasses.items.weapons.staff_nft_item.StaffNFTItem",
    "key": "Quarterstaff",
    "base_damage": "d8",
    "material": "bronze",       # tier 1 — physically wood, but a proper weapon
    "damage_type": DamageType.BLUDGEONING,
    "speed": 2,
    "weight": 2.0,
    "max_durability": 1440,
}
```

## Ninja Weapons Balance — Remort-Gated Specialists

Ninja weapons require the Ninja class, which requires remort. They should be noticeably better than standard weapons (~20-25% premium) as a reward for the remort investment, but not double. Each has a distinct role that doesn't overlap with standard weapons.

### Ninja Weapon Summary (Adamantine GM)

| Weapon | Base | Attacks | Dmg/Tick | Parries | Unique Mechanic | Speed |
|---|---|---|---|---|---|---|
| **Ninjato** | d8 | 2 | 28 (+14 riposte) | 2 block + 1 riposte | Auto-crit opener (assassinate), two-handed | 1 |
| **Sai** (×2) | d4 | 1 per sai | 14 | 10 block (5 per sai) | Disarm on every parry, no size limit | 2 |
| **Nunchaku** | d4 | 3 | 21 | 0 | Every hit = stun check (DEX vs CON), two-handed, GARGANTUAN only immune | 2 |
| **Shuriken** | d4 | 4 | 28 (~32 eff) | 0 | Consumable + crit 18+, 4 throws/tick | 3 |

### Ninja Weapon Identities

**Ninjato** — The ninja's longsword. A d8 finesse two-handed blade wielded in the Japanese sword style. Mirrors the longsword progression (extra attack at EXPERT, parries at SKILLED/MASTER) but trades one block parry for a riposte at GM. The unique capstone is **auto-crit on combat opener** — the instigator's free attack always crits when wielding a ninjato. Combined with the future assassinate command's damage multiplier, this creates the single deadliest opening strike in the game. In sustained combat the ninjato is longsword-equivalent (28 DPS) with a situational riposte bonus (up to 42 vs 1+ incoming). The ninja doesn't want a long fight — they want one perfect moment. Two-handed — no shield, no off-hand. The 2nd attack comes from `get_extra_attacks()`, not dual-wield.

**Ninjato Mastery Progression:**

| Mastery | Hit | Parries | Extra Attacks | New Unlock |
|---|---|---|---|---|
| UNSKILLED | -2 | 0 | 0 | — |
| BASIC | 0 | 0 | 0 | Assassinate (auto-crit opener) |
| SKILLED | +2 | 1 block | 0 | First parry |
| EXPERT | +4 | 1 block | +1 | Second attack |
| MASTER | +6 | 2 block | +1 | Second parry + parry advantage |
| GM | +8 | 2 block + 1 riposte | +1 | Riposte |

**Sai** — The disarm specialist. No other weapon in the game can strip an enemy's weapon. d4 base with only 1 attack — the sai is a pure utility weapon, not a damage dealer (7 DPS at GM, lowest in the game). The sai's power is its **parry-to-disarm** mechanic: every successful parry triggers a contested disarm roll (DEX + mastery vs target STR + mastery). Dual-wield two sais for up to 10 parries at GM — the highest parry count in the game. No size restriction on disarm — if it's wielding a weapon, the sai can take it, even from a gargantuan creature. The sai ninja in a PvP arena can disarm an entire opposing team in 2 rounds. Counter-play: unarmed/animal attacks bypass parries, magic bypasses parries, re-equipping costs an action but gets the weapon back.

**Sai Mastery Progression (per sai):**

| Mastery | Hit | Attacks | Parries | Disarm on Parry | New Unlock |
|---|---|---|---|---|---|
| UNSKILLED | -2 | 1 | 0 | no | — |
| BASIC | 0 | 1 | 1 | yes | First parry + disarm |
| SKILLED | +2 | 1 | 2 | yes | Second parry |
| EXPERT | +4 | 1 | 3 | yes | Third parry |
| MASTER | +6 | 1 | 4 | yes | Fourth parry |
| GM | +8 | 1 | 5 | yes | Fifth parry |

Dual-wielding two sais doubles the parry count (up to 10 at GM). Each parry is a contested roll (not auto-block), and each successful parry is a second contested roll for disarm. Two contested rolls to strip a weapon — the sai demands high DEX to be effective.

**Nunchaku** — The stun-lock weapon. Two-handed, d4 base — low damage per hit but lots of attacks, and **every hit is a stun check**. The attack's hit roll is contested against the target's d20 + CON — no separate stun roll needed. More attacks = more stun chances. Compare against club (light stagger, flat %, one-handed) and greatclub (heavy stagger, flat %, two-handed). The nunchaku is skill-dependent (DEX vs CON) rather than luck-dependent (flat %). A high-DEX ninja against a low-CON target is a near-permanent stun-lock. At MASTER+ a decisive win (margin ≥5) knocks the target PRONE instead. Two-handed means no shield, no off-hand — pure CC. Only **GARGANTUAN** is immune (unlike other stun weapons which cut off at HUGE+) — the remort premium lets ninjas stun-lock ogres and giants. `can_dual_wield = False`.

**Nunchaku Mastery Progression:**

| Mastery | Hit | Extra Attacks | Total Attacks | Stun Duration | Prone (≥5) | New Unlock |
|---|---|---|---|---|---|---|
| UNSKILLED | -2 | 0 | 1 | — | no | — |
| BASIC | 0 | 0 | 1 | 1 round | no | Stun on every hit |
| SKILLED | +2 | +1 | 2 | 1 round | no | Second attack |
| EXPERT | +4 | +1 | 2 | 2 rounds | no | Longer stun |
| MASTER | +6 | +2 | 3 | 2 rounds | yes | Third attack + prone |
| GM | +8 | +2 | 3 | 3 rounds | yes | 3-round stun/prone |

**Shuriken** — Ranged burst. d4 base with 4 throws at GM and crit on 18+. Each throw consumes the shuriken — hit lodges in target's inventory, miss drops to floor. Both recoverable after combat. The logistical cost (carry 20+, recover after fights) is the balancing factor against the bow's unlimited ammunition. At GM with crits, 4 throws at d4 Adamantine (avg 7 each) = 28 base, effective ~32 with 15% crit rate. Above the bow's 28 but you burn through 4 shuriken per tick — a pouch of 20 lasts 5 ticks. No special mechanics — pure damage with a supply constraint. The ninja empties the pouch fast, then switches to ninjato for the sustained fight.

**Shuriken Mastery Progression:**

| Mastery | Hit | Throws | Crit Threshold | New Unlock |
|---|---|---|---|---|
| UNSKILLED | -2 | 1 | 20 | — |
| BASIC | 0 | 1 | 20 | — |
| SKILLED | +2 | 2 | 19 (-1) | Second throw + crit bonus |
| EXPERT | +4 | 2 | 19 | — (hit bonus only) |
| MASTER | +6 | 3 | 18 (-2) | Third throw + better crits |
| GM | +8 | 4 | 18 | Fourth throw |

### Ninja Loadout Choices

Each ninja weapon is a complete combat style — the ninja picks one for the fight:

| Loadout | Hands | Role |
|---|---|---|
| **Ninjato** | two-handed | Assassin — auto-crit opener, sustained d8 damage, parries + riposte |
| **Sai + Sai** | dual-wield | Disarmer — strip weapons, high parry count, minimal damage |
| **Nunchaku** | two-handed | Stunner — flurry of stun checks, CC supremacy |
| **Shuriken** | one-handed (thrown) | Ranged — consumable burst, crit-fishing |

## Lookup Resolution

A single module (e.g. `utils/damage_tables.py` or `world/damage_tables.py`) holds the tables as nested dicts:

```python
DAMAGE_TABLES = {
    "d4": {
        "wood":       {UNSKILLED: "1",   BASIC: "1d2", SKILLED: "1d3", EXPERT: "1d4", MASTER: "1d5", GM: "2d3"},
        "bronze":     {UNSKILLED: "1",   BASIC: "1d3", SKILLED: "1d4", EXPERT: "1d5", MASTER: "1d6", GM: "2d3"},
        "iron":       {UNSKILLED: "1d2", BASIC: "1d4", SKILLED: "1d5", EXPERT: "1d6", MASTER: "1d7", GM: "2d4"},
        "steel":      {UNSKILLED: "1d2", BASIC: "1d5", SKILLED: "1d6", EXPERT: "1d7", MASTER: "2d4", GM: "2d5"},
        "adamantine": {UNSKILLED: "1d3", BASIC: "1d6", SKILLED: "1d7", EXPERT: "2d4", MASTER: "2d5", GM: "2d6"},
    },
    "d8": {
        ...
    },
}

def get_damage_dice(base_damage: str, material: str, mastery: MasteryLevel) -> str:
    """Resolve damage dice string from base die, material, and wielder mastery."""
    return DAMAGE_TABLES[base_damage][material][mastery]
```

`WeaponNFTItem.get_damage_roll()` calls this function instead of reading from a per-instance `damage` dict.

## Integration Points

### WeaponNFTItem

- Remove the `damage` AttributeProperty (mastery→dice dict) from the class.
- Add `base_damage` and `material` AttributeProperties (set from prototype on spawn).
- `get_damage_roll(mastery)` resolves via `get_damage_dice(self.base_damage, self.material, mastery)`.

### Existing Prototypes

- Migrate all weapon prototypes: replace `damage: {...}` with `base_damage` + `material`.
- Verify each weapon's 5e base die classification is correct.

### Enchanted / Legendary Weapons

Enchanted weapons and named legendaries can still override damage if needed — either via a `damage_bonus` (e.g. "+1d6 fire") layered on top, or in rare cases a direct `damage` dict override that bypasses the table entirely. The table handles the 95% case; exceptions remain possible.

### Special Weapons

Some weapons have fixed damage regardless of material (e.g. Blowgun = always 1, Bola = always 1). These can either:
- Use a special `"fixed"` material entry in the table, or
- Continue to use a `damage` dict override on the prototype.

## Wood Progression — Design Notes

Wood/timber weapons are training weapons. They exist to give new players something to swing in tutorials and early zones before they can afford bronze. The wood→bronze gap provides a meaningful upgrade at low mastery tiers (BASIC/SKILLED) where those materials are actually in use.

For d4 base weapons (daggers), wood and bronze share the same GM ceiling — the dice range is too compressed to differentiate. For d6+ base weapons, the natural progression has enough room and wood/bronze GM values differ normally.

## Future: Non-Metal Progressions

The metal progression (wood→bronze→iron→steel→adamantine) covers the majority of melee weapons. Future material tracks may include:

- **Ranged ammunition** — arrow/bolt material may follow a parallel track (wood→bone→iron→steel→adamantine tips)
- **Natural materials** — bone, chitin, dragonscale for druid/ranger thematic weapons
- **Magical materials** — enchanted variants that add elemental damage on top of the base table

These would be additional material entries in the same lookup table structure, not separate systems.
