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
- **Linode VPS:** existing server, already runs Angular SSR app — being dual-purposed as SSH relay

## Current Status

- Ubuntu 24.04 Desktop installed on OPi eMMC
- Reverse SSH tunnel plan written (see `ReverseSshTunnel.md`)
- **Next:** execute tunnel plan — Linode config first, then OPi config

## Implementation Phases

Full instructions in `ReverseSshTunnel.md`. Summary:

1. **Linode config** — create `tunneluser`, harden sshd, fail2ban, unattended-upgrades
2. **OPi config** — harden sshd (port 2222), generate tunnel keypair, autossh systemd service
3. **Terminus config** — ProxyJump setup for one-step Android→Linode→OPi connection
4. **Verification** — security checklist in Part 4 of the plan

## Placeholder Values (replace before running)

| Placeholder | Meaning |
|---|---|
| `YOUR_LINODE_IP` | Linode public IPv4 |
| `YOUR_ADMIN_USERNAME` | Linode admin user |
| `YOUR_PI_USERNAME` | OPi user |

## Decisions Made

- Ubuntu Rockchip 24.04 for OPi (dedicated image, kernel 6.1)
- Reverse SSH tunnel over Tailscale/ZeroTier (simpler, no extra software on Linode)
- autossh + systemd for tunnel persistence
- `tunneluser` with nologin shell + Match block for isolation
- Port 19222 on Linode (localhost-only, not exposed)
- OPi SSH on port 2222
- fail2ban + UFW + key-only auth on both machines
- No ports exposed to internet on either side beyond SSH(22) and web(80/443) on Linode
