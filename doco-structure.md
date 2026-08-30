# Documentation Structure

This project uses **four distinct documentation surfaces**. Each has a different audience, lifespan,
and update cadence. Putting content in the wrong place is a slow-burning cost: the wrong people read
it (or miss it), it goes stale, or it crowds out what the surface is actually for.

This document explains what belongs where, when to update each, and the conventions that keep them
coherent.

## The four surfaces

### 1. `/README.md` — for humans landing on the repo

**Audience.** Someone who just clicked through to the repository. A prospective user, a curious
passer-by, a future contributor. No prior context.

**Purpose.** Answer the first-ninety-seconds questions: What is this? Is it for me? What's its status
(usable / in development / abandoned)? How do I install or try it? Where do I learn more?

**Tone.** Big-picture, plain language, skim-friendly. One browser tab, no scrolling fatigue.

**Does NOT belong here.** Architecture detail (→ `design/`), implementation specifics, working notes,
LLM agent instructions (→ `CLAUDE.md`).

**Cadence.** Stable. Edit when something *user-facing* changes (status, positioning, install steps).

### 2. `/CLAUDE.md` — the always-loaded operating rules for LLM agents

**Audience.** Claude (and other LLM agents) picking up a task here, with no memory of prior sessions.

**Mechanics that define it.** `CLAUDE.md` is loaded **in full, every session**, and read as
instructions to obey. Every token is paid on every session — so it must stay **small and
high-leverage**; bloat dilutes the signal and gets load-bearing rules missed.

