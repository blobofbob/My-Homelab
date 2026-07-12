# 04 — Tailscale

Tailscale is an encrypted mesh VPN built on WireGuard. Every device you add to your tailnet gets Tailscale IP and a MagicDNS hostname, and they can all reach each other directly regardless of what network they are on.

For this homelab, Tailscale does three things: provides secure SSH access to the Pi from anywhere, acts as the transport layer for all private services (Pi-hole admin, OpenClaw, n8n, Vaultwarden), and optionally lets other devices route their internet traffic through the Pi as an exit node.

**Prerequisites:** [01 — Initial Setup](01-initial-setup.md) complete.

---

## Installation

The official one-line installer handles the apt repository and package install: [6]

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Bring Tailscale up and authenticate:

```bash
sudo tailscale up
```

This prints a URL — open it in a browser, log in to your Tailscale account, and the Pi will appear in your tailnet. Once authenticated, confirm it is connected:

```bash
tailscale status
```

Note down the Tailscale IP and MagicDNS hostname, which can be found in the admin panel at `https://login.tailscale.com/admin/machines` — you will use them throughout the remaining guides:

```bash
# Tailscale IP
tailscale ip -4
```

---

## SSH over Tailscale

Tailscale SSH lets you connect to the Pi with a simple browser based authentication (logging into your account). Enable it: [7]

```bash
sudo tailscale up --ssh
```

From another device on the tailnet:

```bash
ssh your-username@your-pi's-magicdns-ipv4
```

> **Why this is better than "normal" SSH:** SSH exposes port 22 to the internet, however Tailscale's SSH only accepts connections from devices authenticated on your tailnet, so with the docker rules that will be configured later, it removes the need for a password when logging in.

After confirming SSH over Tailscale works, it is safe to restrict port 22 in UFW to tailnet-only access. This is handled in [11 — UFW](11-ufw.md) with a single `ufw allow in on tailscale0` rule that covers SSH alongside every other tailnet services.

---

## Exit node (optional)

An exit node lets other devices on your tailnet route all their internet traffic through the Pi. This means your phone or laptop can use your home internet connection while away — useful for accessing local-network resources or for privacy on untrusted networks.

Enable the Pi as an exit node: [8]

```bash
sudo tailscale up --ssh --advertise-exit-node
```

Then in the Tailscale admin panel at `https://login.tailscale.com/admin/machines`, find the Pi and approve it as an exit node. On each device that should use it, select the Pi as the exit node in the Tailscale app.

You also need to enable IP forwarding on the Pi for exit node traffic to route correctly:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## MagicDNS and HTTPS certificates

MagicDNS gives every device on the tailnet a stable hostname (e.g. `raspberrypi.tail7aae4f.ts.net`) that resolves to its Tailscale IP. Tailscale can also issue valid TLS certificates for these hostnames — this is what allows the private services (Pi-hole, OpenClaw, n8n, Vaultwarden) to be served over HTTPS without Certbot or a public domain.

In the Tailscale admin panel at `https://login.tailscale.com/admin/dns`, confirm both **MagicDNS** and **HTTPS Certificates** are enabled.

---

## Tailscale Serve

Tailscale Serve is a built-in reverse proxy that forwards HTTPS requests on the tailnet to a local port on the Pi. It is how every private service in this homelab gets its tailnet URL. [10]

The general form of the command:

```bash
tailscale serve --bg --https=PORT http://YOUR_PI'S_MDNS_IP:PORT
```

- `--bg` runs the rule in the background and persists it across reboots.
- `--https=PORT` is the port Tailscale uses to foward your service on the tailnet.


**Example — Pi-hole web admin:**

```bash
tailscale serve --bg --https=8446 http://100.101.7.56:8080
```

Pi-hole's web admin is then reachable at `https://raspberrypi.tail7aae4f.ts.net:8446/admin` from any device on the tailnet.

Verify any Serve rule with:

```bash
tailscale serve status
```

Serve rules persist natively across reboots.

The full set of Serve rules for all services is in the [Reference].

---

## Useful commands

```bash
# Check connection status
tailscale status

# Show Tailscale IP
tailscale ip -4

# Show all active Serve rules
tailscale serve status

# Ping another tailnet device by MagicDNS hostname
tailscale ping raspberrypi.tail7aae4f.ts.net

# Check Tailscale daemon logs
sudo journalctl -u tailscaled -f

# Re-authenticate (if the node expires)
sudo tailscale up
```

---

## Troubleshooting

**`tailscale status` shows the node as offline**

The Tailscale daemon may have stopped. Restart it:

```bash
sudo systemctl restart tailscaled
sudo tailscale up
```

---

**Serve rule not working after adding it**

Check the rule is listed:

```bash
tailscale serve status
```

If it is listed but the URL is not reachable, verify the destination port is actually listening:

```bash
sudo ss -tlnp | grep DESTINATION_PORT
```

If nothing is listening on that port, the upstream service is not running.

---

**Next:** [05 — Tor](05-tor.md)

**Sources:** [6] [7] [8] [9] [10]
