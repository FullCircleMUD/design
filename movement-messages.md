# Movement Messages

How the game tells a room that someone came or went. Every movement message — walking through an
exit, flying between rooms, changing height inside one — is emitted from a single seam so that the
callers who need silence can get it, and so that new movement modes are added as a rule rather than
as another parallel message source. For the exit classes themselves see
[exit-architecture.md](exit-architecture.md); for the height mechanics that decide the verb see
[vertical-movement.md](vertical-movement.md).

> **The seam is built.** `announce_move_from` / `announce_move_to` on `BaseActor` are the single
> emission point, the rule table below picks the verb from the mover's state, and callers pass their
> own wording as `msg_from` / `msg_to`. Every actor movement path routes through it.
>
> Three extensions remain, each described where it belongs below: a **"rides" verb** for mounted
> movement, **darkness** redaction, and routing the last two quiet callers — **procedural-dungeon
> transitions** and the **NFT/pet mirror moves** — through the seam.

## Why one seam

Evennia gates its own movement announcements on the `quiet` argument to `move_to()`: when a caller
passes `quiet=True`, `announce_move_from()` and `announce_move_to()` are skipped. Most game code
relies on this today — teleports, flee, dungeon transitions and the inventory paths overwhelmingly
move objects quietly and write their own text, because a bare "X is leaving here, heading for there"
is wrong for those cases. That is the state as it stands; the default below changes what a new or
revisited caller should do, without requiring every existing quiet caller to be touched at once.

Messages emitted anywhere else cannot be suppressed that way. `at_traverse()` and
`at_post_traverse()` on an exit fire regardless of `quiet`, so a message emitted there is
unconditional by construction — invisible to every caller that has already decided the move should
be silent. That is the defect behind the doubled door messages, and it is the reason **all** movement
messaging belongs in the two announce methods and nowhere else.

**The seam centralises emission, not wording.** Those are two jobs, and only the first has to live in
one place. Emission is what must be shared: excluding the mover, filtering for hidden and invisible
actors, muffling for sleepers, resolving each recipient's view of who moved, and honouring `quiet`.
Wording can come from either of two sources, and the split between them follows a clear line:

- **Movement modes come from a central rule table**, because they are properties of the *mover* —
  airborne, in water, mounted — and no caller is in a position to declare them. A plain walk runs
  command → `at_traverse()` → `move_to()`, and the command never touches `move_to()`, so on the
  common path there is nothing there to pass wording in with. The seam reads the mover's state
  instead.
- **A command's own intent comes from the command**, passed in as text. Fleeing, panicking, being
  dragged out by a collapse — these belong to the caller, and it passes them as kwargs when it calls
  `at_traverse()`. See Caller-supplied wording below.

Those two are the whole system. There is no third mechanism, and in particular no flag that selects
between stored messages: a caller in a position to set such a flag is equally able to pass the text
itself, so the flag would be an indirection with nothing on the other end. Where several callers want
identical wording they share a named constant and pass it — the vocabulary stays defined once, and
the choice of it stays with the caller.

**Default: every inter-room movement message goes through this seam.** Because a caller can always
pass its own text, there is no longer a case where distinctive wording justifies writing a private
`msg_contents()` call and silencing the seam with `quiet=True`. The remaining legitimate exceptions
are moves that are not really movement — inventory transfers via give/get/drop, and relocations like
teleport where nobody is "leaving" or "arriving" in the sense this document describes.

This default is scoped to moves between rooms — anything that calls `move_to()` with a destination.
**Intra-room movement is not part of this seam and is out of scope by construction, not by
exception.** Climbing and flying/swimming up or down change height inside the current room: there is
no destination room, so `move_to()` is never called and `announce_move_from`/`announce_move_to`
never run. Those events message the room directly, the same way a door announces its own
opening — see Height changes inside a room below.

## What a movement message says

Two messages per traversal, one to each room, the mover excluded from both:

```
source room       Fred leaves north.
destination room  Fred arrives from the south.
```

Three things are deliberately absent.

**Room names.** "Fred is leaving The Harvest Moon, heading for Market Square - Northwest" tells
bystanders where someone is going, which they have no in-world way to know. It also degrades badly
where room names legitimately repeat — several adjacent rooms share the name `Old Trade Way West`,
and the stock message renders that as "leaving Old Trade Way West, heading for Old Trade Way West".

**Door names.** A door announces its own opening and closing; the movement message does not need to
repeat it. This also removes a naming inconsistency, because the two sides of a doorway are separate
objects with separate names — an observer would otherwise watch "a wooden door" open and then see
someone arrive through "a sturdy wooden door".

