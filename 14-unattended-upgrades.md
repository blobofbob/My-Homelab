# 14 — unattended-upgrades

unattended-upgrades automatically installs package updates and, when needed, reboots. This guide
covers configuring it for **every** apt repository on this Pi — not just Debian's own — since by
default it only covers Debian's security/updates origins and silently ignores third-party repos
unless each is explicitly listed.

**Prerequisites:** [12 — Lynis](12-lynis.md) complete (notification layer).

---

## Pre-flight checks

Confirm the system timezone — the scheduled reboot time later in this guide is interpreted in
**local** system time, and the UK moves between GMT and BST, so it matters whether the system clock
is actually set to `Europe/London` or just `UTC`:

```bash
timedatectl
```

Check for a conflicting tool. `cron-apt` installed alongside `unattended-upgrades` is a known cause
of apt-lock contention and random failures:

```bash
dpkg -l | grep cron-apt
```

Install:

```bash
sudo apt install -y unattended-upgrades apt-listchanges
```

---

## Finding the real origin strings

Debian's default `Allowed-Origins`/`Origins-Pattern` config only matches Debian's own repos. A
third-party repo being present in `sources.list.d/` does **not** make unattended-upgrades consider
it — each one needs an explicit pattern entry, matched against the real metadata in its Release
file, not guessed:

```bash
apt-cache policy | grep -A1 -E "docker|tailscale|torproject|cisofy|raspberrypi"
```

This is worth doing on your own system rather than copying the values below verbatim — different
repos populate different fields. On this Pi:

| Repo | Codename tracks distro? | Matching field |
|---|---|---|
| Tor Project | Yes (`trixie`) | `codename=${distro_codename}` |
| Docker | No `codename=` field at all | `archive=${distro_codename}` |
| Raspberry Pi Foundation | No `codename=` field at all | `archive=stable` |
| Tailscale | No — pinned to `bullseye` | `codename=bullseye` (hardcoded) |
| CISOfy (Lynis) | No — pinned to `stable` | `codename=stable` (hardcoded) |

Some repos (Docker, Raspberry Pi Foundation here) populate `archive`/`suite` instead of `codename` —
using the wrong field means the pattern silently never matches, with no error.

Check the exact accepted field names and syntax directly from the shipped config before writing
anything, since the comments document it precisely:

```bash
grep -A20 "Origins-Pattern" /etc/apt/apt.conf.d/50unattended-upgrades | head -30
```

---

## Configuration

Written as a local override (`52unattended-upgrades-local`), leaving the package's own
`50unattended-upgrades` untouched — same reasoning as everywhere else in this project:

```bash
sudo tee /etc/apt/apt.conf.d/52unattended-upgrades-local > /dev/null << 'EOF'
#clear Unattended-Upgrade::Origins-Pattern;
Unattended-Upgrade::Origins-Pattern {
        "origin=Debian,codename=${distro_codename},label=Debian";
        "origin=Debian,codename=${distro_codename},label=Debian-Security";
        "origin=Debian,codename=${distro_codename}-security,label=Debian-Security";
        "origin=Debian,codename=${distro_codename}-updates";
        "origin=TorProject,codename=${distro_codename}";
        "origin=Docker,archive=${distro_codename}";
        "origin=Tailscale,codename=bullseye";
        "origin=CISOfy,codename=stable";
        "origin=Raspberry Pi Foundation,archive=stable";
};

Unattended-Upgrade::Mail "YOUR_RECIPIENT_EMAIL";
Unattended-Upgrade::MailReport "only-on-error";

Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";

Unattended-Upgrade::Remove-Unused-Dependencies "true";
EOF
```

- `#clear` replaces the package default's `Origins-Pattern` list rather than appending to it.
- `MailReport "only-on-error"`, not `"on-change"` — with every repo in scope, something upgrades
  most nights; `"on-change"` would mean a nightly email regardless of whether anything actually
  needs attention.
