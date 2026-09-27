# Design principles

The first principles currently applied across FCM development: how each aspect of the game is built,
independent of any one component.

## 1. DRY — don't repeat yourself

### 1a. Parsing input and filtering contents

- Parsing player input, and searching or filtering an object's contents, starts with the
  [parser and filter inventory](parser-filter-inventory.md).
- If nothing fits, write a helper in the right place, where it becomes the standard for the whole
  build. Never a one-off.
- Log every new parser or filter helper in the inventory.
