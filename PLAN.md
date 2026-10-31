# Power Outage Tracker — Project Plan

Goal: know exactly when the electricity at home went off, when it came back, and how long it was gone.

## 1. How it works

```
 [ESP32 on mains power] --wifi--> [Router on UPS] --internet--> [Server] <--HTTPS-- [PWA on phone]
        |                                                         |
   "I'm alive" every 60 s                           detects gaps, stores outages
```

1. The ESP32 is plugged into a normal wall socket (no UPS), so it dies when the power goes off.
2. While powered, it sends a **heartbeat** to the server every 60 s.
3. If the server gets no heartbeat for ~3 minutes, it opens an **outage** with `off time = last heartbeat`.
4. When power returns, the ESP32 boots and sends a heartbeat right away. The server closes the outage with `on time = now − device uptime`.
5. The PWA shows live status, history, statistics, and lets you edit, delete or annotate outages.

### Key design decisions

| Decision | Why |
|---|---|
| **Heartbeat every 60 s** (not 5 min) | Accuracy of the off-time equals the interval. 60 s costs almost nothing (≈1,440 tiny requests/day). Configurable. |
| **Outage threshold = 3 missed heartbeats** | Ignores single lost packets or short wifi hiccups. |
| **`boot_id` + `uptime` in every heartbeat** | Tells a **power outage** apart from an **internet/wifi outage**. A new `boot_id` means the device restarted, so the power really went off. The same `boot_id` with growing uptime means power was fine and only the network failed. Without this, every internet drop would be recorded as a power cut. |
| **On-time = server time − uptime** | Correct even if the internet comes back later than the power. |
| **The server must NOT run in your house** (or must be on the UPS) | A server that loses power can't record the outage. Use a cheap VPS/free cloud tier, or a Raspberry Pi plugged into the UPS. |
| **Server owns all timestamps** | The ESP32 doesn't need an exact clock; server time (NTP-synced) is the source of truth. |

### Accuracy

- **Off time:** between the last heartbeat and last heartbeat + 60 s. Store `last heartbeat`, optionally show "± 1 min".
- **On time:** accurate to a few seconds (boot time + wifi connect time).
- Want an exact off time? See the optional "last gasp" upgrade in section 2.4.

---

## 2. ESP32 device

### 2.1 Shopping list (≈ $5–15)

| Item | Notes |
|---|---|
| ESP32 board | ESP32-C3 SuperMini, ESP32-DevKitC or Wemos/LOLIN D1 Mini ESP32. Any ESP32 with wifi works. |
| 5 V USB phone charger + USB cable | Safest option: no mains wiring. Plug into a regular socket (**not** the UPS). |
| Small enclosure (optional) | 3D-printed or a plastic project box. |
| LED + 220 Ω resistor (optional) | Status light, if the board has no usable onboard LED. |

> Safety: don't wire the ESP32 to 230 V directly. A certified USB charger is enough. Plugging it into a socket also lets you choose which circuit/phase you monitor.

### 2.2 Software tools

- **PlatformIO** (VS Code extension) with the **Arduino** framework. Simplest to start; ESP-IDF is possible later.
- Libraries: `WiFi`, `HTTPClient`/`WiFiClientSecure`, `ArduinoJson`, `WiFiManager` (setup portal), `ArduinoOTA` or HTTP OTA (optional).

### 2.3 Firmware behaviour

1. On boot: generate a random `boot_id` (`esp_random()`), and start the uptime counter.
2. Connect to wifi. If no credentials are saved, open a **WiFiManager captive portal** (an "OutageTracker-Setup" access point) to enter wifi SSID/password, server URL and device API key. Save them to NVS (flash).
3. Send a heartbeat **immediately**, then every `HEARTBEAT_INTERVAL` (60 s):
   ```http
   POST https://your-server/api/v1/heartbeat
   X-Device-Key: <secret>
   {"device_id":"home-1","boot_id":"a1b2c3d4","uptime_s":12345,"seq":205,"rssi":-61,"fw":"1.0.0"}
   ```
4. Retry on failure with backoff. Reconnect wifi automatically.
5. **Hardware watchdog**: reboot if the firmware hangs.
6. LED: blinking = connecting, solid = OK, fast blink = server error.
7. Optional: the server's reply can change the interval, so you don't need to reflash.
8. Optional: OTA firmware updates.

Don't write to flash on each heartbeat; it wears out the flash. Nothing needs to be stored per heartbeat.

### 2.4 Optional v2 upgrade: "last gasp" (exact off time)

