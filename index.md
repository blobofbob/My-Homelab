# Raspberry Pi Homelab

This is a complete documentation of my self-hosted homelab running on a Raspberry Pi 4. It covers every service I run, every decision I made, and why I made it — written both as a personal record and as a reproducible guide for anyone who wants to build something similar.

Everything here is based on what I actually did on my own Pi. Where something is specific to my setup, I say so. I try to make the steps as general as possible, so they should work on any Raspberry Pi 4 running Debian 13 Trixie.

---

## The hardware

A Raspberry Pi 4 with 8 GB of RAM on a ~29 GB SD card, running Debian 13 Trixie (64-bit). Wi-Fi is disabled — the Pi runs on ethernet only. It sits at home and is reachable from anywhere via Tailscale.

---

## What's running

| Service | What it does |
|---|---|
| Pi-hole | Blocks ads across my entire network and tailnet |
| Apache + Certbot | Serves my CV website publicly over HTTPS |
| Tailscale | Encrypted VPN that ties everything together |
| Tor | Runs a relay and mirrors my CV site as a .onion address |
| Docker | Container for the three services below |
| UFW | Deny-by-default firewall |
| OpenClaw | AI agent I use for chat and automation |
| n8n | Workflow automation, connected to OpenClaw |
| Vaultwarden | Self-hosted password manager |
| Minecraft | Optional — a server I run occasionally |
| Flipper Zero | Not on the Pi, but documented here alongside it |

---

## How to use these guides

The guides are written to be followed in order, since later sections depend on earlier ones — Docker must be installed before OpenClaw, UFW must be configured after all Docker stacks are running, and so on. If you are setting up from scratch, start at [01 — Initial Setup](guides/01-initial-setup.md) and work through sequentially.

If you are looking for a specific service and already have the prerequisites in place, each guide is self-contained enough to follow independently. Anything it depends on is noted at the top.

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
9. [OpenClaw + n8n](guides/9-openclaw-n8n.md) — AI stack
10. [Vaultwarden](guides/10-vaultwarden.md) — Self-hosted password manager
11. [UFW](guides/11-ufw.md) — Firewall configuration alongside Docker

---

[Reference](reference.md) — Port map, file paths, and all sources
