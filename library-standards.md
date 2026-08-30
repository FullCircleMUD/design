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

## Naming

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

## Licensing

- **BSD-3-Clause** for all libraries in this folder. The LICENSE file at repo root contains the full text.
- **Source files carry an SPDX header** on the first line: `# SPDX-License-Identifier: BSD-3-Clause`.

## Testing

### Test-first

Libraries are built test-first. The order is: **agree the cases, write the tests, write the
implementation to pass them.**

The cases live in `docs/test-plan.md` before any test is written. It carries:

- **Stable case IDs**, prefixed by the function or surface they cover (`WC-01`, `PL-05`). Never
  renumber — retire an ID rather than reuse it.
- **A `Test function` column**, filled in as each test lands. An empty cell means the case is agreed
  but not yet covered; that column is the coverage trail an auditor reads.
- **A fixtures table** — the fake objects the suite needs, named and purposed.
- **`[TBD — needs discussion: …]` against the specific case** whose behaviour is unresolved, plus an
  **Open decisions** section collecting them. A case with open behaviour is still listed; it just
  isn't written yet.

The plan is a commitment, not a wishlist: a case in the table is a case the library will cover.

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

A library's documentation follows the umbrella's four-surface model — see
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
3. **Where to read first** — numbered reading order.
4. **Load-bearing architectural principles** — numbered. Constraints every implementation decision must
   respect.
5. **Out of scope** — concrete rulings *or* "decided as questions arise" depending on project maturity.
6. **Working conventions** — editing design docs, license, SPDX headers.
7. **Documentation discipline (load-bearing)** — rules about what gets captured where:
   only-what-was-discussed, flag-open-questions, smaller-is-better.
8. **Repository layout** — current ASCII tree.
9. **Tools and environment** — Python version, runtime deps, test framework.

**Two principles every library's section 4 should include:**

- *The library does not own game concepts.*
- *No FCM-specific assumptions.*

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
