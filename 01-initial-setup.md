# 01 — Initial Setup
 
Before installing anything, get the system fully up to date.
 
```bash
sudo apt update && sudo apt upgrade -y
```
 
The `&&` ensures the upgrade only runs if the update succeeds. Follow any prompts that appear — some packages may ask how to handle config file changes; keeping the current version is usually the right choice.

Then check if you are indeed running a 64 bit version of Debian:

```bash
dpkg --print-architecture
```
if it returns `arm64`, then you are running a 64 bit version. This is especially important if you would like to run the mineccraft server.
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
you will be prompted to input your password, type the password you chose for your user when you initially set-up your Pi
 
You can find the Pi's local IP with:
 
```bash
hostname -I
```
 
Once [Tailscale](04-tailscale.md) is set up, you will be able to SSH into the Pi from any device on your tailnet (that has a CLI) using its Tailscale IP, with no additionall faffing. For now, local IP is fine.
 
---
 
## Ethernet over Wi-Fi
 
Wi-Fi is unreliable for a server running 24/7 and a wired ethernet connection is faster, more stable, and simpler to reason about.
 
If you want to disable Wi-Fi entirely once ethernet is working:
 
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
 