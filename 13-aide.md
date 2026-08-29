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

> **A second use for this trace, beyond confirming exclusions: confirming inclusions.** The same
> `--limit`/`--log-level=rule` method works in reverse — pointed at a path you want to be *sure* is
> still watched (not excluded by an overly broad regex), it prints `ADD` verdicts instead of
> `do NOT add` ones. When writing an exclusion that only *partially* matches a directory's contents
> (e.g. excluding dpkg's bookkeeping files but not its executable maintainer scripts), trace both
> the excluded and the retained subset separately and confirm each gets the verdict you expect.
> Getting the count/verdict split right here is better than discovering a security-relevant file
> has been excluded from the database.

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
!/var/lib/tor/cached-certs$
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

> **`/var/lib/tor/keys/` (the relay's own signing keypair) is deliberately *not* excluded here,
> distinct from `/var/lib/tor/hidden_service/` (the .onion identity key) — and the two are easy to
> conflate in a moment of alarm.** Tor relays rotate their medium-term
> `ed25519_signing_cert`/`ed25519_signing_secret_key` periodically as normal operation; seeing these
> flagged as changed in a report is expected and not on its own a compromise signal. The hidden
> service's `hs_ed25519_secret_key` is a *different* key with a *different* rotation expectation —
> it should essentially never change. When a report shows Tor-related changes, check which directory
> they're actually under before reacting: confirm the hidden service address itself is unaffected
> with `sudo cat /var/lib/tor/hidden_service/hostname` against the value in
> [reference.md](../reference.md), rather than assuming either "it's just Tor, ignore it" or
> "any Tor key change is an incident."

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

**CrowdSec's own operational files** — database WAL/SHM and its own logs ([15](15-crowdsec.md)).
Note what is *not* here: `crowdsec.db` itself and the Hub's scenario/pattern data
(`/var/lib/crowdsec/data/*.txt`, `*.json`) are deliberately left watched, not excluded — see the
note below the fragment:

```bash
sudo tee /etc/aide/aide.conf.d/31_aide_crowdsec > /dev/null << 'EOF'
!/var/lib/crowdsec/data/crowdsec\.db-shm$
!/var/lib/crowdsec/data/crowdsec\.db-wal$
!/var/log/crowdsec-firewall-bouncer\.log$
!/var/log/crowdsec\.log$
!/var/log/crowdsec_api\.log$
EOF
```

> **Why `crowdsec.db` and the Hub data files stay watched, unlike most other services' data
> directories in this guide.** These looked, at first glance, like the same category of
> "high-churn, no security value" data as Pi-hole's gravity DB or Docker's storage layer — but
> they're not. `crowdsec.db` holds live ban/decision state; the Hub `.txt`/`.json` files
> (`sqli_probe_patterns.txt`, `backdoors.txt`, etc.) are the actual detection signatures CrowdSec
> matches traffic against. Root-level tampering with either — quietly stripping a pattern from a
> detection file, for instance — would silently degrade CrowdSec without touching any config file
> this guide already tracks elsewhere. The disk-churn argument that justifies excluding e.g.
> Pi-hole's gravity backups doesn't apply the same way here: these files change rarely (only on
> deliberate `cscli` operations, not automatically — see the note on the Hub-update timer below),
> so watching them costs little and closes a real gap.

---

## A different kind of noise: randomized names that regenerate on every restart

Everything excluded so far is *high-churn content at a fixed path*. There's a second, distinct
pattern worth knowing to recognize on sight: a file or directory whose **name itself** is a fresh
random string every time a service restarts. These show up in a report as one entry Added and a
different entry Removed in the same run — never a clean "changed" — because the old name and the
new name genuinely are different database entries. Excluding the fixed path doesn't help here, since
there is no fixed path; the exclusion has to match the *pattern* the name follows instead.

Two examples found running this exact setup:

**Tor's systemd `PrivateTmp` sandboxing** — systemd gives the `tor@default` service its own private
`/tmp` and `/var/tmp` namespace, named `systemd-private-<32-hex-chars>-tor@default.service-<random>`.
A fresh hex string gets generated on every service restart:

```bash
sudo tee /etc/aide/aide.conf.d/31_aide_tor_privatetmp > /dev/null << 'EOF'
!/tmp/systemd-private-[0-9a-f]+-tor@default\.service-[[:alnum:]]+$
!/tmp/systemd-private-[0-9a-f]+-tor@default\.service-[[:alnum:]]+/.*$
!/var/tmp/systemd-private-[0-9a-f]+-tor@default\.service-[[:alnum:]]+$
!/var/tmp/systemd-private-[0-9a-f]+-tor@default\.service-[[:alnum:]]+/.*$
EOF
```

**CrowdSec's notification plugin socket** — each notification plugin (our `notification-email`,
[15](15-crowdsec.md)) runs as a subprocess talking to the main `crowdsec` engine over a Unix socket
at `/tmp/plugin<random digits>`, regenerated every time the `crowdsec` service restarts:

```bash
sudo tee /etc/aide/aide.conf.d/31_aide_crowdsec_notif_socket > /dev/null << 'EOF'
!/tmp/plugin[0-9]+$
EOF
```

Confirmed via `sudo lsof /tmp/pluginNNNNNN` before writing the rule — owned by CrowdSec's
notification process (user `nobody`), not something unrelated squatting on a similar-looking name.
Any other systemd-`PrivateTmp`-confined service, or any other tool with a plugin/subprocess
architecture, is likely to show the same Added+Removed churn pattern under a different name — the
fix is always the same shape: identify the fixed part of the name, wildcard the random part.

If a newly-added exclusion like this still shows an Added+Removed pair on the *next* check after
being applied, that's not necessarily the rule failing — it can be the database catching up: the
*old* randomly-named entry drops out (shows as Removed) the first time it's no longer tracked, while
a genuinely new one appearing afterward would mean the rule isn't matching newly-created names. The
`--limit` + `--log-level=rule` trace from earlier in this guide is the way to tell these apart for
certain rather than guess from the report alone.

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

**Package management churn** — apt/dpkg/man caches, and dpkg's per-package bookkeeping. All fully
reconstructable from files that stay watched elsewhere (the caches are derived from the real
packages/libraries; the dpkg bookkeeping is metadata, not code). **Deliberately excludes only the
non-executable bookkeeping files** — `.list`, `.md5sums`, `.shlibs`, `.symbols`, `.triggers`,
`.conffiles`. The maintainer scripts (`.postinst`, `.postrm`, `.preinst`, `.prerm`) are root-executed
code and stay watched; a change there with no matching entry in `apt`'s own history log
(`/var/log/apt/history.log`) would be a real supply-chain-tamper signal worth investigating:

```bash
sudo tee /etc/aide/aide.conf.d/31_aide_pkg_mgmt_cache > /dev/null << 'EOF'
!/var/cache/apt/pkgcache\.bin$
!/var/cache/apt/srcpkgcache\.bin$
!/var/cache/apt/archives$
!/var/cache/apt/archives/.*$
!/var/cache/man$
!/var/cache/man/.*$
!/var/cache/ldconfig/aux-cache$
!/etc/ld\.so\.cache$
!/var/backups/dpkg\.arch\.[0-9]+\.gz$
!/var/lib/dpkg/info/.*\.list$
!/var/lib/dpkg/info/.*\.md5sums$
!/var/lib/dpkg/info/.*\.shlibs$
!/var/lib/dpkg/info/.*\.symbols$
!/var/lib/dpkg/info/.*\.triggers$
!/var/lib/dpkg/info/.*\.conffiles$
!/var/lib/apt/periodic/.*$
!/var/lib/unattended-upgrades/kept-back$
!/var/lib/systemd/timers/stamp-.*$
!/var/backups/dpkg\.arch$
!/var/backups/dpkg\.arch\.[0-9]+(\.gz)?$
EOF
```

**Live logs only** — the *actively-growing* log file for each of these, never the rotated/archived
copies. This is the same convention already used above for `pihole.log`/`audit.log`/
`letsencrypt.log`, extended to a few more services: a live log changing daily is expected noise, but
an already-rotated, compressed copy changing after the fact is not — that would suggest evidence
tampering, not routine operation, so rotated copies are deliberately left watched everywhere in this
guide:

```bash
sudo tee /etc/aide/aide.conf.d/31_aide_live_logs_only > /dev/null << 'EOF'
!/var/log/apache2/access\.log$
!/var/log/apache2/error\.log$
!/var/log/unattended-upgrades/unattended-upgrades\.log$
!/var/log/unattended-upgrades/unattended-upgrades-dpkg\.log$
!/var/log/lynis\.log$
!/var/log/lynis-report\.dat$
EOF
```

`msmtp.log` was added in a follow-up pass once it was confirmed to have no logrotate config
(`/etc/logrotate.d/` — checked, absent), so it grows unbounded and the live-only exclusion applies
the same way:

```bash
sudo tee -a /etc/aide/aide.conf.d/31_aide_live_logs_only > /dev/null << 'EOF'
!/var/log/msmtp\.log$
EOF
```

> `lynis.log` and `lynis-report.dat` are the one exception to "live vs. rotated" above — per
> [12 — Lynis](12-lynis.md) they overwrite in place on every run rather than accumulating dated
> files, so there's no rotated-copy concept to preserve for these two; excluding them outright is
> correct, not a gap.
>
> `msmtp.log` has no logrotate config on this system (`/etc/logrotate.d/` — checked, absent) and
> will grow unbounded. That's a minor, separate disk-space consideration on a ~29GB SD card, not an
> AIDE issue — worth adding a logrotate config for it at some point, tracked here as a known gap
> rather than silently ignored.

Deliberately **not** excluded, despite surfacing as noise in a real diagnostic pass on this system:
`/var/lib/cloud/*` (cloud-init state — inert on this Pi today, but it's the on-disk mirror of
`/boot/firmware/user-data`, a FAT32 boot-partition file that would execute as root on next boot if
tampered with; cheap to keep watched given what it sits downstream of), `/home/*/.bash_history` and
shell/tool config files (deliberately watched — a homelab-specific choice to prioritize catching
unusual account activity over reducing noise, at the cost of a "changed" entry every session).

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

## Diagnosing a large or unexpected diff

A worked example, since "the report shows a lot more changes than I expected" is the single most
useful case to have a documented method for, not just a documented conclusion. This happened once on
this system: a nightly check reported 65 Added, 66 Removed, and 1263 Changed entries — large enough
to warrant real investigation rather than a shrug.

**1 — Never trust the aggregate numbers. Pull the actual sections.**

```bash
grep -n "^Added entries:\|^Removed entries:\|^Changed entries:\|^Detailed information" /var/log/aide/aide.log
```

Use the line numbers this returns to slice out each section with `sed -n 'START,ENDp'`. Read Added
and Removed first — they're usually much shorter than Changed and quicker to eyeball against
whatever you already expect to have changed recently (a remediation session, a manual purge, etc).

**2 — Cross-check against real system logs, not memory.** `/var/log/apt/history.log` (grouped by
transaction, includes `Commandline:` and `Requested-By:` for manual runs) and
`/var/log/unattended-upgrades/unattended-upgrades.log` together give a complete, timestamped account
of every package change on the system. A large Changed count is often simply several days' worth of
unattended-upgrades transactions that hadn't been checked against yet, not a sign anything is wrong
— but confirm this from the actual logs, don't assume it.

**3 — For a large Changed section, don't read 1000+ lines by eye.** Extract every changed path and
check its package ownership programmatically:

```bash
sed -n 'START,ENDp' /var/log/aide/aide.log | grep -oP '(?<=: ).*' | sort -u > /tmp/aide-changed-paths.txt

KNOWN_PKGS='pkg-one|pkg-two|pkg-three'   # every package name from step 2's log review, |-separated

> /tmp/aide-unexplained.txt
while read -r path; do
  pkg=$(dpkg -S "$path" 2>/dev/null | cut -d: -f1)
  if [ -z "$pkg" ]; then
    echo "UNOWNED: $path" >> /tmp/aide-unexplained.txt
  elif ! echo "$pkg" | grep -qE "$KNOWN_PKGS"; then
    echo "UNEXPECTED ($pkg): $path" >> /tmp/aide-unexplained.txt
  fi
done < /tmp/aide-changed-paths.txt
```

`UNOWNED` isn't inherently bad — dpkg doesn't own its own bookkeeping files, logs, or most of
`/var/lib/*`'s runtime state, so a large `UNOWNED` count is expected. `UNEXPECTED` (owned by a real
package, but one not in the known-changed list) is the bucket that actually needs reading line by
line before concluding anything.

