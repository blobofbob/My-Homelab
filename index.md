# Raspberry Pi Homelab
 
This is a complete documentation of my self-hosted homelab running on a Raspberry Pi 4. It covers every service I run and is written both as a personal record and as a reproducible guide for anyone who wants to build something similar.
 
Everything here is based on what I actually did on my own Pi. I try to make the steps as general as possible, so they should work on any Raspberry Pi 4 running Debian 13 Trixie.
 
---
 
## The hardware
 
A Raspberry Pi 4 with 8 GB of RAM on a ~29 GB SD card, running Debian 13 Trixie (64-bit). Wi-Fi is disabled — the Pi runs on ethernet only. It sits at home and is reachable from anywhere via Tailscale.
 
---
 
## What's running
 
| Service | What it does |
|---|---|
| Pi-hole | Blocks ads across my entire network and tailnet |
| Apache + Certbot | Serves these guides publicly over HTTPS |
| Tailscale | Encrypted VPN that ties everything together |
| Tor | Runs a relay and mirrors these guides as a .onion address |
| Docker | Container for the three services below |
| UFW | Deny-by-default firewall |
| OpenClaw | AI agent I use for chat and automation |
| n8n | Workflow automation, connected to OpenClaw |
| Vaultwarden | Self-hosted password manager |
| Minecraft | Optional — a server I run occasionally |
| Flipper Zero | Not on the Pi, but documented here alongside it |
 
---
 
## How to use these guides
 
The guide numbers reflect **dependency order, not the order I actually built things in** — each guide states what it needs already in place at the top, and later guides depend on earlier ones (Docker must be installed before OpenClaw, UFW must be configured after all Docker stacks are running, and so on). If you are setting up from scratch, following the numbers in order satisfies every dependency automatically. Start at [01 — Initial Setup](guides/01-initial-setup.md) and work through sequentially.
 
If you are looking for a specific service and already have the prerequisites in place, each guide is self-contained enough to follow independently. Anything it depends on is noted at the top.

**One hard exception: [11 — UFW](guides/11-ufw.md) must always be done last**, after Docker and both container stacks ([09](guides/09-openclaw-n8n.md), [10](guides/10-vaultwarden.md)) are already running — not just "recommended last" like the general ordering above, but a firm requirement. Setting it up earlier risks the same silent container-networking failure documented in guides 08 and 11.
 
The [Reference](reference.md) page has the complete port table, file paths, and all sources in one place. Every guide links back to it rather than repeating that information inline.
 
---
 
## Guides
 
1. [Initial Setup](guides/01-initial-setup.md) — System update and SSH baseline
2. [Pi-hole](guides/02-pi-hole.md) — Ad blocking for the network and tailnet
3. [Apache + Certbot](guides/03-apache-certbot.md) — Public web server with TLS
4. [Tailscale](guides/04-tailscale.md) — VPN, SSH, exit node, and Serve
5. [Tor](guides/05-tor.md) — Relay and .onion hidden service
6. [Minecraft](guides/06-minecraft.md) — Java server, PaperMC, Bedrock compatibility
7. [Flipper Zero](guides/07-flipper-zero.md) — Firmware and setup
8. [Docker](guides/08-docker.md) — Installation and maintenance
9. [OpenClaw + n8n](guides/09-openclaw-n8n.md) — AI stack
10. [Vaultwarden](guides/10-vaultwarden.md) — Self-hosted password manager
11. [UFW](guides/11-ufw.md) — Firewall configuration. **Must be done last**, only after guides 9 and 10 are running
---
 
[Reference](reference.md) — Port map, file paths, and all sources