# Flock-You Copilot Instructions

## Project Overview

Flock-You is a BLE-only surveillance device detector targeting Flock Safety cameras, Raven gunshot detectors, and related hardware. It runs on a **Seeed Studio XIAO ESP32-S3** and has two components:

- **Firmware** (`main/main.cpp`): ESP-IDF C++ — the entire firmware is a single file built with CMake
- **Flask companion app** (`api/`): Python desktop dashboard for live serial ingestion and data analysis

---

## Build & Flash Commands (Firmware)

Requires [ESP-IDF v5.1+](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/get-started/).

```bash
idf.py set-target esp32s3   # first time only
idf.py build                # compile (fetches managed components on first run)
idf.py -p PORT flash        # flash firmware
idf.py -p PORT monitor      # serial monitor (USB CDC, 115200 baud)
idf.py -p PORT flash monitor  # flash then monitor
```

There is no test suite for the firmware.

> The pre-migration Arduino/PlatformIO source is preserved in `src/main.cpp` for reference.

## Flask Companion App

```bash
cd api
pip install -r requirements.txt
python flockyou.py        # starts at http://localhost:5000
```

---

## Architecture

### Radio Coexistence

The ESP32-S3 cannot simultaneously scan BLE and serve WiFi AP at full capacity, but uses ESP32 coexistence mode:
- **WiFi AP** (`flockyou` / `flockyou123`) serves the web dashboard at `192.168.4.1`
- **BLE** scans continuously in the background

**Companion mode** switches this behavior: when the DeFlock mobile app connects via BLE GATT or a serial host is detected, `fyCompanionChangePending` is set to `true` in the BLE callback, and `fyMainTask()` calls `fyOnCompanionChange()` to disable WiFi AP (via `esp_wifi_stop()`) and boost BLE scan duty cycle. This deferred pattern exists because WiFi operations must not be called from BLE callback context. Serial host detection runs in a dedicated `fySerialMonitorTask` that blocks on `fgetc(stdin)` (USB CDC VFS).

### Detection Pipeline

All detection is BLE advertisement-based (no WiFi sniffing). Priority order:

1. **Raven UUID** — service UUID fingerprinting against `raven_service_uuids[]`; firmware version estimated from which UUIDs appear (1.1.x uses legacy Health/Location UUIDs, 1.2.x adds GPS service, 1.3.x adds Power service)
2. **High-confidence MAC prefix** — `flock_mac_prefixes[]` (21 OUIs, direct Flock Safety registrations)
3. **Device name** — case-insensitive substring: `FS Ext Battery`, `Penguin`, `Flock`, `Pigvision`
4. **Manufacturer ID** — `0x09C8` (XUNTONG), sourced from wgreenberg/flock-you
5. **Manufacturer MAC** — `flock_mfr_mac_prefixes[]` (Liteon/USI contract manufacturers — lower confidence)
6. **SoundThinking MAC** — `soundthinking_mac_prefixes[]` (`d4:11:d6`)

### Detection Storage

Static array `FYDetection fyDet[MAX_DETECTIONS]` (max 200 entries) protected by a FreeRTOS mutex (`fyMutex`). Detections are deduplicated by MAC address; re-sightings update RSSI, `lastSeen`, count, and GPS. GPS is re-attached on every sighting to capture device movement.

### Session Persistence (SPIFFS)

- `/session.json` — current session, auto-saved every 15 seconds (only when count changes)
- `/prev_session.json` — previous session, copied on boot before overwriting current

Custom partition table (`partitions.csv`) allocates 6 MB for app and ~2 MB for SPIFFS.

### GPS (Phone Wardriving)

Phone GPS is pushed via browser Geolocation API to the ESP32's `/api/gps` endpoint. The ESP32 stores the last GPS fix and marks it stale after 30 seconds (`GPS_STALE_MS`). GPS is attached to each detection at the time of first sighting and updated on every re-sighting.

### Flask App (`api/flockyou.py`)

- Uses `Flask-SocketIO` (threading mode) for real-time WebSocket updates to the dashboard
- Reads JSON detection records from USB serial (ESP32 serial output format)
- Cumulative detections persisted to `api/data/cumulative_detections.pkl` (pickle)
- Settings persisted to `api/data/settings.json`
- GPS dongle support via `pyserial` — parses NMEA GPGGA sentences
- Matches GPS readings to detections using a 30-second temporal threshold (`GPS_MATCH_THRESHOLD`)

---

