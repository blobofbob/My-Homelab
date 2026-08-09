# 13 — AIDE

AIDE (Advanced Intrusion Detection Environment) is a file integrity monitor — it builds a database
of checksums and metadata for the filesystem, then compares against it on every check, flagging
anything that changed. It doesn't prevent anything; it's a tripwire.

**Prerequisites:** [12 — Lynis](12-lynis.md) complete (notification layer).

---

## How Debian's AIDE package actually works

This matters before writing any config: Debian's `aide-common` package does **not** ship a curated
include-list. It ships a watch-everything-under-`/`-by-default ruleset (a catch-all rule at the very
end of the config), with hundreds of small exclusion fragments layered on top for known-noisy paths
belonging to Debian-packaged services. Anything installed from a **third-party repo** — Docker,
Pi-hole, Tor, Tailscale, CrowdSec, none of which are Debian packages with their own AIDE integration
— has no exclusion fragment waiting for it, and gets fully hashed by the catch-all unless you write
one yourself. On a disk-constrained SD card, that means Docker's entire image/container storage
layer alone can balloon the database by gigabytes and bury real signal in daily noise.

---

## Installing

```bash
sudo apt install -y aide aide-common
```

`aide-common` provides `aideinit` (the correct way to build the database — see below), the
`aide.conf.d/` fragment system, and an automatic daily cron job that we're about to replace.

**Disable the automatic daily job** in favor of our own, precisely-timed cron entry:

```bash
sudo sed -i 's/^#CRON_DAILY_RUN=yes/CRON_DAILY_RUN=no/' /etc/default/aide
```

> The package's own daily job (`/etc/cron.daily/aide`) has an independent mail recipient set via
> `MAILTO=` in `/etc/default/aide` — a **separate mechanism** from root's personal crontab `MAILTO`
> used everywhere else in this project. Disabling it and rolling our own keeps everything on one
> notification path.

---

## The negative-rule syntax — read this before writing any exclusion

Per `aide.conf(5)`: a negative (exclude) rule written with a trailing type letter —
`!/some/path$ d` — is a **restricted** negative rule. Restricted rules only exclude entries that
literally match the regex; children of an excluded directory are still recursed into and picked up
by whatever broader rule applies to them (eventually the catch-all). Adding a type letter doesn't
"restrict to directories" the way it reads intuitively — it punches a hole straight through the
exclusion for everything inside.

The correct form for excluding a directory's entire contents is **two plain, unrestricted lines**:

```
!/some/path$
!/some/path/.*$
```

The first line excludes the directory entry itself; the second excludes everything beneath it
explicitly. (A bare `!/some/path$` alone is not reliably sufficient either — verified empirically on
this system; the explicit `.*$` wildcard is what actually stops the leak.) For plain files with no
children (a single log or database file), one line is enough.

**To verify any exclusion actually works without a full 20–40 minute reinit**, use AIDE's own
rule-tracing mode, scoped to just the path in question:

```bash
sudo aide --config /etc/aide/aide.conf --init --limit "^/path/to/test" --log-level=rule > /tmp/test.log 2>&1
grep -c "do NOT add '/path/to/test/" /tmp/test.log
rm -f /tmp/test.log
```

A count roughly matching the number of files actually in that path confirms every one of them is
being correctly rejected — direct evidence from AIDE's own decision log, not an inference from
report counts.

---

## Scope — exclusion fragments

Everything below is Debian's default config **plus** explicit exclusions for the third-party
services on this Pi. Each fragment goes in `/etc/aide/aide.conf.d/`, named to sort before the final
catch-all (`99_aide_root`) so filename order doesn't matter for these — negative rules apply
regardless of position, but keeping the numbering consistent with Debian's own convention keeps the
directory readable.

**Docker's storage backend** — image layers, container writable layers, and the two homelab data
directories that live inside container bind mounts (Vaultwarden's database, OpenClaw's workspace) —
all high-churn, no security value in diffing:

```bash
sudo tee /etc/aide/aide.conf.d/31_aide_docker > /dev/null << 'EOF'
!/var/lib/docker$
!/var/lib/docker/.*$
!/var/lib/containerd$
!/var/lib/containerd/.*$
!/home/[^/]+/ai-stack/openclaw-data$
!/home/[^/]+/ai-stack/openclaw-data/.*$
!/home/[^/]+/vaultwarden/vaultwarden-data$
!/home/[^/]+/vaultwarden/vaultwarden-data/.*$
EOF
```

**Pi-hole** — rotating config/gravity backups, SQLite WAL/SHM files, list cache, migration
artifacts. `dnsmasq.conf`, `adlists.list`, `pihole.toml`, and `hosts` stay watched — those are real config:

