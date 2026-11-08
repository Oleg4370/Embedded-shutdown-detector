# Power Outage Tracker — Project Plan

Goal: know exactly when the electricity at home went off, when it came back, and how long it was gone. This must also work when the internet provider goes down during long outages.

## 1. How it works

```
 [ESP32 + RTC on mains power] --wifi--> [Router on UPS] --internet--> [Server] <--HTTPS-- [PWA on phone]
        |                                                               |
   - "I'm alive" heartbeat every 60 s                      - live status from heartbeats
   - keeps its own outage log in flash                     - stores the final outage history
   - uploads the log when the server is reachable           - merges device log with its own view
```

The device and the server each detect outages:

1. **Device (source of truth for history).** The ESP32 has a battery-backed real-time clock (RTC), so it always knows the time, even right after a power cut and with no internet.
   - While powered, it saves a "last alive" timestamp to flash every 60 s.
   - On boot (power returned), it creates an outage record: `off = last alive`, `on = now (RTC)`.
   - The record goes into a local **outbox queue** in flash. It stays there until the server confirms it received it.
2. **Server (live view and fallback).** The device sends a heartbeat every 60 s.
   - If heartbeats stop for ~3 min, the server marks the device **unreachable** and opens a provisional outage from the last heartbeat.
   - When the device's own records arrive (immediately, or hours later after an internet outage), they replace the provisional entry.
3. The **PWA** shows live status, history and statistics. You can edit, delete or annotate outages there.

### Problem this solves: internet outage during long power cuts

The provider's equipment has its own backup battery, but it doesn't last 12+ hours. So during a long outage:

```
time ──►
power:     ████████░░░░░░░░░░░░░░░░░░░░░░██████░░░░░░██████████████
internet:  ██████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░█████████
                  ▲                     ▲      ▲     ▲      ▲  ▲
             power off            ISP dies   power on  off  on  internet back
                                                                 └─ device uploads queued records
```

- The server only sees "no heartbeats from power-off until internet back".
- The device recorded every power off/on with exact RTC timestamps, including outages that happened while the internet was down.
- After the internet returns, the device uploads the queued records. The server replaces its provisional outage with the real ones.

### Key design decisions

| Decision | Why |
|---|---|
| **DS3231 RTC module with a coin cell** | Without internet, the ESP32 can't get the time (no NTP). Its own clock resets on every power loss. An RTC keeps accurate time (±1 min/year) through power cuts. It is synced from NTP whenever the internet is available. |
| **"Last alive" checkpoint in flash every 60 s** | When power drops, the device dies instantly and can't write anything. The last checkpoint is the off time (±60 s). |
| **Outbox queue in flash, deleted only after the server confirms** | No data is lost if the server or internet is unreachable for hours or days, or if power fails again before upload. |
| **Unique ID per record (`boot_id`)** | Uploads can be retried safely; the server ignores duplicates. |
| **Heartbeat every 60 s, unreachable after 3 missed** | Live status in the app, plus a fallback if the device or its storage fails. |
| **"Unreachable" ≠ "power off" in the UI** | A missing heartbeat can mean no power **or** no internet. The app says "device unreachable since…" until device data confirms what happened. |
| **Server runs outside the house** (or on the UPS) | It must stay up during outages. |
| All times stored in **UTC** | Displayed in the home time zone. |

### Accuracy

- **Off time:** within 60 s (the checkpoint interval).
- **On time:** within a few seconds (RTC time at boot).
- Both work without internet. The optional "last gasp" upgrade in section 2.6 makes the off time exact.

---

## 2. ESP32 device

### 2.1 Shopping list (≈ $8–20)

| Item | Notes |
|---|---|
| ESP32 board | ESP32-C3 SuperMini, ESP32-DevKitC or LOLIN D1 Mini ESP32. |
| **DS3231 RTC module + CR2032 coin cell** | ~$2–3 (e.g. the "ZS-042" board or a mini DS3231 module). Connects over I²C. **Required** for offline timestamps. |
| 5 V USB phone charger + USB cable | Plug into a regular socket, **not** the UPS. No mains wiring needed. |
| 4 Dupont wires | For RTC to ESP32: 3.3V, GND, SDA, SCL. |
| Small enclosure (optional) | 3D-printed or a plastic project box. |
| LED + 220 Ω resistor (optional) | Status light. |

