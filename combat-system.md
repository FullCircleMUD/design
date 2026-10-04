# Combat system

How the combat system works.

It is loosely based on CircleMUD: a single register of every combatant, looped through every combat
round.

## Combat cycle

One tick a second; a round starts on a tick once the last has finished and the minimum round length
has passed.

| # | Step | Code |
|---|---|---|
| 1 | Every real second, the calendar clock fires `combat_tick`. | `register_combat_tick` |
| 2 | `seconds_since_start += 1` | `CombatManager.on_combat_tick` |
| 3 | If `round_finished` and `seconds_since_start >= MIN_ROUND_TICKS`, start a round. Otherwise wait for the next tick. | `CombatManager.on_combat_tick` |
| 4 | `seconds_since_start = 0`, `round_finished = False` | `CombatManager.start_round` |
| 5 | Run the round. | `CombatManager.run_round` |
| 6 | `round_finished = True` | `CombatManager.start_round` |