```bash
sudo tee /etc/aide/aide.conf.d/31_aide_pihole > /dev/null << 'EOF'
!/etc/pihole/config_backups$
!/etc/pihole/config_backups/.*$
!/etc/pihole/gravity_backups$
!/etc/pihole/gravity_backups/.*$
!/etc/pihole/gravity\.db$
!/etc/pihole/gravity_old\.db$
!/etc/pihole/dhcp\.leases$
!/etc/pihole/cli_pw$
!/etc/pihole/listsCache$
!/etc/pihole/listsCache/.*$
!/etc/pihole/pihole-FTL\.db$
!/etc/pihole/pihole-FTL\.db-wal$
!/etc/pihole/pihole-FTL\.db-shm$
!/etc/pihole/macvendor\.db$
!/etc/pihole/install\.log$
!/etc/pihole/migration_backup$
!/etc/pihole/migration_backup/.*$
!/etc/pihole/migration_backup_v6$
!/etc/pihole/migration_backup_v6/.*$
EOF
```

**Tor** — relay consensus/descriptor caches and the diff-cache directory, all operational churn from
normal relay operation. `/var/lib/tor` itself keeps a positive `VarDir` rule (tracks permissions and
existence, ignores mtime churn from excluded children) rather than a blanket exclude — and critically,
**the hidden service directory is never excluded**. Its private key should almost never change; a
silent change there would mean the .onion address was compromised or swapped, which is exactly what
this tool exists to catch:

```bash
sudo tee /etc/aide/aide.conf.d/31_aide_tor > /dev/null << 'EOF'
/var/lib/tor$ d VarDir
!/var/lib/tor/cached-consensus$
!/var/lib/tor/cached-descriptors$
!/var/lib/tor/cached-descriptors\.new$
!/var/lib/tor/cached-microdesc-consensus$
!/var/lib/tor/cached-microdescs$
!/var/lib/tor/cached-microdescs\.new$
!/var/lib/tor/diff-cache$
!/var/lib/tor/diff-cache/.*$
!/var/lib/tor/lock$
!/var/lib/tor/state$
EOF
```

**Tailscale** — netmap cache and rotated daemon logs:

```bash
sudo tee /etc/aide/aide.conf.d/31_aide_tailscale > /dev/null << 'EOF'
!/var/lib/tailscale/profile-data/[^/]+/netmap-cache$
!/var/lib/tailscale/profile-data/[^/]+/netmap-cache/.*$
!/var/lib/tailscale/tailscaled\.log1\.txt$
!/var/lib/tailscale/tailscaled\.log2\.txt$
EOF
```

**`/run`** — tmpfs, fully wiped every boot. Docker, containerd, cloud-init, Tor, and Tailscale all
leave runtime sockets/state here; none of it can persist a compromise across a reboot, so it's
excluded wholesale rather than chased path-by-path:

```bash
sudo tee /etc/aide/aide.conf.d/31_aide_run_thirdparty > /dev/null << 'EOF'
!/run$
!/run/.*$
EOF
```

**Kernel modules** — rewritten wholesale on every kernel update (already logged separately by
unattended-upgrades, [14](14-unattended-upgrades.md)):

```bash
sudo tee /etc/aide/aide.conf.d/33_aide_kernel_modules > /dev/null << 'EOF'
!/usr/lib/modules$
!/usr/lib/modules/.*$
EOF
```

**CrowdSec's own operational files** — database WAL/SHM and its own logs ([15](15-crowdsec.md)):

```bash
sudo tee /etc/aide/aide.conf.d/31_aide_crowdsec > /dev/null << 'EOF'
!/var/lib/crowdsec/data/crowdsec\.db-shm$
!/var/lib/crowdsec/data/crowdsec\.db-wal$
!/var/log/crowdsec-firewall-bouncer\.log$
!/var/log/crowdsec\.log$
!/var/log/crowdsec_api\.log$
EOF
```

**Misc recurring noise** — Pi-hole's shared-memory files, cert-renewal logs, AIDE's own init log,
tmux sockets:

```bash
sudo tee /etc/aide/aide.conf.d/31_aide_misc_churn > /dev/null << 'EOF'
!/dev/shm/FTL-.*$
!/var/log/aide/aideinit\.log$
!/var/log/audit/audit\.log$
!/var/log/pihole/pihole\.log$
!/var/log/letsencrypt/letsencrypt\.log$
!/tmp/tmux-[0-9]+$
!/tmp/tmux-[0-9]+/.*$
!/var/lib/letsencrypt/http_challenges$
!/var/lib/systemd/timers/stamp-certbot\.timer$
EOF
```

