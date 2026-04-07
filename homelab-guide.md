# Raspberry Pi Homelab Setup Guide

A complete walkthrough for setting up Pi-hole, Tailscale VPN, an Apache web server, and a Tor relay on a Raspberry Pi, based on what I saw, did, and my personal experience. In case of doubt, refer to official documentation as I might have missed something. They are available in the sources.

---

## 1. Initial Setup — Update the System

Before installing anything, make sure your Pi is fully up to date:

```bash
sudo apt update && sudo apt upgrade
```

The `&&` ensures the second command only runs if the first succeeds. Follow any prompts that appear during the upgrade.

I recommend setting up SSH for a more comfortable experience.

---

## 2. Pi-hole — Network-Wide Ad Blocker

### Installation

Run the one-line installer:

```bash
sudo curl -sSL https://install.pi-hole.net | bash
```

The installer will walk you through a series of prompts:

- **Network interface** — select your active interface (usually `eth0` for wired, `wlan0` for Wi-Fi — I highly recommend using ethernet rather than Wi-Fi).
- **Upstream DNS provider** — this is the DNS server Pi-hole itself uses to resolve queries (e.g. Cloudflare `1.1.1.1`, Google `8.8.8.8`, or Quad9 `9.9.9.9`). You can change this later.
- **Block lists** — the default StevenBlack list is really good; you can add more afterwards.
- **Web admin interface** — select On.
- **Web server** — select On.
- **Log queries** — recommended, useful for debugging.
- **Privacy level** — level 0 logs everything; increase if you want less detail stored.

At the end of installation, Pi-hole will display a generated admin password — **save this**.

### Accessing the Web Interface

Once installed, the admin dashboard is available at:

- `http://192.168.1.223/admin` *(replace with your Pi's actual local IP)*
- `http://pi.hole/admin`

### Pointing Devices to Pi-hole

The best way of doing so is to set your router's DNS server to your Pi's local IP address and to make your local IP address static — or else your Wi-Fi may magically stop working from time to time. This routes every device on the network through Pi-hole automatically without configuring each device individually. The exact setting is usually under DHCP / DNS or WAN settings in your router's admin panel. For per-device setup, refer to [1].

### Adding Blocklists

After logging into the admin panel, go to **Adlists** under the Group Management menu. Paste in blocklist URLs and then run:

```bash
pihole -g
```

This pulls down and processes the lists. I have included a set of blocklists at [2].

### Changing the Web Interface Port

By default, Pi-hole listens on ports 80 and 443 — the same ports that my Apache web server uses — which causes a conflict. There are two methods of changing this.

**Option 1 — CLI (recommended):**

```bash
sudo pihole-FTL --config webserver.port "8080o,8443os,[::]:8080o,[::]:8443os"
```

**Option 2 — Edit the config file directly:**

Open `/etc/pihole/pihole.toml`, find the `[webserver]` section, and update it:

```toml
[webserver]
  port = "8080o,8443os,[::]:8080o,[::]:8443os"
```

**Port syntax notes:**

- The `o` suffix means "optional" — Pi-hole won't error if that port can't be bound.
- The `s` suffix marks a port as TLS/SSL.
- Setting the value to an empty string disables the web server and API entirely.

After making changes, restart Pi-hole's FTL service:

```bash
sudo systemctl restart pihole-FTL
```

The admin panel will now be at `http://<your-pi-hole-ip>:8080/admin`.

---

## 3. Apache Web Server

### Installation

```bash
sudo apt install apache2 -y
```

Apache should start automatically. You can verify its status with:

```bash
sudo systemctl status apache2
```

And control it with:

```bash
sudo systemctl start apache2
sudo systemctl stop apache2
sudo systemctl restart apache2
```

### Serving a Web Page

Your site's files live in `/var/www/html/`. The default page is `index.html` — replace or edit it to serve your own content:

```bash
sudo nano /var/www/html/index.html
```

To verify everything is working, open a browser and navigate to your Pi's local IP address — you should see your page.

### Enabling Required Modules

```bash
sudo a2enmod rewrite
sudo a2enmod ssl
sudo systemctl restart apache2
```

`a2enmod rewrite` allows you to change ugly URLs (`/index.php?id=5`) into clean ones (`/articles/my-post`) and `a2enmod ssl` allows the server to handle encrypted connections.
if you would like to have a domain name, you will have to purchase it

### Virtual Hosts *(Optional — I have not done it, but it sounds cool)*

If you want to serve multiple sites, create a config file for each in `/etc/apache2/sites-available/`:

```bash
sudo nano /etc/apache2/sites-available/mysite.conf
```

```apache
<VirtualHost *:80>
    ServerName mysite.local
    DocumentRoot /var/www/mysite
    ErrorLog ${APACHE_LOG_DIR}/mysite_error.log
    CustomLog ${APACHE_LOG_DIR}/mysite_access.log combined
</VirtualHost>
```

Enable it and reload:

```bash
sudo a2ensite mysite.conf
sudo systemctl reload apache2
```

Full setup walkthrough at [3] and [4].

---

## 4. TLS Certificate with Certbot

To serve your site over HTTPS, use Certbot to obtain a free certificate from Let's Encrypt.

### Installation

```bash
sudo apt install certbot python3-certbot-apache -y
```

### Obtaining a Certificate

Your Pi needs to be reachable from the internet on port 80 for this step — make sure your router is forwarding port 80 to your Pi's local IP.

```bash
sudo certbot --apache
```

Certbot will ask for an email address (for renewal reminders) and whether to redirect HTTP to HTTPS — choose redirect.

### Auto-renewal

Certbot installs a systemd timer to handle renewals automatically. Test it with:

```bash
sudo certbot renew --dry-run
```

Full guide at [5].

---

## 5. Tailscale VPN

Tailscale creates an encrypted mesh network between all your devices, letting you access your Pi securely from anywhere without opening ports to the internet.

### Installation

Run the official installer [6]:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Then bring Tailscale up and authenticate:

```bash
sudo tailscale up
```

This prints a URL — open it in a browser to log in and authorise the device. Your Pi will then appear in the Tailscale admin panel at `login.tailscale.com/admin/machines`.

### SSH via Tailscale

Enable Tailscale's built-in SSH so you can reach your Pi from anywhere on your tailnet without managing SSH keys:

```bash
sudo tailscale up --ssh
```

From any other Tailscale-connected device you can then:

```bash
ssh pi@<tailscale-ip-of-pi>
```

The Pi's Tailscale IP is shown in the admin panel [7].

### Set the Pi as an Exit Node

This routes all traffic from your other devices through your Pi when away from home — useful for using Pi-hole and avoiding untrusted Wi-Fi.

> **IMPORTANT:** You must enable IP forwarding to advertise a Linux device as an exit node.

First, enable IP forwarding on the Pi:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p /etc/sysctl.conf
```

Then advertise the Pi as an exit node:

```bash
sudo tailscale up --advertise-exit-node
```

Finally, go to the Tailscale admin panel, find the Pi under **Machines**, click the three-dot menu, and enable **Use as exit node**. Full guide at [8].

### Integrate Pi-hole with Tailscale

In the Tailscale admin panel, navigate to **DNS** and set your Pi's Tailscale IP as a global nameserver. This ensures all tailnet devices use Pi-hole for DNS. Full integration guide at [9].

### Tailscale Serve

Tailscale Serve lets you expose a local service to other devices on your tailnet without opening any ports to the internet:

```bash
tailscale serve https / http://localhost:80
```

This proxies HTTPS requests on your tailnet directly to Apache running locally. Full documentation at [10].

---

## 6. Tor — Onion Service & Relay

### Install Tor

```bash
sudo apt install tor -y
```

### Onion Service

To expose your web server as a `.onion` site, add the following to `/etc/tor/torrc`:

```
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServicePort 80 127.0.0.1:80
```

Restart Tor:

```bash
sudo systemctl restart tor
```

Tor generates your `.onion` address automatically. Retrieve it with:

```bash
sudo cat /var/lib/tor/hidden_service/hostname
```
You can also install OnionShare for a simpler way to share files over Tor:

```bash
sudo apt install onionshare -y
```

### Tor Relay

Running a relay contributes bandwidth to the Tor network. First, add the official Tor Project repository to get the latest version [11]:

```bash
sudo apt install apt-transport-https -y

# Add the signing key
wget -qO- https://deb.torproject.org/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/tor-archive-keyring.gpg >/dev/null

# Add the repository
echo "deb [signed-by=/usr/share/keyrings/tor-archive-keyring.gpg] \
  https://deb.torproject.org/torproject.org bullseye main" \
  | sudo tee /etc/apt/sources.list.d/tor.list

sudo apt update && sudo apt install tor deb.torproject.org-keyring -y
```

Edit `/etc/tor/torrc` and add your relay configuration:

```
Nickname      MyRelay          # Choose a unique name
ContactInfo   you@example.com
ORPort        9001             # 9001 keeps port 443 free for Apache HTTPS
ExitRelay     0                # 0 = middle/guard relay only (recommended)
RelayBandwidthRate  1 MBytes
RelayBandwidthBurst 2 MBytes
```

> **Port note:** Using `ORPort 9001` keeps port 443 available for Apache's HTTPS traffic.

Restart Tor and verify it's running:

```bash
sudo systemctl restart tor
sudo systemctl status tor
```

After a few hours, check that your relay has registered with the network by searching your nickname at https://metrics.torproject.org/rs.html. Post-install tips available at [12].

---

## Sources

| # | URL |
|---|-----|
| [1] | https://discourse.pi-hole.net/t/how-do-i-configure-my-devices-to-use-pi-hole-as-their-dns-server/245 |
| [2] | https://burstbytes.com.au/best-2026-pi-hole-blocklists-and-how-to-install-them/ |
| [3] | https://www.tomshardware.com/news/raspberry-pi-web-server,40174.html |
| [4] | https://www.sunfounder.com/blogs/news/raspberry-pi-apache-server-setup-step-by-step-installation-and-configuration |
| [5] | https://pimylifeup.com/raspberry-pi-ssl-lets-encrypt/ |
| [6] | https://tailscale.com/download/linux/rpi-bullseye |
| [7] | https://tailscale.com/learn/how-to-ssh-into-a-raspberry-pi#enabling-tailscale-ssh-access |
| [8] | https://tailscale.com/kb/1103/exit-nodes?tab=linux |
| [9] | https://tailscale.com/kb/1114/pi-hole |
| [10] | https://tailscale.com/docs/features/tailscale-serve |
| [11] | https://community.torproject.org/relay/setup/guard/debian-ubuntu/ |
| [12] | https://community.torproject.org/relay/setup/post-install/ |
