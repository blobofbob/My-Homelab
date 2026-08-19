# 12 — Lynis

Lynis is a CIS-style security auditing tool — it inspects the running system against hundreds of
hardening checks and reports warnings and suggestions. It doesn't fix anything on its own; it's a
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

## Remediation — acting on findings

Lynis doesn't fix anything itself — every run just produces `/var/log/lynis-report.dat` for you to
triage. Not every warning or suggestion applies to every setup; some are genuinely worth fixing,
some are already covered by tools elsewhere in this project, and some are false positives specific
to a particular architecture. Worked example from a real triage pass on this Pi, since "what to
actually do with a Lynis report" is the harder half of using this tool.

### Triage first, fix second

Group findings into four buckets before touching anything:

- **Real gaps, worth fixing** — anything the rest of your stack doesn't already cover
- **Already covered elsewhere** — e.g. Apache DoS/WAF suggestions (`HTTP-6640`/`HTTP-6643`) if
  CrowdSec ([15](15-crowdsec.md)) already has the `apache2`/`http-cve` collections installed;
  external log host / process accounting suggestions if you're deliberately deferring that to
  separate hardware (Suricata, centralized logging)
- **False positives for this architecture** — Lynis doesn't know your setup. Two examples found
  here:
  - `PKGS-7388` ("can't find any security repository") — the repo was genuinely present
    (`grep -r security /etc/apt/sources.list.d/`), just in Debian's newer deb822 `.sources` format,
    which Lynis doesn't appear to parse the same way as classic one-line `sources.list` files.
  - `AUTH-9328` (umask suggestion, 022 vs 0002) — check what's actually running before assuming the
    louder default is safer:
    ```bash
    umask
    id YOUR_USERNAME
    grep "^UMASK\|^USERGROUPS_ENAB" /etc/login.defs
    ```
    If your primary group ID matches your user ID and no one else is in it (Debian's default
    "user private group" scheme), a group-writable umask isn't actually shared with anyone —
    0002 is fine as-is.
- **Not practical here** — e.g. separate partitions for `/home`/`/var` would need a reinstall on a
  single SD card

### Sysctl hardening (`KRNL-6000`)

Two commonly-flagged values, both safe defense-in-depth on a homelab box — neither is relied on by
Docker, Apache, Pi-hole, Tor, or Tailscale:

```bash
sudo tee /etc/sysctl.d/60-lynis-hardening.conf > /dev/null << 'EOF'
# Prevent unprivileged loading of TTY line disciplines via TIOCSETD ioctl —
# a known local privilege-escalation vector.
dev.tty.ldisc_autoload = 0

# Disallow opening FIFOs not owned by the caller in world-writable sticky
# directories (e.g. /tmp) — closes a symlink/hardlink attack class.
fs.protected_fifos = 2
EOF
sudo sysctl --system
```

