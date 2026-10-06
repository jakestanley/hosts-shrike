# Sunshine

[Sunshine](https://github.com/LizardByte/Sunshine) is a self-hosted
game stream host that speaks the Moonlight protocol. It runs on
`shrike` alongside Steam Remote Play, giving lower-latency streaming
to phones/tablets/dedicated clients without going through Steam.

## What the role does

- **Install** — `sunshine` and `vigembus` chocolatey packages (over
  SSH as `ansible`, same transport as the `services` role). ViGEmBus is a virtual
  gamepad bus driver; Sunshine refuses to start without it ("Fatal:
  ViGEmBus is not installed or running"). A host reboot may be needed
  the first time the driver lands.
- **Firewall** — inbound TCP `47984,47989,47990,48010` and UDP
  `47998-48000,48010` on domain + private profiles.
- **Config** — renders `C:\Program Files\Sunshine\config\sunshine.conf`
  from `roles/sunshine/templates/sunshine.conf.j2`. Changes notify a
  handler that restarts `SunshineService`.
- **Service** — ensures `SunshineService` is started and set to
  `auto`.

## Tunables

In `roles/sunshine/defaults/main.yml`:

- `sunshine_name` — display name shown to Moonlight clients.
- `sunshine_port` — base port (Sunshine derives the rest from this).
- `sunshine_min_log_level` — `verbose | debug | info | warning | error | fatal | none`.
- `sunshine_firewall_tcp_ports` / `sunshine_firewall_udp_ports` —
  must be kept in step with `sunshine_port` if you change it.

## Applying

Bundled with `shrike-bootstrap.yml`. For a focused run:

```sh
cd ansible
ansible-playbook playbooks/shrike-sunshine.yml
```

## After install — manual one-time setup

Sunshine's web UI credentials and pairing flow are interactive and
not managed here (same posture as the Steam Remote Play pairing
prompt):

1. On `shrike`, open https://localhost:47990 (self-signed cert; accept
   the warning). Set the web UI username + password on first visit.
2. From the controlling device, install Moonlight, point it at
   `shrike.stanley.arpa`, and enter the PIN that the Sunshine web UI
   displays during pairing.
3. Add the games / desktop shortcut you want to stream under
   **Applications** in the web UI.

## Headless operation

Sunshine, like Steam Remote Play, needs a real display target. `shrike`
uses a hardware dummy HDMI plug for headless streaming — if the plug
is unseated, Sunshine encodes a black frame. See
[docs/gaming.md](gaming.md) for the same caveat applied to Steam
Remote Play.

## Troubleshooting

- **Service won't start after a config change**: check
  `C:\Program Files\Sunshine\config\sunshine.log` — most config errors
  are reported there with a line number, and the service exits early
  rather than running with broken config.
- **Moonlight can't see the host**: confirm the firewall rules are
  present (`Get-NetFirewallRule -DisplayName 'Sunshine (TCP)'`) and
  that the client is on a network reachable by domain or private
  profile (Sunshine relies on mDNS discovery on UDP 5353 — already
  open by default in the private profile).