- `Remove-Unused-Dependencies "true"` — the package default is `false`. Without this, old kernel
  versions accumulate indefinitely; on a small SD card that adds up fast (worth an
  `sudo apt autoremove --dry-run` check the first time you enable this, to see what's already
  backlogged).

Add any repo you deliberately don't want auto-updated by simply leaving it out of the pattern list —
nothing else to configure.

Enable the periodic trigger:

```bash
sudo tee /etc/apt/apt.conf.d/20auto-upgrades > /dev/null << 'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
EOF
```

---

## Scheduling — this is systemd timers, not cron

Unlike Lynis and AIDE (which have no built-in scheduler), unattended-upgrades hooks into
`apt-daily.timer` and `apt-daily-upgrade.timer` — systemd timers shipped by the `apt` package
itself, not something this project sets up from scratch. Their defaults (~6am with hours of random
delay) are the wrong window here; retiming via drop-in overrides rather than editing the timer units
directly:

```bash
sudo mkdir -p /etc/systemd/system/apt-daily.timer.d
sudo tee /etc/systemd/system/apt-daily.timer.d/override.conf > /dev/null << 'EOF'
[Timer]
OnCalendar=
OnCalendar=*-*-* 01:00:00
RandomizedDelaySec=0
EOF
sudo mkdir -p /etc/systemd/system/apt-daily-upgrade.timer.d
sudo tee /etc/systemd/system/apt-daily-upgrade.timer.d/override.conf > /dev/null << 'EOF'
[Timer]
OnCalendar=
OnCalendar=*-*-* 01:30:00
RandomizedDelaySec=0
EOF
sudo systemctl daemon-reload
sudo systemctl restart apt-daily.timer apt-daily-upgrade.timer
```

01:00 (list update) → 01:30 (install) → AIDE's 02:00 check ([13](13-aide.md)) → 03:00 reboot if one's
needed → Lynis's 04:00 Sunday audit ([12](12-lynis.md)). Nothing overlaps.

The blank `OnCalendar=` line before the real value is required — it clears the package-default
schedule before setting the new one; without it, both schedules apply.

---

## Verifying

Dry run before trusting any of this live:

```bash
sudo unattended-upgrade --dry-run --debug 2>&1 | grep -iE "not allowed|pkgs that look like|Packages that will be upgraded"
```

Confirm packages from third-party repos actually appear under "Packages that will be upgraded" —
not just Debian's own. Anything still showing under "not allowed" has a pattern that doesn't match;
recheck its real origin fields with `apt-cache policy`.

Confirm the retimed timers actually took effect:

```bash
systemctl list-timers apt-daily.timer apt-daily-upgrade.timer
```

Should show next-trigger times around 01:00 and 01:30, not the ~6am default.

---

## Useful commands

```bash
# Dry run with full debug output
sudo unattended-upgrade --dry-run --debug

# Real run, invoked manually
sudo unattended-upgrade

# Check what autoremove would clean up, without removing anything yet
sudo apt autoremove --dry-run

# View recent activity
sudo tail -30 /var/log/unattended-upgrades/unattended-upgrades.log
```

---

## Troubleshooting

**A third-party package never gets upgraded**

Its `Origins-Pattern` entry likely targets the wrong field. Re-check with
`apt-cache policy | grep -A1 reponame` — some repos use `archive=`/`suite=` instead of `codename=`,
and using the wrong one fails silently.

**Reboot happened at the wrong time, or not at all**

Check `timedatectl` — `Automatic-Reboot-Time` is interpreted in system local time. If the system
clock is UTC rather than the intended timezone, "03:00" means something different than expected.

**AIDE reports a huge pile of changes the morning after this runs**

Expected if a kernel or many packages updated — see the scope-scheduling note in
[13 — AIDE](13-aide.md). Confirm via `sudo tail -30 /var/log/unattended-upgrades/unattended-upgrades.log`
what actually installed before assuming anything is wrong.

---

**Next:** [15 — CrowdSec](15-crowdsec.md)

**Sources:** [37] [38] [39]