> ZS-042 modules have a charging circuit meant for rechargeable LIR2032 cells. With a normal CR2032, remove the charging diode/resistor (a common, simple mod), or buy a module without charging.

### 2.2 Wiring

```
DS3231      ESP32
VCC   ───── 3.3V
GND   ───── GND
SDA   ───── SDA pin (e.g. GPIO8 on ESP32-C3 SuperMini, GPIO21 on DevKitC)
SCL   ───── SCL pin (e.g. GPIO9 on ESP32-C3 SuperMini, GPIO22 on DevKitC)
```

### 2.3 Software tools

- **PlatformIO** (VS Code extension), **Arduino** framework.
- Libraries: `WiFi`, `HTTPClient`/`WiFiClientSecure`, `ArduinoJson`, `RTClib` (DS3231), `LittleFS` (flash file system), `Preferences` (NVS), `WiFiManager` (setup portal), optional OTA.

### 2.4 Firmware behaviour

**On boot:**
1. Read the time from the RTC. If the RTC lost time (battery dead or never set), mark `time_valid = false` (see 2.5).
2. Generate a random `boot_id`.
3. Read the last "last alive" checkpoint from flash. If there is one, create an outage record and add it to the outbox:
   ```json
   {"id":"<boot_id>","type":"outage","off":"2026-11-08T03:14:00Z","on":"2026-11-08T17:42:05Z","time_valid":true}
   ```
4. Connect to wifi. If no credentials are saved, open the WiFiManager portal to enter wifi, server URL and API key.

**Main loop:**
- **Every 60 s:** write the "last alive" checkpoint (current RTC time) to flash.
- **Every 60 s:** send a heartbeat if wifi is up:
  ```http
  POST /api/v1/heartbeat
  X-Device-Key: <secret>
  {"device_id":"home-1","boot_id":"a1b2c3d4","uptime_s":12345,"rtc_time":"...","pending":3,"rssi":-61,"fw":"1.1.0"}
  ```
- **Outbox flush:** whenever the outbox isn't empty and the server is reachable, upload up to 50 records in one request: `POST /api/v1/events`. Delete only the IDs the server returns as accepted. If the server is unreachable, retry with backoff (30 s → 1 min → 2 min … max 10 min).
- **Time sync:** when the internet is available, sync NTP every 6 h and update the RTC if it is off by more than 2 s.
- **Internet up/down log (optional):** record `net_down` / `net_up` events. This lets the app also show internet-outage periods.
- Hardware watchdog, automatic wifi reconnect, status LED.

### 2.5 Local storage details

| What | Where | Size / wear |
|---|---|---|
| Config (wifi, server URL, API key) | NVS (`Preferences`) | Tiny. |
| "Last alive" checkpoint | NVS, overwriting one key | 1 write/min. NVS spreads writes across its pages (wear leveling): with a 64 KB+ NVS partition, ~525k writes/year use only a small part of the flash's ~100k erase cycles per page. That's years of life. For unlimited writes, an FRAM chip (FM24CL64, ~$3) is an optional upgrade. |
| Outbox queue | LittleFS file `/outbox.jsonl`, one JSON line per record | ~100 bytes/record. 1 MB holds ~10,000 records (years of outages). If it ever fills up, the oldest records are dropped first. |

**Crash safety:** write records with "write new file → rename" or append + flush. A power cut during a write then can't corrupt the queue. A record is only deleted after the server confirms it.

**Without valid RTC time** (coin cell dead): fall back to relative time. A record from the *current* boot is sent with `uptime_s` and the server computes `on = server time − uptime`. A gap that spans earlier boots is marked `approximate`, and the app shows a "replace RTC battery" warning.

### 2.6 Optional upgrade: "last gasp" (exact off time)

Add a small LiPo + TP4056 charger (or a 1–5 F supercapacitor) and a voltage divider from USB 5 V to a GPIO. When 5 V disappears, the device writes an exact `power_off` record to flash (and sends it, if online), then sleeps. This replaces the ±60 s checkpoint with an exact time.

---

## 3. Server

### 3.1 Recommended stack

