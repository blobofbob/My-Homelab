# 05 — Tor
 
Tor is an anonymity network that routes traffic through a series of encrypted relays. The tor network helps anonymise whistleblowers and circumvent censorship around the world free of charge thanks to volunteers that run relays like the one i set up (and that you will in a bit). This guide sets up two things: a **middle relay** that contributes bandwidth to the Tor network, and a **hidden service** that makes these guides accessible with .onion address.
 
**Prerequisites:** [03 — Apache + Certbot](03-apache-certbot.md) complete. The hidden service points at Apache on port 80 — Apache must be running before Tor can serve it. You only need the prerequisite if you want to serve a webpage over tor.
 
---
 
## Installing Tor
 
The Tor package in the default Debian repositories is outdated. Install from the Tor Project's own apt repository instead. [11]
 
**1 — Add the Tor Project GPG key:**
 
```bash
sudo curl -fsSL https://deb.torproject.org/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc \
  | sudo gpg --dearmor -o /usr/share/keyrings/tor-archive-keyring.gpg
```
 
**2 — Add the repository:**
 
```bash
CODENAME=$(. /etc/os-release && echo "$VERSION_CODENAME")
echo "deb [signed-by=/usr/share/keyrings/tor-archive-keyring.gpg] \
https://deb.torproject.org/torproject.org $CODENAME main" \
  | sudo tee /etc/apt/sources.list.d/tor.list
```
 
Using `$VERSION_CODENAME` from the OS release file means this command works correctly on any Debian release without hardcoding the codename.
 
**3 — Install:**
 
```bash
sudo apt update && sudo apt install tor deb.torproject.org-keyring -y
```
 
Verify Tor is running:
 
```bash
sudo systemctl status tor
```
 
---
 
## Configuring the relay
 
Open the main Tor config file:
 
```bash
sudo nano /etc/tor/torrc
```
 
Add the following block. Most of the file is commented-out examples — add these lines at the bottom or uncomment and edit the relevant sections: [11]
 
```
# Relay settings
ORPort 9001
Nickname YourRelayNickname
ContactInfo your-email@example.com
ExitPolicy reject *:*
```
 
- `ORPort 9001` is the port other relays use to connect to yours. Port 9001 must be forwarded on your router. The default is 443, however it you have Apache, you will need it to serve your webpage over https
- `Nickname` is a public label for your relay — it appears on relay tracking sites.
- `ContactInfo` is how the Tor Project contacts you if your relay misbehaves. It is public.
- `ExitPolicy reject *:*` makes this a middle relay — it passes traffic along but never makes the final connection to a destination. Running an exit relay is significantly more complex and carries more legal exposure; a middle relay contributes meaningfully to the network without that overhead.
Apply the config:
 
```bash
sudo systemctl restart tor
```
 
**Verify the relay is working:** [12]
 
```bash
sudo journalctl -u tor -f
```
 
Look for a line like:
 
```
Jun 18 00:00:39 raspberrypi systemd[1]: Starting tor.service - Anonymizing overlay network for TCP (multi-instance-master)...
Jun 18 00:00:40 raspberrypi systemd[1]: Finished tor.service - Anonymizing overlay network for TCP (multi-instance-master).
```
 
This confirms that the relay is up and accepting connections. It can take a few minutes after restart to appear.
 
Your relay's fingerprint and stats will appear on `https://metrics.torproject.org/rs.html` within a few hours. New relays take several days to ramp up to full traffic as the network builds trust in them.
 
---
 
## Setting up the hidden service
 
A hidden service makes a local port accessible as a .onion address, routed entirely through Tor. The hidden service here points at Apache on port 80, creating a .onion mirror of my website.
 
Add the following to `/etc/tor/torrc`, below the relay config:
 
```bash
sudo nano /etc/tor/torrc
```
 
```
# Hidden service
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServicePort 80 127.0.0.1:80
```
 
- `HiddenServiceDir` is where Tor stores the hidden service keys and hostname. Tor creates this directory automatically on first start.
- `HiddenServicePort 80 127.0.0.1:80` maps port 80 on the .onion address to Apache on localhost port 80.
> **The hidden service directory is root-protected.** You will need `sudo` to read anything inside it. Also, if `/var/lib/tor/hidden_service/` already exists from a previous setup, Tor will use it and preserve the existing .onion address. If the name is taken by something else, Tor will refuse to start — check with `sudo ls /var/lib/tor/` and rename or remove the conflicting directory first.
 
Restart Tor to generate the hidden service keys:
 
```bash
sudo systemctl restart tor
```
 
Get your .onion address:
 
```bash
sudo cat /var/lib/tor/hidden_service/hostname
```
 
This is the permanent address for your hidden service as long as you keep the keys in `hidden_service/`. Losing the keys means losing the address — back up that directory.
 
To verify it works, visit your .onion address in the Tor Browser.
 
---
 
## OnionShare 
 
OnionShare lets you share individual files or host a simple site over Tor without configuring a hidden service manually. It is not needed for the setup above — the hidden service is already handled by torrc — but it is useful if you want to share files ad hoc over Tor.
 
Install via apt:
 
```bash
sudo apt install onionshare-cli -y
```
 
> **Why not OnionShare?.** Onionshare (without -cli) pulls in approximately 800 MB of GNOME GUI dependencies that serve no purpose on a headless server.
 
---
 
## Useful commands
 
```bash
# Check Tor status
sudo systemctl status tor
 
# Tail Tor logs (useful when verifying the relay)
sudo journalctl -u tor -f
 
# Read your .onion address
sudo cat /var/lib/tor/hidden_service/hostname
 
# Read your relay fingerprint
sudo cat /var/lib/tor/fingerprint
 
# Reload config without full restart
sudo systemctl reload tor
```
 
---
 
## Troubleshooting
 
**`Self-testing indicates your ORPort is reachable` never appears**
 
Port 9001 is not reachable from the internet. Confirm the port forwarding rule on your router is pointing to the Pi's local IP and that the protocol is TCP. If UFW is already active, confirm port 9001 is allowed:
 
```bash
sudo ufw status
```
 
---
 
**Tor fails to start after adding the hidden service config**
 
Check the logs for the specific error:
 
```bash
sudo journalctl -u tor -n 50
```
 
Common causes: the `hidden_service` path does not exist and Tor cannot create it (permissions issue), or a conflicting directory already exists at that path. Run `sudo ls /var/lib/tor/` to see what is there.
 
---
 
**The .onion address changed after a reinstall**
 
The .onion address is derived from the private key in `hidden_service`. If that directory was deleted or replaced, Tor generated a new key and a new address. The old address is unrecoverable without the original key. Back up `hidden_service` to avoid this.
 
---
 
**Next:** [06 — Minecraft](06-minecraft.md)
 
**Sources:** [11] [12]