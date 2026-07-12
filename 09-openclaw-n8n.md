# 09 — OpenClaw + n8n

OpenClaw is a personal AI agent framework — it connects to messaging channels, maintains persistent context in a workspace, and can execute tasks via tools and skills. n8n is a workflow automation engine that runs alongside it as a background automation layer. Both run as Docker containers bound to the Tailscale IP, invisible on the local network.

**Prerequisites:** [04 — Tailscale](04-tailscale.md) and [08 — Docker](08-docker.md) complete.

**What we're adding:**

| Service | Port |
|---|---|---|
| OpenClaw | 18789 |
| n8n | 5678 |
| Tailscale Serve (OpenClaw HTTPS) | 8443 |
| Tailscale Serve (n8n HTTPS) | 8444 |

> **Why HTTPS from the start?** OpenClaw's Control UI uses browser APIs that require a secure context — they are blocked over plain HTTP. Tailscale Serve provides valid HTTPS certificates automatically for the tailnet, no domain or Certbot needed. Setting it up before starting the stack means every URL baked into configs is correct from day one — retrofitting `WEBHOOK_URL` after workflows already exist means fixing the same value in multiple places.

---

## Step 1 — Set up Tailscale Serve

Confirm your Tailscale IP and MagicDNS hostname (already known from [04 — Tailscale](04-tailscale.md)):

```bash
tailscale ip -4
tailscale status | grep $(hostname)
```

Create the two Serve rules:

```bash
tailscale serve --bg --https=8443 http://100.101.7.56:18789
tailscale serve --bg --https=8444 http://100.101.7.56:5678
```

> **Why point at the Tailscale IP, not `localhost`?** Docker containers are bound to the Tailscale IP. Pointing Tailscale Serve at `localhost` gets a connection refused — nothing is listening there from Docker's perspective.

Verify:

```bash
tailscale serve status
```

Expected output:

```
https://raspberrypi.tail7aae4f.ts.net:8443 (tailnet only)
|-- / proxy http://100.101.7.56:18789

https://raspberrypi.tail7aae4f.ts.net:8444 (tailnet only)
|-- / proxy http://100.101.7.56:5678
```

Tailscale persists Serve rules across reboots — no need to re-run these after a restart.

---

## Step 2 — Create the project directory

```bash
mkdir -p ~/ai-stack
cd ~/ai-stack
```

Pre-create the OpenClaw data directory before starting the stack:

```bash
mkdir -p ~/ai-stack/openclaw-data
sudo chown -R 1000:1000 ~/ai-stack/openclaw-data
```

> **Why?** Docker creates mounted directories as `root` if they don't already exist. The OpenClaw container runs as user `node` (UID 1000) and crashes immediately with `EACCES: permission denied` if the directory is root-owned.

---

## Step 3 — Create the environment file

```bash
nano ~/ai-stack/.env
```

```env
# Your Tailscale IP from 04 — Tailscale
TAILSCALE_IP=100.101.7.56

# Google AI Studio key — get one at aistudio.google.com/apikey
GEMINI_API_KEY=your_gemini_key_here

# Groq key (backup provider) — get one at console.groq.com/keys
GROQ_API_KEY=your_groq_key_here

# n8n webhook URL — must use the Tailscale Serve HTTPS address from Step 1
# n8n embeds this URL in webhook payloads so external services can call back
WEBHOOK_URL=https://raspberrypi.tail7aae4f.ts.net:8444

# Silence n8n telemetry — prevents ECONNREFUSED log spam from failed
# analytics calls to external servers unreachable from behind the VPN
N8N_DIAGNOSTICS_ENABLED=false
```

Protect the file:

```bash
chmod 600 ~/ai-stack/.env
```

---

## Step 4 — Create the Docker Compose file

```bash
nano ~/ai-stack/docker-compose.yml
```