Add a small energy reserve so the ESP32 can report the moment power drops:
- A small LiPo + TP4056 charger module (or a 1–5 F supercapacitor), plus
- a voltage divider from the USB 5 V line to a GPIO to detect "USB power lost".

When 5 V disappears, the device immediately sends `{"event":"power_lost"}` and then goes to sleep. This gives an exact off time. The heartbeat logic stays as the fallback.

---

## 3. Server

### 3.1 Recommended stack

| Part | Choice | Alternatives |
|---|---|---|
| Language | **TypeScript + Node.js (Fastify)**: same language as the PWA | Python + FastAPI |
| Database | **SQLite** (one file; plenty for one house) | PostgreSQL |
| ORM | Drizzle or Prisma | — |
| Hosting | Small VPS (Hetzner/DigitalOcean ~$4–5/mo) or Oracle Cloud free tier, in **Docker** | Raspberry Pi on the UPS + Cloudflare Tunnel |
| HTTPS | **Caddy** reverse proxy (automatic Let's Encrypt). PWAs require HTTPS. | Cloudflare |

The server also hosts the built PWA files, so there is a single deployment.

### 3.2 Data model

```
devices
  id, name, api_key_hash, heartbeat_interval_s (60), offline_threshold_s (180),
  last_seen_at, last_boot_id, last_uptime_s, last_rssi, fw_version, created_at

outages
  id, device_id,
  started_at            -- off time
  ended_at   (nullable) -- on time; NULL = ongoing
  duration_s (computed when closed)
  kind       -- 'power' | 'network' | 'unknown'
  source     -- 'auto' | 'manual'
  note       (text)
  edited     (bool), created_at, updated_at, deleted_at (soft delete)

monitoring_periods          -- for "stop/start server"
  id, paused_at, resumed_at (nullable), reason

settings / users
  single user: username, password_hash; push subscriptions
```

Optional: a `heartbeats` table with raw heartbeats, auto-deleted after 30 days. It's useful for debugging. Otherwise only `last_seen` is needed.

### 3.3 Outage detection logic (the core)

Device status: `ONLINE` → `OFFLINE` → `ONLINE`, or `PAUSED` when monitoring is stopped.

**Watchdog job (every 30 s):**
```
if monitoring enabled
   and device ONLINE
   and now - last_seen_at > offline_threshold_s:
       create outage(started_at = last_seen_at, ended_at = NULL, kind = 'unknown')
       status = OFFLINE
       send push notification "Power off since HH:MM" (optional)
```

**On heartbeat:**
```
validate API key
if open outage exists:
    if boot_id != last_boot_id:          # device rebooted -> real power outage
        ended_at = now - uptime_s
        kind = 'power'
    else:                                 # same boot -> only network was down
        ended_at = now
        kind = 'network'                  # hidden by default in UI / stats
    duration_s = ended_at - started_at
    push "Power is back, was off 2h 13m"
elif boot_id != last_boot_id and last_seen_at is set:
    # rebooted but the gap was shorter than the threshold (short blink or manual replug)
    optionally record a short 'power' outage from last_seen_at to now - uptime_s
update last_seen_at, last_boot_id, uptime, rssi
```

**Pause ("stop server"):** while monitoring is paused, the watchdog creates no outages, and heartbeats only update `last_seen`. On resume, the device needs one fresh heartbeat before detection starts again, so the time you were unplugged isn't counted. The HTTP server itself keeps running so the PWA still works. Stopping the process or container entirely is possible but not recommended.

**Server restart safety:** on startup, if `now − last_seen` is already past the threshold, open the outage from `last_seen`. Then a server outage doesn't lose data.

### 3.4 REST API

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/v1/heartbeat` | Device heartbeat (device API key) |
| POST | `/api/auth/login` / `logout` | PWA login → httpOnly session cookie |
| GET | `/api/status` | Current state, last seen, ongoing outage, monitoring on/off, device info |
| GET | `/api/outages?from=&to=&kind=&page=` | List outages |
| POST | `/api/outages` | Manually add an outage |
| PATCH | `/api/outages/:id` | Edit start/end/note/kind |
| DELETE | `/api/outages/:id` | Delete (soft delete) |
| GET | `/api/stats?period=today\|week\|7d\|month\|30d\|year\|custom&from=&to=` | Total off time, count, longest, average, availability %, per-day totals |
| POST | `/api/monitoring` `{enabled: bool, reason?}` | Pause/resume monitoring |
| GET | `/api/export.csv?from=&to=` | Export |
| POST | `/api/push/subscribe` | Web Push subscription (optional) |

Notes:
- Stats clip outages to the period boundaries, so an outage that crosses midnight is split correctly between days.
- Use the home **time zone** (e.g. `Europe/Kyiv`) as a setting for "today/week/month" boundaries. Store everything in UTC.

### 3.5 Security

- Device: a long random API key, stored hashed on the server, sent over HTTPS only.
- PWA: single-user login (bcrypt/argon2 password), httpOnly + Secure cookie, rate-limited login.
- Daily SQLite backup (e.g. `sqlite3 .backup` via cron to another location).

---

## 4. PWA web app

### 4.1 Stack

- **React + Vite + TypeScript**, **vite-plugin-pwa** (manifest + service worker, installable on the phone home screen).
- UI: Tailwind CSS (or MUI). Charts: Recharts or Chart.js. Dates: date-fns / dayjs with time zone support.
- Data fetching: TanStack Query, auto-refresh status every 30 s.

### 4.2 Screens

1. **Dashboard**
   - Large status card: 🟢 *Power ON* / 🔴 *Power OFF since 14:32 (1h 12m, live timer)* / ⏸ *Monitoring paused*.
   - Last heartbeat, wifi signal, firmware version.
   - Quick totals: today, this week, this month (off time and number of outages).
2. **History**
   - List grouped by day: `14:32 → 16:45 · 2h 13m · note`. An ongoing outage is shown at the top.
   - Filters: period (today, current week, last 7 days, month, year, custom range) and kind (power/network).
   - Tap an outage to edit start/end, add a note, change kind, or delete (with confirm and undo).
   - "+ Add outage" button for manual entries.
3. **Statistics**
   - Period selector: current week / last 7 days / current month / last 30 days / year / custom.
   - Total time without power, number of outages, longest, average, % time with power.
   - Bar chart of off-hours per day (or per month for the year view).
   - Optional: a timeline/heatmap of the day (hours × days), handy for seeing scheduled-outage patterns.
4. **Settings**
   - **Monitoring on/off** toggle (the "stop/start server"), with an optional reason.
   - Device info, heartbeat interval, offline threshold, time zone.
   - Push notifications on/off.
   - Export CSV. Change password.

### 4.3 PWA specifics

- Manifest: name, icons (192/512), `display: standalone`, theme color.
- Service worker: cache the app shell for instant load. Show the last known data when offline, clearly marked as stale.
- **Web Push** (optional but very useful): a notification when the power goes off or comes back. Works on Android, and on iOS 16.4+ once the app is installed to the home screen.

---

## 5. Repository structure

```
/firmware     PlatformIO project (ESP32)
/server       Fastify + SQLite API (+ serves the built PWA)
/web          React + Vite PWA
/deploy       docker-compose.yml, Caddyfile, backup script
/docs         wiring photo/diagram, API description
PLAN.md
```

---

## 6. Implementation phases

| # | Phase | Result |
|---|---|---|
| 1 | **Server MVP**: heartbeat endpoint, SQLite, detection logic + unit tests, `/api/status`, `/api/outages` | Outages recorded (test with `curl` simulating a device) |
| 2 | **Firmware MVP**: wifi + heartbeat with `boot_id`/uptime, hardcoded config | Real device reporting |
| 3 | **Deploy**: Docker + Caddy on a VPS, HTTPS, backups | Runs 24/7 outside the house |
| 4 | **PWA v1**: login, dashboard, history with edit/delete/note, pause toggle | Usable daily |
| 5 | **Stats**: periods, charts, CSV export | Full reporting |
| 6 | **Polish**: WiFiManager setup portal, OTA, Web Push notifications, LED status | Comfortable to use |
| 7 | *(optional)* Last-gasp hardware, multiple devices/phases, Telegram bot notifications | Extras |

### Testing checklist

- Unplug the ESP32 for 10 min → outage ≈ 10 min recorded, kind = power.
- Turn off the router wifi for 10 min while the ESP32 stays powered → recorded as **network**, not power.
- Unplug for 1 min (below the threshold) → short outage handled as designed.
- Pause monitoring → unplug → replug → resume → no outage recorded.
- Restart the server during an outage → outage still recorded correctly.
- An outage crossing midnight or a month boundary → stats split correctly.

---

## 7. Open questions to decide

1. Hosting: VPS / free cloud tier, or a Raspberry Pi on the UPS?
2. Heartbeat interval: 60 s (recommended) or something else?
3. One device or several (e.g. one per phase in a 3-phase house)? The data model already supports several.
4. Notifications: Web Push, Telegram bot, or both?
5. Is the "last gasp" hardware upgrade worth it for exact off times?
