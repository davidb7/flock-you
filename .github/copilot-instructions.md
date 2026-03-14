# Flock-You Copilot Instructions

## Project Overview

Flock-You is a BLE-only surveillance device detector targeting Flock Safety cameras, Raven gunshot detectors, and related hardware. It runs on a **Seeed Studio XIAO ESP32-S3** and has two components:

- **Firmware** (`src/main.cpp`): Arduino/PlatformIO C++ — the entire firmware is a single file
- **Flask companion app** (`api/`): Python desktop dashboard for live serial ingestion and data analysis

---

## Build & Flash Commands (Firmware)

Requires [PlatformIO](https://platformio.org/).

```bash
pio run                  # build
pio run -t upload        # build and flash
pio device monitor       # open serial monitor at 115200 baud
pio run -t uploadfs      # flash SPIFFS filesystem
```

There is no test suite for the firmware.

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

**Companion mode** switches this behavior: when the DeFlock mobile app connects via BLE GATT or a serial host is detected, `fyCompanionChangePending` is set to `true` in the BLE callback, and `loop()` calls `fyOnCompanionChange()` to disable WiFi AP and boost BLE scan duty cycle. This deferred pattern exists because WiFi operations must not be called from BLE callback context.

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

### Firmware (`src/main.cpp`)

- **All globals/functions prefixed `fy`**: `fyDet`, `fyMutex`, `fyServer`, `fyGPSLat`, `fyBeep()`, etc.
- **Section headers**: `// ===...===` comment blocks divide the file into logical sections (CONFIGURATION, DETECTION PATTERNS, AUDIO SYSTEM, etc.)
- **MAC comparison**: always `strncasecmp(mac_str, prefix, 8)` for 8-character OUI prefix matching
- **JSON name sanitization**: `"` and `\` characters in device names are replaced with `_` before storage
- **Mutex discipline**: always acquire `fyMutex` with a 100ms timeout; release in all code paths including early returns
- **Audio**: `fyBeep()` and `fyCaw()` are no-ops when `fyBuzzerOn == false`; all audio must check this

### OUI Reference Files

- `oui.txt` (root) — IEEE OUI database for reference when updating MAC prefix lists
- `api/oui.txt` — copy used by the Flask app for MAC vendor lookup
- `datasets/raven_configurations.json` — Raven BLE GATT characteristic map organized by firmware version (1.1.7, 1.2.0, 1.3.1); used as reference when updating Raven detection logic

### MAC Prefix Confidence Tiers

When adding new OUIs, place them in the appropriate array:
- `flock_mac_prefixes[]` — confirmed Flock Safety IEEE registrations or exclusive-use OUIs
- `flock_mfr_mac_prefixes[]` — shared contract manufacturer OUIs (Liteon, USI) — may false positive on non-Flock devices
- `soundthinking_mac_prefixes[]` — SoundThinking/ShotSpotter registrations

### Flask API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/detections` | GET | Get all detections (filterable) |
| `/api/detections` | POST | Ingest detection from ESP32 serial |
| `/api/gps/ports` | GET | List available serial ports |
| `/api/gps/connect` | POST | Connect GPS dongle |
| `/api/export/csv` | GET | Export as CSV |
| `/api/export/kml` | GET | Export as KML (Google Earth) |