```yaml
services:
  openclaw:
    image: ghcr.io/openclaw/openclaw:latest
    ports:
      - "${TAILSCALE_IP}:18789:18789"  # Tailscale Serve proxies to this
      - "127.0.0.1:18789:18789"        # Local CLI commands and health checks
    environment:
      - GEMINI_API_KEY=${GEMINI_API_KEY}
      - GROQ_API_KEY=${GROQ_API_KEY}
      - OPENCLAW_GATEWAY_BIND=lan
      - OPENCLAW_DISABLE_BONJOUR=1
    volumes:
      - ./openclaw-data:/home/node/.openclaw
    restart: unless-stopped

  n8n:
    image: n8nio/n8n:latest
    ports:
      - "${TAILSCALE_IP}:5678:5678"
    environment:
      - NODE_ENV=production
      - WEBHOOK_URL=${WEBHOOK_URL}
      - N8N_DIAGNOSTICS_ENABLED=${N8N_DIAGNOSTICS_ENABLED}
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - n8n-data:/home/node/.n8n
    restart: unless-stopped

volumes:
  n8n-data:
```

> **Why `OPENCLAW_GATEWAY_BIND=lan`?** Without this, the gateway process inside the container binds only to its own internal loopback (`127.0.0.1`). Docker's port publishing maps host ports to container ports, but if the container-side process only listens on loopback, external connections are refused even though the port appears published in `docker ps`. Set as an environment variable — not via `openclaw config set` — because OpenClaw overwrites `openclaw.json` frequently and environment variables always take precedence.

> **Why `OPENCLAW_DISABLE_BONJOUR=1`?** Docker bridge networking does not forward mDNS multicast packets. OpenClaw's Bonjour plugin tries to advertise the gateway over local mDNS, gets stuck probing, and crashes the entire gateway roughly every 37 seconds. This is documented in the official OpenClaw Docker guide.

> **Why `127.0.0.1:18789:18789` in addition to the Tailscale IP binding?** Allows `docker compose exec openclaw openclaw ...` CLI commands and `curl http://127.0.0.1:18789/healthz` health checks to work from the Pi host.

> **Why `extra_hosts: host.docker.internal:host-gateway`?** On Linux Docker, `host.docker.internal` is not automatically resolved inside containers. This maps it to the Docker host gateway IP so n8n can reach services on the Pi host machine if a workflow ever needs to.

> **Why no `N8N_BASIC_AUTH_ACTIVE`?** Deprecated in n8n 2.x. n8n handles auth through its own user system — the account is created via the UI on first launch.

---

## Step 5 — Start the stack

```bash
cd ~/ai-stack
docker compose pull
docker compose up -d
docker compose ps
```

Both containers should show `Up` or `Up (healthy)`.

**Healthy OpenClaw logs:**

```
[gateway] agent model: google/gemma-4-31b-it
[gateway] ready (5 plugins: acpx, browser, device-pair, phone-control, talk-voice)
```

5 plugins, not 6 — Bonjour is correctly absent.

**Healthy n8n logs:**

```
Editor is now accessible via:
http://100.101.7.56:5678
```

The `model-pricing OpenRouter/LiteLLM fetch failed` warnings are harmless — n8n tries to fetch pricing data from the internet and can't reach it. They do not affect functionality.

---

## Step 6 — Run the onboarding wizard

The onboarding wizard is the official, recommended way to configure OpenClaw. Use `--flow manual` to get full control over every value — port, bind, auth, and model — rather than having them auto-generated silently.

```bash
docker compose exec -it openclaw openclaw onboard
```

> **Why `-it`?** The flags attaches an interactive terminal — required for the wizard's prompts to display correctly inside Docker.

The wizard has 9 steps:

**Step 1 — Flow selection**

| Prompt | Answer |
|---|---|
| Quickstart or Manual? | Manual |

> Manual gives full control over port, bind, auth mode, and model. Quickstart auto-generates values silently — avoid it.

**Step 2 — Existing config detection**

| Prompt | Answer |
|---|---|
| Config already exists — what to do? | Keep / Use existing values |

> The gateway token was generated on first start. Choosing Keep preserves it. Choosing Reset wipes it and forces a new token — avoid unless factory resetting.

**Step 3 — Model / Auth**

| Prompt | Answer |
|---|---|
| Provider | Google (Gemini) |
| API key | Paste your API key |
| Default model | `google/gemma-4-31b-it` |
| Add a second provider? | Yes |
| Second provider | Groq |
| Groq API key | Paste your API key |

