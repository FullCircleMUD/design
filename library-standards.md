# Library Standards

Conventions for the reusable libraries developed under `libraries/`. Every library in that folder
follows these standards, so a session creating a new one produces a result structurally consistent with
its siblings. This is a **project-level standard held at the umbrella** — libraries *follow* it; they do
not carry or reference a copy of it. Where a convention has a deeper authoritative source, this file
points there rather than duplicating (e.g. [doco-structure.md](doco-structure.md) for the documentation
surfaces).

**These are rules for LLMs, not constraints on the developer.** They exist so a session building a
library arrives at something consistent with its siblings instead of choosing afresh each time. The
human developer overrides any of them at their own discretion, without justifying it here — an
instruction from them outranks this document. Where a decision they made departs from a rule, record
what was decided in the library's `CLAUDE.md`, not an argument for why the rule should have won.

The `libraries/` folder may also contain **auxiliary repos directly required for library development** —
test-content and fixture repos (e.g. `evennia-world-builder-test-yaml/` holds private YAML fixtures for
world-builder's tests). These standards apply only to the libraries themselves; auxiliary repos are not
bound by them.

## When this applies

These standards apply to libraries developed in `libraries/<library-name>/`. Such libraries are:

- **Internal-first** — FCM is the primary consumer; refactor freely as needs evolve, no semver promises
  to outside users. Standalone use is *consumption*, not independent development.
- **Evennia-flavoured** — designed to extend or work alongside Evennia.
- **BSD-3-Clause licensed** — matching Evennia's licensing context.

If a future library has materially different framing (a pure-Python library with no Evennia coupling,
say), some standards may relax. State the deliberate divergence in that library's `CLAUDE.md`.

## Two families — `evennia-` and `fcm-`

The prefix on a library's name is a claim about who can use it, and the two claims are different.

**`evennia-*` libraries are game-agnostic.** They extend Evennia and know nothing about FCM. Anyone
running Evennia could install one and it would work. **Everything else in this document is written for
them** — where a rule below says "the library", read "an `evennia-*` library" unless the text says
otherwise.

**`fcm-*` libraries embed FCM's game concepts, deliberately.** Some of what FCM does cannot be
abstracted into a general library without abstracting away the thing itself — the XRPL ownership layer
and the telemetry-driven spawn economy are both like that. The prefix is the claim being withdrawn: an
`fcm-*` library does not offer itself as something that drops into an arbitrary Evennia install, so it
is free to name FCM concepts in its own code. `fcm-xrpl` and `fcm-telemetry-spawn` are the two.

The reason for the family is *why* it exists rather than the volume of coupling: a library goes in it
when abstracting the game concepts out would destroy the library, not when doing so would merely be
work.

**Everything structural and procedural is identical across both families.** Src layout, `pyproject.toml`
shape, the logging shim, database aliases and routers, the test framework, test-first and its test plan,
the documentation surfaces, the `CLAUDE.md` shape, `docs/` structure, interoperability — all the same.
The family decides what the code is allowed to *know*, and nothing else.

Two consequences, both narrow:

- **The two scope principles do not apply to an `fcm-*` library.** See *CLAUDE.md structure* below.
- **Licensing and distribution.** `fcm-xrpl` is not licensed: no `LICENSE`, no SPDX headers,
  `pyproject.toml` declares `Proprietary`, and it is pip-installable through a private provider —
  probably GitHub — rather than PyPI. `[TBD — needs discussion: whether that is a rule for the `fcm-*`
  family or a per-library call. The `library-standards-linter` encodes BSD-3-Clause either way, so an
  unlicensed library reports two errors and a warning until this is settled and the linter taught.]`

## Naming

- **Prefix**: `evennia-` or `fcm-`, per *Two families* above. The prefix is chosen when the library is
  created and states what the code is allowed to know.
- **Repo name and PyPI distribution name**: hyphenated, lowercase, descriptive — `evennia-world-builder`,
  `evennia-shards`.
- **Python import name**: underscored equivalent — `evennia_world_builder`, `evennia_shards`. Required
  because Python identifiers can't contain hyphens.
- **Repo location**: `libraries/<library-name>/`. Each library is its own git repo with its own GitHub
  remote.
- **Mixin class names carry a prefix naming the library they came from** — `ArchivableObjectMixin`,
  `ScalingCharacterMixin`, `SurvivalMixin`. A consumer's typeclass declaration is a list of mixins from
  several libraries and Evennia, and the prefix is what says which is whose when one of them
  misbehaves.

  **At least one full word, never an abbreviation.** Where the library's name is a single word, that
  word is the prefix. Where it is several, one of them is enough — `evennia-portal-multiplex` would
  give `Multiplex…`, not `PortalMultiplex…`. Abbreviations are the thing to avoid: they read fine to
  whoever picked them and to nobody afterwards.

## Source layout

Use the **src layout**. Modern best practice; PyPA-recommended; forces `pip install -e .` for
development, which catches packaging bugs early.

```
<library-name>/
├── pyproject.toml
├── runtests.py
├── README.md
├── CLAUDE.md
├── LICENSE                   # BSD 3-Clause
├── .gitignore
├── docs/                     # content wiki — how the library works (see doco-structure.md)
│   └── ...
├── src/
│   └── <library_name>/       # the package
│       ├── __init__.py
│       ├── log.py            # the logging shim — see below
│       ├── contrib/          # ONLY if contrib modules exist — see below
│       └── tests.py          # tests live inside the package (Django convention)
├── tests/                    # standalone test infrastructure
│   ├── __init__.py
│   ├── test_settings.py
│   └── urls.py
└── examples/                 # demo gamedirs for integration testing
```

## contrib/ — conditional

### The problem it resolves

Every library here holds the same principle: **the library provides infrastructure, the consumer owns
the game.** Rooms, exits, characters, items, zones and economies belong to the game, not to a library
that many different games might use.

That principle collides with reality whenever a library exposes a primitive that *almost every
consumer will use the same way*. Two bad options present themselves:

- **Put it in core.** The library now owns a game concept, breaches its own scope, and forces one
  shape on every consumer — including the ones whose game doesn't work that way.
- **Leave it out entirely.** Every consumer reinvents the same wheel from the same primitive, with no
  reference implementation to work from and no shared vocabulary for discussing it.

**`contrib/` is the third option.** Core stays game-agnostic and ships only the primitive. Contrib
carries a working implementation of the common case, opt-in and clearly marked as one answer rather
than the answer. A consumer can import it as-is, copy and adapt it, or simply read it to understand
how the primitive is meant to be used — and a consumer whose game is shaped differently ignores it
entirely and loses nothing.

This is Evennia's own convention: community members contribute modules for other community members to
use and to learn from. Adopting it means an FCM library stays a good citizen of that ecosystem while
keeping its core honest.

### The rule

**The folder exists only when there are contrib modules in it.** Do not scaffold an empty `contrib/`
"for later" — its presence is the signal that opt-in modules are available.

Nothing in `contrib/` may be imported by core, and core must remain fully functional with the
directory absent. If core needs it, it isn't contrib.

### Worked example

> **Future work.** This example describes the finished state. No `contrib/` exists in
> `evennia-shards` today — the split is agreed, the module is not yet written. Remove this note once
> it lands.

`evennia-shards` exposes `cross_shard_move()` — a primitive that relocates a player between shards
and returns what actually happened. A consumer can drive it from anywhere: an exit, a portal, a ship,
a teleport pad, a login-time placement rule.

Walkable doors between shards are the anticipated common case, so a `CrossShardExit` typeclass ships
as **contrib**. Core never learns what an exit is; consumers who want doors get a working one;
consumers whose world crosses shards some other way are unaffected.

[TBD — needs discussion: whether contrib modules carry their own tests and `docs/` entries, or are
documented inline.]

## pyproject.toml

Standard shape (adapt names per library):

```toml
[build-system]
requires = ["setuptools>=61"]
build-backend = "setuptools.build_meta"

[project]
name = "<library-name>"
version = "0.0.1"
description = "..."
readme = "README.md"
license = {text = "BSD-3-Clause"}
authors = [{name = "Tim Baird"}]
requires-python = ">=3.10"
dependencies = [
    "evennia",  # plus any other runtime deps
]

[tool.setuptools.packages.find]
where = ["src"]
include = ["<library_name>*"]
```

Key points:

- **`pyproject.toml` is the only build/dependency declaration.** No `setup.py`, no `setup.cfg`, no
  `requirements.txt`.
- **Runtime dependencies in `[project] dependencies`**, not anywhere else.
- **Optional dev tools** (e.g. `ruff`) go in `[project.optional-dependencies] dev = [...]` and are
  installed via `pip install -e ".[dev]"`.
- **License declaration must match the LICENSE file** (BSD-3-Clause).
- **`requires-python = ">=3.10"`** is the minimum (matches Evennia). Raise per library only if there's a
  reason.

## Reading settings

Two kinds of setting, and they are handled differently. **What separates them is whether the library
can supply a default.**

| | Has a library default | Required, no default |
|---|---|---|
| How it is read | an accessor in `config.py`, returning the default when unset | an accessor in `config.py`, a plain read and nothing else |
| Checked at boot | no | yes, in `check_settings()`, and the instance does not start without it |
| Missing is | fine — the default applies | a refusal |

Both are read through `config.py`. What differs is where the knowledge sits: for a defaulted setting the
accessor holds the fallback, and for a required one the accessor holds nothing at all — the checking has
already happened at boot.

### A setting with a default — an accessor in `config.py`

Every setting the library can fall back on gets a named accessor in `src/<library_name>/config.py`, and
both library code and consumer code call that rather than the setting. Six libraries do it this way —
`evennia-shards`, `evennia-ai-memory`, `evennia-message-bus`, `evennia-mob-spawner`,
`evennia-world-builder`, `fcm-xrpl` — and the shape is the same in all of them:

```python
SETTING_TICK_SECONDS = "MOB_SPAWNER_TICK_SECONDS"
DEFAULT_TICK_SECONDS = 60


def get_tick_seconds() -> int:
    """Return ``MOB_SPAWNER_TICK_SECONDS``, defaulting to 60."""
    from django.conf import settings

    return int(getattr(settings, SETTING_TICK_SECONDS, DEFAULT_TICK_SECONDS))
```

**A setting with a default is never checked at boot.** Booting without it is the case the default
exists for, so there is nothing to refuse.

Four things are load-bearing:

- **A direct read raises `AttributeError` for the consumer who declared nothing**, which is the common
  case. The accessor is what makes a setting genuinely optional.
- **The default is a module-level constant, named and next to its accessor.** It is the single source of
  truth for that fallback, and a comment there is where the *reason* for the default lives — which is
  the thing a consumer deciding whether to override actually needs.
- **The `from django.conf import settings` is inside the function**, not at module scope. A settings
  import at module level runs when the library is first imported, which can be while the consumer's
  settings module is still executing.
- **The setting name is prefixed with the library's own vocabulary** — `SHARDS_`, `MOB_SPAWNER_`,
  `XRPL_` — because a consumer running several of our libraries has one settings namespace.

**A setting a helper reads at settings-import time must be declared above the call.** Where a consumer
calls a library helper from their own settings module — a `DATABASES` entry, say — a setting declared
*below* that call does not exist yet, and the accessor quietly returns the default rather than raising.
Document the ordering next to the accessor; do not engineer around it.

### A required setting — validated at boot, then read directly

A setting has no safe default when there is no value the library could pick that is correct. An
instance that does not know its own name, or its role in a deployment, cannot behave correctly in any
direction — so refusing is the honest answer, and guessing hides the mistake until it costs more.

**Validate it once, in `AppConfig.ready()`.** One guard clause per setting: read it, test it, collect
what is wrong. Raise at the end, with everything.

```python
def check_settings():
    """Refuse to start when a required setting is missing or unusable."""
    from django.conf import settings
    from django.core.exceptions import ImproperlyConfigured

    problems = []

    shards = getattr(settings, SETTING_SHARDS, None)
    if not shards or isinstance(shards, str):
        problems.append(
            f"{SETTING_SHARDS} is {shards!r}. It must list every shard in the "
            f"deployment; a bare string is read one letter at a time and "
            f"matches nothing."
        )

    ...one clause per setting...

    if problems:
        raise ImproperlyConfigured(" ".join(problems))
```

**Every problem in one raise.** A consumer installing the library has typically got more than one thing
to set, and stopping at the first turns that into fix-restart-fix-restart, once per setting. Collecting
them means a run either starts or hands back the whole list, and a list that has been worked through
starts.

`getattr` with a `None` default rather than `settings.NAME`, so an undeclared setting reaches the
message that says what to add instead of raising `AttributeError`. `not value` covers unset and empty
together. A value that must be a sequence of names gets `isinstance(value, str)` as well, because a
string satisfies every other test — it is iterable, it has a length, and membership against it silently
succeeds one letter at a time.

**A check that depends on an earlier one needs a fallback**, or the second clause crashes on the value
the first just rejected. Give the rejected value a harmless stand-in — an empty tuple where a sequence
was expected — so the remaining clauses still run and still report.

Every required setting is checked here, in one function, in the same place in every library. Four guard
clauses in a row beat four one-line `check_x()` functions and four calls in `ready()`.

**Then give it an accessor that only reads.**

```python
def get_start_location_shard():
    """Return ``SCALING_START_LOCATION_SHARD``. Checked at boot."""
    from django.conf import settings

    return settings.SCALING_START_LOCATION_SHARD
```

No `getattr` fallback and no `if`: boot has already guaranteed the setting is there and usable, so the
accessor's whole job is to defer the read.

**Deferral is the reason the accessor exists.** A read inside a function body is deferred already, so
that much could be a direct `settings.NAME`. A read at *module scope* is not — a class attribute
default, a module constant, anything evaluated when the file is imported, which happens while Django is
still populating apps and possibly before the consumer's settings module has finished. Those need a
callable, and a named accessor is that callable:

```python
current_shard = AttributeProperty(default=get_start_location_shard, strattr=True)
```

Without an accessor the same deferral needs a lambda at every such call site —
`default=lambda: settings.SCALING_START_LOCATION_SHARD` — which is the same function with no name and
no docstring, scattered wherever it is needed.

Having one for every required setting rather than only the ones that need deferring keeps the setting
name in one place and means a reader never has to work out which kind it is holding.

**Do not put the validation inside the accessor and call the accessor from `ready()`.** It works, and it
costs a re-validation on every subsequent read of a value that cannot have changed. It also gives the
accessor a contract that includes raising, for a condition that can no longer occur.

The check being at boot is the point. Validation deferred to first use fires whenever that is — the
first tick, the first player to connect — so a misconfigured instance starts cleanly, runs until
something exercises that path, and then fails somewhere that says nothing about the setting.

## Consumer-authored config — a setting names a module

Some libraries need more from a consumer than a value. A list of meter stages, a set of rules, a table
of definitions — content the consumer writes as Python, which the library reads and works from.

**That is a setting naming a module path, resolved by the library.** Nothing else. The library does not
read a file from a location it chose, and it does not create a folder in the consumer's gamedir.

```python
SURVIVAL_HUNGER_STAGES = "world.survival_stages.HUNGER"
```

Which kind of setting it is follows the rules above: a definition the library cannot invent has no
safe default, so it is checked in `check_settings()` and refused at boot.

**The shape is fixed; the call that resolves it is the library's choice.** One setting naming a dotted
path, resolved in `config.py` behind a named accessor, checked once at boot. Which loader gets used
depends on what is being loaded, and any of these is compliant:

| Call | Loads | On failure |
|---|---|---|
| `evennia.utils.utils.class_from_module` | a class | raises `ImportError` |
| `django.utils.module_loading.import_string` | a class or a variable | raises `ImportError` |
| `evennia.utils.utils.variable_from_module` | a variable | **returns `None`** |

**The one requirement is that a failure is loud.** `variable_from_module` is the one to watch: it
returns `None` for a missing module, a missing name, and a name whose value genuinely is `None`, with
nothing to tell them apart — `mod_import` catches `ImportError` and returns `None`, despite a
docstring saying it logs. Use it only where the default is right, or check the result and raise.

Anything else a library needs is fine on the same terms: resolve it, and either it raises or you turn
what it does into an exception yourself.

**A consumer already knows this shape**, because Evennia uses it throughout —
`BASE_CHARACTER_TYPECLASS`, `PROTOTYPE_MODULES`, `CMDSET_CHARACTER`, `LOCK_FUNC_MODULES`. A library
that invents its own mechanism is asking them to learn a second one for no gain.

**Where those modules sit in the gamedir is the consumer's business and no part of ours.** A game
running ten of our libraries may well want them gathered in one folder rather than scattered; that is
theirs to decide, and it costs them nothing but the paths they put in their own settings. A library
that creates directories at boot takes that decision away and leaves something behind in a repo it does
not own.

The wider rule this follows from: **every kind of thing a library needs to store already has an
answer**, and a library inventing a fourth is what this section exists to prevent.

| What the library needs | Where it goes |
|---|---|
| Config or definitions the consumer authors | A setting naming a module path |
| Data of the library's own | The game database, or its own alias behind its own router |
| Log output | `settings.LOG_DIR`, through the shim in `log.py` |

## Reading and writing object state

**Prefer to validate in `at_set()`, wherever an attribute has a rule that can be stated.** A number
that cannot go negative, a value from a fixed set, a type that should be coerced — the check belongs
in the descriptor rather than at each call site. It then runs once, wherever the value came from, and
it fails at the assignment that caused it rather than later, inside whatever arithmetic consumed it.

This is a preference and not always achievable. Some attributes have no constraint worth stating, and
some can only be checked against state the descriptor cannot see.

**Where a property is not validated in `at_set()`, a comment at the declaration says why** — checked
somewhere else, checkable only against state the descriptor cannot reach, or deliberately unconstrained.
That comment is what stops the decision being taken again: a session reading it sees the reasoning and
moves on.

**An unvalidated property with no such comment is an open question, not a settled one.** The first
thing to ask of it is whether it should be validated in `at_set()`. Decide that, and where the answer
is no, write the comment — so the next session inherits the decision instead of re-deriving it.

**An `AttributeProperty` carrying validation must be set by assignment, never through `.db`.** The
validation lives in the descriptor, and `.db` does not go through the descriptor.

`AttributeProperty` is a descriptor with an `at_set()` hook, which is where a library checks or
normalises what it stores. `obj.thing = value` runs it. `obj.db.thing = value` goes through the
`AttributeHandler` instead, never reaches the descriptor, and stores the value unchecked. Evennia
documents it: `at_set` "will only fire if you actually set the Attribute via this
`AttributeProperty`".

By way of example, a property whose `at_set()` refuses a negative number refuses `obj.weight = -5` and
accepts `obj.db.weight = -5` without complaint.

**So a library sets its own attributes by assignment throughout**, whether or not a given one
validates today. An unvalidated property is one commit away from a validated one, and every `.db`
write that was harmless before is then silently wrong, with nothing failing to say so.

**The bypass cannot be closed from the library side.** Document it rather than defending against it: a
consumer who writes through `.db` gets whatever they set. Pin it with a case, so the limit is stated
rather than discovered.

## Overriding a hook or a method

**An override calls `super()` unless there is an explicit reason not to.** A consumer's typeclass is a
stack of mixins from several libraries plus Evennia, and an override that does not call up removes
everyone below it from the chain. Nothing reports that: the other library simply stops working, in a
game where it was installed correctly.

**Where an override deliberately does not call `super()`, a comment in the method says why.** Blocking
the chain is occasionally right — a mixin whose whole job is to refuse something. The comment is what
stops the decision being taken again.

**An override with no `super()` call and no such comment is an open question, not a settled one.** The
first thing to ask of it is whether it should call up. Decide that, and where the answer is no, write
the comment — so the next session inherits the decision instead of re-deriving it.

Note the ordering where an override both consults `super()` and adds its own veto: take the parent's
answer first and pass on a refusal, rather than returning your own result over the top of it.

## Whose problem is it — when a library raises

Three kinds of failure, three different answers. The question a library asks before raising is **whose
problem this is**, and it is answerable in every case.

| The problem | What the library does |
|---|---|
| The consumer has not configured what the library requires | Collect it and raise, with everything else that is wrong |
| The library itself is broken | Raise. Loudly, immediately, with the real traceback |
| Something outside the library is broken | Say nothing. Let it raise wherever it naturally would |

**A consumer's configuration is the library's business.** A required setting missing, a typeclass without
the mixin the library needs, two settings that contradict each other — the library knows the contract and
the consumer cannot be expected to. It refuses at boot, and it names every problem at once so a consumer
gets one list rather than one restart per mistake. See *Reading settings* above.

**The library's own bugs are the library's business.** Never swallow one. A library that hides its own
failure is a library nobody can debug, and the consumer is entitled to find out that the fault was ours.

**Everything else is not the library's business, and it must stay out of the way.** A consumer's own code
failing — a broken module, a typo in a class they wrote, an error in a hook they implemented — is theirs
to see and theirs to fix. If the library is holding that error when it surfaces, it lets it go rather
than re-raising it. Three reasons, and all three matter:

- **The traceback stops being honest.** Re-raised through the library, the stack reads
  `library.ready() → library.check → their module`. The bottom line is still their bug, but the name in
  the middle is ours, and that is what a reader anchors on.
- **It sends them into the wrong code.** The natural move is to open the frame whose name you recognise.
  Every minute spent reading library source is a minute not spent on the actual fault.
- **It costs nothing to stay quiet.** The error surfaces on its own the moment anything else touches
  that code — from Evennia, or from their own call site, with a traceback that names only them.

The cost, stated plainly: a check that could not run did not run. That is acceptable, because whatever
broke their module is stopping them anyway, and the next boot after they fix it runs the check properly.

**Distinguish by content, not by exception type.** The same `TypeError` can be a mixin ordering conflict
the library caused and can explain, or an unrelated bug in a consumer's module. Catch narrowly, test the
message for what identifies it as the library's own, translate that case, and let everything else go.

## One call, one return type

**A function returns the same type however it was reached.** Not one type when it created something and
another when it found it already there; not one type on the first call and another on the second. The
caller writes one line to handle the result, and that line is either right or wrong — never right on
Tuesday.

The failure is quiet, which is what makes it worth a standard. Code that compares the result to
something works for as long as the other branch is never taken, and the first time it is, the comparison
returns false instead of raising. `evennia-archive` had exactly this: `archive()` handed back a record
whose identity was a `uuid.UUID` when the row already existed and a `str` when it had just been created,
because Django does not coerce a field until the row is reloaded. Nothing noticed until archiving twice
became the normal case.

**Coerce at the boundary, not in the storage.** The fix is not to change the column — `archive_id` stays
a `UUIDField`, which is the right storage type and 16 bytes rather than 36 on Postgres. It is to hand
back what the library's API says it deals in. Everywhere else in that library speaks strings: the mixins
mint and store them, and both finders coerce. So the return does too.

**Vary it only for a clear and specific reason, and document it where a caller will read it.** The
distinction is whether the caller can see the reason. Returning the thing or `None` — "the archive id,
or `None` when nothing is archived under that name" — is one signature with a sentinel in it, stated in
the docstring and covered by a case. The caller knows both outcomes and which one they are handling.
That is a contract.

What this standard refuses is a type that varies with an internal branch nobody outside can see: which
one you get depends on state the caller has no view of, so there is no line they can write that is
reliably correct. If a return genuinely has to vary, the reason goes in the docstring in the same
sentence as the types, and each shape gets a case.

## Logging

**Every library logs to a file of its own, through a shim in `src/<library_name>/log.py`.** A library
that writes into the main server log makes an operator search for its lines among everything else the
game emitted; one that uses stdlib `logging` with no handler configured emits records nobody ever sees.
Neither is acceptable, and neither is a per-library judgement call — this is the pattern, and a session
bootstrapping a new library copies it rather than choosing again.

The shim is small and its shape is fixed:

- **One public function**, named for the library — `bus_log`, `ai_memory_log`. Signature
  `(message: str, level: str = "INFO", trace: bool = False) -> None`.
- **A lazy `from evennia.utils import logger` inside a `try`.** Outside an Evennia engine — the test
  suite, a standalone tool — an `ImportError` is swallowed and the call is a silent no-op. It does not
  fall back to stderr or to a local file: a library that logs somewhere unexpected is worse than one
  that stays quiet.
- **`logger.log_file(f"[{level}] {message}", filename="<library>.log")`.** The file lands in the running
  instance's `settings.LOG_DIR` beside `server.log`.
- **No timestamp of its own.** `log_file` already prefixes `<timestamp> [-] ` in UTC, the same format
  the rest of the server logs use, so a library line and a `server.log` line can be read against each
  other. Adding another would stamp every line twice.
- **Levels are `INFO` / `WARN` / `ERROR`**, and anything else coerces to `INFO`. A log call must never
  raise into its caller, so an unknown level degrades rather than rejecting.
- **`trace=True` appends `traceback.format_exc()`**, for calls made from inside an `except` block.
  Outside one, `format_exc()` returns `"NoneType: None"` and the shim suppresses it rather than logging
  noise.

The shim is internal — not part of the consumer-facing API, and not re-exported from `__init__.py`.

**This is the one place a library may import Evennia when it otherwise has no need to.** A library
whose logic is framework-neutral still logs through the shim; it asserts the narrow rule (only `log.py`
imports Evennia) as a test case rather than the broad one.

Copy [evennia-message-bus's `log.py`](../libraries/evennia-message-bus/src/evennia_message_bus/log.py)
and change the function name and the filename. It is verbatim across the libraries that have it, and
should stay that way — a difference between two copies is a defect, not a variation.

## Database aliases and routers

**A library that owns tables puts them on an alias of its own, behind its own router, when its data
has to outlive the game database or be reachable from more than one instance.** Either alone is enough:

- **It must survive a rebuild.** A game database gets wiped and rebuilt from source; anything the
  library is holding that matters afterwards cannot be in there. It also means a consumer can move that
  data onto separate hardware without the library knowing.
- **More than one instance reads it.** A game database belongs to one instance. Data that has to cross
  between them needs somewhere both can see.

**A library whose data is scoped to one instance and worth nothing after a wipe puts its tables in the
game database.** An alias would cost a database, a router and a migration step a consumer has to
configure, to protect rows that are stale within seconds and meaningless after a restart. Say so in the
library's `CLAUDE.md` and pin it with a case, so it is not later "fixed" into an alias by someone
applying the first rule without reading the reason.

### The router

Three rules, all load-bearing:

- **Answer only for your own app label.** Return `None` for every model you do not own. Django consults
  routers in order and takes the first non-`None` answer, so a router that answers for a foreign model
  silently captures its queries and sends them to the wrong database. A consumer running two of our
  libraries has two routers in the list; each must leave the other's models alone.
- **`allow_migrate` returns `False` for your app on every other alias.** Without it a plain
  `evennia migrate` creates your tables in the game database as well, and the separation you just built
  exists only on paper.
- **`allow_relation` expresses no opinion unless both models are yours.** A library holding no foreign
  key to anything should return `None` throughout; one with relations between its own models may return
  `True` for that case and `None` otherwise.

### How a consumer installs it

A library's setup snippet is pasted into a settings file whose prior state the library cannot see. It
might be the first router the consumer has ever added, or the fourth. **The documented form has to be
correct in both cases**, and the obvious one is not:

```python
DATABASE_ROUTERS += ["your_library.db_router.YourRouter"]
```

Evennia's default settings do not define `DATABASE_ROUTERS`, so that works only when something else has
already created the list. Install this library first and it raises `NameError` before the server
starts; install it third and it works — which is worse, because the instruction then looks correct
until the day someone follows it on a clean gamedir.

Document this instead, and copy it verbatim between libraries so a consumer running several sees one
familiar shape:

```python
_YOUR_ROUTER = "your_library.db_router.YourRouter"
DATABASE_ROUTERS = list(globals().get("DATABASE_ROUTERS", []))
if _YOUR_ROUTER not in DATABASE_ROUTERS:
    DATABASE_ROUTERS.append(_YOUR_ROUTER)
```

It builds the list whether or not one exists, appends rather than replacing — so it cannot silently
drop a router another library added — and the membership check makes re-running it harmless.

### Resolving the alias

A library that needs an alias should ship a helper the consumer calls in their settings, rather than
making them hand-write a `DATABASES` entry:

```python
DATABASES["your_alias"] = your_library_database(os.path.join(GAME_DIR, "your_library.db3"))
```

Three rungs, in order: a `DATABASE_URL_<LIBRARY>` environment variable naming a database of its own,
then `DATABASE_URL` to share the game's, then the SQLite path passed in. Which rung is *right* depends
on something no single instance can see, so the helper does not guess and does not warn — it ships a
companion `describe_*_database()` that names the resolved database and the rung that produced it, and
the library writes that line to its log at startup. Two instances that should share a database are then
confirmed by reading two log lines rather than by reasoning about where each variable was set.

That description must carry the database name and host only. The configuration holds credentials parsed
out of a URL, and they must never reach a log file.

`evennia-message-bus` and `evennia-ai-memory` both implement this; copy from either.

## Licensing

- **BSD-3-Clause** for all libraries in this folder. The LICENSE file at repo root contains the full text.
- **Source files carry an SPDX header** on the first line: `# SPDX-License-Identifier: BSD-3-Clause`.

## Testing

### Test-first

Libraries follow the project's test-first order — see
[test-first-process.md](test-first-process.md) for the process and the rationale. What is
library-specific is where the plan lives and what it carries.

The cases live in `docs/test-plan.md` before any test is written. It carries:

- **Stable case IDs**, prefixed by the function or surface they cover (`WC-01`, `PL-05`). Never
  renumber — retire an ID rather than reuse it.
- **A `Test function` column**, filled in as each test lands. An empty cell means the case is agreed
  but not yet covered; that column is the coverage trail an auditor reads, and it is checked **both
  ways** — see [test-first-process.md](test-first-process.md) for the rules and what counts as a test.
- **A fixtures table** — the fake objects the suite needs, named and purposed.
- **`[TBD — needs discussion: …]` against the specific case** whose behaviour is unresolved, plus an
  **Open decisions** section collecting them. A case with open behaviour is still listed, but it does
  not pass: an unresolved case is an error until the decision is made.

The plan is a commitment, not a wishlist: a case in the table is a case the library will cover. The
`library-standards-linter` checks it by delegating to the `test-plan-linter`, so a library's plan is
held to the same rules as any other.

[evennia-targeting's test plan](../libraries/evennia-targeting/docs/test-plan.md) is the reference
shape.

### Test framework

**Django's test runner** via a standalone `runtests.py`. Not pytest. Modelled on evennia-shards:

1. **`runtests.py`** at repo root — entry point. Bootstraps Django + Evennia, runs Django's test runner
   against the package. No consumer gamedir required.
2. **`tests/test_settings.py`** — minimal Django settings. Imports `evennia.settings_default`, adds the
   library to `INSTALLED_APPS`, configures an in-memory SQLite test DB.
3. **`tests/urls.py`** — empty URL config (`urlpatterns = []`); tests don't expose HTTP routes.
4. **`src/<library_name>/tests.py`** — actual test code, inside the package. Discovered by Django's runner.

Tests use `django.test.TestCase` (DB-aware, transactional) or stdlib `unittest.TestCase` (pure Python).

## Development environment

Each library is developed against a **dedicated venv** at `<library-name>/venv/`, with its own Evennia
install. This keeps library development standalone — independent of any consumer game's environment.

Setup from a fresh clone:

```bash
cd libraries/<library-name>
python -m venv venv
# Activate (Linux/macOS):  source venv/bin/activate
# Activate (Windows PS):   .\venv\Scripts\Activate.ps1
pip install evennia
pip install -e .
python runtests.py
```

`venv/` is gitignored.

### `examples/` — demo gamedirs for integration testing

Where unit tests via `runtests.py` cover the library's logic in isolation, the `examples/` directory
holds **demo Evennia gamedirs** that exercise the library end-to-end against real Evennia objects and a
real database.

Conventions:

- One subdirectory per demo gamedir (`examples/demo_world/`, etc.).
- Each gamedir is a normal Evennia gamedir created via `evennia --init`, configured to use the library.
- Demo gamedirs are run with `evennia start` from inside the gamedir directory.
- Demo gamedirs serve the library, not a real consumer game. No FCM concepts; just enough Evennia world
  content to exercise the library's surface.

## Documentation surfaces

A library's documentation follows the umbrella's documentation-surface model — see
[doco-structure.md](doco-structure.md) — and carries only the **reduced set** appropriate to a
stand-alone-reusable sub-repo:

- **`README.md`** — humans landing on the repo (and standalone consumers).
- **`CLAUDE.md`** — repo-specific agent context.
- **`docs/`** — the content wiki: how the library works (`INDEX.md` + kebab-case topic files).

A library does **not** carry the project's documentation conventions (no per-repo conventions meta-doc)
and has **no memory surface of its own** — those are project-level concerns held once, at the umbrella.
The library follows the umbrella conventions; it does not restate them. Full rationale: the
"stand-alone-reusable sub-repos may self-document" exception in [doco-structure.md](doco-structure.md).

## CLAUDE.md structure

Standard sections, in order:

1. **What this project is** — one paragraph + tagline.
2. **Project status** — link to `docs/progress.md` for current state. Don't duplicate status content
   here; it ages badly.
3. **Where to read first** — numbered reading order. `docs/test-plan.md` is always in it, high, marked
   as where a behavioural change starts.
4. **Load-bearing architectural principles** — numbered. Constraints every implementation decision must
   respect.
5. **Out of scope** — concrete rulings *or* "decided as questions arise" depending on project maturity.
6. **Working conventions** — editing design docs, license, SPDX headers.
7. **Documentation discipline (load-bearing)** — rules about what gets captured where:
   only-what-was-discussed, flag-open-questions, smaller-is-better.
8. **Repository layout** — current ASCII tree.
9. **Tools and environment** — Python version, runtime deps, test framework.

**Three principles every `evennia-*` library's section 4 should include:**

- *The library does not own game concepts.*
- *No FCM-specific assumptions.*
- *Test-first* — a case lands in `docs/test-plan.md`, then the test, then the code. Points at
  [test-first-process.md](test-first-process.md).

**An `fcm-*` library carries the third and not the first two**, and says so — a principle stating that
FCM concepts belong in it, and that the two sibling principles deliberately do not apply. Adding them
back later, on the reasonable-looking grounds that every other library has them, would break the
library; the principle exists to stop that. See *Two families* above.

Other principles are library-specific.

## docs/ structure

Required:

- **`INDEX.md`** — lists every design document with a one-line description, organised by category.
- **`progress.md`** — reverse-chronological milestone log with links to evidence.
- **`test-plan.md`** — every test case the library commits to covering, and the test function
  covering it. See [Testing](#testing).
- **`interoperability.md`** — this library against every sibling library. See below.
- **`archive/`** — historical context. Material here is preserved per the "don't delete; supersede" rule.

Conventions for new design documents are the umbrella's — see [doco-structure.md](doco-structure.md):
kebab-case filename; first line an `# H1 Title` matching the filename; a one-paragraph summary as the
second block; index every document in `INDEX.md` (an un-indexed document is invisible).

## Describing a process — the step list comes first

Wherever a document explains how something works — a boot sequence, a request through the layers, a
handoff between components — **a step-by-step list comes before the prose that explains it.** Not
instead of: the prose still carries the reasoning. The list is what a reader can hold in their head,
and it is what they came for.

**Every step is attributed.** Each one says who does it — this library, Evennia, or a named sibling
library. That is usually the question being asked: *which of these is mine to change?* Attribution
turns a description into a map of responsibility.

**Plain language, one step per line.** What happens, in the order it happens. A step that needs a
paragraph to justify it gets the paragraph below, not on the line.

**Say where the gaps are.** A process that is not finished marks the steps that do not exist yet, so
the list doubles as the work queue. Say plainly when there are none.

```markdown
## A Server booting, step by step

Every step from starting a Server to it being reachable, and who owns each one — **[library]** for
this library, **[Evennia]** for Evennia or Twisted. No gaps: this part is complete.

- **[Evennia]** `django.setup()` runs, which runs every installed app's `ready()`
- **[library]** `ready()` repoints the class settings Evennia resolves later
- **[Evennia]** the AMP client dials the Portal, retrying with backoff if it cannot
- **[library]** the Server asks the Portal whether its announcement was recorded
- **[gap]** <a step that does not exist yet, named as such>
```

**One list per process, not per document.** Distinct processes get distinct lists, each with its own
prose below it, however many of them a document holds. They interrelate; they are still separate
things a reader follows one at a time. When a document carries so many that it stops being readable,
split it into a document per process rather than merging the lists.

Tags are whatever names the parties — `[library]`, `[Evennia]`, `[gap]`, or a sibling's name where two
libraries share a flow. Use the same vocabulary throughout a document, so two lists can be read
against each other.

## Library interoperability

Every library carries `docs/interoperability.md`, covering **all** the libraries in `libraries/` —
including itself. A reader deciding whether two of our libraries can be co-installed gets a definite
statement from either side rather than inferring from silence.

**One template, no permutations.** Every copy carries the same sibling sections in the same order, so
the files can be read side by side and a missing consideration is visible as a gap. The library's own
entry says *"This library."* and nothing else.

Sibling order is alphabetical:

```markdown
# Interoperability

<One paragraph: what this library does that could constrain, or be constrained by, a sibling —
ORM writes, thread dispatch, persistent scripts, settings it reads.>

## evennia-mob-spawner
## evennia-shards
## evennia-targeting
## evennia-world-builder
## evennia-yaml-reader
```

Each sibling section opens by naming the **relationship** — one of:

- **Hard dependency** — imported unconditionally; the sibling must be installed.
- **Optional integration** — imported behind a `try` with a documented fallback; both co-installed and
  standalone are supported.
- **No coupling** — neither imports the other.

Then either the considerations, or an explicit clearance. A clearance states *why* it is clear, in terms
of what this library actually does:

> **No coupling.** No constraints — this library issues no ORM writes and dispatches nothing off the
> reactor thread, so nothing it does is visible to the tenancy layer.

"No known issues" on its own is not a clearance; it is the void the document exists to remove.

**Each constraint is documented once, by the library that owns it** — the one whose data model or API
the rule constrains. The other library's section links to it rather than restating, so the pair cannot
drift. Owning a constraint does not depend on which library triggers it.

## Documentation discipline

Follows the umbrella's working-discipline rules (the umbrella `CLAUDE.md` and [doco-structure.md](doco-structure.md)); each library's `CLAUDE.md` carries a discipline section reflecting them.

## Bootstrap checklist

When creating a new library in this folder:

- [ ] Create the GitHub repo and clone to `libraries/<library-name>/`.
- [ ] Add `LICENSE` (BSD-3-Clause).
- [ ] Add `.gitignore` (Python standard).
- [ ] Adapt `CLAUDE.md` from a sibling library; rewrite project-specific sections, keep the standard shape.
- [ ] Create `docs/INDEX.md` and `docs/progress.md` with current state.
- [ ] Create `docs/interoperability.md` from the template — every sibling section present, each either
      a stated consideration or an explicit clearance.
- [ ] Write `README.md` answering: what is it, status, is it for me, install, learn more.
- [ ] Populate `pyproject.toml` using the standard shape.
- [ ] Create `src/<library_name>/__init__.py` with `__version__ = "0.0.1"`.
- [ ] Copy `log.py` from a sibling; rename the function and the log filename. See *Logging*.
- [ ] If the library owns tables: decide where they go. Its own alias if the data must outlive the game
      database or be read by more than one instance — then add the router and resolution helper, and
      document the append-don't-assign setup form. Otherwise the game database, said so in `CLAUDE.md`
      and pinned by a case. See *Database aliases and routers*.
- [ ] Adapt `runtests.py`, `tests/test_settings.py`, `tests/urls.py` from a sibling.
- [ ] Create `docs/test-plan.md` with the fixtures table and the first surface's cases (IDs assigned,
      `Test function` column empty).
- [ ] Add `src/<library_name>/tests.py` with one smoke test that proves install + runner work end-to-end.
- [ ] Create dedicated venv: `python -m venv venv` at repo root, activate it, then `pip install evennia`.
- [ ] Verify `pip install -e .` and `python runtests.py` both succeed.
- [ ] Initial commit.

## Living examples

When in doubt about a convention not covered here, look at how an existing library does it:

- **[evennia-shards](../libraries/evennia-shards/)** — split-deployment / sharding library; working MVP.
  Reference for the test-runner pattern, src layout, and `pyproject.toml` shape.
- **[evennia-world-builder](../libraries/evennia-world-builder/)** — declarative YAML world authoring.
  Reference for early-stage repo bootstrap.