**Purpose.** The small set of **always-on, imperative** rules + orientation:
- what the project is, in one paragraph;
- where to read first / the repo's layout;
- load-bearing **principles** that constrain *every* task;
- the **out-of-scope** list, so the agent never proposes deliberately-excluded things;
- broad project-specific conventions;
- **pointers** (not duplicated content) to `MEMORY` and `design/` — including a **required-reading line
  for [`design/INDEX.md`](#4-design--the-technical--knowledge-wiki)**. Unlike `MEMORY.md`, the docs index is
  **not** auto-injected each session, so the always-loaded `CLAUDE.md` must explicitly tell the agent to
  read it (then load the relevant doc on demand — the same index-then-load pattern as `MEMORY`).

**Does NOT belong here.** Situational knowledge that isn't relevant every session (→ `MEMORY`),
generic language/framework advice, deep detail (→ `design/`), user-facing framing (→ `README`).

**Cadence.** Stable. A rule lands here when it must bind **every** session — often by *graduating up*
from `MEMORY` (see the split below). Don't let it grow into a wiki.

### 3. `MEMORY.md` (+ `.claude/memory/`) — the agent's accumulated durable knowledge

**Audience.** Claude, across sessions. The **index** (`MEMORY.md`, one line per memory) is loaded
every session; the **topic files** are pulled in **on demand** when a task touches them.

**Mechanics that define it.** Because only the index is always-loaded and the detail is lazy-loaded,
`MEMORY` can hold a **large, growing** body of knowledge cheaply — the situational counterpart to
CLAUDE.md's always-on rules.

**Purpose.** Durable, **declarative** decisions / agreements / facts — *"principles, not state":*
- durable decisions and the **why** behind them;
- user preferences and agreements;
- component- or area-specific facts you should **recall when working there**.

**Does NOT belong here.** Universal rules that bind *every* session (→ `CLAUDE.md`); **ephemeral or
repo-derivable state** — paths, "what's built", commit SHAs (git/code already holds it; it goes stale
on creation); deep architectural rationale (→ `design/`).

**Cadence.** Living. Claude writes to it as decisions accrue — one focused topic file per memory, a
one-line entry in `MEMORY.md` so it's discoverable.

### 4. `design/` — the technical / knowledge wiki

**Audience.** Anyone working on the project's architecture, design, or substance — humans and agents
alike.

**Purpose.** The full record. Architecture, decisions, rationale, plans, post-mortems, glossaries —
the place where deep *why* lives. (For a non-software project, the same surface holds the project's
substantive knowledge base; the role is identical even if the content isn't code.)

**Structure.**
- **`INDEX.md`** — entry point: every document with a one-line description, organised by category. It
  is **required reading**, surfaced by a pointer in the always-loaded `CLAUDE.md` (the index is not
  auto-loaded; the pointer is what makes it reliably read). An un-indexed document is invisible.
- **Content documents** — one focused topic per file, kebab-case (`schema-design.md`).

**One wiki surface, at the top level only.** In a multi-repo (umbrella) project the wiki lives **only at
the top level** — a single shared documentation surface for the umbrella *and* every sub-repo. In FCM
that surface is the `design` repo, cloned into the umbrella root as `design/`. Sub-repos hold **no** wiki
of their own (you always launch from the umbrella, so one surface serves all). In a single-repo project
it is a `docs/` folder at that repo's root. Either way there is exactly one, at the top.

**Exception — stand-alone-reusable sub-repos may self-document.** A sub-repo whose code could
reasonably be used **outside this project** — for example a library a third party might install into an
unrelated project — **may** (not must) carry its own internal `docs/` so the repo is understandable in
isolation by someone who never sees the umbrella. The criterion is **stand-alone reusability**: a
general-purpose library that stands on its own qualifies; a sub-repo that makes sense *only* within this
project does not, so its documentation lives in `design/`. A self-documenting sub-repo follows the
same conventions as `design/` (an `INDEX.md` entry point, one kebab-case topic per file), and
`design/INDEX.md` should list it so it stays discoverable.

**A self-documenting sub-repo carries a _reduced_ set, not the full suite.** Its internal docs are only
what is needed to understand and use the repo in isolation: a `README` (for whoever lands on it),
repo-specific agent context (`CLAUDE.md`), and content `docs/` (an `INDEX.md` plus kebab-case topic files
describing how the code works). It does **not** restate the project's documentation conventions — those
live once, at the umbrella — and it has **no memory surface of its own**; durable agent memory is a
project-level concern held at the umbrella. The sub-repo *follows* the umbrella's conventions; it never
carries a copy of them.

**Does NOT belong here.** Agent instructions (`CLAUDE.md`), user-facing framing (`README.md`),
throwaway scratch notes (if it isn't worth indexing, it isn't worth keeping).

**Cadence.** Active — documents are added/updated as topics arise; history is preserved (supersede,
don't delete).

## The `CLAUDE.md` ⇄ `MEMORY` split (the LLM-facing division)

Both are for the agent, so the boundary matters. Two axes settle it:

- **Always-on vs situational.** `CLAUDE.md` is loaded in full *every* session → only universal,
  every-task content earns a place (keep it small). `MEMORY` is indexed-now / detail-on-demand →
  the larger body of *sometimes-relevant* durable knowledge.
- **Imperative vs declarative.** `CLAUDE.md` = rules you *apply* ("do X, never Y, read Z first").
  `MEMORY` = knowledge you *recall* ("we chose X because Y; the user prefers Z").

**The graduation flow:** a decision is made in conversation → if durable, it's recorded in `MEMORY`
→ **if it must bind every session, it graduates up into `CLAUDE.md`.** If `CLAUDE.md` accumulates
situational detail, that **demotes down into `MEMORY`.** `CLAUDE.md` trends small-and-universal;
`MEMORY` accumulates-and-specialises.

## Quick decision guide

| Question | ⇒ goes in |
|---|---|
| Will a non-developer visitor want to see this? | `README.md` |
| Is it a rule/constraint that must steer **every** agent session? | `CLAUDE.md` |
| Is it a durable decision/agreement/fact to **recall when working on a specific thing**? | `MEMORY` (topic file + index line) |
| Is it architectural detail, decision rationale, or a plan? | `design/` (and indexed in `INDEX.md`) |
| Is it ephemeral or derivable from the repo (paths, "what's built")? | Don't write it — git/code has it |
| Useful for the *current task only*? | Don't write it — keep it in the conversation |
| Is it a comparison, ranking, or self-assessment of our own work? | Don't write it — see *the reader test* |

If content seems to belong in two places, prefer the more durable/detailed surface (`design/` or
`MEMORY`) and **link** to it from the always-on ones (`README`, `CLAUDE.md`), which stay short.

## The reader test

Every sentence on every surface has to answer one of two questions:

1. **How does this work?**
2. **How do I use this in my project?**

A reader — human or LLM — arrives wanting to understand the thing or integrate it. Content that serves
neither purpose is noise, however true it is.

What this rules out:

- **Comparisons between our own repos.** "The lighter library", "unlike the other libraries", "more
  capable than X". A reader of one repo's docs does not need a league table of the others, and a
  comparative only means anything to someone already holding both in their head — precisely the reader
  who didn't need telling.
- **Rankings and self-assessment.** Whether something is clean, elegant, robust, or better than what it
  replaced. None of it changes what the reader types.
- **Justification aimed at the reader's approval** rather than at their understanding.

What survives, and why the distinction is not "drop all rationale": the **why behind a constraint
passes the test**, because it tells a later session what must not be undone. The difference is whether
the sentence states a fact about *this* thing or a fact about some other thing's relative standing:

| Fails | Passes | Because |
|---|---|---|
| "Keeps the heavy dependency out of the lighter library" | "Depends on Django, pgvector and numpy — a consumer wanting only chat never installs them" | The second is what it costs to install |
| "A cleaner design than owning the provider" | "Holds no API key or base URL; the consumer injects the embedder" | The second is how you wire it up |
| "Our most mature library" | "The reference for the test-runner pattern" | The second tells you why to open it |

Apply the test to a sentence you are about to write, not only to one you are reviewing. When it fails,
the usual fix is not deletion — it is restating the same point as a fact about the thing being
documented.

## Conventions for `design/` documents

- **Filename:** kebab-case, descriptive, stable (`schema-design.md`, not `notes-2026-04.md`).
- **First line:** an `# H1 Title` matching the filename.
- **Second block:** a one-paragraph summary — what shows in a preview pane and gets quoted in `INDEX.md`.
- **Cross-references:** relative links to other docs (`[schema design](schema-design.md)`); leading
  slash to root files (`[CLAUDE.md](../CLAUDE.md)`).
- **Third-party code references:** cite a dependency's source as an inline-code path relative to that
  package's own root (e.g. `` `package/module/file` ``, `` `ClassName.method()` ``) — never as a working
  link into a local install directory (a virtualenv, `node_modules`, vendor folder, or other
  machine-specific path). The dependency's real home is upstream, not this repo; a link into a local
  install breaks across machines and operating systems. Link upstream (a permalink to the project's
  repository) only if you can pin a stable revision.
- **No stale-prone metrics.** Don't cite counts that drift and are trivially re-derived — test
  counts, line/file counts, coverage percentages. They go stale the moment the code changes, and anyone
  who needs the number can run the tool in seconds. State the *capability* ("covered by a test suite"),
  not the *tally* ("390 tests").

  Better still, **link to the document that cannot drift**. A library's `docs/test-plan.md` lists every
  case and the test covering it, and the linter fails if the two disagree in either direction — so a
  link to it stays true for free, while a count is wrong the next time a case lands. Prefer "every case
  the suite covers is listed in `docs/test-plan.md`" — as a link — over any sentence carrying a number.
- **No counts of things, anywhere — including headings.** The rule above extends past derived metrics
  to any tally a doc states as fact: how many databases, how many surfaces, how many public functions,
  how many message types. Every one of them is correct until the day something is added, and then it is
  wrong in a heading, in a table of contents, and in whatever else linked to it. Frame around *the set*
  rather than its size — "The databases", not "The five databases"; "exports the public functions", not
  "exports all ten public functions". Where the plurality itself matters, say "multiple" or "several".
  The exception is a number that is part of a decision rather than a description: "two database
  backends, SQLite and PostgreSQL" is naming a deliberate choice, not counting a collection.
- **No volatile state.** Don't record what is currently enabled, disabled, running, or switched
  off — it can flip within hours, and a later reader has no way to tell whether the doc or the
  code is right. Describe how the thing is *designed* to work; if it isn't built or isn't
  running, describe what it needs to do. Then name the file that is authoritative for current
  status, so a reader checks there rather than trusting the doc. "Which scripts run on which
  role is declared in `server/conf/at_server_startstop.py`" stays true; "the telemetry pipeline
  is enabled" is stale the next time someone comments a line out. This is the runtime sibling of
  *no stale-prone metrics* above.
- **Write the design as finished; carry status in a removable note.** A design doc describes how the
  game *works*, in present tense, whether or not the code has caught up yet. Where a part isn't built,
  put a short blockquote note directly above it saying so — never hedge the prose itself into "will
  be" / "is planned to". Shipping the feature is then a one-line edit: delete the note. The
  description was already correct.

  This keeps the doc a **target** rather than a status report. Progress notes are cheap to add and
  remove; a design rewritten every time implementation moves becomes a changelog, drifts, and loses
  the authority that makes it worth consulting. It also makes remaining work visible — the notes are
  the to-do list, and an absence of notes means the doc and the code agree.

  The design prose is stable against *implementation progress*, not against *design change*. If a
  decision is genuinely revisited, the prose changes — deliberately, as its own act, with the *why*
  captured. What must never happen is the design quietly eroding to match whatever got built.

  **Events go to [progress.md](progress.md), not into the design doc.** "Verified on Windows in May",
  "validated on macOS under these conditions", "ran the suite and it passed" — these are milestones,
  not design, and a design doc that accumulates them becomes a status report. Record the design in the
  topic doc; record what has actually been proven, and what hasn't, in the progress log. Each library
  keeps its own `docs/progress.md` on the same rule.

- **Index it.** Add a one-line entry to `INDEX.md`. An un-indexed document is invisible.
- **Capture the *why*.** A doc that says *what* without *why* is obsoleted by the first strong
  counter-argument. The *why* is what makes it durable.
- **Don't delete; supersede.** Leave a superseded doc in place with a note linking forward.

## Why four surfaces and not one

A single `README` is too long for visitors or too shallow for contributors. `CLAUDE.md` alone either
pulls in agent-irrelevant marketing or omits depth. `MEMORY` exists because an agent needs durable
recall that *accumulates* without bloating the always-loaded rules. `design/` without an entry point is
invisible. Four surfaces, each focused on one audience and one cadence, linking where helpful: the
cost is a little discipline about what goes where; the benefit is each surface stays useful for the
long haul.
