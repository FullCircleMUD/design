# Library Standards

Conventions for the reusable libraries developed under `libraries/`. Every library in that folder
follows these standards, so a session creating a new one produces a result structurally consistent with
its siblings. This is a **project-level standard held at the umbrella** — libraries *follow* it; they do
not carry or reference a copy of it. Where a convention has a deeper authoritative source, this file
points there rather than duplicating (e.g. [doco-structure.md](doco-structure.md) for the documentation
surfaces).

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

## Reading settings — always through an accessor in `config.py`

**A library never reads `settings.SOMETHING` directly.** Every setting it consumes gets a named accessor
in `src/<library_name>/config.py`, and both library code and consumer code call that rather than the
setting. Six libraries do it this way — `evennia-shards`, `evennia-ai-memory`, `evennia-message-bus`,
`evennia-mob-spawner`, `evennia-world-builder`, `fcm-xrpl` — and the shape is the same in all of them:

```python
SETTING_TICK_SECONDS = "MOB_SPAWNER_TICK_SECONDS"
DEFAULT_TICK_SECONDS = 60


def get_tick_seconds() -> int:
    """Return ``MOB_SPAWNER_TICK_SECONDS``, defaulting to 60."""
    from django.conf import settings

    return int(getattr(settings, SETTING_TICK_SECONDS, DEFAULT_TICK_SECONDS))
```

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

Where a setting is **required** rather than defaulted, the accessor raises `ImproperlyConfigured` naming
the setting and where to put it, and `AppConfig.ready()` calls it so an unconfigured consumer cannot
boot. `evennia-message-bus`'s `check_instance_id` and `evennia-ai-memory`'s `validate_settings` are the
two worked examples.

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

`[TBD — needs discussion: whether the shim eventually becomes a shared dependency rather than a file
copied into each library. Copying is deliberate for now — the shim is small, it has no reason to
change, and a shared logging package would be a dependency every library carries for thirty lines.]`

## Database aliases and routers

A library that owns tables puts them on **an alias of its own, behind its own router**, rather than in
the consuming game's database. The reason is the same one every time: a game database gets rebuilt, and
what the library stored should survive that. It also means a consumer can move the library's data onto
separate hardware without the library knowing.

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
- [ ] If the library owns tables: add its alias, router and resolution helper, and document the
      append-don't-assign setup form. See *Database aliases and routers*.
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
