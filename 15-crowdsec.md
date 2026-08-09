# 15 — CrowdSec

CrowdSec is a log-based intrusion detection engine — it parses logs for attack patterns (brute
force, HTTP probing, etc.), and via a "bouncer" component, auto-blocks offending IPs at the
firewall. It also participates in a shared community blocklist: IPs flagged by other CrowdSec users
worldwide get blocked here too, and vice versa.

**Prerequisites:** [11 — UFW](11-ufw.md) active, [12 — Lynis](12-lynis.md) complete (notification
layer).

---

## Mandatory reading before installing: how the bouncer interacts with UFW and Docker

This project has hit two firewall-related incidents before this guide existed: `iptables-persistent`
installed alongside UFW silently removed UFW's own rules, and a `DOCKER-USER` `ACCEPT` rule once
broke exit-node internet access. CrowdSec's firewall bouncer inserts rules into iptables directly,
so it's worth being precise about what it touches before installing it.

**The bouncer's default scope is `INPUT` only** — `FORWARD` and `DOCKER-USER` are commented out by
default in its config. CrowdSec's own docs say `DOCKER-USER` is only needed for containers
port-forwarded from the router to the WAN. None of the containers in this project are: OpenClaw,
n8n, and Vaultwarden are bound to the Tailscale IP + `127.0.0.1` only, never `0.0.0.0`, reached via
Tailscale Serve rather than port-forwarding. **`DOCKER-USER` should stay untouched** — the same
lesson as the earlier incident, from the opposite direction.

**UFW reload does not wipe the bouncer's rule.** UFW only flushes and rebuilds its own custom chains
on enable/reload — it doesn't clear rules other software inserts directly into the built-in `INPUT`
chain (the same mechanism that lets fail2ban coexist with UFW long-term). Confirmed on
this system: the rule survived a `ufw reload`, and Docker's own internet access was unaffected
immediately after.

---

## Installing the engine

```bash
curl -s https://install.crowdsec.net | sudo sh
sudo apt install -y crowdsec
```

The installer runs an automatic setup pass that detects installed services and enables matching
collections — check what it actually installed rather than assume:

```bash
sudo cscli collections list
```

On this Pi, auto-detection correctly picked up `crowdsecurity/linux` (baseline), `crowdsecurity/sshd`
(sshd is active, tailnet-only via UFW), `crowdsecurity/apache2` (WAN-facing), plus
`auditd`/`base-http-scenarios`/`http-cve`/`whitelist-good-actors`. If any of those are missing on
your system:

```bash
sudo cscli collections install crowdsecurity/linux
sudo cscli collections install crowdsecurity/sshd
sudo cscli collections install crowdsecurity/apache2
```

There is **no official Tor collection** — checked the Hub's full category list directly, no Tor
entry exists. Tor relay logs are outside CrowdSec's scope here.

---

## Port conflict — check before starting the service

CrowdSec's Local API (LAPI) defaults to port 8080 — the same port Pi-hole's web admin uses on this
Pi ([02 — Pi-hole](02-pi-hole.md)). Check for the conflict and move it before it becomes confusing:

```bash
grep listen_uri /etc/crowdsec/config.yaml
```

If it shows `8080`, move it — 8090 keeps it clear of both Pi-hole (8080) and the Tailscale Serve
range (8443–8446):

```bash
sudo sed -i 's/listen_uri: 127.0.0.1:8080/listen_uri: 127.0.0.1:8090/' /etc/crowdsec/config.yaml
```

The bouncer's own credentials file needs the same port, or it can't reach the LAPI:

```bash
sudo sed -i 's|http://127.0.0.1:8080|http://127.0.0.1:8090|' /etc/crowdsec/local_api_credentials.yaml
```

```bash
sudo systemctl start crowdsec
sudo systemctl status crowdsec
```

Confirm `active (running)` before continuing.

---

## Installing the firewall bouncer

```bash
sudo apt install -y crowdsec-firewall-bouncer-iptables
```

Check the shipped config rather than assume its defaults:

```bash
cat /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml
```

Two things worth confirming, not assuming:

- `iptables_chains` should already default to `[INPUT]` only, with `FORWARD` and `DOCKER-USER`
  commented out — matches the scope decided above. No edit needed if so.
- The file also ships an `nftables:` block with `enabled: true`. This looks like it could conflict
  with `mode: iptables`, but it's inert boilerplate present in every default config regardless of
  which mode is active.

`ipset` is a hard dependency for iptables mode — confirm it installed automatically:

```bash
which ipset
```

Start it and confirm the rule actually landed, not just that the service is running:

```bash
sudo systemctl enable --now crowdsec-firewall-bouncer
sudo iptables -L INPUT -n --line-numbers | grep -i crowdsec
```

Should show the CrowdSec chain at position 1 — ahead of UFW's own rules, so a banned IP gets dropped
before UFW ever evaluates it.

---

## Verifying end-to-end

```bash
sudo cscli decisions add --ip 192.0.2.1 --duration 1h --reason "manual test ban"
sudo cscli decisions list
```

Confirm it actually landed in the live ipset, not just CrowdSec's own decision log:

```bash
sudo ipset list crowdsec-blacklists-0 | grep 192.0.2.1
sudo ipset list crowdsec-blacklists-1 | grep 192.0.2.1
```