| Part | Choice |
|---|---|
| Language | **TypeScript + Node.js (Fastify)** (Python + FastAPI also fine) |
| Database | **SQLite** |
| ORM | Drizzle or Prisma |
| Hosting | Small VPS (~$4–5/mo) or free cloud tier in **Docker**, or a Raspberry Pi on the UPS + Cloudflare Tunnel |
| HTTPS | **Caddy** (automatic Let's Encrypt) |

The server also hosts the built PWA.

### 3.2 Data model

```
devices
  id, name, api_key_hash, heartbeat_interval_s (60), offline_threshold_s (180),
  status ('online' | 'unreachable' | 'paused'),
  last_seen_at, last_boot_id, last_uptime_s, pending_events, rssi, fw_version,
  rtc_ok (bool), created_at

outages
  id, device_id,
  started_at, ended_at (NULL = ongoing / unknown yet), duration_s,
  kind        -- 'power' | 'network'
  source      -- 'device' (from device log) | 'server' (provisional, from heartbeat gap) | 'manual'
  confidence  -- 'exact' | 'approximate' | 'provisional'
  device_event_id (unique, nullable)  -- for idempotent uploads
  note, edited (bool), created_at, updated_at, deleted_at

device_events        -- raw uploaded records, for audit/debug (optional)
  id (device event id), device_id, type, payload_json, received_at

monitoring_periods   -- pause/resume
  id, paused_at, resumed_at, reason

users / settings / push_subscriptions
```

### 3.3 Detection and merge logic

**Watchdog (every 30 s):**
```
if monitoring on and device online and now - last_seen_at > threshold:
    status = 'unreachable'
    create outage(started_at = last_seen_at, ended_at = NULL,
                  source = 'server', confidence = 'provisional')
    push: "Home device unreachable since 14:32 (power or internet)"
```

**Heartbeat received:**
```
update last_seen_at, boot_id, uptime, pending_events, ...
status = 'online'
if open provisional outage exists:
    if boot_id is the same as before:   # device never rebooted -> power was fine
        close it: kind = 'network', ended_at = now
    else:
        # device rebooted -> power outage; wait for the device's records,
        # which normally arrive in the same minute
        leave it open until events arrive (or close with ended_at = now - uptime
        after 10 min as a fallback, confidence = 'approximate')
```

**Events upload `POST /api/v1/events`:**
```
for each record:
    if device_event_id already exists -> accepted (duplicate), skip
    if type == 'outage':
        find server-provisional outages for this device overlapping [off - 2 min, on + 2 min]
        - if a provisional outage covers several device outages (internet was down
          between them), the part between device outages was a network outage:
          replace the provisional entry with the device outages, and optionally
          save the gaps as kind='network'
        - keep the user's note/edits if the provisional entry was edited
        insert outage(source='device', confidence = time_valid ? 'exact' : 'approximate')
    if type in ('net_down','net_up'): store as network periods (optional)
return {"accepted": [ids...]}
```
All of this runs in one database transaction, so a failed request changes nothing and the device simply retries.

**Pause ("stop server"):** while paused, no provisional outages are created. Device records with `off`/`on` inside a paused period are stored as hidden (or ignored, configurable). The HTTP server keeps running.

**Server restart safety:** on startup, apply the watchdog check immediately.

### 3.4 REST API

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/v1/heartbeat` | Device heartbeat (device API key) |
| POST | `/api/v1/events` | **Batch upload of the device's queued records → returns accepted IDs** |
| GET | `/api/v1/time` | Server time (backup time source for the device) |
| POST | `/api/auth/login` / `logout` | PWA login (httpOnly cookie) |
| GET | `/api/status` | State (online/unreachable/paused), last seen, ongoing outage, pending device events, RTC health |
| GET | `/api/outages?from=&to=&kind=&page=` | List |
| POST | `/api/outages` | Manual add |
| PATCH | `/api/outages/:id` | Edit start/end/note/kind |
| DELETE | `/api/outages/:id` | Soft delete |
| GET | `/api/stats?period=today\|week\|7d\|month\|30d\|year\|custom&from=&to=` | Totals, count, longest, average, availability %, per-day breakdown |
| POST | `/api/monitoring` `{enabled, reason?}` | Pause/resume |
| GET | `/api/export.csv` | Export |
| POST | `/api/push/subscribe` | Web Push |

Stats count only `kind = 'power'` by default. They split outages correctly at day/month boundaries, using the home time zone.

### 3.5 Security

- Device API key (long, random, stored hashed), HTTPS only.
- Single-user login (argon2/bcrypt), Secure httpOnly cookie, rate-limited login.
- Daily SQLite backup.

---

## 4. PWA web app

### 4.1 Stack

React + Vite + TypeScript, **vite-plugin-pwa**, Tailwind CSS, Recharts/Chart.js, dayjs (time zones), TanStack Query (status refresh every 30 s).

### 4.2 Screens

1. **Dashboard**
   - Status card:
     - 🟢 **Power ON**
     - 🟠 **Device unreachable since 03:14 (14h 28m)**: "power or internet is down; the exact data will sync when the device is back online"
     - 🔴 **Power OFF**: only shown when it is confirmed (e.g. by a last-gasp record)
     - ⏸ **Monitoring paused**
   - Last heartbeat, wifi signal, firmware, **queued records on device**, **RTC battery OK / replace**.
   - Quick totals: today, this week, this month.
2. **History**
   - List grouped by day: `03:14 → 17:42 · 14h 28m · note`, with badges: *provisional*, *approximate*, *manual*, *network*.
   - Filters: period (today, current week, last 7 days, month, year, custom) and kind (power / network / all).
   - Tap to edit start/end, add a note, change kind, or delete (with undo). "+ Add outage" button.
3. **Statistics**
   - Period selector, total time without power, count, longest, average, % time with power.
   - Bar chart per day (per month for a year view). Optional hour × day heatmap.
   - Optional: internet downtime shown separately.
4. **Settings**
   - **Monitoring on/off** toggle ("stop/start server") with an optional reason.
   - Device info, intervals, time zone, push notifications, CSV export, password.

### 4.3 PWA specifics

Manifest + icons, standalone display. The service worker caches the app shell and the last data (shown as stale when offline). Web Push for: unreachable, power back, data synced ("Sync complete: 2 outages added, 14h 28m total").

---

## 5. Repository structure

```
/firmware     PlatformIO project (ESP32 + DS3231)
/server       Fastify + SQLite API (+ serves the built PWA)
/web          React + Vite PWA
/deploy       docker-compose.yml, Caddyfile, backup script
/docs         wiring diagram, API description
PLAN.md
```

---

## 6. Implementation phases

| # | Phase | Result |
|---|---|---|
| 1 | **Server MVP**: heartbeat + events endpoints, SQLite, watchdog, merge logic with unit tests, status/outages API | Logic tested with `curl`/scripts simulating a device |
| 2 | **Firmware MVP**: wifi, heartbeat, DS3231 + NTP sync, checkpoint, outbox in LittleFS, upload + retry | Device survives power and internet outages without data loss |
| 3 | **Deploy**: Docker + Caddy on a VPS, HTTPS, backups | Runs 24/7 |
| 4 | **PWA v1**: login, dashboard, history with edit/delete/note, pause toggle | Daily use |
| 5 | **Stats**: periods, charts, CSV export | Full reporting |
| 6 | **Polish**: WiFiManager portal, OTA, Web Push, LED, RTC battery warning | Comfort |
| 7 | *(optional)* last-gasp hardware, FRAM, several devices/phases, Telegram bot | Extras |

### Testing checklist

- Unplug the ESP32 for 10 min → 1 power outage ≈ 10 min, `source = device`, `confidence = exact`.
- Unplug the router's WAN/internet cable (simulates the provider outage), then unplug and replug the ESP32 **several times**, then restore the internet → every outage appears with correct times, and the provisional entry is replaced.
- Internet down for 30 min while the ESP32 stays powered → recorded as **network**, no power outage.
- Server down for 1 h during an outage → records upload when the server is back; no duplicates.
- Power fails again **during** an upload → no lost or duplicated records after the next boot.
- Remove the RTC coin cell → records marked approximate, warning shown in the app.
- Pause monitoring → unplug → replug → resume → no outage counted.
- Outage across midnight or a month boundary → stats split correctly.

---

## 7. Open questions

1. Hosting: VPS / free cloud tier, or a Raspberry Pi on the UPS?
2. Heartbeat/checkpoint interval: 60 s OK?
3. One device or several (per phase)?
4. Notifications: Web Push, Telegram, or both?
5. Track internet-outage periods in the app too, or only power?
6. Is the "last gasp" upgrade worth it for exact off times?
