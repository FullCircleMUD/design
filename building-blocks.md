# Building blocks

The game is built from libraries and components, assembled into typeclasses and prototypes.

| Block | What it is | Where |
|---|---|---|
| Library | An external package, installed into the environment through `requirements.txt`. See the [library catalogue](library-catalogue.md). | `requirements.txt` |
| Component | The game's main building block. Modular; may depend on libraries and other components, declared in `DEPENDS_ON` in its `__init__.py`. See the [component catalogue](component-catalogue.md). | `components/` |
| Typeclass | A persistent game object's class, composed from component and library mixins. | `typeclasses/` |
| Prototype | A template spawning an object from a typeclass with preset values. | `world/prototypes/` |

Paths are relative to `src/router/`.
