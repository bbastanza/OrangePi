# Orange Pi 5B Setup — Progress Tracker

## Phase 1: Flash Ubuntu 24.04 Desktop
- [x] Download Ubuntu 24.04 desktop image for OPi5B
- [x] Decompress .img.xz -> .img
- [x] Wipe SD card (wipefs + sgdisk + dd zero)
- [x] Flash image to SD card (dd, 7.4GB, verified)
- [x] Flash eMMC from SD card
- [ ] First boot from eMMC — complete Ubuntu setup wizard
- [ ] Clone this repo + install Claude Code on the board

## Phase 2: System Setup
- [ ] apt update/upgrade
- [ ] Install essentials (curl, wget, git, build-essential)
- [ ] Set hostname to `openclaw-host`
- [ ] Install & enable SSH (key-only auth)
- [ ] Verify networking (Wi-Fi + Ethernet)

## Phase 3: Node.js 22
- [ ] Add NodeSource apt repo
- [ ] Install Node.js 22
- [ ] Verify `node --version` shows v22.x

## Phase 4: OpenClaw
- [ ] `npm install -g openclaw@latest`
- [ ] `openclaw onboard --install-daemon`
- [ ] Configure agent model (Anthropic API key)
- [ ] Install Chromium for browser control
- [ ] Connect WhatsApp channel

## Phase 5: Harden
- [ ] UFW: deny all inbound, allow SSH
- [ ] Disable SSH password auth (key-only)
- [ ] `loginctl enable-linger`
- [ ] Bind OpenClaw gateway to localhost only
- [ ] Unattended upgrades

## Phase 6: Remote Desktop (Headless)
- [ ] Install xrdp
- [ ] Configure virtual sessions
- [ ] SSH tunnel setup for xrdp access
- [ ] Test headless operation (no monitor)

## Phase 7: Verify
- [ ] Reboot, confirm OpenClaw daemon autostart
- [ ] Test WhatsApp messaging
- [ ] Test browser control
- [ ] Monitor resource usage (htop)
- [ ] Check thermals

## Future
- [ ] Signal channel (JVM + ARM64 libsignal)
- [ ] Tailscale for remote access outside LAN
