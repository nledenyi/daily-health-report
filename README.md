# daily-health-report

Opinionated, single-file daily health report for a **Proxmox VE** host (single-node or small homelab). Runs once a day, produces three outputs in one pass:

- **Telegram message** — one-line KPI per section, with a worst-severity icon
- **HTML email** — compact dark-theme tables, one section per concern
- **Markdown log** — appended to a dated file, suitable for grep/tail/git

...and drives case RGB lighting (green / amber / red) via an external hook, so the severity of the last report is visible at a glance.

Designed to be **boring**: no daemons, no databases, no agents, no UI. A single bash script, one systemd timer, one config file. Readable by anyone who knows shell.

## What it checks

| Area | Signals |
|------|---------|
| ZFS pools + pve-root FS | health, capacity thresholds (80/90/85/95%) |
| Disks (SMART) | health status, temperature, hours, uncorrectable errors, reallocated sectors, **missing-disk detection by serial number** |
| ZFS replication | sanoid snapshot coverage across VM/LXC datasets, sanoid+syncoid service exit status, syncoid dataset age |
| Proxmox backups | vzdump coverage (jobs × guests), last-backup age |
| Guests | VM and LXC status, memory, disk %, pending `apt` updates, HASS OS special-case (`/mnt/data` instead of `/`) |
| Postfix | deferred mail queue |
| Docker | container health across a Docker VM and a docker-in-LXC, configurable critical-container patterns (CRIT on down) and non-critical ones (WARN on unhealthy/crash) |
| Service health | Home Assistant API, PostgreSQL (via `pg_isready` in LXC), NVIDIA/AMD GPU driver presence, **LXC GPU cgroup major-number mismatches** (catches silent breakage after kernel/driver upgrades) |
| TLS | certificate expiry on the Docker host's 443 |
| Syncthing | API reachability, per-folder pull errors, peer offline > 24h |
| `claude-remote.service` | systemd active + `claude remote-control` process alive |
| Updates | `unattended-upgrades` status, reboot-required flag, package NEWS, PVE + kernel available-update detection |
| ZFS scrub | last-run + errors per pool |
| Ollama | version + GitHub release check, models, GPU offload, request count + errors over 24h, tokens, tok/s, idle time |
| Temperatures | CPU, NVMe, NVIDIA GPU, AMD GPU (+ VRAM + sleep state), 10G NIC, disk min-max |
| System summary | uptime, load, RAM, PVE + kernel versions, upgradable-package count, LXC update roll-up |

Each check runs in a dedicated `check_<name>` bash function. Adding a new one typically means writing ~5-10 lines plus one call.

## Architecture

Three concerns, sharply separated:

```
┌──────────────────────────────────────────────────────┐
│  Gather phase — check_storage_pools, check_disks,    │
│  check_guests, check_docker, check_ollama, etc.      │
│  Each function sets its per-section globals AND      │
│  calls emit_issue LEVEL "msg" for any issues.        │
└──────────────────────────────────────────────────────┘
                          │
                          ▼
          ISSUES=("CRIT|..." "WARN|..." "INFO|...")
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
   Markdown renderer  Telegram        HTML email
   (append to log)    (KPI lines)     (dark-theme
                                       tables)
                          │
                          ▼
                 set-case-rgb worst-severity
```

- **One severity ladder**: `CRIT | WARN | INFO`. Telegram icon, HTML color, RGB lighting, email subject all derive from the worst level in `ISSUES`.
- **Issue emission is centralized**: every check calls `emit_issue LEVEL "human-readable message"`. Adding a new issue-only check touches exactly one file, in one place.
- **Per-section OK flags are set upstream** (`POOL_OK`, `SMART_OK`, `GUEST_OK`, `CAP_OK`, `SANOID_OK`, `SYNCOID_OK`) — the renderers never re-derive them from raw data blobs. This was the biggest lever in the refactor from the original version.
- **Each check function comments its `Exports:` and any `Depends on:`** upstream state. See the top of each `check_*` in `daily-health-report`.

## Install

Requirements: a Proxmox VE 8+ host running `bash`, `python3`, `smartctl`, `sensors`, `zfs`/`zpool`, `pct`, `qm`, `pvesh`, `postqueue`, `curl`, `mail` (bsd-mailx / s-nail). GPU checks require `nvidia-smi` and/or the amdgpu sysfs interface; they no-op otherwise.

