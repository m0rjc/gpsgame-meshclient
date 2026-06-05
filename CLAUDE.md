# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build System

This is a PlatformIO project targeting embedded firmware. There is no Makefile, CMake, or other build system.

**First-time setup** — MeshCore is a git submodule and must be initialised before any build attempt:

```sh
git submodule update --init --recursive
```

**Build a specific environment:**
```sh
pio run -e <environment>
```

**Flash to a connected board:**
```sh
pio run -e <environment> -t upload
```

**Monitor serial output:**
```sh
pio device monitor -e <environment>
```

**Target environments** (defined in `variants/*/platformio.ini`):
| Environment | Board | Role |
|---|---|---|
| `t1000e_gps_device` | Seeed T1000-E (nRF52840) | Player device — GPS+LoRa, compact form factor |
| `heltec_v3_gps_gateway` | Heltec WiFi LoRa 32 V3 (ESP32-S3) | Gateway — WiFi bridge to game server |

The `heltec_v3_gps_device` environment exists in the variant config but is not a target for this project (ESP32 power consumption rules it out as a player device).

## Local Configuration Overrides

WiFi credentials, admin passwords, upload ports, and other machine-specific values belong in `platformio.local.ini` (already git-ignored). Copy the relevant `build_flags` from the variant `platformio.ini` and override there. Never commit secrets to the variant files.

## Adding Board Variants

1. Create `variants/<board-name>/platformio.ini`
2. Define a board base section that `extends` the appropriate platform base (`esp32_base`, `nrf52_base`, `rp2040_base`, `arduino_base`)
3. Set `MC_VARIANT = <variant-name>` in `build_flags` — MeshCore's `build_as_lib.py` uses this to select the correct HAL files from `meshcore/variants/<variant>/`
4. Define `[env:...]` environments for device and/or gateway targets using `build_src_filter` to include `src/device/` or `src/gateway/`

## MeshCore Library

MeshCore lives in `meshcore/` as a git submodule. PlatformIO picks it up via `lib_deps = file://./meshcore`. MeshCore's `build_as_lib.py` build script selects board HALs and helpers at compile time based on `MC_VARIANT` and platform defines (`ESP32_PLATFORM`, `NRF52_PLATFORM`, etc.).

## Project Structure

- `src/device/` — device firmware (state machine, GPS, geofence evaluation, mesh requests)
- `src/gateway/` — gateway firmware (room server, WebSocket bridge client)
- `variants/` — per-board PlatformIO configs
- `meshcore/` — MeshCore library (git submodule; must be initialised)
- `mock_game/` — placeholder for Go mock game server (not yet implemented)
- `doc/` — design documents

## Design Documentation

All major design decisions are captured in `doc/`. Read these before implementing or proposing significant changes:

- `doc/architecture.md` — system overview, component roles, startup sequence
- `doc/protocol.md` — packet formats, message types, coordinate encoding (`latlon_t` = 24-bit, ~1.2 cm resolution)
- `doc/device-design.md` — state machine (Uninitialised → Unenrolled → Syncing → Active), NVS persistence, LED patterns, retry parameters
- `doc/gateway-design.md` — bridge design, WebSocket-over-TLS (preferred) vs HTTP polling fallback
- `doc/implementation-plan.md` — phased work plan and mock game server API (`/enroll`, `/assign/{node_id}`, `/event`, `/status`)
- `doc/future-directions.md` — backlog of well-understood stories not yet committed to design
- `doc/research/field-research.md` — walk test results and radio experiments (informs frequency and antenna choices)
- `doc/research/geofence-packing.md` — varint delta encoding analysis for `GEOFENCE_SEGMENT` packets

## Current Status

Firmware source code has not yet been written. `src/device/` and `src/gateway/` contain only `.gitkeep` placeholders. The mock game server (`mock_game/`) is also a placeholder. Implementation follows the phased plan in `doc/implementation-plan.md`.

## LoRa Parameters

Default radio config (defined in root `platformio.ini`, override per-variant if needed):
- Frequency: 869.525 MHz (preferred for game use — see below)
- Bandwidth: 62.5 kHz
- Spreading factor: 8

**Frequency note:** The project was initialised with 869.618 MHz (the main MeshCore community channel). Field testing (2026-06-04) showed that channel carries heavy ambient mesh flood traffic — 4.7% TX duty cycle, 54% duplicate rate — almost entirely from relaying other people's traffic rather than game messages. Retuning to 869.525 MHz dropped TX duty cycle to 0.36% (13× reduction) with zero duplicates. The `platformio.ini` default should be updated to 869.525 MHz. Note: 869.525 MHz is also used by Meshtastic; test in your area before committing to it as the final game frequency.
