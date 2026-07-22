# 10 — Vaultwarden

Vaultwarden is a lightweight, self-hosted, Bitwarden-compatible password manager. This guide deploys it as a Docker container bound to the Tailscale IP, reachable only from devices on the tailnet.

**Prerequisites:** [04 — Tailscale](04-tailscale.md) and [08 — Docker](08-docker.md) complete.

**What we're adding:**

| Service | Docker host port | Tailscale Serve port |
|---|---|---|
| Vaultwarden | 8096 (bound to Tailscale IP) | 8445 |

---

## Step 1 — Check your Tailscale addresses

Already known from [04 — Tailscale](04-tailscale.md), but worth reconfirming:

```bash
tailscale ip -4
tailscale status | grep $(hostname)
```

Confirm HTTPS certificates are still enabled at `https://login.tailscale.com/admin/dns` — both **MagicDNS** and **HTTPS Certificates** must be on.

---

## Step 2 — Set up Tailscale Serve

Setting this up before starting the stack means `DOMAIN` is correct from the start.

> **Why point at the Tailscale IP, not `localhost`?** The Vaultwarden container is bound to the Tailscale IP. Pointing Serve at `localhost` gets a connection refused — nothing is listening there from Docker's perspective.

> **Why port 8445?** OpenClaw's Tailscale Serve is on 8443, n8n's is on 8444. 8445 is the next free port.

```bash
tailscale serve --bg --https=8445 http://100.101.7.56:8096
```

Verify:

```bash
tailscale serve status
```

Expected output:

```
https://raspberrypi.tail7aae4f.ts.net:8445 (tailnet only)
|-- / proxy http://100.101.7.56:8096
```

Tailscale persists Serve rules natively across reboots — no action needed after a restart.

---

## Step 3 — Create the project directory

```bash
mkdir -p ~/vaultwarden
cd ~/vaultwarden
```

---

## Step 4 — Create the environment file

```bash
nano ~/vaultwarden/.env
```

Start with a temporary plain-text token — it gets replaced with a proper Argon2 hash in Step 7 once the container is running.

```env
# Your Tailscale IP from Step 1
TAILSCALE_IP=100.101.7.56

# Temporary plain-text token — replaced with an Argon2 hash in Step 7
ADMIN_TOKEN=change-me-after-first-start

# Must exactly match the Tailscale Serve HTTPS URL from Step 2
DOMAIN=https://raspberrypi.tail7aae4f.ts.net:8445
```

Protect the file:

```bash
chmod 600 ~/vaultwarden/.env
```

> **Why set `DOMAIN` here?** Vaultwarden embeds this URL in invitation emails, WebAuthn/passkey responses, attachment links, and push notification payloads. An incorrect `DOMAIN` silently breaks browser extensions, mobile apps, and passkey registration.

---

## Step 5 — Create the Docker Compose file

```bash
nano ~/vaultwarden/docker-compose.yml
```

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    ports:
      - "${TAILSCALE_IP}:8096:80"   # Tailscale Serve proxies to this
      - "127.0.0.1:8096:80"         # Local health checks from the Pi host
    environment:
      - DOMAIN=${DOMAIN}
      - ADMIN_TOKEN=${ADMIN_TOKEN}
      - SIGNUPS_ALLOWED=false
    volumes:
      - ./vaultwarden-data:/data
```

> **Why `SIGNUPS_ALLOWED=false`** The account is created through the admin panel in Step 9. Starting with registrations disabled means there is never a window where another device on the tailnet could register before you.

> **Why two port bindings?** The Tailscale IP binding is what Tailscale Serve proxies to. The `127.0.0.1` binding allows `curl http://127.0.0.1:8096` health checks from the Pi host itself without going through Tailscale.

---

## Step 6 — Start the stack

```bash
cd ~/vaultwarden
docker compose pull
docker compose up -d
docker compose ps
```

The container should show `Up`. Check the logs to confirm it started correctly:

```bash
docker compose logs vaultwarden
```

**Healthy output:**

```
vaultwarden | Rocket has launched from http://0.0.0.0:80
```

You should also see this warning in the logs — expected at this stage, fixed in Step 7:

```
[NOTICE] You are using a plain text `ADMIN_TOKEN` which is insecure.
```

