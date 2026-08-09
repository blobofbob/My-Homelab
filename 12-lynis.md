# 12 — Lynis

Lynis is a CIS-style security auditing tool — it inspects the running system against hundreds of
hardening checks, reports, warnings and suggestions. It doesn't fix anything on its own; it's a
periodic health check, run here weekly.

This guide also sets up the **notification layer** — a lightweight mail relay that Lynis, AIDE
([13](13-aide.md)), unattended-upgrades ([14](14-unattended-upgrades.md)), and CrowdSec
([15](15-crowdsec.md)) all send through. It only needs setting up once, here, and every later guide
just points at it.

**Prerequisites:** [01 — Initial Setup](01-initial-setup.md) complete.

---

## Notification layer — msmtp

msmtp is a send-only mail relay — no daemon, no inbox, nothing to receive. It fits a system that
only ever needs to *send* alerts, and it's a much smaller footprint than a full MTA like Postfix on
an SD card that's already tight on space.

**1 — Generate a Gmail app password.**

Go to `https://myaccount.google.com/apppasswords`, name it something like `pi-alerts`, and copy the
16-character password — it's shown only once.

**2 — Install:**

```bash
sudo apt update
sudo apt install -y msmtp msmtp-mta bsd-mailx ca-certificates
```

During install, apt may show an AppArmor confinement prompt for msmtp — **decline it**. msmtp's
upstream author recommends disabling it if you hit "permission denied" errors, and Debian ships it
disabled by default since 1.8.15 for the same reason: the profile has documented bugs around a
custom `logfile` directive and system-wide (`/etc/msmtprc`, not `~/.msmtprc`) configs — both of
which this setup uses.

**3 — Create the config:**

```bash
sudo tee /etc/msmtprc > /dev/null << 'EOF'
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        /var/log/msmtp.log

account        pi-alerts
host           smtp.gmail.com
port           587
from           YOUR_GMAIL_ADDRESS@gmail.com
user           YOUR_GMAIL_ADDRESS@gmail.com
password       YOUR_APP_PASSWORD

account default : pi-alerts
EOF
```

**4 — Lock down permissions.** It holds a plaintext credential — same reasoning as any other
service's `.env` file in this project:

```bash
sudo chown root:root /etc/msmtprc
sudo chmod 600 /etc/msmtprc
```

> **This file is now root-only readable by design.** Every consumer of it (Lynis, AIDE,
> unattended-upgrades, CrowdSec — all run as root via cron or systemd) can read it fine. If you ever
> test it manually, use `sudo` — testing as your own login user will fail with `no configuration
> file available`, which looks like a broken config but is actually the permission model working
> as intended.

**5 — Test it:**

```bash
echo "msmtp test from raspberrypi" | sudo mail -s "Pi notification test" YOUR_RECIPIENT_EMAIL
```

If it doesn't arrive, check `sudo tail -5 /var/log/msmtp.log` for the actual SMTP error before
guessing further.

**6 — Route root's cron mail through it.** This one line means every future cron job under root
mails you automatically, without needing to pipe through `mail -s` individually:

```bash
(sudo crontab -l 2>/dev/null; echo "MAILTO=YOUR_RECIPIENT_EMAIL") | sudo crontab -
```

---

## Installing Lynis

Debian's own repo version is badly outdated. Install from CISOfy's own repo instead — same
signed-keyring pattern as the Tor repo in [05 — Tor](05-tor.md):

```bash
curl -fsSL https://packages.cisofy.com/keys/cisofy-software-public.key | sudo gpg --dearmor -o /usr/share/keyrings/cisofy-lynis-keyring.gpg
echo "deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/cisofy-lynis-keyring.gpg] https://packages.cisofy.com/community/lynis/deb/ stable main" | sudo tee /etc/apt/sources.list.d/cisofy-lynis.list
sudo apt update
sudo apt install -y lynis
```

---

## Running a baseline audit

```bash
sudo lynis audit system
```

This takes a few minutes. At the end, review the warnings and suggestions — some will be genuinely
worth acting on, others (like sshd listening on `0.0.0.0`) may already be mitigated elsewhere (UFW
restricting it to the tailnet, in that example) and are false positives for this specific setup.
Lynis doesn't know your architecture; use judgement, not a checklist mentality.

To review findings after any run, without wading through the full interactive output:

```bash
sudo grep -E "^warning|^suggestion" /var/log/lynis-report.dat
```

(`lynis show warnings` is **not** a real subcommand, despite appearing in some third-party
cheatsheets — this `grep` against the report file is the actual method.)

---

## Scheduling

```bash
(sudo crontab -l 2>/dev/null; echo "0 4 * * 0 $(command -v lynis) audit system --cronjob") | sudo crontab -
```

Sunday 04:00 — weekly (daily is unnecessary CPU/disk load for a Pi, and this scheduling note applies
across all four hardening guides): after unattended-upgrades' 03:00 reboot window
([14](14-unattended-upgrades.md)), low enough frequency to not matter if it overlaps with anything
else. `--cronjob` suppresses interactive prompts and color codes, meant for exactly this use case.

Lynis's default report/log locations (`/var/log/lynis-report.dat`, `/var/log/lynis.log`) overwrite
on each run rather than accumulating dated files — disk usage stays flat, and the weekly email
itself is the historical record.

---

## Useful commands

```bash
# Manual audit
sudo lynis audit system

# Review current warnings/suggestions
sudo grep -E "^warning|^suggestion" /var/log/lynis-report.dat

# Check installed version
lynis show version

# View root's crontab
sudo crontab -l
```

---

## Troubleshooting

**Cron job doesn't mail anything**

Prove cron's own mail pipeline works, independent of Lynis, with a one-minute throwaway job:

```bash
(sudo crontab -l 2>/dev/null; echo "* * * * * echo cron-mailto-test") | sudo crontab -
```

Wait for the email, then remove it:

```bash
sudo crontab -l | grep -v "cron-mailto-test" | sudo crontab -
```

If that arrives but the real Lynis job's mail doesn't, the issue is Lynis-specific, not the
notification layer — check `sudo crontab -l` for a typo in the actual command line.

---

**Duplicate crontab entries**

If a line ends up added twice (from re-running an append command), dedupe without touching anything
else:

```bash
sudo crontab -l | awk '!seen[$0]++' | sudo crontab -
```

---

**Next:** [13 — AIDE](13-aide.md)

**Sources:** [32] [33] [34] [35]
