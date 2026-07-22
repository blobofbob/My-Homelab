# 11 — UFW

UFW (Uncomplicated Firewall) is a frontend for iptables that enforces a deny-by-default policy on the Pi. Set up last, after every Docker stack is already running.

**Prerequisites:** [08 — Docker](08-docker.md) complete, and all three Docker stacks running — [09 — OpenClaw + n8n](09-openclaw-n8n.md) and [10 — Vaultwarden](10-vaultwarden.md).

---

## How Docker and UFW coexist

Docker manages its own iptables chains for container networking, independently of UFW. Containers stay off the local network because they're bound to the Tailscale IP, not `0.0.0.0` — not because of anything UFW does.

> **⚠ Enabling or reloading UFW can silently break container internet access, even when every rule looks correct.** Confirmed on this system: `iptables -L DOCKER-FORWARD` showed Docker's own bridge `ACCEPT` rules present and correct, yet containers still got `ENETUNREACH` on every outbound connection. **A full container restart fixes it**

`iptables-persistent` must never be installed alongside UFW — the two manage iptables independently, and installing it automatically deletes UFW.

---

## Configuring UFW

**1 — Set default policies:**

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed
```

**2 — Tailscale — allow all tailnet traffic:**

```bash
sudo ufw allow in on tailscale0
```

This single rule covers SSH (port 22), the Pi-hole web admin (via Tailscale Serve), and every other tailnet-only service. No per-port rules are needed for tailnet services.

**3 — Public services:**

```bash
sudo ufw allow 80/tcp    # Apache HTTP
sudo ufw allow 443/tcp   # Apache HTTPS
sudo ufw allow 9001/tcp  # Tor relay (ORPort)
```

**4 — Pi-hole DNS — local subnet only:**

```bash
sudo ufw allow from 192.168.11.0/24 to any port 53
```

**5 — Docker containers → Pi INPUT:**

```bash
sudo ufw allow in from 172.16.0.0/12
```

Docker assigns bridge network addresses from the `172.16.0.0/12` range. This rule lets containers reach host services (Pi-hole DNS on port 53) without naming any specific bridge interface, and stays correct if networks are recreated. It only covers container → Pi traffic — not container → internet, which Docker's own chains handle (see the warning above).

**6 — Minecraft (optional):**

```bash
sudo ufw allow 25565/tcp  # Minecraft Java
sudo ufw allow 19132/udp  # Bedrock (GeyserMC)
sudo ufw allow 19133/udp  # Bedrock (secondary)
```

Only add these if running the Minecraft server publicly.

**7 — Enable UFW, then restart every container stack:**

```bash
sudo ufw enable
docker compose -f ~/ai-stack/docker-compose.yml restart
docker compose -f ~/vaultwarden/docker-compose.yml restart
```

UFW will warn that enabling it may interrupt existing SSH connections — not a risk here since SSH is over Tailscale, covered by the rule added in Step 2. The container restart is what prevents the silent forwarding failure described above; don't skip it.

**8 — Verify:**

```bash
sudo ufw status verbose
```

Expected:

```
Status: active

To                         Action      From
--                         ------      ----
Anywhere on tailscale0     ALLOW       Anywhere
25565/tcp                  ALLOW       Anywhere
19132/udp                  ALLOW       Anywhere
19133/udp                  ALLOW       Anywhere
9001/tcp                   ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
53                         ALLOW       192.168.11.0/24
Anywhere                   ALLOW       172.16.0.0/12
Anywhere (v6) on tailscale0 ALLOW       Anywhere (v6)
25565/tcp (v6)             ALLOW       Anywhere (v6)
19132/udp (v6)             ALLOW       Anywhere (v6)
19133/udp (v6)             ALLOW       Anywhere (v6)
9001/tcp (v6)              ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
443/tcp (v6)               ALLOW       Anywhere (v6)
```

Then confirm containers can reach the internet:

```bash
docker compose -f ~/ai-stack/docker-compose.yml exec openclaw \
  curl -s https://example.com -o /dev/null -w "%{http_code}"
```

Expected: `200`.

---

## Useful commands

```bash
# View active rules
sudo ufw status verbose

# Add a rule
sudo ufw allow PORT/PROTOCOL

# Delete a rule
sudo ufw delete allow PORT/PROTOCOL

# Disable temporarily (e.g. for debugging)
sudo ufw disable

# Re-enable
sudo ufw enable

# Reload after manual changes to before.rules — always restart containers after this too
sudo ufw reload
```

---

## Troubleshooting

**SSH connection drops after enabling UFW**

The `tailscale0` rule wasn't added before enabling. Recover through the Pi's local console or local network:

```bash
sudo ufw allow in on tailscale0
sudo ufw reload
```

---

**Containers can't reach the internet after enabling or reloading UFW**

Restart them — this is the confirmed fix, not just a thing to try:

```bash
docker compose -f ~/ai-stack/docker-compose.yml restart
docker compose -f ~/vaultwarden/docker-compose.yml restart
```

If that doesn't resolve it, a bridge may genuinely have no `ACCEPT` rule rather than a stale one — a different, rarer problem. Check:

```bash
sudo iptables -L DOCKER-FORWARD -v
```

If a bridge is missing an `ACCEPT` line entirely (not just showing a zero packet count), add one to `/etc/ufw/before.rules`, inside the `*filter` block before `COMMIT`:

```bash
ip link show | grep br-
sudo nano /etc/ufw/before.rules
```

```
-A ufw-before-forward -i br-XXXXXXXXXXXX -j ACCEPT
-A ufw-before-forward -o br-XXXXXXXXXXXX -j ACCEPT
```

```bash
sudo ufw reload
docker compose -f ~/ai-stack/docker-compose.yml restart
```

Bridge names are regenerated by Docker on network recreation — re-check `ip link show | grep br-` after any full stack rebuild and update `before.rules` if they changed.

---

**Pi-hole DNS stops working after a Docker restart**

```bash
sudo systemctl restart pihole-FTL
```

---

*This is the last guide in the series. For the complete port map and file paths, see the [Reference](../reference.md).*