**Any hint of who can see it.** Visibility is handled upstream: `RoomBase.msg_contents()` filters
messages from hidden and invisible actors and muffles them for sleepers, provided `from_obj` is
passed. The announce methods pass it.

## Verb resolution

The verb comes from an ordered list of rules describing how the mover is travelling, first match
wins, falling through to walking. Adding a mode means appending a rule, not editing a conditional.

| Rule | Condition | Departure | Arrival |
|---|---|---|---|
| airborne | `room_vertical_position > 0` | `Fred flies north.` | `Fred flies in from the south.` |
| in water | `room_vertical_position < 0`, or `== 0` in a room whose `max_depth < 0` | `Fred swims north.` | `Fred swims in from the south.` |
| default | — | `Fred leaves north.` | `Fred arrives from the south.` |

Height alone does not decide whether a character is swimming: position `0` is the water's surface in
a room that has depth, and the ground in a room that does not. The rule therefore reads the room's
`max_depth` alongside the character's position, so someone floating on the surface swims away rather
than walking away.

A predicate receives `(mover, room)` and nothing else. That is the constraint that keeps the split
honest: if a mode can't be recognised from the mover and the room they are in, it isn't mover state,
and it belongs to whichever caller does know about it.

**Every rule here must be state the seam can observe for itself**, because these are the only
messages the common path can produce. Nothing keyed on how the move was requested belongs in this
table — that is the caller's to say, and the caller can say it directly.

## Caller-supplied wording