```bash
git clone https://github.com/nledenyi/daily-health-report.git
cd daily-health-report

# 1. Stage the config
sudo mkdir -p /etc/daily-health
sudo cp config.example /etc/daily-health/config
sudo chmod 0644 /etc/daily-health/config
sudo $EDITOR /etc/daily-health/config      # fill in IPs, VMIDs, disk serials, email domain, etc.

# 2. Install the script + systemd units
sudo install -m 0755 -o root -g root daily-health-report /usr/local/bin/daily-health-report
sudo install -m 0644 daily-health-report.service /etc/systemd/system/
sudo install -m 0644 daily-health-report.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now daily-health-report.timer

# 3. (Optional) External hooks — the script shells out to these if present
#   /usr/local/bin/notify-telegram  "<message>"     # sends to Telegram
#   /usr/local/bin/set-case-rgb     <ok|warn|crit>  # sets case LED color
# If either is missing, the corresponding output is skipped silently.

# 4. Run on demand to verify
sudo /usr/local/bin/daily-health-report
```

The default timer fires at 08:00 local time with `Persistent=true` (so a missed wake-up catches up when the host comes online).

## Configuration

Every environment-specific value lives in `/etc/daily-health/config` (bash-sourced). See `config.example` — it's short, commented, and every field is documented inline. At a glance:

- **Identity**: `PVE_NODE`, `LOG_DIR`, `MAIL_FROM`
- **Service endpoints**: `HA_URL`, `OLLAMA_API`, `OLLAMA_METRICS`, `SYNCTHING_URL`, `SYNCTHING_CONFIG_XML`
- **Storage**: `ZFS_POOLS_REGEX`, `SYNCOID_DATASET`, `VZDUMP_LOG_GLOB`
- **Guests**: `VM_IDS`, `VM_EXPECTED_STOPPED`, `HASS_OS_VM_IDS`
- **Service LXCs**: `POSTGRES_LXC`, `DOCKER_VM`, `DOCKER_IN_LXC`, `OLLAMA_LXC`
- **GPU**: `NVIDIA_UVM_LXCS`, `AMD_KFD_LXCS` (LXCs that should have matching cgroup device allows)
- **Docker**: `DOCKER_CRITICAL_PATTERNS`, `DOCKER_CRITICAL_EXCLUDE` (bash glob patterns)
- **Expected disks**: `EXPECTED_DISKS` (associative array, serial → description)

Most fields accept empty string / empty array to disable their check. Unconfigured probes are silently skipped so you can start minimal and add coverage as you grow.

## Adding a new check

Issue-only (most common case):

```bash
check_my_thing() {
    if something_is_wrong; then
        emit_issue WARN "descriptive message"
    fi
}
check_my_thing
```

Drop it next to the existing `check_*` functions in `daily-health-report`. That's the whole change — the issue appears in all three outputs, contributes to severity, and drives RGB color automatically.

If your check also needs to render its own section (table or summary), you'll additionally touch the markdown, telegram, and HTML renderer blocks — they're at the bottom of the file, labelled with `# ====== Markdown Log ======`, `# ====== Telegram ======`, `# ====== HTML Email ======`. An empty `SECTIONS=()` registry is in place at the top of the file for a future registry-driven renderer; for now, adding rendered sections is still a 2-3 place edit.

## Design notes

- **Single file, no dependencies on this repo.** Config is the only non-committed runtime input.
- **Fail soft.** Almost every probe is wrapped in `2>/dev/null`; an unreachable service is a WARN/CRIT in the report, not a script crash. There's no `set -e` (deliberately — many `grep -q` chains would trip it).
- **Python stdlib for JSON.** Bash for everything else. No third-party pip packages. When a check needs to talk to a REST API (Ollama, Syncthing, GitHub releases), inline `python3 -c` with `urllib` handles it.
- **`while read … <<< "$(…)"` not pipes.** Piping into a `while` loop runs the loop in a subshell, losing the variable assignments that the whole script relies on. Here-string form keeps assignments in the parent shell.
- **One worst-severity policy.** Any `CRIT` → red. Any `WARN` → amber. Otherwise green. `INFO` never escalates color.

## Not included

- `notify-telegram` and `set-case-rgb` are external hooks. The first can be a 3-line curl wrapper; the second is motherboard/GPU-specific (mine uses the MSI Mystic Light USB HID protocol + I2C on an ASUS card). If either isn't present, the corresponding output is skipped cleanly.
- No metrics export (Prometheus etc.). If you want dashboards, scrape something else and let this handle the daily digest role.
- No templating engine for the HTML. The email is built by string concatenation in bash. This is ugly but portable.

## License

MIT — see [LICENSE](LICENSE).

---

Written for my own single-node Proxmox at home, made public because it's the kind of thing I'd have wanted to find when I started. Open an issue if you hit a snag porting it, or if something in the config would genuinely benefit from being surfaced.