No UFW rule is needed for the Vaultwarden container. If UFW has already been set up ([11 — UFW](11-ufw.md)), the `172.16.0.0/12` rule added there covers Docker's bridge subnet regardless of the bridge's generated name.

---

## Step 7 — Secure the admin token

Now that the container is running, use the Vaultwarden binary inside it to generate a proper Argon2 hash:

```bash
docker exec -it vaultwarden /vaultwarden hash
```

You will be prompted to enter and confirm a password — this is what you type to log in to the admin panel. The output looks like:

```
Generate an Argon2id PHC string using the 'bitwarden' preset:
Password:
Confirm Password:
ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$...$...'
```

Copy everything between the single quotes.

Open `.env` and replace the temporary token:

```bash
nano ~/vaultwarden/.env
```

```env
TAILSCALE_IP=100.101.7.56
ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$...$...'
DOMAIN=https://raspberrypi.tail7aae4f.ts.net:8445
```

> **Why single quotes around `ADMIN_TOKEN`?** The Argon2 hash contains `$` signs that a shell would try to interpolate as variable names. Single quotes in a `.env` file prevent that — Docker Compose strips the quotes correctly before passing the value to the container. Do not double the `$` signs.

Apply the change:

```bash
docker compose up -d
```

Confirm the plain-text warning is gone:

```bash
docker compose logs vaultwarden | grep NOTICE
```

No output means the Argon2 hash was accepted correctly.

> **Remember:** to log in to the admin panel, always enter the **password you chose above** — not the hash string itself.

---

## Step 8 — Configure SMTP

SMTP enables invitation emails, account verification, 2FA codes, password hints, and emergency access requests.

### Get a Gmail app password

Google does not allow the regular account password for SMTP — an **app password** is needed, which requires 2-Step Verification enabled on the Google account.

**1 —** Go to `https://myaccount.google.com/apppasswords`.

**2 —** Name it (e.g. `vaultwarden`) and click **Create**.

**3 —** Copy the 16-character password — it's shown only once.

### Add SMTP variables to `.env`

```bash
nano ~/vaultwarden/.env
```

Add below the existing variables:

```env
# SMTP
SMTP_HOST=smtp.gmail.com
SMTP_FROM=your-address@gmail.com
SMTP_USERNAME=your-address@gmail.com
SMTP_PASSWORD=your-16-char-app-password
SIGNUPS_VERIFY=true
```

### Add SMTP variables to `docker-compose.yml`

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    ports:
      - "${TAILSCALE_IP}:8096:80"
      - "127.0.0.1:8096:80"
    environment:
      - DOMAIN=${DOMAIN}
      - ADMIN_TOKEN=${ADMIN_TOKEN}
      - SIGNUPS_ALLOWED=false
      - SMTP_HOST=${SMTP_HOST}
      - SMTP_FROM=${SMTP_FROM}
      - SMTP_PORT=587
      - SMTP_SECURITY=starttls
      - SMTP_USERNAME=${SMTP_USERNAME}
      - SMTP_PASSWORD=${SMTP_PASSWORD}
      - SIGNUPS_VERIFY=${SIGNUPS_VERIFY}
    volumes:
      - ./vaultwarden-data:/data
```

> **Why port 587 and `starttls`?** Port 587 is the standard submission port for outbound email. STARTTLS upgrades the connection to TLS after it opens. Port 465 with `force_tls` is the alternative if a provider requires implicit TLS from the start — Gmail supports both, but 587 is preferred.

Apply the changes:

```bash
docker compose up -d
```

### Test SMTP

Visit the admin panel:

```
https://raspberrypi.tail7aae4f.ts.net:8445/admin
```

Scroll to **SMTP Email Settings**, enter an email address in **Send test email**, and click send. A test email should arrive within a minute.

If it fails, the error is shown inline. Check the logs for more detail:

```bash
docker compose logs vaultwarden | grep -i smtp
```

---

## Step 9 — First-time setup

> **Warning — `config.json` override:** If **Save Settings** is clicked anywhere in the admin panel, Vaultwarden writes a `config.json` to `./vaultwarden-data/` that permanently overrides all environment variables — including `ADMIN_TOKEN`. If the admin token stops working after a settings save, either update the token through the admin panel UI itself, or delete `config.json` and restart to revert to environment-variable-only config:
>
> ```bash
> rm ~/vaultwarden/vaultwarden-data/config.json
> docker compose restart vaultwarden
> ```

Visit from any device on the tailnet:

```
https://raspberrypi.tail7aae4f.ts.net:8445
```

This shows the Vaultwarden login page. Because `SIGNUPS_ALLOWED=false` is set, self-registration is disabled — the account is created through the admin panel instead.

**1 —** Visit the admin panel and log in with the password from Step 7:

```
https://raspberrypi.tail7aae4f.ts.net:8445/admin
```

**2 —** Go to **Users → Invite User** and invite yourself with your email address.

**3 —** Open the invitation email and complete registration.

---

## Useful commands

```bash
# Stop the stack
docker compose down