**Validate everything before initializing** (fast syntax check, no filesystem scan):

```bash
sudo aide --config /etc/aide/aide.conf --config-check
```

---

## Reporting

By default AIDE only writes its report to stdout — fine for cron's `MAILTO`, but leaves no
persistent, grep-able record. AIDE supports multiple simultaneous `report_url` destinations
natively:

```bash
sudo mkdir -p /var/log/aide
sudo tee /etc/aide/aide.conf.d/32_aide_reporting > /dev/null << 'EOF'
report_url=stdout
report_url=file:/var/log/aide/aide.log
EOF
```

Add rotation — otherwise this log grows forever:

```bash
sudo tee /etc/logrotate.d/aide-check > /dev/null << 'EOF'
/var/log/aide/aide.log {
    weekly
    rotate 6
    compress
    delaycompress
    missingok
    notifempty
    create 0640 _aide adm
}
EOF
```

> **Ownership matters here.** `aideinit` runs the actual scan as an unprivileged `_aide` system
> user, deliberately — a full-filesystem-hashing process is a meaningful attack surface if AIDE
> itself were ever compromised, so it doesn't need root to do its job. If `/var/log/aide/aide.log`
> ever ends up `root`-owned (e.g. from a manual `sudo aide --check` run, which executes as root with
> no privilege-dropping), `aideinit` will fail with a permission error trying to write to it. Fix:
> `sudo chown _aide:adm /var/log/aide/aide.log`. The logrotate config above prevents this from
> recurring on every rotation.

---

## Initializing the database

```bash
df -h /
```

Check free space first — AIDE hashes a large portion of the filesystem. Then, wrapped in `tmux`
since this can take 20–40 minutes and an SSH disconnect would otherwise kill it mid-run:

```bash
tmux new-session -d -s aideinit "sudo aideinit -y -f"
```

`-y -f` auto-confirm the overwrite prompts — required for a detached session, since there's nobody
there to answer them interactively. Reattach anytime with:

```bash
tmux attach-session -t aideinit
```

Confirm success:

```bash
sudo tail -5 /var/log/aide/aideinit.log
```

Looking for `AIDE --init return code 0`.

---

## Scheduling

```bash
(sudo crontab -l 2>/dev/null; echo "0 2 * * * /usr/bin/aide --config /etc/aide/aide.conf --check") | sudo crontab -
```

02:00 daily — before unattended-upgrades' 03:00 reboot window ([14](14-unattended-upgrades.md)),
after any 01:00–01:30 patch installation. This means legitimate overnight patches show up
as "changed" in that night's report — expected, and useful confirmation the tool is actually
working, not noise to suppress.

Verify a manual run against the database:

```bash
sudo aide --config /etc/aide/aide.conf --check
```

`--config` is required explicitly — AIDE's compiled-in default config path is `/etc/aide.conf`
(no subdirectory); Debian relocates it to `/etc/aide/aide.conf` specifically to support the
`aide.conf.d/` fragment system, and the raw binary has no way to know that on its own. Only the
`aideinit`/`aide-common` wrapper scripts know to pass it automatically.

---

## Useful commands

```bash
# Manual check against the current database
sudo aide --config /etc/aide/aide.conf --check

# Config syntax check (fast, no scan)
sudo aide --config /etc/aide/aide.conf --config-check

# Fast, scoped verification of one exclusion rule
sudo aide --config /etc/aide/aide.conf --init --limit "^/path" --log-level=rule

# Rebuild the database after a legitimate, expected change
sudo aideinit -y -f

# Pull just the summary line from the last report
sudo grep -A3 "^Summary:" /var/log/aide/aide.log
```

---

## Troubleshooting

**`ERROR: missing configuration`**

Missing `--config /etc/aide/aide.conf` — see the scheduling section above.

**Report is enormous for what should be a small change**

`report_level=changed_attributes` (the default) prints full per-attribute detail for every changed
file — expected verbosity, not a bug, when many files genuinely changed (a kernel update, for
example). Pull just the summary counts first:

```bash
sudo grep -A3 "^Summary:" /var/log/aide/aide.log
```

Then find the real section boundaries before extracting further, rather than guessing at the report
format:

```bash
grep -n "^Changed entries:" /var/log/aide/aide.log
```

**An exclusion doesn't seem to be working**

Don't wait for a full reinit to find out — use the `--limit` + `--log-level=rule` method from
earlier in this guide. It shows AIDE's actual per-file decision directly.

**`aideinit` fails with a permission error writing to the log**

See the ownership note in the Reporting section above.

---

**Next:** [14 — unattended-upgrades](14-unattended-upgrades.md)

**Sources:** [36]
