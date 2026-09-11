# AGENTS

Conventions for AI agents (and humans) working in this repo.

## Repo purpose

This repo is the canonical source of truth for provisioning the Windows
host `shrike`. It contains:

- An Ansible bootstrap (`ansible/`) for everything that can be managed
  cleanly over WinRM (services, registry, networking, AppX bloatware
  removal, homelab service clones) and the desktop apps role driven
  over SSH-as-mail (`roles/desktop_apps/`).
- Docs (`docs/`) covering the one-time manual setup that bootstraps the
  SSH transport and autologon.

Two transports for two reasons:

- **WinRM as `ansible`** (local admin) for the standard machine config.
  Network logon, no AppX provisioning needed.
- **SSH key auth as `mail`** (MSA user) for winget. SSH key auth
  bypasses Windows LSA password auth, so the MSA passwordless setting
  doesn't block it; the session runs in the interactive-equivalent
  context that AppX/winget require.

## Running Ansible from the controller

`ansible-playbook` is not on the host `PATH` — the controller environment
is provided by `shell.nix`. Enter it first:

```sh
nix-shell                                                # from repo root
cd ansible
ansible-playbook playbooks/<playbook>.yml [--ask-pass]
```

The `shellHook` installs the collections from `ansible/requirements.yml`
into `.ansible/collections` on first entry. Use this for any deploy or
re-run (services role, desktop apps, SSH bootstrap, etc.).

## Adding a new desktop app

Desktop GUI apps are managed by the `desktop_apps` role over SSH.
The canonical list is `ansible/roles/desktop_apps/defaults/main.yml`.

To add a new app:

1. Find its id with `winget search <name>` on a Windows machine, or
   check https://winget.run / https://winstall.app.
2. Add an entry to `desktop_packages` in
   `ansible/roles/desktop_apps/defaults/main.yml`, keeping the
   alphabetisation within its source group. Use `source: 'winget'`
   for the community repo and `source: 'msstore'` for Microsoft Store
   items.
3. Add a trailing `# Display name` comment if the package id is opaque
   (anything from `msstore` always needs one — they use numeric ids).
4. The user applies it with
   `ansible-playbook playbooks/shrike-desktop-apps.yml`. The role is
   idempotent; re-runs are safe.

## Desktop apps: winget install hangs on MSIX packages

If a `winget install` invocation from the `desktop_apps` role hangs
indefinitely (no CPU, no network, no visible installer child process),
suspect an MSIX package whose installer would trigger an interactive
close dialog when a running instance of the app is present. The dialog
never renders under the non-interactive SSH-as-mail transport, so the
call blocks until the winget PID is killed by hand. `AgileBits.1Password`
is the reference case; expect the same pattern from any MSIX / Store-
packaged app that self-updates outside of winget.

The role's install task pre-checks `winget list --exact --id <id>` per
package and short-circuits with an "already installed" marker on match,
which sidesteps the failure mode entirely once the app is present. So:

- Do NOT swap to a `.MSI` id "for a real unattended path" — for
  1Password specifically there is no such id on the default winget
  source (only `AgileBits.1Password`, `.Beta`, `.CLI`); the current
  `AgileBits.1Password` manifest already ships an MSIX bundle.
- Do NOT drop the entry from `desktop_packages` — the pre-check keeps
  the play idempotent, and leaving the id in place still gets it
  installed on a fresh rebuild (when the app isn't running yet, the
  install call itself works fine).
- If a first-time install on a rebuild does hang, the interactive
  fallback is: SSH in interactively (or use the console), close the
  running app, re-run the play.

## Removing a desktop app

Remove the entry from `desktop_packages` in
`ansible/roles/desktop_apps/defaults/main.yml`. The role does not
uninstall — uninstallation is a separate, manual step
(`winget uninstall --exact --id <id>` from an interactive shell or
SSH session as `mail`). Removing from the list just stops re-installing
on a fresh rebuild.

If the app needs to be actively uninstalled from existing hosts (e.g.
because we no longer want it provisioned for new user accounts), the
Ansible `common` role's "Remove default Microsoft bloatware AppX
packages" task uses native `Remove-AppxPackage` and works over WinRM —
extend that list instead. AppX-only path; for classic Win32 installers,
manual uninstall remains the answer.

## Adding a Chocolatey package

`ansible/roles/services/tasks/main.yml` uses `chocolatey.chocolatey.win_chocolatey`
for packages the homelab services consume programmatically (git, python,
nssm, ffmpeg, ollama). The `sunshine` role uses it for its own install.
New entries should go into the role that owns the consuming surface; do
not introduce Chocolatey into `common`, `gaming`, or `desktop_apps`.

## Adding a manual maintenance procedure

Periodic tasks the user runs by hand (driver updates, etc.) live in
`docs/maintenance.md`. First-time-only setup procedures live in
`docs/unattended.md` and similar focused docs. Do not mix the two.

## Conventional commits

Prefix commit messages with `feat:`, `fix:`, `refactor:`, `docs:`,
`chore:`, etc. Keep commits atomic where reasonable. Long fix-forward
debugging sessions may be squashed into a single commit once the run
succeeds — do not commit each attempted fix.
