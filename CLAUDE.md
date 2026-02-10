# CLAUDE.md

## Project Overview

Orange Pi 5B + Linode VPS setup. Goal: access OPi from anywhere via reverse SSH tunnel through Linode relay.

## Architecture

```
[Terminus/Android] → SSH → [Linode :22] → localhost:19222 → [Orange Pi :2222]
                                ↑                                    |
                         reverse tunnel kept alive by ──────────────┘
                         OPi's autossh systemd service
```

## Hardware

- **Orange Pi 5B:** RK3588S, 8GB RAM, 64GB eMMC, WiFi6+BT5, Mali-G610, 6T NPU
- **Linode VPS:** Fedora 43 Server, already runs Angular SSR app — being dual-purposed as SSH relay

## Current Status

- Ubuntu 24.04 Desktop installed on OPi eMMC
- Reverse SSH tunnel plan written (see `ReverseSshTunnel.md`)
- **Part 1 (Linode config) COMPLETE**
- **Next:** Part 2 — Orange Pi config (harden sshd, generate tunnel keypair, autossh service)

## Implementation Phases

Full instructions in `ReverseSshTunnel.md`. Summary:

1. ~~**Linode config**~~ — `tunneluser` created, sshd hardened, firewalld tightened, fail2ban active (19 bans), dnf5-automatic timer running
2. **OPi config** — harden sshd (port 2222), generate tunnel keypair, autossh systemd service
3. **Terminus config** — ProxyJump setup for one-step Android→Linode→OPi connection
4. **Verification** — security checklist in Part 4 of the plan

## Placeholder Values (replace before running)

| Placeholder | Meaning |
|---|---|
| `YOUR_LINODE_IP` | Linode public IPv4 |
| `YOUR_ADMIN_USERNAME` | Linode admin user |
| `YOUR_PI_USERNAME` | OPi user |

## Linode Details (for Part 2 reference)

- **IP:** 66.175.223.124
- **Admin user:** bbastanza
- **Tunnel user:** tunneluser (uid 1001, nologin)
- **Tunnel port:** 19222 (localhost-only)
- **sshd drop-ins:** `/etc/ssh/sshd_config.d/50-redhat.conf` (X11Forwarding fixed to no)
- **dnf5-automatic** (not dnf-automatic) — Fedora 43 uses dnf5

## Decisions Made

- Ubuntu Rockchip 24.04 for OPi (dedicated image, kernel 6.1)
- Reverse SSH tunnel over Tailscale/ZeroTier (simpler, no extra software on Linode)
- autossh + systemd for tunnel persistence
- `tunneluser` with nologin shell + Match block for isolation
- Port 19222 on Linode (localhost-only, not exposed)
- OPi SSH on port 2222
- fail2ban + firewalld (Linode) / UFW (OPi) + key-only auth on both machines
- No ports exposed to internet on either side beyond SSH(22) and web(80/443) on Linode