> Gemma 4 31B has a 256K context window — the primary reason to prefer it over other providers. Groq stays as a fast, low-latency fallback.

**Step 4 — Workspace**

| Prompt | Answer |
|---|---|
| Workspace directory | Press Enter (accept default: `/home/node/.openclaw/workspace`) |

**Step 5 — Gateway**

| Prompt | Answer |
|---|---|
| Port | `18789` |
| Bind | `lan` |
| Auth mode | `token` |
| Token | Press Enter (keep existing) |
| Tailscale exposure | `off` |

> **Why `lan` and not `loopback`?** With `loopback`, the gateway process inside the container only listens on its own internal `127.0.0.1`. Docker's port publishing maps host ports to container ports, but if nothing is listening on the container-side, connections are refused even though the port appears published. `lan` makes the gateway listen on all container interfaces, including the one Docker bridges to the host.

> **Why `tailscale off`?** The native Tailscale integration (`serve` mode) requires the `tailscale` CLI inside the container, which the standard Docker image does not include. Tailscale Serve is handled externally, in Step 1 of this guide.

**Step 6 — Channels**

| Prompt | Answer |
|---|---|
| Set up a channel now? | Skip (or configure WhatsApp/Telegram if desired) |

> Channels can be added at any time via `openclaw onboard` or `openclaw configure`. Skip here unless you are ready to connect one now — Telegram setup is covered separately in Step 11 of this guide.

**Step 7 — Daemon install**

| Prompt | Answer |
|---|---|
| Install as a system daemon? | No |

> Docker manages the process lifecycle via `restart: unless-stopped`. Installing a systemd daemon inside the container as well would create two competing process managers for the same service.

**Step 8 — Health check**

This step runs automatically. The wizard starts the gateway and waits for it to respond to `openclaw health`. A spinner appears — this is normal and takes up to 30 seconds on first run.

**Step 9 — Skills**

| Prompt | Answer |
|---|---|
| Install skills? | Skip for now (or choose npm/pnpm if you want agent tools) |

> Skills add capabilities to the agent (web browsing, code execution, etc). They can be installed at any time via `openclaw skills`.

**Finish**

The wizard prints a summary and next steps. Verify the model was written correctly:

```bash
docker compose exec openclaw openclaw config get agents.defaults.model
# Should print: google/gemma-4-31b-it
```

Then restart to ensure all env vars and config are in sync:

```bash
docker compose restart openclaw
```

---

## Step 7 — Get the gateway token

OpenClaw generates a token on first start. It is needed to log in and to approve device pairing.

```bash
docker compose exec openclaw openclaw dashboard --no-open
```

This prints the full Control UI URL with the token embedded as `?token=...`. Copy the token value from the URL.

---

## Step 8 — Access the UIs

| Service | URL |
|---|---|
| OpenClaw | `https://raspberrypi.tail7aae4f.ts.net:8443` |
| n8n | `https://raspberrypi.tail7aae4f.ts.net:8444` |

**n8n:** On first launch, prompts for an owner account (email + password). These become the permanent login credentials.

**OpenClaw:** Paste the gateway token from Step 7. The next screen says "device pairing required" — proceed to Step 9.

---

## Step 9 — Pair your browser device

OpenClaw requires each browser session to be explicitly approved before it's trusted. This prevents anyone who can reach the URL from accessing the gateway without approval.

> **Why not `/pair` in the chat?** The `/pair` slash command generates a mobile app pairing code (for the iOS app). It does not approve browser sessions. Browser pairing uses the `devices` CLI.

**1 —** Open `https://raspberrypi.tail7aae4f.ts.net:8443`. The pairing screen shows a `requestId`.

**2 —** On the Pi, approve it:

```bash
docker compose exec openclaw openclaw devices approve --latest
```

**3 —** Refresh the browser immediately. You are now logged in.

---

## Step 10 — Workspace bootstrap

On the first chat, the agent responds with a setup conversation instead of a normal greeting. **This is expected — it is not an error.**

> **What is happening:** OpenClaw creates a `BOOTSTRAP.md` in the workspace on first run. The agent reads it and runs a one-time identity setup — name, personality, preferences. Once done, it deletes `BOOTSTRAP.md` and starts responding normally.

