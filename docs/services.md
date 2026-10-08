# Services

The `services` role clones each homelab service repo under `C:\homelab\`,
seeds its `.env`, and runs each repo's `scripts/up.ps1` to install the NSSM
service.

## Run

```sh
ansible-playbook playbooks/shrike-bootstrap.yml
```

The whole playbook is idempotent so a full re-run is the standard
"deploy new service code" workflow. If you only want to re-run a
specific step, use `--start-at-task`:

```sh
ansible-playbook playbooks/shrike-bootstrap.yml --list-tasks
ansible-playbook playbooks/shrike-bootstrap.yml \
    --start-at-task "Install or update NSSM services via each repo's up.ps1"
```

## What gets installed

`ansible/roles/services/defaults/main.yml` defines the list:

- `homelab-demucs` — Demucs separation API
- `homelab-ollama` — Ollama HTTP wrapper
- `homelab-relay` — RTMP stream relay (MediaMTX, NVENC transcoding,
  archive to `D:\RelayRecordings`)

## homelab-relay

- **Never run the services role during a broadcast.** It passes `-Restart`
  to every `up.ps1`, which restarts all of the relay's NSSM services and
  drops the stream. To change relay config mid-broadcast, edit `.env` on
  shrike and run `C:\homelab\homelab-relay\scripts\up.ps1` without
  `-Restart` (elevated). That restarts only what changed.
- **`INGEST_KEY` is set by hand, once.** It must equal batw's
  `STREAM_KEY_LIVE` (letters, digits, `_ . -` only), and `up.ps1` refuses
  the `.env.example` placeholder. The first deploy clones and seeds `.env`,
  then fails at the config check. Set the key in
  `C:\homelab\homelab-relay\.env` and re-run. The role never overwrites an
  existing `.env`, so later deploys need no input. Platform stream keys are
  optional; an output with no key is disabled.
- The first run needs internet: `up.ps1` downloads the pinned MediaMTX and
  ffmpeg into `tools\`, and pip installs into `.venv`.
- Logs: `C:\homelab\homelab-relay\logs\`. Services:
  `homelab-relay-{mediamtx,recorder,status,out-*}`.

Override `services_root` or `homelab_services` from inventory if you want a
different install location or a different set of repos.

## Prerequisites managed by the role

- `git` (clone + pull)
- `python3` (interpreter for the repo venvs)
- `nssm` (each repo's `up.ps1` uses it)

Installed via Chocolatey at the top of the role; idempotent.

## Prerequisites NOT managed by the role

- **NVIDIA drivers + `nvidia-smi` on PATH** — required by Telegraf's
  `nvidia_smi` input and `homelab-demucs`. Install via GeForce Experience or the NVIDIA driver
  installer.
- **Ollama** — required by `homelab-ollama`. Install from
  <https://ollama.com/download/windows>. The seed task auto-fills
  `OLLAMA_EXE` in `.env` if `ollama.exe` is on PATH at clone time.
- **CUDA-enabled PyTorch in `homelab-demucs`'s venv** — Demucs needs torch
  built against CUDA. The current `up.ps1` installs from `requirements.txt`;
  if CUDA isn't picked up automatically, install torch manually in the
  venv from <https://pytorch.org/get-started/locally/> and re-run.

## `.env` files

On first clone, `.env` is seeded from `.env.example` with three layers of
substitution:

1. `*_PYTHON_EXE` lines are rewritten to the discovered `python.exe` path.
2. `OLLAMA_EXE` is rewritten to the discovered `ollama.exe` path (if found).
3. Any keys listed under a service's `env_overrides` in
   `roles/services/defaults/main.yml` are rewritten to the override value.

Example (`defaults/main.yml`):

```yaml
homelab_services:
  - name: homelab-demucs
    url: https://github.com/jakestanley/homelab-demucs.git
    env_overrides:
      STORAGE_ROOT: D:\demucs
```

Other host-specific paths (`DATA_ROOT`, port overrides, etc.) keep their
`.env.example` values. Edit `C:\homelab\<repo>\.env` directly, then re-run
the playbook to re-apply.

The seed task only runs when `.env` is missing — your edits never get
clobbered. To re-trigger seeding for one service, delete its `.env` and
re-run the playbook.

## Where state lives

| Service | NSSM service name | Logs | Data |
|----------------|------------------|-----------------------------------|----------------------------|
| homelab-demucs | `homelab-demucs` | `C:\homelab\homelab-demucs\logs\` | `STORAGE_ROOT` from `.env` |
| homelab-ollama | `homelab-ollama` | `C:\homelab\homelab-ollama\logs\` | `STATE_DIR` from `.env`    |

## Inspecting a single service

```powershell
Get-Service homelab-demucs
nssm get homelab-demucs AppEnvironmentExtra
Get-Content C:\homelab\homelab-demucs\logs\homelab-demucs-stdout.log -Tail 50
```