**4 — Don't assume "many lines reviewed and explained" means the header count and the detailed
listing agree — check.** On this system, the detailed listing for the Changed section actually
contained 1287 lines against a reported `Changed entries: 1263` header — a real 24-line discrepancy.
Before writing it off, confirm the block itself is clean (no stray separator/header text leaked into
the count):

```bash
sed -n 'START,ENDp' /var/log/aide/aide.log | grep "^[a-z]" | cut -c1 | sort | uniq -c
```

All entries should reduce to a small set of legitimate one-character type codes (`f`, `d`, `l`, and
so on). If they do, and the first/last lines of the block are genuine filesystem paths rather than
formatting artifacts, the discrepancy is a cosmetic quirk in how AIDE's summary header tallies
against the full listing — not a sign of anything hidden. This project has not found a case where
the detailed listing itself was inaccurate; only the header count has been observed to drift from
it. Trust the listing, not the header, when the two disagree.

**5 — Any path that touches something in the "never exclude" category (Tor's hidden-service key,
CrowdSec's ban database or Hub signature files, and similarly sensitive paths noted throughout this
guide) gets independently verified on its own terms, regardless of how well everything else in the
diff is explained.** A fully-explained diff elsewhere in the report is not evidence about an
unrelated sensitive path — check it directly.

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
sudo grep -A5 "^Summary:" /var/log/aide/aide.log
```

---

## Troubleshooting

**`ERROR: missing configuration`**

Missing `--config /etc/aide/aide.conf` — see the scheduling section above.

**`sudo grep -A5 "^Summary:" /var/log/aide/aide.log` produces no output at all**

Not a broken command — a genuinely clean check (no added, removed, or changed entries) skips the
`Summary:` block entirely because it does not exist. Instead, use the following command if you want to be absolutely sure you have a clean check 

```bash
sudo grep -q "found NO differences" /var/log/aide/aide.log && echo "clean"
```

**Report is enormous for what should be a small change**

`report_level=changed_attributes` (the default) prints full per-attribute detail for every changed
file — expected verbosity, not a bug, when many files genuinely changed (a kernel update, for
example). Pull just the summary counts first:

```bash
sudo grep -A5 "^Summary:" /var/log/aide/aide.log
```

Then find the real section boundaries before extracting further, rather than guessing at the report
format:

```bash
grep -n "^Changed entries:" /var/log/aide/aide.log
```

See also the [Diagnosing a large or unexpected diff](#diagnosing-a-large-or-unexpected-diff) section
above for the full method, not just the summary-pulling step.

**An exclusion doesn't seem to be working**

Don't wait for a full reinit to find out — use the `--limit` + `--log-level=rule` method from
earlier in this guide. It shows AIDE's actual per-file decision directly.

**`aideinit` fails with a permission error writing to the log**

See the ownership note in the Reporting section above.

---

**Next:** [14 — unattended-upgrades](14-unattended-upgrades.md)

**Sources:** [36]