Verify the *live* values, not just that the file exists — other `sysctl.d` files (Debian's own
defaults, RPi-specific config, Tailscale's) apply in the same pass, and whichever file sorts last
wins for a given key:

```bash
sysctl dev.tty.ldisc_autoload fs.protected_fifos
```

### Unused protocol modules

```bash
lsmod | grep -E "dccp|sctp|rds|tipc"
```

Confirm none are currently loaded first — blacklisting doesn't unload something already running.
Then:

```bash
sudo tee /etc/modprobe.d/blacklist-unused-protocols.conf > /dev/null << 'EOF'
install dccp /bin/false
install sctp /bin/false
install rds /bin/false
install tipc /bin/false
EOF
```

The `install MODULE /bin/false` form, not a bare `blacklist` line — `blacklist` alone only stops
*automatic* loading via aliases; it doesn't stop an explicit `modprobe MODULE` call. The fake-install
trick blocks both.

### Old package cleanup (`PKGS-7346`)

```bash
dpkg -l | grep '^rc' | awk '{print $2}'
```

Review the list before purging — on this Pi it was old kernel version configs plus
`netfilter-persistent`, the package behind `iptables-persistent`. Its presence here (removed, not
purged, not active) is just overdue cleanup, not a live conflict — but worth double-checking it's
genuinely inactive given this project's own documented incident of `iptables-persistent` silently
deleting UFW's rules ([11 — UFW](11-ufw.md)):

```bash
dpkg -l | grep '^rc' | awk '{print $2}' | xargs sudo apt purge -y
```

### Wiring auditd into CrowdSec (`ACCT-9630`)

If CrowdSec ([15](15-crowdsec.md)) is already installed, its auto-detected `auditd` collection
expects real `execve` events to parse — an empty ruleset (`sudo auditctl -l` showing "No rules")
means it's logging nothing.

Check the acquisition config is already pointed at the right log first — `cscli setup`'s
auto-detection usually handles this on install:

```bash
cat /etc/crowdsec/acquis.d/*.yaml 2>/dev/null | grep -A2 auditd
```

Then add the rule itself. On ARM (this Pi), be deliberate about which syscalls you audit — `b64`/
`b32` in `-F arch=` are architecture-independent shorthand (auditctl auto-detects the machine's
native word size, confirmed via the `auditctl` man page), so that part is safe. But several syscalls
common in x86-oriented CIS-benchmark rule templates (bare `chmod`, `creat`, `open`) don't exist on
arm64 at all — only the newer `*at` variants do — and will fail to load with "Syscall name unknown"
if copied wholesale. `execve` is universal across architectures, so a minimal rule avoids the
problem entirely rather than needing an ARM-specific rewrite of a bigger ruleset:

```bash
sudo tee /etc/audit/rules.d/10-crowdsec-execve.rules > /dev/null << 'EOF'
-a exit,always -F arch=b64 -S execve
EOF
sudo augenrules --load
```

Verify it loaded and is actually seeing events (a growing `backlog` count across repeated checks
confirms real activity, not just a registered rule):

```bash
sudo auditctl -l
```

### SSH hardening (`SSH-7408`)

Lynis's suggestions here are generic — apply the ones that don't depend on how you actually use SSH
without a second thought, but check your real usage before touching the rest:

| Setting | Depends on |
|---|---|
| `AllowTcpForwarding`, `AllowAgentForwarding` | Do you ever tunnel through this Pi or jump to another host from it? |
| `MaxSessions` | How many simultaneous SSH connections do you actually run? (Lynis suggests 2 — if that's your literal ceiling with zero slack, consider 3) |
| `Port` | Skip this one on a Tailscale-only setup — the real protection is UFW + Tailscale, not port obscurity. Changing it adds config complexity for no meaningful security gain here. |

Everything else (`ClientAliveCountMax`, `LogLevel`, `MaxAuthTries`, `TCPKeepAlive`,
`X11Forwarding` on a headless box) is safe to apply regardless:

```bash
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf > /dev/null << 'EOF'
AllowTcpForwarding no
AllowAgentForwarding no
ClientAliveCountMax 2
LogLevel VERBOSE
MaxAuthTries 3
MaxSessions 3
TCPKeepAlive no
X11Forwarding no
EOF
```

**Test before reloading — a syntax error here risks a lockout:**

```bash
sudo sshd -t
```

Silence means valid. Then apply live without dropping your current session:

```bash
sudo systemctl reload sshd
```

`reload`, not `restart` — re-reads config without ever closing the connection you're using to run
the command, so a mistake surfaces as "still connected, changes didn't apply" rather than "locked
out."

Confirm the running config actually picked up the changes (`sshd -T` reads live state, not just the
file on disk):

```bash
sudo sshd -T | grep -iE "allowtcpforwarding|allowagentforwarding|clientalivecountmax|loglevel|maxauthtries|maxsessions|tcpkeepalive|x11forwarding"
```

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

**Cron job silently doesn't mail anything**

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