> **Moving an actor? Read
> [exit-architecture.md § Moving actors through exits](exit-architecture.md#moving-actors-through-exits)
> first.** Actors move through exits by calling `at_traverse()` on the exit, not `move_to()` — a
> direct `move_to()` silently skips every door, height, size, encumbrance and trap check. The
> wording channel below is the same either way: `at_traverse()` forwards `move_type` and `**kwargs`
> straight through, so nothing here is lost by going via the exit. `move_to()` remains correct for
> objects and for teleports.

A caller can pass its own lines through the seam rather than around it, via `msg_from` and `msg_to`.
Both ride `**kwargs` to the announce pair, so a single call can give each side different text —
Evennia's own `msg=` cannot, since it applies to both.

```python
chosen.at_traverse(
    caller,
    chosen.destination,
    move_type="flee",
    msg_from="{name} panics and flees {direction} for no apparent reason!",
    msg_to="{name} arrives {direction}, in a panic.",
)
```

The exit passes itself as `exit_obj`, which is what `{direction}` is resolved from.

**An override is a template, not a finished string.** `{name}` is bound to the mover as an *object*,
so it resolves per recipient through `get_display_name()` — which is what redacts it to "Someone" for
anyone who cannot see. A caller that formats the name in itself — `f"{caller.key} flees north!"` —
has defeated that before the seam ever sees it.

`{direction}` is supplied too, carrying the direction logic described below so an override never has
to work it out. It differs by side, because different wording reads naturally in each: outbound it is
the bare direction (`north`), inbound it is the whole phrase (`from the south`, `from below`).

**Callers can add placeholders of their own** by passing a `msg_mapping`, which is merged into the
seam's:

```python
mob.move_to(
    destination,
    exit_obj=chosen,
    msg_mapping={"pursuer": guard},
    msg_from="{name} flees {direction}, {pursuer} close behind!",
)
```

Entries backed by an object resolve per recipient exactly as `{name}` does, so a pursuer nobody can
see is redacted for them too. The seam's own keys are applied last and win, so `{name}` cannot be
rebound to something that skips that resolution.

An override replaces the composed line outright, including the party form: a caller asking for
specific words gets exactly those words.

Passing text through the seam rather than emitting it directly is what buys the guarantees in Why one
seam above — exclusion of the mover, visibility filtering, per-recipient names, and `quiet`
suppression. A private `msg_contents()` call gets none of them.

## Direction phrasing

Ten directions are in use — the eight compass points plus up and down. The compass points take the
plain direction word on both sides. Up and down do not read naturally that way on arrival:

| Direction | Departure | Arrival |
|---|---|---|
| north … northwest | `Fred leaves north.` | `Fred arrives from the south.` |
| up | `Fred leaves up.` | `Fred arrives from below.` |
| down | `Fred leaves down.` | `Fred arrives from above.` |

The arrival direction is taken from the reciprocal exit — the exit in the room being entered that
leads back where the mover came from — using its own `direction` attribute. Evennia's
`announce_move_to()` already resolves that exit and offers it in its mapping. One-way exits have no
reciprocal, and there the direction falls back to the opposite of the exit traversed, via
`OPPOSITES` in `utils/exit_helpers.py`.

`in` and `out` are defined in `OPPOSITES` but unused by any authored exit. The design does not cover
them; a rule can be added if they are ever built.

## Height changes inside a room

Climbing, flying and swimming up or down move a character between height layers of the same room
without changing which room they're in. These do not call `move_to()` and are outside the seam this
document defines — see the scope note above. Whatever room message they send, if any, is written
directly by the command, the same way `ExitDoor` writes its own open/close text.

Today: climbing and falling already emit a room message. Flying and swimming up/down do not — they
speak only to the mover, so nobody in the room sees a character take off, gain height, dive or
surface. Giving them one is a legitimate follow-on, but it's a `cmd_fly.py`/`cmd_swim.py` change
alongside their existing `room.msg_contents()`-shaped siblings, not a rule in the inter-room
dispatch above.

## Followers and groups

A follower cascade (`get_followers()`) moves every follower in the chain immediately after the
leader. Left as-is, each of those moves announces itself, so a party of five reads as ten room
messages for what is, from a bystander's point of view, one event.

The leader's own move carries the group instead. When the leader has followers in the room, the
subject becomes the party rather than the person:

```
source room       Fred's party leaves north.
destination room  Fred's party arrives from the south.
```

The verb still comes from the rules, resolved against the leader. That keeps a mixed
walking/flying/swimming group to one line rather than one per mode, while preserving anything the
verb genuinely needs to convey — a party fleeing reads as `Fred's party flees north!`, not as an
ordinary departure.

A leader moving alone (no followers in the room) gets the ordinary line — "party" wording only
appears when there is a party.

Followers themselves move quietly (`quiet=True` — no announce pair from their own `move_to()`) and
receive only the existing direct message, now carrying the direction:

```
You follow Fred north.
```

There is no arrival message for a follower. This keeps the room-facing side to one line per group
per room, and the follower-facing side to one line per move, with nothing duplicated between them.

## Flee

Flee is the worked example of caller-supplied wording, and of the shared-constant pattern.

Every flee passes its wording through `at_traverse()`, so every flee can say what it means. Combat
flee and the out-of-combat panic run pass different text; nothing about the move is inferred:

```
combat    source room       Fred flees north!
          destination room  Fred flees in from the south!

panic     source room       Fred panics and flees north for no apparent reason!
          destination room  Fred arrives north, in a panic.
```

Every flee runs one implementation — `combat_utils.flee_from_combat()` — so the wordings cannot
drift apart. A caller supplies a `FleeWording` and nothing else: the voluntary `flee` command, a
wimpy threshold firing, and a creature compelled by the frightened effect all pass through the same
code with words of their own. Combat flee's pair is a named constant shared by the callers that
must read alike, rather than each spelling it out.

Both rooms hear a flee, and `{direction}` resolves without the caller doing anything, because
`at_traverse()` passes the exit as `exit_obj` itself. A caller that derived direction from
`chosen.key` would get the exit's *name* and a message reading "You flee Old Trade Way West!".

The private message to the fleeing character (`"You flee north!"` / `"You panic and flee north!"`)
is untouched by any of this — `announce_move_from`/`announce_move_to` exclude the mover and never
speak to them.

## Decided: no first-person echo

The mover receives the new room's description on arrival and nothing else. No "You leave north." is
added for the person moving.

The arrival description *is* the confirmation that the move worked, and a refused move produces its
own message saying why. An echo would be a third line telling the mover what two already told them.

## Deferred — not in this pass

Both of these need doing. They are sequenced after the core, not decided against.

- **A "rides" verb.** `mount()` leaves `following` alone, so a ridden animal travels as an ordinary
  follower and is silenced by the same `quiet=True` — a mounted move already reads as one event, not
  two. What is missing is only the wording: the rule table has verbs for airborne and in-water and
  nothing for mounted, so a rider departs with the walking verb. Adding a rule is a natural
  extension of this same dispatch.
Darkness was listed here too and has since been answered by the wider pass it was waiting on. An
unlit room does read "Someone arrives from the south", because `{name}` is bound as an object and
`UnseenNameMixin.get_display_name` substitutes the mover's `unseen_name` for any recipient who
cannot see them — see [room-architecture.md](room-architecture.md) § Name Redaction. Nothing in the
movement seam had to change for it.

## Remaining quiet callers

Two paths still move actors quietly and write their own text: **procedural-dungeon entry and exit**,
and the **NFT/pet mirror moves**. Both are candidates for the same treatment under the "everything
goes through the seam" default above, to be weighed individually rather than rewritten as a batch.

Nothing in the exit hierarchy emits movement text — every exit type relies on the announce pair
alone. `ExitDoor` keeps its open, close and lock messages, which are about the door rather than
about the movement.