(Two ipsets exist by design — CrowdSec rotates between them for atomic updates without a coverage
gap.) Clean up the test ban:

```bash
sudo cscli decisions delete --ip 192.0.2.1
```

**Full regression check** — the exact thing the earlier `DOCKER-USER` incident broke, re-verified
here:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://100.101.7.56:8080/admin
docker compose -f ~/ai-stack/docker-compose.yml exec openclaw curl -s https://example.com -o /dev/null -w "%{http_code}\n"
tailscale serve status
```

Pi-hole should respond (a 308 redirect is normal, not an error), the container should get a 200, and
every Tailscale Serve endpoint should still be registered.

**Stress-test the UFW-survival claim directly, don't just trust the docs:**

```bash
sudo ufw reload
sudo iptables -L INPUT -n --line-numbers | grep -i crowdsec
docker compose -f ~/ai-stack/docker-compose.yml exec openclaw curl -s https://example.com -o /dev/null -w "%{http_code}\n"
```

The rule and Docker's internet access should both survive the reload untouched.

---

## Notifications

CrowdSec uses its own plugin system (`notification-plugins`) rather than local `mail`/cron — the
email plugin connects to SMTP directly. Check the shipped template first:

```bash
cat /etc/crowdsec/notifications/email.yaml
```

Fill in the SMTP fields — reusing the same Gmail app password from
[12 — Lynis](12-lynis.md)'s notification layer keeps one credential per system-level tool rather than
a separate one per component:

```bash
sudo nano /etc/crowdsec/notifications/email.yaml
```

Set `smtp_host: smtp.gmail.com`, `smtp_port: 587`, `auth_type: login`,
`encryption_type: "starttls"`, and fill in `smtp_username`, `smtp_password`, `sender_email` (your
Gmail address and app password — visible in `/etc/msmtprc` if you need to copy them across) and
`receiver_emails` (your own address).

**Wire it into the remediation profiles** — this needs *two* edits, not one. The profile file ships
with both the plugin reference **and its parent key** commented out:

```yaml
# notifications:
#   - email_default
```

Uncommenting only the list item leaves it as an orphaned entry under a still-commented parent key —
invalid YAML, and the service will fail to start with `value is not allowed in this context`. Both
lines need uncommenting:

```bash
sudo sed -i 's/^# notifications:/notifications:/' /etc/crowdsec/profiles.yaml
sudo sed -i 's/#   - email_default/  - email_default/' /etc/crowdsec/profiles.yaml
```

Validate before restarting — catches YAML mistakes without a failed service restart:

```bash
sudo crowdsec -c /etc/crowdsec/config.yaml -t -error
```

No output means it passed. Then:

```bash
sudo systemctl restart crowdsec
sudo systemctl status crowdsec
```

Test with a real ban and check your inbox:

```bash
sudo cscli decisions add --ip 192.0.2.1 --duration 1h --reason "notification test"
sudo cscli decisions delete --ip 192.0.2.1
```

---

## Community blocklist

Confirm it's actually active rather than assume:

```bash
sudo cscli capi status
```

Should confirm the Central API connection and that community blocklist pulling is enabled — this is
separate from, and doesn't require, the optional (free) console account at app.crowdsec.net.

To see whether an entry currently blocked came from the community blocklist versus this Pi's own
local detection:

```bash
sudo cscli decisions list -o human
```

`Source: crowdsec` means a local scenario caught it directly (e.g. `crowdsecurity/http-probing`
against Apache); a `CAPI` source means it came from the shared blocklist. Absence of `CAPI` entries
isn't a problem on its own — it just means nothing currently on the shared list has hit this Pi yet.

---

## Useful commands

```bash
# Service status
sudo systemctl status crowdsec crowdsec-firewall-bouncer

# List active decisions (bans)
sudo cscli decisions list

# Add / remove a manual ban
sudo cscli decisions add --ip X.X.X.X --duration 1h --reason "..."
sudo cscli decisions delete --ip X.X.X.X

# List installed collections/parsers/scenarios
sudo cscli collections list
sudo cscli parsers list
sudo cscli scenarios list

# Confirm CAPI / community blocklist status
sudo cscli capi status

# Config validation without restarting the service
sudo crowdsec -c /etc/crowdsec/config.yaml -t -error
```

---

## Troubleshooting

**crowdsec fails to start, port already in use**

Almost certainly the Pi-hole 8080 conflict — see the Port conflict section above.

**crowdsec fails to start after editing `profiles.yaml`**

Run `sudo crowdsec -c /etc/crowdsec/config.yaml -t -error` and read the line number in the error
directly rather than guessing — a YAML indentation or comment mistake is the most common cause.

**Notification email never arrives**

```bash
sudo journalctl -u crowdsec -n 30 --no-pager
```

Check the SMTP credentials in `email.yaml` match `/etc/msmtprc` exactly, and that both
`notifications:` and `- email_default` are uncommented in `profiles.yaml`.

**Unsure whether a rule survived a UFW reload**

Don't assume either way — check directly:

```bash
sudo iptables -L INPUT -n --line-numbers | grep -i crowdsec
```

---

This is the last guide in the series. For the complete port map and file paths, see the
[Reference](reference.md).

**Sources:** [40] [41] [42] [43]
