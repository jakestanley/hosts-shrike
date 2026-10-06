# Sleep-on-LAN

A small HTTP server (and optional UDP listener) on `shrike` that puts the
host to sleep when called. Upstream: <https://github.com/SR-G/sleep-on-lan>.

The HTTP endpoint is the primary interface for tools like Upsnap.

## What the role does

`ansible/roles/common/tasks/sleep-on-lan.yml`, tagged `networking`:

1. Ensures NSSM is installed (it's the service wrapper).
2. Downloads the release zip pinned by `sol_version` in
   `roles/common/defaults/main.yml`.
3. Extracts to `C:\Program Files\SleepOnLan\` (the zip's `windows_amd64/`
   subdir ends up at `C:\Program Files\SleepOnLan\windows_amd64\`).
4. Registers `SleepOnLan` as an NSSM service, starts it, sets it to auto.
5. Opens inbound TCP 8009 (HTTP) and UDP 9 (magic-packet) on domain +
   private profiles.

## HTTP endpoints

The bundled `sol.json` makes SOL listen on TCP 8009 by default:

| URL | Effect |
|---|---|
| `GET /` | Index page; shows local IP / MAC |
| `GET /sleep` | Sleep the host |
| `GET /state/local` | Reports whether the host is alive |
| `GET /quit` | Exits the SOL process |

So to sleep `shrike` from anywhere on the LAN:

```sh
curl http://shrike.stanley.arpa:8009/sleep
```

No auth — anyone who can reach TCP 8009 can sleep the host. Fine for a
trusted LAN; don't expose 8009 externally.

## Upsnap integration

Point Upsnap's shutdown / sleep action at the HTTP endpoint:

```
http://10.92.8.4:8009/sleep
```

Field name varies by Upsnap fork — look for "shutdown command",
"HTTP URL", or "ping URL on shutdown". Method is `GET`.

## UDP magic-packet path (optional)

SOL also accepts standard WoL magic packets on UDP 9, but expects the
target MAC bytes to be **reversed**. Same-tool, same-port as WoL — the
reverse is how SOL distinguishes a sleep request from a wake request.

```sh
# wake shrike (normal MAC)
wakeonlan AA:BB:CC:DD:EE:FF

# sleep shrike (reversed MAC, UDP 9)
wakeonlan -p 9 FF:EE:DD:CC:BB:AA
```

Use this if you have a tool that already speaks WoL but not HTTP. For
Upsnap, HTTP is simpler.

## Bumping the upstream version

Edit `sol_version` in `ansible/roles/common/defaults/main.yml` and re-run:

```sh
ansible-playbook playbooks/shrike-bootstrap.yml
```
