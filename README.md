# L337-Merlin

A set of shell scripts for ASUS routers running [Asuswrt-Merlin](https://www.asuswrt-merlin.net/), providing event-driven DDNS updates, WAN telemetry, reconnect classification, and supporting utilities.

Designed for reliability over complexity: event-driven where possible, no external package managers, no polling daemons.

## Features

- **DDNS updates** — Event-driven updates via any DynDNS-compatible provider; duplicate suppression prevents hammering on rapid reconnects
- **WAN event logging** — Structured log of disconnects, reconnects, and downtime
- **Reconnect classification** — Distinguishes `expected_reboot`, `unexpected_boot`, and `runtime_reconnect` events
- **DHCP lease logging** — Records WAN IP, gateway, DNS, and lease details on each connection
- **Static ARP persistence** — Maintains Layer 2 mappings for Wake-on-LAN targets across reboots
- **Log rotation** — Keeps logs bounded; runs via cron
- **Statistics tools** — CLI tools for reviewing WAN history and DHCP lease data

## Requirements

- ASUS router running [Asuswrt-Merlin](https://www.asuswrt-merlin.net/) (developed on 3004.388.x, RT-AX88U)
- JFFS partition enabled (*Administration > System > Enable JFFS custom scripts and configs*)
- `curl` available at `/usr/sbin/curl` (standard on Merlin)

## Before you install

L337-Merlin takes ownership of four Merlin event hook scripts:

```
ddns-start    nat-start    services-start    wan-event
```

**These files will be overwritten** during install and every subsequent update. If any of these hooks already exist on your router with other functionality, you must resolve the conflict before installing — do not merge unknown hooks with L337 scripts.

To check for existing hooks before installing:

```sh
ls -la /jffs/scripts/
```

If any of the four files listed above exist, review their contents and back them up before proceeding.

## Installation

### First-time bootstrap (SSH into router)

1. **Copy and edit the config:**
   ```sh
   curl -fsSL https://raw.githubusercontent.com/jensbrak/L337-Merlin/main/L337.conf.example \
       -o /jffs/configs/L337.conf
   vi /jffs/configs/L337.conf
   ```

2. **Bootstrap the updater and pull all scripts:**
   ```sh
   curl -fsSL https://raw.githubusercontent.com/jensbrak/L337-Merlin/main/L337-update \
       -o /jffs/scripts/L337-update \
       && chmod +x /jffs/scripts/L337-update \
       && /jffs/scripts/L337-update
   ```

3. **Reboot** (or register cron jobs manually):
   ```sh
   /jffs/scripts/services-start
   ```

### Updating

```sh
/jffs/scripts/L337-update
```

Pulls all scripts from `main`. Your `/jffs/configs/L337.conf` is never modified by the updater.

## Configuration

All user-specific settings live in `/jffs/configs/L337.conf`. Copy `L337.conf.example` as a starting point — it documents every variable with defaults and descriptions.

The config file is sourced as a shell script. Key settings:

| Variable | Description |
|---|---|
| `DDNS_USER` / `DDNS_PASS` | DDNS provider credentials |
| `DDNS_HOSTNAME` | Hostname to update |
| `DDNS_UPDATE_URL` | Provider's DynDNS-compatible endpoint URL |
| `L337_DDNS_DOMAIN` | Domain verified by `L337-verify-ddns` |
| `L337_ARP_DEVICES` | WoL targets (IP, MAC, name per line) |
| `L337_DDNS_COOLDOWN` | Duplicate suppression window in seconds (default: 120) |
| `L337_BOOT_RECONNECT_THRESHOLD` | Boot-related reconnect window in seconds (default: 600) |
| `L337_CRON_PLANNED_REBOOT` | Scheduled reboot cron expression (leave empty to disable) |

## Scripts

### Merlin event hooks
These filenames are required by the firmware and cannot be changed.

| Script | Trigger | Purpose |
|---|---|---|
| `ddns-start` | DDNS event | Updates DDNS provider with current WAN IP |
| `wan-event` | WAN connect/disconnect | Logs events, classifies reconnects, invokes DDNS update |
| `nat-start` | After NAT rules loaded | Binds static ARP entries for WoL targets |
| `services-start` | After services start | Registers L337 cron jobs |

### L337 utilities

| Script | Purpose |
|---|---|
| `L337-log-dhcp-lease` | Logs WAN DHCP lease details (called by `wan-event`) |
| `L337-planned-reboot` | Marks a planned reboot and reboots; called by cron |
| `L337-rotate-wan-log` | Trims log files to configured line caps; called by cron |
| `L337-verify-ddns` | Checks DNS resolution against WAN IP; triggers update on mismatch |
| `L337-wan-lease-stats` | CLI: DHCP lease history summary |
| `L337-wan-stats` | CLI: WAN reconnect and downtime statistics |
| `L337-wan-weekly-report` | CLI: Combined stats report |
| `L337-update` | Pulls latest scripts from this repository |

## Log files

| File | Content | Cap |
|---|---|---|
| `/jffs/var/log/wan-history.log` | WAN events (connect, disconnect, reconnect) | 2000 lines |
| `/jffs/var/log/wan-dhcp-leases.log` | DHCP lease snapshots | 1000 lines |
| `/jffs/var/log/planned-reboot.marker` | Reboot intent marker (transient) | — |

## Design principles

- **Event-driven over polling** — scripts run in response to firmware events, not cron where avoidable
- **Quiet on success** — only failures and meaningful state changes are logged
- **No external dependencies** — BusyBox sh only; no Entware, no external packages
- **Idempotent** — safe to run repeatedly without side effects
- **Config-driven** — all personal and tunable values live in `L337.conf`, not in scripts

## License

MIT
