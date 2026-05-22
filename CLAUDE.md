# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

An Arduino library that connects ESP32/SAMD microcontrollers to Acaia and Bookoo espresso scales over BLE, using the ArduinoBLE library. The primary artifact is the `AcaiaArduinoBLE` library itself, plus a complete embedded application (`examples/shotStopper/`) that automates espresso shot stopping by weight.

## Building and Flashing

**Compiling:** Use the Arduino IDE or `arduino-cli`. Target boards are ESP32S3, ESP32C3, or Arduino Nano ESP32 (SAMD). The `shotStopper` example requires the `ArduinoBLE` and `EEPROM` libraries.

**Flashing pre-built binaries** (no compiler needed):
```
cd examples/shotStopper/build
flash.bat   # interactive — picks COM port, prompts for machine type
```
Requires Python with `esptool` installed (`pip install esptool`). The script puts the board in download mode (hold BOOT, plug USB, release BOOT), erases flash, then writes bootloader + partitions + app at the correct offsets (`0x0`, `0x8000`, `0x10000`).

**OTA update** (after initial flash): Use the BLE app to set WiFi credentials and trigger OTA mode; then navigate to the device IP for a web-based firmware upload.

## Library Architecture

`AcaiaArduinoBLE.h/.cpp` — the library proper. Three scale variants are handled via the `scale_type` enum:
- `OLD` — pre-2021 Lunar (characteristic UUID `2a80`, 10/14-byte weight packets)
- `NEW` — 2021+ Lunar and Pyxis (long UUID `49535343-...`, 13/17-byte packets)
- `GENERIC` — Bookoo Themis, Felicita Arc (characteristic UUID `ff11`/`ff12`, 20-byte packets)

The variant is detected at connection time by probing which characteristic is subscribable. Heartbeats are only required for OLD/NEW types (every 2750 ms); GENERIC scales do not need them.

`newWeightAvailable()` must be called every loop iteration — it pulls the BLE notification buffer and parses the weight. It also handles timeout detection (5 s without a packet triggers reconnect). `getWeight()` only returns the last parsed value; it does not poll.

**IMPORTANT — two copies of the library must stay in sync:**
`AcaiaArduinoBLE.cpp/.h` at the repo root is the installable Arduino library. `examples/shotStopper/AcaiaArduinoBLE.cpp/.h` is a local copy compiled directly with the sketch (Arduino builds all `.cpp` files in the sketch directory). Any change to the library must be applied to both copies.

### Scale BLE Packet Formats

#### OLD type (`scale_type::OLD`, pre-2021 Lunar)
- Write/read characteristic: `2a80`
- Weight packet length: 10 or 14 bytes
- Weight bytes `[2:3]` (little-endian uint16), scale exponent at `[6]`, sign at `[7]` (bit 1 set = negative)
- Formula: `((input[3]<<8)|input[2]) / pow10[input[6]] * (input[7]&0x02 ? -1 : 1)`
- Requires IDENTIFY + NOTIFICATION_REQUEST on connect, heartbeat every 2750 ms

#### NEW type (`scale_type::NEW`, 2021+ Lunar, Pyxis)
- Write characteristic: `49535343-8841-43f4-a8d4-ecbe34729bb3`
- Read/notify characteristic: `49535343-1e4d-4bd9-ba61-23c647249616`
- Weight packet length: 13 or 17 bytes, `input[4] == 0x05` identifies weight frame
- Weight bytes `[5:6]` (little-endian uint16), scale exponent at `[9]`, sign at `[10]` (bit 1 set = negative)
- Formula: `((input[6]<<8)|input[5]) / pow10[input[9]] * (input[10]&0x02 ? -1 : 1)`
- Requires IDENTIFY + NOTIFICATION_REQUEST on connect, heartbeat every 2750 ms

