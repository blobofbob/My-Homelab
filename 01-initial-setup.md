# 01 — Initial Setup

Before installing anything, get the system fully up to date.

```bash
sudo apt update && sudo apt upgrade -y
```

The `&&` ensures the upgrade only runs if the update succeeds. Follow any prompts that appear — some packages may ask how to handle config file changes; keeping the current version is usually the right choice.

---

## SSH

I strongly recommend setting up SSH before going any further. Working over SSH from a proper terminal is significantly more comfortable than typing directly at the Pi, and every subsequent guide assumes you are connected this way.

The standard Debian installer enables SSH by default if you select it during setup. If it is not running:

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

Connect from another machine:

```bash
ssh your-username@your-pi-local-ip
```

You can find the Pi's local IP with:

```bash
hostname -I
```

Once [Tailscale](04-tailscale.md) is set up, you will be able to SSH into the Pi from anywhere using its MagicDNS hostname, with no port forwarding needed. For now, local IP is fine.

---

## Ethernet over Wi-Fi

The Pi's Wi-Fi is unreliable for a server running 24/7 — connection drops under load, and the antenna sits close to the SD card and other interference sources. A wired ethernet connection is faster, more stable, and simpler to reason about.

If you want to disable Wi-Fi entirely once ethernet is confirmed working:

```bash
sudo rfkill block wifi
```

To confirm it is blocked:

```bash
rfkill list
```

Wi-Fi should show `Soft blocked: yes`. This persists across reboots.

---

**Next:** [02 — Pi-hole](02-pi-hole.md)
