# Doom

`playbooks/shrike-doom.yml` runs the `doom` role over the SSH-as-`mail`
transport (same as the desktop apps playbook). It installs:

| App         | Source                                   | Install location                                    |
|-------------|------------------------------------------|-----------------------------------------------------|
| UZDoom      | winget `UZDoom.UZDoom` (portable zip)    | winget's portable package dir                       |
| dsda-doom   | GitHub release `dsda-org/dsda-doom`      | `%LOCALAPPDATA%\Programs\dsda-doom`                 |
| Doom Runner | GitHub release `Youda008/DoomRunner`     | `%LOCALAPPDATA%\Programs\DoomRunner`                |

All three are portable, so the role adds a Start menu shortcut for each
in `mail`'s profile.

GitHub releases are pinned by tag and sha256 in
`ansible/roles/doom/defaults/main.yml`; to upgrade, bump `version`,
`asset`, `sha256` (and `archive_root` if versioned) together.

## Doom Runner engines

The role registers UZDoom and dsda-doom as engines in Doom Runner's
`options.json`, which it keeps next to its exe. Engines already listed
(by exe path), and every other setting, are left untouched, so
configuration done in the GUI survives re-runs.

- Close Doom Runner before running the play. It rewrites `options.json`
  on exit, so the task skips while it's running.
- On a fresh install (never launched) the task writes a minimal
  `options.json` seed; Doom Runner fills in the rest with defaults and
  rewrites it on first exit.

## Not managed

- **IWADs** (`DOOM.WAD`, `DOOM2.WAD`, ...): the engines ship without
  game data. Add the IWAD directory in Doom Runner's Initial Setup.
- Doom Runner presets, mod lists and a default engine.
