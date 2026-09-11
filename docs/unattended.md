# Manual one-time setup

First-time setup steps for `shrike` that aren't worth automating, or
that automation can't reach. Periodic / maintenance procedures live in
`docs/maintenance.md`.

## Contents

- [Unattended boot (autologon)](#unattended-boot-autologon)
- [SSH transport for desktop apps](#ssh-transport-for-desktop-apps)

---

## Unattended boot (autologon)

Goal: cold boot → desktop → ready for Steam Remote Play / SSH, no
intervention at the keyboard.

### Why this is manual

Encoding autologon in Ansible requires the MSA password at every
playbook run, even when the credential is already armed and stable. The
configuration doesn't drift in practice (the stored LSA secret is valid
until the MSA password rotates, which is rare), so the cost of keeping
it as code outweighs the rebuild convenience.

### Constraints

- **MSA login is preserved** for software entitlement (Office, Steam,
  Dropbox account state, etc.).
- The user is `mail` — the truncated local form of `mail@jakestanley.co.uk`.
  Confirm with `whoami` on shrike if unsure.

### One-time setup

Run from an elevated PowerShell on shrike:

```powershell
# 1. Re-allow MSA password sign-in. Windows defaults this to passwordless-only
#    after Hello enrollment, which blocks autologon. Setting it to 0 lets the
#    stored LSA secret be honoured at boot.
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\PasswordLess\Device" `
    /v DevicePasswordLessBuildVersion /t REG_DWORD /d 0 /f

# 2. Install Sysinternals Autologon (one-off tool, ~50KB).
winget install -e --id Microsoft.Sysinternals.Autologon

# 3. Arm autologon. Uses the same password you'd type at the MSA lock screen.
#    Stores it encrypted in LSA secrets, not plaintext in the registry.
autologon mail $env:COMPUTERNAME '<your-MSA-password>' /accepteula
```

Reboot to verify. If you land on the desktop, you're done.

### When you'll need to redo this

- **You change the MSA password.** Re-run step 3 with the new password.
- **A Windows update flips `DevicePasswordLessBuildVersion` back to 2.**
  Re-run step 1.

### Verifying it's armed

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AutoAdminLogon
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
```

`AutoAdminLogon=1` and `DefaultUserName=mail` confirms autologon is armed.
The password lives in LSA secrets, not in the registry, so it won't show
up in a `reg query`.

### Undoing it

```powershell
autologon -d
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\PasswordLess\Device" `
    /v DevicePasswordLessBuildVersion /t REG_DWORD /d 2 /f
```

---

## SSH transport for desktop apps

Used by `playbooks/shrike-desktop-apps.yml` to install/upgrade the
canonical desktop app set (see `ansible/roles/desktop_apps/defaults/main.yml`).

### Why SSH instead of WinRM

WinRM as the `ansible` local admin can't execute `winget` — the
`Microsoft.DesktopAppInstaller` AppX package's WindowsApps directory
denies execute access to users who have never logged in interactively
(see `README.md` "Platform limitations"). OpenSSH on Windows runs
sessions in interactive-equivalent context, so a user that has logged
in once on the desktop (`mail`) has AppX provisioned and can invoke
`winget`.

The MSA passwordless setting on `mail` does not block SSH **key auth** —
keys are validated by `sshd` and the session token is built via Win32
APIs without consulting Windows password authentication. The same MSA
setting that breaks WinRM/NTLM is invisible to SSH keys.

### One-time setup

Uses your existing controller ed25519 keypair at `~/.ssh/id_ed25519`.
No new key is generated. From the controller:

```sh
nix-shell
cd ansible
ansible-playbook playbooks/shrike-ssh-bootstrap.yml --ask-pass
```

The playbook installs OpenSSH Server, opens the firewall, sets
PowerShell as the default shell, writes your public key into
`C:\ProgramData\ssh\administrators_authorized_keys` (the file sshd
uses for admin-group users — `mail` is one), and applies the strict
ACLs sshd requires.

Pass `-e ssh_public_key_path=/some/other/path.pub` if you want to
publish a different key.

### Verify

```sh
ssh mail@shrike.stanley.arpa whoami
# expect: mail
```

No password prompt. If you get one, key placement or ACLs failed —
check `C:\ProgramData\ssh\administrators_authorized_keys` on shrike
and the sshd event log.

### Apply the desktop app set

```sh
ansible-playbook playbooks/shrike-desktop-apps.yml
```

No prompts. The role is idempotent — re-runs are safe. The task pre-checks
`winget list` per package and skips the install call entirely when
already present, which also sidesteps the MSIX-hang failure mode (some
packaged apps, notably `AgileBits.1Password`, hang the winget install
CLI when a running instance would need an interactive close dialog).
See `AGENTS.md` "Adding a new desktop app" for how to modify the list.