## Key Conventions

### Firmware (`main/main.cpp`)

- **All globals/functions prefixed `fy`**: `fyDet`, `fyMutex`, `fyServer`, `fyGPSLat`, `fyBeep()`, etc.
- **Section headers**: `// ===...===` comment blocks divide the file into logical sections (CONFIGURATION, DETECTION PATTERNS, AUDIO SYSTEM, etc.)
- **MAC comparison**: always `strncasecmp(mac_str, prefix, 8)` for 8-character OUI prefix matching
- **JSON name sanitization**: `"` and `\` characters in device names are replaced with `_` before storage
- **Mutex discipline**: always acquire `fyMutex` with a 100ms timeout; release in all code paths including early returns
- **Audio**: `fyBeep()` and `fyCaw()` are no-ops when `fyBuzzerOn == false`; all audio must check this
- **Entry point**: `extern "C" void app_main()` (replaces Arduino `setup()`); the main loop runs in `fyMainTask` via `xTaskCreate`
- **Platform shims**: `millis()` is an inline shim over `esp_timer_get_time()`; `delay(N)` expands to `vTaskDelay(pdMS_TO_TICKS(N))`
- **Audio peripheral**: tone generation uses LEDC (`ledc_timer_config` / `ledc_channel_config` / `ledc_set_freq` / `ledc_update_duty`)
- **WiFi**: `esp_wifi_init` / `esp_wifi_set_mode` / `esp_wifi_set_config` / `esp_wifi_start`; companion mode calls `esp_wifi_stop()` / `esp_wifi_start()` — config persists in RAM, no re-init needed
- **Filesystem**: `esp_vfs_spiffs_register()` mounts at `/spiffs`; files opened with standard `fopen("/spiffs/...")` returning `FILE*`
- **HTTP server**: `esp_http_server`; chunked responses via `httpd_resp_send_chunk()`; ended with `httpd_resp_send_chunk(req, NULL, 0)`
- **Legacy source**: `src/main.cpp` (Arduino/PlatformIO) is preserved as reference only; it is not built

### OUI Reference Files

- `oui.txt` (root) — IEEE OUI database for reference when updating MAC prefix lists
- `api/oui.txt` — copy used by the Flask app for MAC vendor lookup
- `datasets/raven_configurations.json` — Raven BLE GATT characteristic map organized by firmware version (1.1.7, 1.2.0, 1.3.1); used as reference when updating Raven detection logic

### MAC Prefix Confidence Tiers

When adding new OUIs, place them in the appropriate array:
- `flock_mac_prefixes[]` — confirmed Flock Safety IEEE registrations or exclusive-use OUIs
- `flock_mfr_mac_prefixes[]` — shared contract manufacturer OUIs (Liteon, USI) — may false positive on non-Flock devices
- `soundthinking_mac_prefixes[]` — SoundThinking/ShotSpotter registrations

### Firmware Web Endpoints (`192.168.4.1`)

Registered in `fySetupServer()` using `esp_http_server`. All routes are GET; responses are chunked where output size is variable.

| Endpoint | Description |
|----------|-------------|
| `/` | HTML dashboard (self-contained, inline JS polled every 2.5s) |
| `/api/detections` | Current session detections as JSON array |
| `/api/stats` | `{total, raven, ble, gps_valid, gps_age, gps_tagged}` |
| `/api/gps` | Accept GPS fix: `?lat=&lon=&acc=` (pushed by dashboard JS) |
| `/api/patterns` | Detection pattern arrays (MAC prefixes, UUIDs, names, mfr IDs) |
| `/api/export/json` | Download current session as JSON |
| `/api/export/csv` | Download current session as CSV |
| `/api/export/kml` | Download current session as KML (GPS-tagged devices only) |
| `/api/history` | Prior session JSON (served from `/spiffs/prev_session.json`) |
| `/api/history/json` | Download prior session as JSON attachment |
| `/api/history/kml` | Download prior session as KML (parsed with ArduinoJson) |
| `/api/clear` | Clear all detections (saves session first, then wipes `fyDet[]`) |

### Flask API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/detections` | GET | Get all detections (filterable) |
| `/api/detections` | POST | Ingest detection from ESP32 serial |
| `/api/gps/ports` | GET | List available serial ports |
| `/api/gps/connect` | POST | Connect GPS dongle |
| `/api/export/csv` | GET | Export as CSV |
| `/api/export/kml` | GET | Export as KML (Google Earth) |