# Restart
docker compose restart vaultwarden

# Update to latest image
docker compose pull && docker compose up -d

# Follow logs live
docker compose logs -f vaultwarden

# Resource usage
docker stats

# Health check from the Pi host
curl -fsS http://127.0.0.1:8096/alive

# Check Tailscale Serve is active
tailscale serve status

# Regenerate admin token hash (if needed)
docker exec -it vaultwarden /vaultwarden hash
```

---

## Troubleshooting

**Vaultwarden isn't reachable at `https://raspberrypi.tail7aae4f.ts.net:8445`**

Check Tailscale Serve is still running:

```bash
tailscale serve status
```

Tailscale Serve rules persist natively — if the rule is genuinely missing, re-run:

```bash
tailscale serve --bg --https=8445 http://100.101.7.56:8096
```

---

**Certificate error when visiting the URL**

HTTPS certificates may not be provisioned yet. Confirm **Enable HTTPS** is toggled on in the Tailscale admin DNS tab. It can take a minute or two on first launch. Force a certificate refresh:

```bash
sudo tailscale cert raspberrypi.tail7aae4f.ts.net
```

---

**Browser extensions or mobile apps can't connect**

Confirm `DOMAIN` in `.env` matches the Tailscale Serve URL exactly, including the port:

```env
DOMAIN=https://raspberrypi.tail7aae4f.ts.net:8445
```

Restart the stack after any change:

```bash
docker compose up -d
```

---

**Admin token login fails after saving settings in the UI**

Vaultwarden has written a `config.json` that overrides `ADMIN_TOKEN`. Update the token through the admin panel UI, or delete `config.json` and restart:

```bash
rm ~/vaultwarden/vaultwarden-data/config.json
docker compose restart vaultwarden
```

---

**Invitation or verification emails aren't arriving**

Check the logs for SMTP errors:

```bash
docker compose logs vaultwarden | grep -i smtp
```

Common causes:
- Wrong app password — generate a new one at `https://myaccount.google.com/apppasswords` and update `SMTP_PASSWORD` in `.env`
- 2-Step Verification not enabled on the Google account — app passwords require it
- `SMTP_FROM` doesn't match `SMTP_USERNAME` — they must be the same Gmail address

After any `.env` change, redeploy:

```bash
docker compose up -d
```

Then use **Send test email** in the admin panel to confirm before inviting yourself.

---

**Pi-hole DNS stops working after Docker restarts**

```bash
sudo systemctl restart pihole-FTL
```

---

## Backups

The entire Vaultwarden database, attachments, and keys live in `~/vaultwarden/vaultwarden-data/`. Back this up regularly — losing it means losing every stored password.

A cron job that compresses and archives it daily, with a companion job that deletes backups older than 14 days:

```bash
# Use an absolute path — tilde (~) does not expand reliably in cron
0 2 * * * tar -czf /backup/vaultwarden-$(date +\%F).tar.gz /home/YOUR_USER/vaultwarden/vaultwarden-data/
5 2 * * * find /backup -name 'vaultwarden-*.tar.gz' -mtime +14 -delete
```

Vaultwarden also exposes a built-in SQLite backup endpoint from the admin panel at `/admin` — useful for on-demand snapshots without stopping the container.

---

*This guide cross-references the main homelab guide. See the [Reference](../reference.md) for the complete port map.*

**Sources:** [31]