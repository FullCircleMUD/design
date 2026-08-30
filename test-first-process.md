# Test-First Process

The standard development process across the whole FCM umbrella — the umbrella itself, the libraries
under `libraries/`, `src/game`, and every other sub-repo. Behaviour is defined in the test plan first,
then the tests are written against it, then the code is written to pass them. This is not a
library-only convention and it is not reserved for greenfield work; it is how work is done here.

## The order

**1. Update the test plan.** Before any other work, the test plan gets the cases that define the
behaviour being added or changed. Each case gets a stable ID and a description of what must be true.
The **Test function** column stays empty — nothing covers it yet.

This is the step that defines the behaviour. Everything downstream implements what the plan says, so
disagreement about what the thing should do gets settled here, in a table, rather than discovered
halfway through an implementation.

**2. Write the tests.** One test per case, filling in the **Test function** column as each lands.

If a test can't be written because the thing it calls doesn't exist yet, **create the placeholder**:
the method, function or class with the real signature and a body of `raise NotImplementedError`. The
test is then writable and honest — it fails for the right reason.

**3. Run them. Failures are expected.** A suite that is red at this point is the process working. The
red tests are the specification, expressed as something executable.

**4. Write the code.** Implement until the tests pass. The definition of done is the plan's cases
going green, not a subjective sense that the feature works.

## Why this order

The plan is a **commitment, not a wishlist** — a case in the table is a case the code will satisfy.
Agreeing the cases before writing anything is what makes that commitment meaningful; a plan written
after the code is a description of whatever got built, which is a different and much weaker document.

The **Test function** column is the coverage trail. It reads in both directions: from a case to the
test proving it, and from a test back to the behaviour it exists to defend. An empty cell is a visible
work item rather than a silent gap, so the plan doubles as the work queue.

Writing the tests before the implementation also constrains the implementation's shape. A surface that
is awkward to test is usually awkward to use, and the process surfaces that while it is still cheap to
change.

## What the plan and the suite must agree on

The trail is checked in both directions, mechanically, by the `test-plan-linter` skill. Four ways the
plan and the suite can disagree, and all four are errors:

- **A case names a test that does not exist.** The column is a coverage claim, so a name resolving to
  nothing — or to a helper rather than a test — is a false one.
- **A test no case names**, a *ghost test*. The case is agreed before the test is written, so a test
  with no case means the plan was skipped. The plan records what the code commits to; a test outside it
  defends behaviour nobody agreed.
- **A case ID used twice.** IDs are referenceable, so reusing one makes a case invisible to the trail.
- **A case still carrying an unresolved marker**, below.

An empty **Test function** cell is not a disagreement. It is the normal in-progress state, and warns
rather than errors.

A **test function**, for this purpose, is a `test_*` function in a `tests.py` or `test_*.py` module,
found by parsing the module. Helpers, fixtures and `setUp` are not tests, and the standalone test
infrastructure (`test_settings.py`, `urls.py`, `conftest.py`) is excluded.

### Unresolved cases do not pass

A case carrying `[TBD — needs discussion: …]` is an **error**, not a warning: the decision is made
before the plan passes. Flagging an open question is still the right move — it just does not ship
unresolved.

To write the marker without raising one, as a document explaining the convention must, escape it with
a backslash: `\[TBD]` is documentation and is ignored.

## Where the plan lives

For a reusable library under `libraries/`, the plan is `docs/test-plan.md` — required by
[library-standards.md](library-standards.md). The `library-standards-linter` checks it by delegating to
the `test-plan-linter`, so the rules above apply identically wherever a plan lives; a consumer supplies
the plan path and the directories holding its tests.

`[TBD — needs discussion: where the test plan lives for repos without their own docs/ wiki, notably
src/game. The libraries' docs/test-plan.md location does not transfer directly, since those repos
document into design/ rather than internally.]`

## Reference shape

[evennia-targeting's test plan](../libraries/evennia-targeting/docs/test-plan.md) is the reference:
the prefix legend, the fixtures table, the per-surface case tables, and the Open decisions section
collecting the `[TBD]`s.
