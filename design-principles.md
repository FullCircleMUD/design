# Design principles

> Always try these first. Varying from one is allowed only when it genuinely cannot work — which
> should be rare — and the code comments must say why it couldn't be done this way.

The first principles currently applied across FCM development: how each aspect of the game is built,
independent of any one component.

## 1. DRY — don't repeat yourself

### 1a. Parsing input and filtering contents

- Parsing player input, and searching or filtering an object's contents, starts with the
  [parser and filter inventory](parser-filter-inventory.md).
- If nothing fits, write a helper in the right place, where it becomes the standard for the whole
  build. Never a one-off.
- Log every new parser or filter helper in the inventory.

## 2. Decoupling

- **Reading another component's data** — through a public read method exported from its package (never reach
  inside another component; use only its public interface).
- **Changing data another component owns** — by signal. The owner's receiver makes the change.
  - *Announcement*, one-to-many: something happened; each listener updates its own data.
  - *Request*, many-to-one: any sender asks the one owner to do a job only it does.
- **Coordinating across components** — work no single component owns — through hooks on the game's
  typeclasses and mixins. A hook may send a signal for any part that is a component's own data.
- The test: name the receiving component and the attribute it changes. If you can't, it's a hook.
- Signals are declared in `components.signals`; sender and receiver import it, never each other.

## 3. Decompose, test each unit, then compose

- Break a complex task into units, test each unit on its own, then compose them.
- Every unit has its own tests. A top-level test alone is not coverage.

## 4. Test in isolation

- A component tests only its own behaviour.
- Another component it touches is mocked or stubbed at its public interface.
- A component sending a signal tests that it sends it, never what the receiver does with it.

## 5. Typeclasses compose, they don't own

- An in-game typeclass declares no attributes of its own. Every attribute comes from a mixin provided
  by a component or library.
- A typeclass may assign values to attributes its mixins declare.
- The only methods on a typeclass are overrides of hooks — Evennia's or a mixin's — that coordinate
  behaviour across components and libraries.

## 6. Commands decide, then execute

- A command decides whether it may run, then calls the method that does the work.
- A brief decision lives in the command. A long or complex one moves to its own helper or helpers.
- The execution method never decides whether to run. By the time it is called, that is settled.

## 7. Static data is a validated record

- Static game data — kit classes, races, languages — is declared as frozen dataclass instances, never
  raw dicts.
- Each record validates its fields on construction, so bad data fails at boot, not at runtime.
- Every dict in a record is a `MappingProxyType` over a copy. `frozen` stops a field being replaced,
  not the dict it holds being changed.
- Records are collected in a `StaticRegistry`.