#### GENERIC type (`scale_type::GENERIC`, Bookoo, Felicita Arc)
- Write characteristic: `ff12`, read/notify characteristic: `ff11`
- Weight packet length: 20 bytes
- Packet structure (0-indexed):
  - `[0:1]` header `0x03 0x0B`
  - `[2:4]` elapsed milliseconds (24-bit big-endian)
  - `[5]` weight unit (gram/ounce)
  - `[6]` sign byte (`0x2D` = ASCII `-` = negative)
  - `[7:9]` weight × 100 (24-bit big-endian, grams)
  - `[10:11]` flow rate × 100 (16-bit big-endian, g/s)
  - `[12]` battery percentage
  - `[13:14]` standby minutes
  - `[15:16]` buzzer level, flow smoothing toggle
  - `[17:19]` padding + XOR checksum
- Formula: `((input[7]<<16)|(input[8]<<8)|input[9]) / 100.0 * (input[6]==0x2D ? -1 : 1)`
- **Does NOT require** IDENTIFY, NOTIFICATION_REQUEST, or heartbeat — subscribe to `ff11` and data flows immediately

### GENERIC/Bookoo Command Protocol

Commands are 6 bytes: `[0x03, 0x0A, CMD, RESERVED, PARAM, CHECKSUM]`

**Checksum rule:** XOR of all 5 payload bytes (indices 0–4).
Example: tare (`CMD=0x01`, `RESERVED=0x00`, `PARAM=0x00`) → `0x03^0x0A^0x01^0x00^0x00 = 0x08` ✓

| Command | CMD byte | Notes |
|---------|----------|-------|
| Tare | `0x01` | Works on Bookoo and Felicita Arc |
| Beep (level 0–5) | `0x02` | `PARAM` = level 0–5; Bookoo-only |
| Timer start | `0x04` | Works on Bookoo and Felicita Arc |
| Timer stop | `0x05` | Works on Bookoo and Felicita Arc |
| Timer reset | `0x06` | Works on Bookoo and Felicita Arc |
| Tare + start timer | `0x07` | Bookoo-only atomic command |
| Flow smoothing toggle | `0x08` | Bookoo-only |

**Note on timer command checksums:** The `START/STOP/RESET_TIMER_GENERIC` byte arrays in the source were originally reverse-engineered from Felicita Arc (via Beanconqueror) and their last bytes do not match the Bookoo XOR formula. They appear to work because Bookoo is lenient about checksum validation for these commands. Do not "fix" these unless you have hardware to verify the result — the Bookoo-specific commands (`TARE_START_TIMER_BOOKOO`, `BEEP_LEVEL_BOOKOO`) use the correct XOR formula and were independently verified.

## ShotStopper Application (`examples/shotStopper/`)

The shotStopper is a self-contained firmware for the ShotStopper PCB (ESP32S3/C3). Key design points:

**GPIO pin mapping** varies by board variant — see the `#ifdef ARDUINO_ESP32S3_DEV / ESP32C3_DEV` block in `shotStopper.ino`.

**Four pre-built binary variants** correspond to the two boolean compile-time flags encoded in the filename:
- `mom0`/`mom1` = `momentary` false/true
- `reed0`/`reed1` = `reedSwitch` false/true

### EEPROM Layout

Total size: 78 bytes. Signature byte `0xAA` at address 0 guards initialization (missing = reinitialize with defaults).

| Address | `#define` | Type | Range / notes |
|---------|-----------|------|---------------|
| 0 | `SIGNATURE_ADDR` | uint8 | `0xAA` = initialized |
| 1 | `ENABLED_ADDR` | uint8 | 0 or 1 |
| 2 | `WEIGHT_ADDR` | uint8 | Goal weight in whole grams (0–255) |
| 3 | `OFFSET_ADDR` | uint8 | `weightOffset * 10`, so range 0.0–25.5 g. **Must be ≥ 0 before writing** — the value is stored unsigned; a negative float would wrap. Clamp to `max(0.0f, value)` before `EEPROM.write()`. |
| 4 | `MOMENTARY_ADDR` | uint8 | 0 or 1 |
| 5 | `REEDSWITCH_ADDR` | uint8 | 0 or 1 |
| 6 | `AUTOTARE_ADDR` | uint8 | 0 or 1 |
| 7 | `MIN_SHOT_DURATION_S_ADDR` | uint8 | seconds |
| 8 | `MAX_SHOT_DURATION_S_ADDR` | uint8 | seconds |
| 9 | `DRIP_DELAY_S_ADDR` | uint8 | seconds |
| 10–41 | `WIFI_SSID_ADDR` | char[32] | null-terminated |
| 42–73 | `WIFI_PASS_ADDR` | char[32] | null-terminated |
| 74 | `TARE_START_TIMER_ADDR` | uint8 | 0 or 1; Bookoo-only feature |
| 75 | `BEEP_ADDR` | uint8 | 0 or 1; Bookoo-only feature |
| 76 | `BEEP_LEVEL_ADDR` | uint8 | 0–5; Bookoo-only feature |
| 77 | *(reserved)* | — | Next available address |