Follow the conversation. The agent confirms when bootstrap is complete.

---

## Step 11 — Connect Telegram

This adds Telegram as a second messaging channel alongside the one already configured. It is new — verify each step against your own gateway before relying on this guide.

**1 — Create the bot in BotFather.**

Open Telegram, chat with `@BotFather` (confirm the handle is exactly that), and run:

```
/newbot
```

Follow the prompts for name and username. BotFather replies with a token in the form `8740053914:ARGRCCYNzoMvPXL4T0_RQFBq4O0BHIJXYFY`. Save it.

**2 — Give the token to Openclaw**
run the following in your terminal

```bash
docker compose exec openclaw openclaw configure
```

you will then be prompted to select a few options, like during the onboarding wizard
| Prompt | Answer |
|---|---|
|  Where will the Gateway run? | local |
|---|---|
| What do you want to configure? | channels |

Then select add or update channels, followed by telegram. It will then ask you for your bot token. paste it in, and then select finish.

|Configure DM access policies now? (default:pairing)| Yes, then select `pairing (recommended)` |

you will then be sent back to the `What do you want to configure?` phase, select `continue`

**3 — Trigger pairing.**

Send any message to the bot on Telegram — "hello" is fine, 

**4 — Find your Telegram user ID.**

```bash
docker compose logs openclaw --tail=30
```

Look for a `from.id` field in the incoming message log. If it's not obvious from the logs, the Telegram Bot API itself will show it directly:

```bash
curl "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates"
```

The `from.id` in the response is your numeric Telegram user ID.

**5 — Approve the pairing request.**

Pairing codes expire after one hour.

```bash
docker compose exec openclaw openclaw pairing list telegram
docker compose exec openclaw openclaw pairing approve telegram <CODE>
```

Message the bot again to confirm it now responds.

**6 — Lock down access.**

By default, Telegram's DM policy (`pairing`) accepts a pairing request from any account that finds the bot. For a personal bot, switch to an explicit allowlist so only your account is accepted going forward. Replace `YOUR_TELEGRAM_USER_ID` with the numeric ID from step 4:

```bash
docker compose exec openclaw openclaw config set channels.telegram.dmPolicy allowlist
docker compose exec openclaw openclaw config set channels.telegram.allowFrom '["YOUR_TELEGRAM_USER_ID"]'
docker compose restart openclaw
```

Message the bot once more to confirm it still responds to you.

---

## Step 12 — Connect n8n to OpenClaw

When building n8n workflows that call OpenClaw, use the container name as the hostname:

```
http://openclaw:18789
```

> **Why the container name and not `host.docker.internal`?** Both containers are defined in the same `docker-compose.yml` and share a Docker network. Container-to-container traffic via service name (`openclaw`) stays inside Docker's virtual network — it never touches the Pi's host network or Tailscale. It's faster, more reliable, and the semantically correct approach for same-compose services.

> **Why not the Tailscale HTTPS URL?** That would route traffic out through Tailscale Serve and back in — an unnecessary round-trip for two containers already on the same machine.

---

## Pi optimisations

These are not required but improve stability on a Pi running long-term services.

### Reduce GPU memory allocation

The Pi reserves GPU memory even when headless. Reduce it to give more RAM to Docker:

```bash
echo "gpu_mem=16" | sudo tee -a /boot/firmware/config.txt
sudo systemctl disable bluetooth
sudo reboot
```

> `/boot/firmware/config.txt` is the correct path on current Raspberry Pi OS. Older guides may reference `/boot/config.txt` — that path is outdated.

### Speed up CLI commands

Repeated `docker compose exec` calls are noticeably faster with the Node compile cache enabled. Add to `~/.bashrc`:

```bash
grep -q 'NODE_COMPILE_CACHE' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
EOF
source ~/.bashrc
```

### Check for CPU throttling

If the Pi is thermal-throttling, it silently slows down:

```bash
vcgencmd get_throttled
# 0x0 = no issues
# Anything else = throttling has occurred — check cooling and power supply
```

### BOOTSTRAP.md reset

If bootstrap has already been completed and you want to reset the agent's identity, delete the file and it regenerates:

