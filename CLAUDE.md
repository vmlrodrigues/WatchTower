# WatchTower — Claude Code Primer

## What This Project Is

WatchTower is an Arduino sketch running on a Keyestudio KS0413 ESP32 board. It generates
a WWVB 60 kHz time signal that synchronises a Sharp atomic wall clock. The clock is a US
model that only supports US time zones (Pacific, Mountain, Central, Eastern).

The owner is in Sydney, Australia (AEST/AEDT) and uses WatchTower to make the clock
display the correct Sydney time by transmitting a carefully offset time signal.

The upstream project is at https://github.com/emmby/WatchTower. This is a fork with
local modifications described below.

---

## Hardware

- **ESP32 board:** Keyestudio KS0413 (Jaycar XC3800), 38-pin
- **H-bridge:** L9110H DIP-8
- **Antenna:** ~70-turn coil of 0.25 mm enamelled wire on a 9 mm × 100 mm ferrite rod
- **Signal pin:** GPIO4 (`PIN_ANTENNA = 4`)
- **Arduino board setting:** ESP32 Dev Module
- **Mac serial port:** `/dev/cu.usbserial-0001`
- **Wall clock:** Sharp WWVB atomic clock — Central Time zone selected, DST off

---

## How the Time Offset Works

The Sharp clock is set to **Central Time (UTC−6), DST off**. It subtracts 6 hours from
whatever time it receives in the WWVB signal.

Sydney is **AEST (UTC+10)** in winter and **AEDT (UTC+11)** in summer. To get the clock
to display the correct Sydney time after it subtracts 6 hours, WatchTower must transmit
Sydney local time plus 6 hours:

| Season | Sydney offset | Transmitted offset | Clock displays |
|--------|--------------|-------------------|----------------|
| AEST (winter) | UTC+10 | UTC+16 | UTC+16 − 6 = UTC+10 ✓ |
| AEDT (summer) | UTC+11 | UTC+17 | UTC+17 − 6 = UTC+11 ✓ |

This is achieved via the POSIX timezone string (negative = ahead of UTC):

```cpp
const char* TZ_INFO = "AEST-16AEDT-17,M10.1.0,M4.1.0/3";
```

Australian DST transitions are encoded in the string:
- Starts: first Sunday of October (`M10.1.0`)
- Ends: first Sunday of April at 03:00 (`M4.1.0/3`)

---

## Changes Made to the Upstream Fork

### 1. PIN_ANTENNA changed to GPIO4
```cpp
const int PIN_ANTENNA = 4;
```

### 2. Timezone set for Sydney via offset trick
```cpp
const char* TZ_INFO = "AEST-16AEDT-17,M10.1.0,M4.1.0/3";
```

### 3. WWVB signal now broadcasts local time, not UTC
The upstream code passed `buf_now_utc` (raw UTC) to `wwvbLogicSignal()`. Since the
timezone string only affects the ESP32's local time display and has no effect on UTC,
this was changed to pass `buf_now_local` so the offset in `TZ_INFO` is actually applied
to the transmitted signal.

```cpp
// Before (upstream):
logicValue = wwvbLogicSignal(
    buf_now_utc.tm_hour, buf_now_utc.tm_min, buf_now_utc.tm_sec, ...

// After (this fork):
logicValue = wwvbLogicSignal(
    buf_now_local.tm_hour, buf_now_local.tm_min, buf_now_local.tm_sec, ...
```

### 4. ArduinoMDNS replaced with ESPmDNS
`ArduinoMDNS` is effectively unmaintained. It was replaced with `ESPmDNS`, which is
bundled into the ESP32 Arduino core and maintained by Espressif. This removed the
need for a `WiFiUDP` object and a manual `mdns.run()` call in `loop()`.

---

## Dependencies

Install via Arduino Library Manager:

- Adafruit NeoPixel ~1.15.5
- ESPUI ~2.2.4
- ESP32Async / ESPAsyncWebServer ~3.11.0
- ESP32Async / AsyncTCP ~3.4.10
- WiFiManager ~2.0.17
- ESPmDNS (built into ESP32 Arduino core — no separate install needed)

---

## Commit Message Convention

This project follows [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/).

### Format

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Types

| Type | Use for |
|------|---------|
| `feat` | A new feature |
| `fix` | A bug fix |
| `docs` | Documentation changes only |
| `style` | Formatting, no logic change |
| `refactor` | Code restructure, no feature or fix |
| `perf` | Performance improvement |
| `test` | Adding or updating tests |
| `build` | Build system or dependency changes |
| `ci` | CI/CD configuration changes |
| `chore` | Routine tasks, maintenance |

### Breaking Changes

Append `!` after the type, or add a `BREAKING CHANGE:` footer:

```
feat!: drop support for legacy signal format

BREAKING CHANGE: TZ_INFO must now use the POSIX negative-offset convention
```

### Examples

```
feat: transmit local time instead of UTC in WWVB signal
fix: correct DST bit calculation for Southern Hemisphere
build: update AsyncTCP to 3.4.10
chore: replace ArduinoMDNS with ESPmDNS from ESP32 core
docs: add offset calculation table to CLAUDE.md
```
