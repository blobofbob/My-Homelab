# Reference — Raspberry Pi Homelab
 
Quick-access tables for every guide. All port numbers, file paths, and sources live here so each guide can cross-reference this page rather than define them independently.
 
---
 
## Hardware
 
| Component | Specification |
|---|---|
| Board | Raspberry Pi 4 Model B |
| RAM | 8 GB |
| Storage | ~29 GB SD card (that comes with the starter kit) |
| OS | Debian 13 Trixie (64-bit) |
| Network | Ethernet only — Wi-Fi disabled via rfkill |
| Tailscale IP | `100.101.7.56` |
| MagicDNS hostname | `raspberrypi.tail7aae4f.ts.net` |
| .onion address | `male3e4xwgo4swc7awciiw2vrqsok26uz3ywucte5kcvrcxsluob7kyd.onion` |
 
> The Tailscale IP and MagicDNS hostname are assigned by Tailscale and specific to this machine. If you are following these guides on your own Pi, substitute your own values wherever `100.101.7.56` and `raspberrypi.tail7aae4f.ts.net` appear.
 
---
 
## Active services
 
| Service | What it does |
|---|---|
| Pi-hole | Network-wide ad blocker and DNS server — covers both the local network and the tailnet |
| Apache | Public web server, serves my website at gabriel.fortin-cara.org |
| Tailscale | Encrypted mesh VPN; provides SSH into my Pi, Tailscale Serve (reverse proxy), and MagicDNS |
| Tor relay | Contributes bandwidth to the Tor anonymity network |
| Tor hidden service | .onion mirror of my website, served by Apache via torrc |
| Docker | Container for OpenClaw, n8n, and Vaultwarden |
| UFW | Deny-by-default firewall; works alongside Docker's own forwarding chains |
| OpenClaw | AI agent framework (Google Gemma 4 31B primary, Groq backup) |
| n8n | Workflow automation engine, integrated with OpenClaw |
| Vaultwarden | Self-hosted password manager |
 
---
 
## Port map
 
### Public — reachable from the internet
 
| Port | Protocol | Service |
|---|---|---|
| 80 | TCP | Apache HTTP |
| 443 | TCP | Apache HTTPS |
| 9001 | TCP | Tor relay (ORPort) |
| 25565 | TCP | Minecraft Java *(optional — only when server is running ,also reachable on the local network)* |
| 19132 | UDP | Minecraft Bedrock *(optional)* |
| 19133 | UDP | Minecraft Bedrock *(optional)* |
 
### Local network only
 
| Port | Protocol | Service | Restriction |
|---|---|---|---|
| 53 | TCP/UDP | Pi-hole DNS | `192.168.11.0/24` only |
 
### Tailscale only — HTTPS via Tailscale Serve
 
| Tailscale Serve port | Forwards from | Service |
|---|---|---|
| 8443 | `100.101.7.56:18789` | OpenClaw web interface |
| 8444 | `100.101.7.56:5678` | n8n editor |
| 8445 | `100.101.7.56:8096` | Vaultwarden |
| 8446 | `100.101.7.56:8080` | Pi-hole web admin |
 
> Pi-hole's web admin listens on port 8080, and Tailscale Serve forwards HTTPS requests on port 8446 to it. Once UFW ([11 — UFW](guides/11-ufw.md)) is configured, port 8080 is not reachable from the local network — only the `tailscale0` interface is allowed in, so port 8446 is the only way to reach the Pi-hole admin panel. Port 22 (SSH) is tailnet-only for the same reason.
 
### Internal Docker ports
 
| Port | Service | Bound to |
|---|---|---|
| 18789 | OpenClaw | `100.101.7.56` + `127.0.0.1` |
| 5678 | n8n | `100.101.7.56` + `127.0.0.1` |
| 8096 | Vaultwarden | `100.101.7.56` + `127.0.0.1` |
 
> Each Docker service is bound to both the Tailscale IP (so Tailscale Serve can proxy to it) and `127.0.0.1` (so health checks and CLI commands can reach it from the Pi host without going through Tailscale).
 
---
 
## Key file paths
 
| Path | What |
|---|---|
| `/etc/tor/torrc` | Tor configuration — relay and hidden service settings |
| `/var/lib/tor/hidden_service/` | Tor hidden service directory (root-protected — requires `sudo` at time of writing) |
| `/etc/apache2/sites-available/` | Apache virtual host config files |
| `/var/www/html/` | Apache document root |
| `/etc/letsencrypt/` | Certbot certificates and renewal config |
| `/etc/ufw/before.rules` | UFW low-level iptables rules (`*filter` block) |
| `~/ai-stack/` | OpenClaw + n8n Docker Compose project |
| `~/ai-stack/.env` | AI stack environment variables |
| `~/ai-stack/openclaw-data/` | OpenClaw persistent data |
| `~/vaultwarden/` | Vaultwarden Docker Compose project |
| `~/vaultwarden/.env` | Vaultwarden environment variables |
| `~/vaultwarden/vaultwarden-data/` | Vaultwarden database, attachments, and encryption keys — **back this up** |
 
---
 
## Sources
 
| # | URL |
|---|-----|
| [1] | https://discourse.pi-hole.net/t/how-do-i-configure-my-devices-to-use-pi-hole-as-their-dns-server/245 |
| [2] | https://burstbytes.com.au/best-2026-pi-hole-blocklists-and-how-to-install-them/ |
| [3] | https://www.tomshardware.com/news/raspberry-pi-web-server,40174.html |
| [4] | https://www.sunfounder.com/blogs/news/raspberry-pi-apache-server-setup-step-by-step-installation-and-configuration |
| [5] | https://pimylifeup.com/raspberry-pi-ssl-lets-encrypt/ |
| [6] | https://tailscale.com/install |
| [7] | https://tailscale.com/learn/how-to-ssh-into-a-raspberry-pi#enabling-tailscale-ssh-access |
| [8] | https://tailscale.com/kb/1103/exit-nodes?tab=linux |
| [9] | https://tailscale.com/kb/1114/pi-hole |
| [10] | https://tailscale.com/kb/1242/tailscale-serve |
| [11] | https://community.torproject.org/relay/setup/guard/debian-ubuntu/ |
| [12] | https://community.torproject.org/relay/setup/post-install/ |
| [13] | https://www.minecraft.net/en-us/download/server |
| [14] | https://forums.raspberrypi.com/viewtopic.php?t=342095 |
| [15] | https://raspberrytips.com/minecraft-server-raspberry-pi/ |
| [16] | https://docs.papermc.io/paper/updating/ |
| [17] | https://docs.papermc.io/paper/next-steps/ |
| [18] | https://papermc.io/downloads/ |
| [19] | https://geysermc.org/download?project=geyser |
| [20] | https://geysermc.org/download/?project=floodgate |
| [21] | https://raspberrypi-guide.github.io/other/Improve-raspberry-pi-security |
| [22] | https://www.youtube.com/watch?v=aFXV-lt-CL4 |
| [23] | https://momentum-fw.dev/update?version=42630e91 |
| [24] | https://www.digitalocean.com/community/tutorials/apache-configuration-error-ah00558-could-not-reliably-determine-the-server-s-fully-qualified-domain-name |
| [25] | https://docs.docker.com/engine/install/debian/ |
| [26] | https://docs.openclaw.ai/install/docker |
| [27] | https://docs.openclaw.ai/install/raspberry-pi |
| [28] | https://docs.n8n.io/ |
| [29] | https://docs.openclaw.ai/providers/groq |
| [30] | https://docs.openclaw.ai/providers/google |
| [31] | https://github.com/dani-garcia/vaultwarden/wiki |