**Adding new settings:** increment `EEPROM_SIZE`, pick address 77+, add to both `loadOrInitEEPROM()` branches, and add a BLE characteristic if it needs to be app-configurable.

### ShotStopper BLE Peripheral (Companion App Interface)

Service UUID: `0x0FFE`. The companion app (`icapurro/shotStopperCompanionApp`) reads and writes characteristics `0xFF10`–`0xFF24`. Characteristics `0xFF25`–`0xFF27` are Bookoo-specific extensions not yet in the companion app.

**Do not change existing UUIDs or their data widths** — the companion app hardcodes all UUIDs and assumes single-byte values for all non-string characteristics.

| UUID | Name | R/W | EEPROM addr | Type | Notes |
|------|------|-----|-------------|------|-------|
| `0xFF10` | enabled | RW | 1 | uint8 0/1 | Disables scale connection and relay when 0 |
| `0xFF11` | goalWeight | RW | 2 | uint8 g | Integer grams only |
| `0xFF12` | reedSwitch | RW | 5 | uint8 0/1 | Switches input GPIO between IN and REED_IN |
| `0xFF13` | momentary | RW | 4 | uint8 0/1 | Momentary vs latching paddle type |
| `0xFF14` | autoTare | RW | 6 | uint8 0/1 | Tare on brew start and 3 s after latching |
| `0xFF15` | minShotDurationS | RW | 7 | uint8 s | Prevents stopping during flush |
| `0xFF16` | maxShotDurationS | RW | 8 | uint8 s | Safety cutoff for latching machines |
| `0xFF17` | dripDelayS | RW | 9 | uint8 s | Post-stop settle time before offset measurement |
| `0xFF18` | firmwareVersion | R | — | uint8 | Currently `2`; app uses this to gate features |
| `0xFF19` | scaleStatus | R+Notify | — | uint8 | `0`=disconnected, `1`=connected |
| `0xFF20` | shotStatus | R+Notify | — | uint8 | `0`=idle, `1`=brewing |
| `0xFF21` | otaModeRequested | RW | — | uint8 0/1 | Writing 1 starts WiFi + web OTA server |
| `0xFF22` | wifiSsid | RW | 10 | char[32] | Written by app before OTA |
| `0xFF23` | wifiPassword | W | 42 | char[32] | Write-only for security |
| `0xFF24` | wifiIp | R+Notify | — | char[16] | `"disconnected"` until WiFi connects |
| `0xFF25` | tareStartTimer | RW | 74 | uint8 0/1 | Use atomic Bookoo tare+start command |
| `0xFF26` | beep | RW | 75 | uint8 0/1 | Enable scale beep feedback (Bookoo) |
| `0xFF27` | beepLevel | RW | 76 | uint8 0–5 | Bookoo beep level |

### Shot Logic Flow

