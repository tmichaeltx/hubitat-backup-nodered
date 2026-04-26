# hubitat-backup-nodered

A self-contained Node-RED flow for automated Hubitat hub management:

- **Daily backup** of all hubs to a local directory, with five-tier retention
- **Update check** — polls hubs for available firmware updates and publishes results to MQTT
- **Manual update trigger** — backs up a hub, applies a firmware update, and polls until the new version is confirmed

No external scripts required. All logic runs natively inside Node-RED using HTTP request and function nodes.

Tested with Hubitat C-5 hubs running firmware 2.4.x and 2.5.x.

---

## Prerequisites

- Node-RED (tested on 4.x)
- An MQTT broker (e.g. Mosquitto)
- A directory on the host mounted into the Node-RED container at `/backup`
### Volume mount example (Docker Compose)

```yaml
volumes:
  - /path/to/your/backup/dir:/backup
```

---

## Setup

### 1. Import the flow

In the Node-RED editor: **Menu → Import → select `flow.json`**

This creates a `hubitat-maintenance` tab with all three groups.

### 2. Configure the MQTT broker node

Open any of the `mqtt out` nodes and set the broker to point to your MQTT broker host and port.

### 3. Set environment variables

Open the `hubitat-maintenance` tab **Properties** (double-click the tab) and fill in the **Env** section with your hub details:

| Variable | Description | Default |
|----------|-------------|---------|
| `HUB_COUNT` | Number of hubs (1–3) | `3` |
| `HUB_1_NAME` | Display name for hub 1 | `Hub One` |
| `HUB_1_ADDR` | Hostname or IP of hub 1 | `hubitat-hub-1.local` |
| `HUB_1_USER` | Admin username for hub 1 | `your-username` |
| `HUB_1_PASS` | Admin password for hub 1 | `your-password` |
| `HUB_2_NAME` | Display name for hub 2 | `Hub Two` |
| `HUB_2_ADDR` | Hostname or IP of hub 2 | `hubitat-hub-2.local` |
| `HUB_2_USER` | Admin username for hub 2 | `your-username` |
| `HUB_2_PASS` | Admin password for hub 2 | `your-password` |
| `HUB_3_NAME` | Display name for hub 3 | `Hub Three` |
| `HUB_3_ADDR` | Hostname or IP of hub 3 | `hubitat-hub-3.local` |
| `HUB_3_USER` | Admin username for hub 3 | `your-username` |
| `HUB_3_PASS` | Admin password for hub 3 | `your-password` |
| `BACKUP_PATH` | Path inside the container where backups are written | `/backup` |
| `RETAIN_T1` | Daily backups to keep for the current build | `3` |
| `RETAIN_T2` | Builds to keep one backup each (most recent) | `3` |
| `RETAIN_T3` | Patches to keep one backup each | `3` |
| `RETAIN_T4` | Minors to keep one backup each | `3` |
| `RETAIN_T5` | Majors to keep one backup each | `3` |
| `RETAIN_MAX` | Hard cap on total backups per hub | `14` |

If you have fewer than 3 hubs, set `HUB_COUNT` to the actual number. The unused hub variables are ignored.

If you have more than 3 hubs, Groups A and B will handle them automatically — just add the additional `HUB_4_NAME` / `HUB_4_ADDR` / etc. env vars and increment `HUB_COUNT`. Group C, however, has one inject node hardcoded per hub. For a 4th hub you would need to duplicate one of the existing inject+function pairs in the editor and update the label to match your 4th hub's name.

---

## Groups

### Group A — Daily Backup (runs at 1:00 AM)

Loops over all hubs. For each:
1. Logs in, downloads the most recent `.lzf` backup from the hub
2. Writes it to `<BACKUP_PATH>/<hub-name>/YYYY-MM-DD-HHMM-<hub-date>~<version>.lzf`
3. Applies the five-tier retention policy
4. Publishes a summary to MQTT topic `homelab/hubitat/backup/status`

### Group B — Update Check (runs at 3:00 AM)

Loops over all hubs. For each:
1. Logs in, calls the update check endpoint
2. Fetches hub details to get the current version
3. Collects results and publishes to MQTT topic `homelab/hubitat/update/available`

### Group C — Manual Update Trigger

Three inject nodes — one per hub. Triggering one:
1. Downloads and saves a pre-update backup
2. Calls the `updatePlatform` endpoint to trigger the firmware download and install
3. Polls hub details every 30 seconds (up to 20 minutes) until the version changes
4. Publishes the result to MQTT topic `homelab/hubitat/update/result`

Note: Hubitat C-5 hubs typically take 15–20 minutes from the update trigger to complete the download and reboot. The poll loop accounts for this.

---

## Retention policy

Backups are stored per hub in `<BACKUP_PATH>/<hub-name>/`. After each backup, the retention cleanup function runs a five-tier keep/delete pass:

| Tier | Kept |
|------|------|
| T1 | `RETAIN_T1` most recent backups for the current build (all 4 version parts match) |
| T2 | 1 backup per distinct build, for the `RETAIN_T2` most recent builds |
| T3 | 1 backup per distinct patch, for the `RETAIN_T3` most recent patches |
| T4 | 1 backup per distinct minor, for the `RETAIN_T4` most recent minors |
| T5 | 1 backup per distinct major, for the `RETAIN_T5` most recent majors |

Anything not assigned to a tier is deleted. If the total kept exceeds `RETAIN_MAX`, the oldest T5 files are dropped first.

---

## MQTT topics

| Topic | Published by | Payload |
|-------|-------------|---------|
| `homelab/hubitat/backup/status` | Group A | `{ timestamp, hubs: [{ hub, status, file }] }` |
| `homelab/hubitat/update/available` | Group B | `{ timestamp, hubs: [{ hub, updateAvailable, version, ... }] }` |
| `homelab/hubitat/update/result` | Group C | `{ timestamp, hub, status, versionBefore, versionAfter, polls }` |

Group C `status` values: `updated` (version changed), `no_change` (20-minute timeout elapsed with no version change).

---

## Hubitat API notes

All calls use **port 8080** except login (port 80).

| Call | Method | Endpoint |
|------|--------|----------|
| Login | POST | `:80/login` — body: `username=X&password=Y` (URL-encoded) |
| List backups | GET | `:8080/hub2/localBackups` |
| Download backup | GET | `:8080/hub/backupDB?fileName=<name>` |
| Check for update | GET | `:8080/hub/cloud/checkForUpdate` |
| Hub details / version | GET | `:8080/hub/details/json` |
| Apply update | GET | `:8080/hub/cloud/updatePlatform` |

The login response sets a session cookie. All subsequent calls attach that cookie. The poll loop re-authenticates before every poll because the hub session cookie is invalidated when the hub reboots during an update.

---

## How this was built

This flow was developed collaboratively using [Claude Code](https://claude.ai/claude-code). The architecture, Hubitat API integration, retention policy, and poll loop behavior were iteratively designed and refined through a working session that included live testing against real hubs — including discovering that C-5 firmware downloads take 15–20 minutes before the hub reboots, which shaped the final poll loop design.

---

## License

MIT
