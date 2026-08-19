# New-Machine Setup — re-cloning the FCM umbrella

How to reconstitute a working FCM environment on a fresh machine. The umbrella repo carries the
coordination layer (`.claude/`, `CLAUDE.md`, `docs/`, memory, the view-gamedir symlinks) but
**gitignores the nested working repos** — so cloning the umbrella does *not* bring them down. Re-clone
them with the manifest below.

## 0. Prerequisites

- **git**, **Node** (for the memory hook; not bundled with Claude Code's native installer — `brew
  install node` / `winget install OpenJS.NodeJS`), and **git-crypt** (`brew install git-crypt` /
  `choco install git-crypt`) for the encrypted game secrets.
- **GitHub access to the `FullCircleMUD` org.** Several repos are **private**. The local `gh` CLI has
  two accounts; the **`FullCircleMUD`** account is the one with access, not `timbaird`. Switch with
  `gh auth switch --user FullCircleMUD` before cloning the private repos (switch back afterward if you
  prefer `timbaird` active).

## 1. Clone the umbrella

```bash
git clone https://github.com/FullCircleMUD/fcm-umbrella.git
cd fcm-umbrella
```

## 2. Re-clone the nested working repos

Folder names must match the umbrella `.gitignore` entries exactly. All remotes are
`https://github.com/FullCircleMUD/<name>.git`.

### Top-level repos (clone into the umbrella root)

| Repo | Private? | Default branch |
|---|---|---|
| cosigner | public | main |
| fullcirclemud | public | main |
| lore | public | main |
| nft_api | public | main |
| transparency | public | main |
| fcm-mobs | **private** | main |
| fcm-world | **private** | main |
| llm-test-harness | **private** | main |
| ops | **private** | main |
| xrpl-tools | **private** | main |

> `design` is intentionally **not** here — its content was migrated into this `docs/` surface and the
> repo retired (the umbrella is now the source of truth for design documentation).

### `libraries/` (clone into `libraries/` — see `libraries/README.md`)

`evennia-mob-spawner`, `evennia-mob-spawner-test-world`, `evennia-mob-spawner-test-yaml`,
`evennia-shards`, `evennia-targeting`, `evennia-world-builder`, `evennia-world-builder-test-yaml`,
`evennia-yaml-reader` — all public, branch `main`.

### `src/`

- `src/game` (public; **git-crypt'd** — see §4). Default branch `main`; active working branch is
  `shards-rework`.
- `src/game-router/` and `src/game-shard1/` are **tracked by the umbrella** (Unix view gamedirs of
  symlinks back to `../game`) — they arrive with the umbrella clone, no separate clone needed. They
  are a Unix-only multi-shard dev convenience; Windows runs all roles directly from `src/game`.
- `src/venv` is **not** tracked — recreate it from the game repo's `requirements.txt`.

For the private repos, ensure the `FullCircleMUD` gh account is active first (§0).

## 3. Memory (portable, self-healing)

Memory lives in `.claude/memory/` (committed). On first launch **from the umbrella root**, the
`SessionStart` hook (`node .claude/hooks/ensure-repo-memory.mjs`) writes the machine-local
`autoMemoryDirectory` into `.claude/settings.local.json` (gitignored) and prints `FIXED … RELAUNCH
REQUIRED` — **relaunch once**, then memory loads from the repo. Accept the workspace-trust prompt.

## 4. Unlock the game secrets (git-crypt)

`src/game` encrypts `server/conf/secret_settings.local` with git-crypt (symmetric key, shared
out-of-band — there is no committed key). A fresh clone is **locked**; unlock before the game runs:

```bash
cd src/game
git-crypt unlock /path/to/fcm-game-gitcrypt.key
```

To produce the key from an already-unlocked clone: `git-crypt export-key <path-outside-repo>/fcm-game-gitcrypt.key`.
Keep the key **outside any repo** and never commit it. (Details: `.claude/memory/gitcrypt_game_secrets.md`.)

## 5. The golden rule

**Always launch Claude from the umbrella root** — even when editing inside a sub-repo. Skills and each
sub-repo's `CLAUDE.md` cascade down on demand, but agents, settings, rules, and memory load only from
the launch dir. Launch inside a sub-repo and you silently lose the umbrella's shared layer.