1. Button/reed press detected (active-low GPIO, 31-sample majority-vote debounce)
2. `setBrewingState(true)` — resets timer, tares scale (or sends atomic `tareStartTimer` command for Bookoo)
3. Each weight notification stores `(time_s, weight)` into an N=10 entry ring buffer (`shot.weight[datapoints % N]`)
4. `calculateEndTime()` fits a least-squares line through the last N points: `m = (N·ΣXY - ΣX·ΣY) / (N·ΣX² - (ΣX)²)`, `b = meanY - m·meanX`
5. Predicted end = `(goalWeight - weightOffset - b) / m`; falls back to `maxShotDurationS` if weight < 10 g or fewer than N readings
6. When `shotTimer >= expected_end_s`, relay fires (latching: GPIO held HIGH; momentary: 300 ms pulse)
7. After `dripDelayS` seconds, final weight is read; `weightOffset` is auto-adjusted by `currentWeight - goalWeight`, clamped to `[0, MAX_OFFSET]` and persisted to EEPROM

**`weightOffset` semantics:** the relay fires at `goalWeight - weightOffset`, so a positive offset causes early stopping to account for in-flight drip. If the final weight consistently overshoots, the offset increases; if it undershoots, it decreases but never goes below 0 (a negative offset would make the relay fire *after* goal weight, which is never useful, and the EEPROM storage is unsigned).

**Ring buffer:** `Shot.weight[N]` and `Shot.time_s[N]` are ring buffers — write index is `datapoints % N`, `datapoints` is the total count (never reset mid-shot). `calculateEndTime` iterates indices `(datapoints - N + i) % N` for `i` in `[0, N)` to read the oldest-to-newest window. Only the last N points are needed; storing the full shot history would waste ~7.9 KB and had no bounds check.

## Versioning

Version string appears in three places — keep them in sync:
1. `library.properties` — `version=`
2. `AcaiaArduinoBLE.h` — `#define LIBRARY_VERSION`
3. Binary filenames in `examples/shotStopper/build/` (`shotStopper_*_vX.Y.Z_*.bin`)

## Adding a New Scale

1. Add a new `scale_type` enum value in `AcaiaArduinoBLE.h`
2. Define characteristic UUIDs as `#define` constants
3. In `AcaiaArduinoBLE::init()`, add a detection branch after the existing `canSubscribe()` checks
4. Add the scale's BLE name prefix (first 5 chars) to `isScaleName()`
5. Add command byte arrays for tare/timer if they differ from existing variants
6. If the scale does **not** need the Acaia IDENTIFY + NOTIFICATION_REQUEST handshake, include the new type in the `_type != GENERIC` guard in `init()`
7. Add a parsing branch in `newWeightAvailable()` for the packet format
8. Apply the same changes to **both** `AcaiaArduinoBLE.cpp` (root) and `examples/shotStopper/AcaiaArduinoBLE.cpp`

## Known Constraints and Gotchas

- **`weightOffset` EEPROM is unsigned:** stored as `uint8_t(weightOffset * 10)`. Always clamp to `max(0.0f, value)` before writing. A negative float silently wraps (e.g. −0.5 → stored as 251 → read back as 25.1 g).
- **`goalWeight` is integer-only:** stored as a single uint8 byte, exposed as a uint8 BLE characteristic. Fractional gram targets are not supported without a breaking protocol change.
- **Watchdog is 30 s:** `scale.init()` blocks for up to 10 s (scan timeout) plus several seconds for connect + attribute discovery. Do not reduce the watchdog below ~20 s without also making init non-blocking.
- **`init()` is blocking:** the 10-second scan loop in `AcaiaArduinoBLE::init()` prevents `BLE.poll()` from running, so the companion app will lose its BLE peripheral connection during reconnect. This is a known limitation.
- **`pow()` is replaced by `powLUT`:** `pow(10, x)` was a ~200-cycle FP call on every weight packet. The lookup table `powLUT[] = {1, 10, 100, 1000}` covers all real exponent values (0–2); index is clamped to 3 as a safety bound.
- **Felicita Arc timer command checksums:** `START/STOP/RESET_TIMER_GENERIC` checksums do not match the Bookoo XOR formula. They were reverse-engineered from Felicita Arc and work because Bookoo validates checksums leniently for these commands. Do not "fix" them without hardware to verify.
- **`Serial.println` in the library:** the library prints diagnostics unconditionally (not behind `_debug`). This is intentional for embedded debugging but adds latency on every BLE event on slow UART rates — `Serial.begin(9600)` is the default in shotStopper.
