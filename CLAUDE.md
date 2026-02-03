# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Setup and configuration project for an **Orange Pi 5B** single-board computer. Goal: run **Ubuntu 24.04 Desktop** as host for an **OpenClaw** instance (personal AI assistant platform).

## Hardware

- **Board:** Orange Pi 5B (RK3588S, 8-core ARM64)
- **RAM:** 8GB LPDDR4
- **Storage:** 64GB eMMC (onboard)
- **Network:** Gigabit Ethernet + Wi-Fi 6 + BT 5.0
- **GPU:** Mali-G610 MP4, **NPU:** 6 TOPS

## Target Stack

- **OS:** Ubuntu 24.04 LTS Desktop (Ubuntu Rockchip, kernel 6.1) — image: `ubuntu-24.04-preinstalled-desktop-arm64-orangepi-5b.img.xz`
- **Runtime:** Node.js 22 (via NodeSource apt repo)
- **App:** OpenClaw (`npm install -g openclaw@latest`) — https://github.com/openclaw/openclaw
- **Browser:** Chromium (for OpenClaw's CDP-based browser control)
- **Daemon:** systemd user service (installed via `openclaw onboard --install-daemon`)
- **Channel:** WhatsApp (Baileys, pure Node.js — no ARM64 issues)
- **LLM:** Anthropic API (Claude) primary, Gemini API available as fallback
- **Remote access:** xrdp (virtual sessions) via SSH tunnel, no ports exposed to internet
- **Security:** UFW deny-all + SSH key-only + localhost-bound gateway + optional Tailscale

## Current Status

- **Ubuntu 24.04 Desktop installed on eMMC** — image flashed to SD, then written to onboard eMMC
- **About to first-boot from eMMC** on the Orange Pi 5B
- Next: complete Ubuntu setup wizard, then proceed to Phase 2 (system setup)
- See TODO.md for full checklist

## If Running on the Orange Pi 5B

This section is for Claude Code sessions running directly on the board after Ubuntu is installed.

### Immediate Next Steps (first boot from eMMC)

1. Complete Ubuntu initial setup wizard (user, language, timezone) if not already done
2. Set up git + clone this repo:
   ```bash
   sudo apt install -y git
   git clone https://github.com/bbastanza/OrangePi.git ~/Projects/OrangePi
   ```
3. Set up push access (no password):
   ```bash
   # Option A: gh CLI (easiest)
   sudo apt install -y gh
   gh auth login
   # Option B: SSH key
   ssh-keygen -t ed25519
   # Add ~/.ssh/id_ed25519.pub to GitHub -> Settings -> SSH Keys
   git remote set-url origin git@github.com:bbastanza/OrangePi.git
   ```
4. Install Claude Code:
   ```bash
   curl -fsSL https://claude.ai/install.sh | bash
   ```
5. Proceed to Phase 2 (system setup) below

### Phase 2: System Setup
```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl wget git build-essential openssh-server
sudo hostnamectl set-hostname openclaw-host
sudo systemctl enable ssh
# Verify Wi-Fi: nmcli dev wifi list / connect via GUI
```

### Phase 3: Node.js 22
```bash
sudo apt install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_22.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list
sudo apt update && sudo apt install -y nodejs
node --version  # should be v22.x
```

### Phase 4: OpenClaw
```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
# Edit ~/.openclaw/openclaw.json to set model + API key
sudo apt install -y chromium-browser
```

### Phase 5: Harden
```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw enable
# Disable password auth:
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
sudo loginctl enable-linger $USER
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### Phase 6: Remote Desktop (Headless)
```bash
sudo apt install -y xrdp
sudo systemctl enable xrdp
# Access via SSH tunnel from remote machine:
# ssh -L 3389:localhost:3389 user@openclaw-host
# Then RDP to localhost:3389
```

### Thermal Monitoring
```bash
cat /sys/class/thermal/thermal_zone*/temp
```

## Decisions Made

- Ubuntu Rockchip 24.04 over Armbian (dedicated OPi5B image, newer kernel 6.1)
- NodeSource over nvm (system-wide install, simpler for systemd service)
- **WhatsApp over Signal** — Baileys is pure Node.js (zero ARM64 issues). Signal deferred to future phase.
- **xrdp over Gnome Remote Desktop** — Gnome RD mirrors physical display, breaks headless. xrdp creates virtual sessions.
- **No ports exposed to internet** — gateway localhost-only, xrdp via SSH tunnel only
- **Cooling:** board already in case with proper cooling
- **API keys:** Anthropic (primary) + Gemini (fallback) both available

## Signal ARM64 Notes (for future reference)

signal-cli depends on libsignal-client (Rust native lib), ships x86_64 only. Options:
- Community ARM64 binaries: https://github.com/exquo/signal-libs-build/ or https://media.projektzentrisch.de/temp/signal-cli/
- Build libsignal from source (Rust + Cargo on the board)
- Must use JVM-based signal-cli (not native) — needs Java runtime (~200MB extra)
- RK3588S may lack SVE2/SVEBITPERM CPU features required by signal-cli-native

## Key Resources

- Ubuntu Rockchip downloads: https://joshua-riek.github.io/ubuntu-rockchip-download/boards/orangepi-5b.html
- Ubuntu Rockchip wiki: https://github.com/Joshua-Riek/ubuntu-rockchip/wiki/Orange-Pi-5B
- OpenClaw repo: https://github.com/openclaw/openclaw
- OpenClaw Signal docs: https://docs.openclaw.ai/channels/signal
- OrangePi install guide: https://orangepi.net/guide-to-install-operation-system-on-orange-pi-5b.html
- signal-cli ARM64 binaries: https://github.com/AsamK/signal-cli/wiki/Provide-native-lib-for-libsignal
- xrdp headless on Orange Pi: https://forum.armbian.com/topic/27407-complete-headless-armbian-and-orange-pi-installation-tutorial-connecting-via-windows-remote-desktop-xrdp/
