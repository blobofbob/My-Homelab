# 03 — Apache + Certbot
 
Apache is the web server that serves these guides publicly. Certbot handles TLS certificates from Let's Encrypt, so the site is accessible over HTTPS with a valid certificate and automatic renewal.
 
**Prerequisites:** [02 — Pi-hole](02-pi-hole.md) complete, specifically the port change away from 80 and 443, since Apache will take those ports. Moreover, if your Pi is not in a de-militarized zone, add port forwarding rules to your router for ports 80 and 443
 
---
 
## Installing Apache
 
```bash
sudo apt install apache2 -y
sudo systemctl enable apache2
sudo systemctl start apache2
```
 
Verify it is running:
 
```bash
sudo systemctl status apache2
```
 
Then visit `http://YOUR_PI_LOCAL_IP` in a browser. You should see the default Apache page. If you do, Apache is running and port 80 is open.
 
---
 
## Fixing the AH00558 warning
 
Apache may log this on start:
 
```
AH00558: apache2: Could not reliably determine the server's fully qualified domain name
```
 
It is harmless but noisy. Fix it by adding a `ServerName` directive to the main Apache config: [24]
 
```bash
echo "ServerName localhost" | sudo tee -a /etc/apache2/apache2.conf
sudo systemctl restart apache2
```
 
The warning should be gone on next start.
 
---
 
## Putting your site on the Pi
 
Apache serves files from `/var/www/html/` by default. Replace the default page with your own content:
 
```bash
sudo rm /var/www/html/index.html
sudo nano /var/www/html/index.html
```
 
File ownership is worth setting correctly so you can edit without `sudo`:
 
```bash
sudo chown -R $USER:$USER /var/www/html
```
 
Your site is now live at `http://YOUR_PI_LOCAL_IP` on the local network, and if your port forwarding rules are active, you should be able to reach it from outside your network at `http://YOUR_PUBLIC_IP`. The next step is making it reachable from the internet under a real domain name.
 
---
 
## Domain and DNS
 
To serve the site publicly over HTTPS, you need a domain name with a DNS A record pointing at your Pi's public IP address. You can find your current public IP at `https://ifconfig.me`.
 
> **Dynamic IP:** Most home ISPs assign a dynamic public IP that changes periodically. If yours does, the A record will go stale when the IP changes and the site will go down until you update it. Options: use a Dynamic DNS service (e.g. DuckDNS, Cloudflare with the API), or check whether your ISP offers a static IP. I manually update the IP when it changes.
 
Once the A record is set, wait for it to propagate (time may vary between registrars), then confirm it resolves correctly:
 
```bash
dig YOUR_DOMAIN +short
```
 
The output should be your Pi's public IP.
 
---
 
## Certbot — getting a TLS certificate
 
Install Certbot with the Apache plugin: [5]
 
```bash
sudo apt install certbot python3-certbot-apache -y
```
 
Run Certbot with your domain. It will verify you own the domain (by placing a temporary file on your site), issue a certificate, and automatically configure Apache to use it:
 
```bash
sudo certbot --apache -d YOUR_DOMAIN
```
 
Certbot will ask for an email address for renewal reminders, and whether to redirect HTTP to HTTPS — select yes to the redirect.
 
When it finishes, your site is live at `https://YOUR_DOMAIN` with a valid certificate. Visit it in a browser to confirm.
 
---
 
## Auto-renewal
 
Let's Encrypt certificates expire after 90 days. Certbot installs a systemd timer that handles renewal automatically — check it is active:
 
```bash
sudo systemctl status certbot.timer
```
 
It should show `active (waiting)`. To confirm the renewal process actually works end-to-end without issuing a new certificate:
 
```bash
sudo certbot renew --dry-run
```
 
No errors means renewal will work when the time comes. Certbot renews certificates automatically when they are within 30 days of expiry.
 
---
 
## Useful commands
 
```bash
# Check Apache status
sudo systemctl status apache2
 
# Restart Apache (after config changes)
sudo systemctl restart apache2
 
# Test Apache config for syntax errors before restarting
sudo apache2ctl configtest
 
# List active virtual hosts
sudo apache2ctl -S
 
# View Apache error log
sudo tail -f /var/log/apache2/error.log
 
# View Apache access log
sudo tail -f /var/log/apache2/access.log
 
# Check certificate status and expiry
sudo certbot certificates
 
# Force renew all certificates
sudo certbot renew --force-renewal
```
 
---
 
## Troubleshooting
 
**Site not reachable from the internet**
 
Check that port 80 and 443 are open on your router and forwarded to the Pi's local IP. UFW is not set up yet at this stage — if you have already set it up, make sure `80/tcp` and `443/tcp` are allowed:
 
```bash
sudo ufw status
```
 
Also confirm the A record has propagated:
 
```bash
dig YOUR_DOMAIN +short
```
 
---
 
**Certificate issuance fails**
 
Certbot verifies domain ownership by placing a temporary file at `http://YOUR_DOMAIN/.well-known/acme-challenge/`. This requires port 80 to be reachable from the internet. If the verification fails, confirm port 80 is forwarded on your router and that your Pi's local IP hasn't changed since you set up the forwarding rule.
 
---
 
**HTTPS works but HTTP does not redirect**
 
If you selected the redirect option during Certbot setup, it creates an Apache config that redirects port 80 to 443. Check it exists:
 
```bash
sudo apache2ctl -S
```
 
If the redirect is missing, re-run Certbot and select the redirect option, or add it manually to the virtual host config in `/etc/apache2/sites-available/`.
 
---
 
**Next:** [04 — Tailscale](04-tailscale.md)
 
**Sources:** [3] [4] [5] [24]