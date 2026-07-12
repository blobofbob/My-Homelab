# 02 — Pi-hole

Pi-hole is a network-wide ad blocker that acts as a DNS sinkhole — DNS queries from devices on your network pass through it, and any domain on a blocklist gets dropped. No request, no ads. It runs as a DNS server on the Pi, and once your router is pointed at it, every device on the network gets ad blocking without any per-device configuration.

**Prerequisites:** [01 — Initial Setup](01-initial-setup.md) complete.

---

## Installation

Run the one-line installer:

```bash
sudo curl -sSL https://install.pi-hole.net | bash
```

The installer walks through a series of prompts:

- **Network interface** — select your active interface. For a wired connection, this is `eth0`.
- **Upstream DNS provider** — the server Pi-hole itself uses to resolve queries it doesn't block. Cloudflare (`1.1.1.1`), Google (`8.8.8.8`), and Quad9 (`9.9.9.9`) are all solid choices. I use Cloudflare and Google as a backup. You can change this later.
- **Block lists** — the default StevenBlack list is a good starting point. You can add more after installation.
- **Web admin interface** — select On.
- **Web server** — select On.
- **Log queries** — recommended; useful for debugging and seeing what's being blocked.
- **Privacy level** — level 0 logs everything. Increase it if you want less stored.

At the end, the installer displays a generated admin password — save this somewhere.

---

## Changing the web interface port 

By default Pi-hole listens on ports 80 and 443, which conflicts with Apache. The port needs to be changed before setting up Apache in the next guide.

**Option 1 — CLI (recommended):**

```bash
sudo pihole-FTL --config webserver.port "8080o,8443os,[::]:8080o,[::]:8443os"
```

**Option 2 — Edit the config file directly:**

Open `/etc/pihole/pihole.toml`, find the `[webserver]` section, and set:

```toml
[webserver]
  port = "8080o,8443os,[::]:8080o,[::]:8443os"
```

**Port syntax:**

- The `o` suffix marks a port as optional — Pi-hole won't error if it can't bind it.
- The `s` suffix marks a port as TLS/HTTPS.
- Setting the value to an empty string disables the web server entirely.

After making the change, restart Pi-hole's FTL service:

```bash
sudo systemctl restart pihole-FTL
```

The admin panel is now at `http://YOUR_PI_LOCAL_IP:8080/admin`.


---

## Pointing devices at Pi-hole

The most effective approach is to set Pi-hole as the DNS server in your **router's DHCP settings**. This routes every device on the network through Pi-hole automatically, without touching individual devices.

The exact location of this setting varies by router, but it is usually under **DHCP**, **LAN settings**, or **DNS** in the admin panel. Set the primary DNS server to your Pi's local IP address.

Also set a **static local IP** for the Pi in your router's DHCP reservation table. If the Pi's local IP changes, every device on the network loses DNS. [1]

For per-device configuration instead of router-wide, refer to [1].

---

## Adding blocklists

After logging in to the admin panel, go to **Group Management → Adlists**. Paste in blocklist URLs and then run:

```bash
pihole -g
```

This pulls down and processes the lists. A curated set of blocklists is available at [2].

---

## Tailscale integration — ad blocking on the tailnet

With the setup above, Pi-hole only blocks ads for devices on your local network. To extend it to devices on your tailnet (phones, laptops, etc. connecting remotely via Tailscale), two things are needed.

**Step 1 — Tell Pi-hole to accept queries from all interfaces.**

By default, Pi-hole only responds to DNS queries that arrive on its configured interface. Tailscale queries arrive on a different interface (`tailscale0`) and are refused. To fix this:

In the Pi-hole web admin, go to **Settings → DNS**. In the upper right corner, toggle **Basic** to **Expert** to reveal advanced settings. Under **Interface settings**, check **Permit all origins**.

> **Why this is safe:** "Permit all origins" sounds broad, but Pi-hole still only responds on the network interfaces the Pi actually has. The Pi is not a public DNS server — it's only reachable from your local network and your tailnet. Enabling this setting simply stops Pi-hole from refusing queries that arrive on the Tailscale interface.

**Step 2 — Set Pi-hole as the global nameserver in Tailscale.**

In the Tailscale admin panel at `https://login.tailscale.com/admin/dns`, add your Pi's Tailscale IP as a **Global nameserver**. This tells every device on your tailnet to send DNS queries to Pi-hole.

Full integration guide: [9].

Once both steps are done, Pi-hole will block ads for every device on your tailnet, whether they are at home or connecting remotely.

---

## Accessing the web admin remotely

Once [Tailscale](04-tailscale.md) and [UFW](11-ufw.md) are set up, the Pi-hole admin panel is accessible securely from anywhere on the tailnet via Tailscale Serve.

The Tailscale Serve rule that enables this:

```bash
sudo tailscale serve --bg --https=8446 http://100.101.7.56:8080
```

This forwards HTTPS requests on port 8446 of the tailnet to Pi-hole's web admin on port 8080. The panel is then reachable at:

```
https://YOUR_MAGICDNS:8446/admin
```

The full Tailscale Serve setup is covered in [04 — Tailscale](04-tailscale.md). The port assignments for all services are in the [Reference](../reference.md#port-map).

---

## Useful commands

```bash
# Update gravity (pull latest blocklists)
pihole -g

# Check Pi-hole service status
sudo systemctl status pihole-FTL

# Restart Pi-hole
sudo systemctl restart pihole-FTL

# Tail Pi-hole logs
sudo journalctl -u pihole-FTL -f

# Check current web server port
sudo pihole-FTL --config webserver.port
```

---

## Troubleshooting

**Admin panel not loading after port change**

Confirm the port change took effect:

```bash
sudo pihole-FTL --config webserver.port
```

If it shows the old value, the config file change may not have been saved correctly. Use the CLI option instead and restart FTL.

---

**DNS stops working after Docker restarts**

Docker rebuilds its iptables rules on restart and can overwrite Pi-hole's. If DNS resolution fails after a Docker operation:

```bash
sudo systemctl restart pihole-FTL
```

This is also why UFW must be configured carefully alongside Docker — see [11 — UFW](11-ufw.md).

---

**Next:** [03 — Apache + Certbot](03-apache-certbot.md)

**Sources:** [1] [2] [9]
