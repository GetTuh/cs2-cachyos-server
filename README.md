# cs2-cachyos-server

Ansible playbook for deploying an aggressively tuned CS2 dedicated server on a fresh CachyOS minimal install. Bare metal only. Targets a GMKtec G3 (Intel N100, 8GB DDR4, 512GB NVMe) but works on any x86-64-v3 CachyOS host.

Tuning applied:
- `linux-cachyos-server` kernel (BORE scheduler, server optimized)
- CachyOS x86-64-v3 native packages
- CPU mitigations disabled, `nohz_full`, `rcu_nocbs` on all cores
- CPU governor forced to `performance` via udev
- BBR congestion control + large network buffers
- zram swap (zstd, 50% RAM), `vm.swappiness=10`
- Unnecessary services disabled (cups, bluetooth, ModemManager)

## Prerequisites

- Fresh CachyOS minimal install (no DE)
- Ansible ≥ 2.14 installed on the server itself
- `community.general` and `ansible.posix` collections

## Quick start

Clone and run directly on the server:

```bash
git clone <repo> cs2-cachyos-server
cd cs2-cachyos-server

ansible-galaxy collection install -r requirements.yml

cp vars.yml.example vars.yml
$EDITOR vars.yml

ansible-playbook -i inventory.ini  -K playbook.yml
```

The playbook is fully idempotent — re-running is safe.

## Vars reference

| Variable | Default | Description |
|---|---|---|
| `cs2_hostname` | `CachyOS CS2 LAN Server` | Server name shown in browser |
| `cs2_maxplayers` | `16` | Max player slots |
| `cs2_map` | `de_dust2` | Starting map |
| `cs2_password` | `` (empty) | Join password, leave empty for open |
| `cs2_gamemode` | `competitive` | `competitive` or `casual` |

## Benchmarking with sv_stressbots

Once the server is running:

```
rcon sv_cheats 1
rcon exec benchmark
```

`benchmark.cfg` loads 16 bots at full `bot_quota`. Monitor with:

```
rcon stats
```

Watch CPU, frametime, and `svms` (server milliseconds per frame). On an N100 expect ~60–90 fps server framerate with 16 bots.

## Manage the service

```bash
systemctl status cs2
journalctl -u cs2 -f
systemctl restart cs2
```

## Updating CS2

The playbook deploys a `cs2update` script to `/usr/local/bin`. Run it as root:

```bash
cs2update
```

```bash
cs2update   # update only (faster)
cs2validate # update + validate file integrity
```

Both stop the service, run steamcmd, then start the server back up.

## Notes

- No GSLT token required — `sv_lan 1` keeps the server LAN-only.
- The playbook removes the stock `linux` kernel and installs `linux-cachyos-server`. Ensure you have a working bootloader (systemd-boot) before running, or you will need a recovery boot.
- `mitigations=off` is intentional for a private LAN server. Do not use on internet-facing machines.
