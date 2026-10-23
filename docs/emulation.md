# Emulation

`playbooks/shrike-emulation.yml` runs the `emulation` role over the
SSH-as-`mail` transport (same as the desktop apps playbook). It
installs:

| App             | Source                                        | Install location                 |
|-----------------|-----------------------------------------------|----------------------------------|
| RetroArch       | winget `Libretro.RetroArch` (machine scope)   | `C:\RetroArch-Win64` (default)   |
| LRPS2 (PS2)     | libretro nightly buildbot `pcsx2_libretro.dll`| RetroArch's `cores\`             |

RetroArch's installer adds its own Start menu entry.

Cores are listed in `retroarch_cores` in
`ansible/roles/emulation/defaults/main.yml`, by core name
(`<name>_libretro.dll`, e.g. `pcsx2`). They come from libretro's
nightly buildbot and aren't pinned: the role installs a core once if
it's missing and never replaces it, so update cores from RetroArch's
**Online Updater → Update Installed Cores**.

## Not managed

- **PS2 BIOS**: LRPS2 needs a BIOS dumped from your own console. Put
  the files in `system\pcsx2\bios\` under RetroArch's install dir.
- Games, RetroArch settings, controller bindings and per-core options.