```bash
rm ~/ai-stack/openclaw-data/workspace/BOOTSTRAP.md
docker compose restart openclaw
```

The file lives on the Pi host at that path — no need to go inside the container.

---

## Useful commands

```bash
# Stop the stack
docker compose down

# Restart one service
docker compose restart openclaw

# Update to latest images
docker compose pull && docker compose up -d

# Follow logs live
docker compose logs -f

# Resource usage
docker stats

# OpenClaw health checks
curl -fsS http://127.0.0.1:18789/healthz   # Is the gateway alive?
curl -fsS http://127.0.0.1:18789/readyz    # Is it ready to accept connections?

# List paired devices
docker compose exec openclaw openclaw devices list

# Re-pair browser (run when pairing screen appears)
docker compose exec openclaw openclaw devices approve --latest

# Run OpenClaw diagnostics (first step for any gateway issue)
docker compose exec openclaw openclaw doctor --non-interactive

# Check current config
docker compose exec openclaw openclaw config get gateway
docker compose exec openclaw openclaw config get agents.defaults.model
```

---

## Troubleshooting

**First step for any OpenClaw issue:**

```bash
docker compose exec openclaw openclaw doctor --non-interactive
```

This runs all health checks and prints a clear summary of what's wrong.

---

**Tailscale Serve returns 502**

Tailscale Serve is running but can't reach the container. Either the container is down, or the Serve rules are pointing at `localhost` instead of the Tailscale IP. Check and re-create:

```bash
docker compose ps
tailscale serve status

# Re-create if needed
tailscale serve --bg --https=8443 off
tailscale serve --bg --https=8444 off
tailscale serve --bg --https=8443 http://100.101.7.56:18789
tailscale serve --bg --https=8444 http://100.101.7.56:5678
```

---

**OpenClaw UI says "origin not allowed"**

```bash
docker compose exec openclaw openclaw config set \
  --merge gateway.controlUi.allowedOrigins \
  '["https://raspberrypi.tail7aae4f.ts.net:8443"]'
docker compose restart openclaw
```

> **Why `--merge`?** Without it, `config set` replaces the entire `allowedOrigins` array, wiping any existing entries. `--merge` appends instead.

---

**OpenClaw crashes every ~37 seconds (CIAO PROBING CANCELLED)**

`OPENCLAW_DISABLE_BONJOUR=1` in the compose file should prevent this. If it still happens, confirm the env var is being passed:

```bash
docker compose exec openclaw env | grep BONJOUR
```

If missing, check `.env` and run `docker compose config` to see the fully resolved configuration. As a fallback:

```bash
docker compose exec openclaw openclaw plugins disable bonjour
docker compose restart openclaw
```

Healthy ready line shows `5 plugins`, not `6`.

---

**`gateway.bind` reverts to loopback**

OpenClaw overwrites `openclaw.json` during some operations. Since `OPENCLAW_GATEWAY_BIND=lan` is in the compose environment it should always win on restart. Confirm it's being passed:

```bash
docker compose exec openclaw env | grep OPENCLAW_GATEWAY_BIND
# Should print: OPENCLAW_GATEWAY_BIND=lan
```

---

**Device pairing commands time out**

Debug in order:

```bash
# 1. Is the container healthy?
docker compose ps

# 2. Are both ports bound?
sudo ss -tlnp | grep 18789
# Should show: 127.0.0.1:18789 AND 100.101.7.56:18789

# 3. Can the Pi reach the gateway?
curl http://127.0.0.1:18789/healthz

# 4. Is it ready?
curl http://127.0.0.1:18789/readyz

# 5. Recent logs
docker compose logs openclaw --tail=30
```

---

**Groq or Gemini not responding**

```bash
# Are the keys in the container?
docker compose exec openclaw env | grep -E "GEMINI|GROQ"

# Is the correct model active?
docker compose logs openclaw | grep "agent model"
# Should show: agent model: google/gemma-4-31b-it

# Force set if wrong
docker compose exec openclaw openclaw models set google/gemma-4-31b-it
docker compose restart openclaw
```

---

**Next:** [10 — Vaultwarden](10-vaultwarden.md)

**Sources:** [26] [27] [28] [29] [30]
