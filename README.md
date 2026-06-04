# GPS Game MeshCore Client

Embedded firmware and gateway code for running GPS-based wide-area games over a
[MeshCore](https://github.com/meshcore-dev/MeshCore) LoRa mesh network.

Designed for events such as Scout wide games and orienteering-style navigation
challenges. Players carry inexpensive, GPS-equipped LoRa devices. A gateway
bridges the mesh to a backend game server over the internet. The firmware and
gateway code in this repository are fully open source; the game server that
awards scores and manages players is a separate project and can remain
proprietary.

## How it works

```
  Player device                Gateway                  Game server
  ┌─────────────┐   LoRa mesh  ┌──────────────┐  HTTPS  ┌────────────────┐
  │ GPS         │◄────────────►│ MeshCore     │◄───────►│ Bridge micro-  │
  │ geofence    │              │ room server  │         │ service        │
  │ state mach. │              │ + WebSocket  │         │                │
  └─────────────┘              └──────────────┘         └────────────────┘
```

1. A player device powers on, listens for a gateway advertisement, then sends an
   `INIT_REQ` to enrol in the game.
2. The game organiser assigns the device via the game UI; the gateway delivers
   geofence segments to the device over the mesh.
3. The device evaluates GPS fixes against stored geofences and reports arrivals,
   button presses, and periodic location pings to the gateway.
4. The gateway forwards events to the bridge microservice, which maps hardware
   IDs to player tokens and calls the game API.

Full protocol and state machine details are in [`doc/`](doc/).

## Repository layout

```
gpsgame-meshclient/
├── doc/                    Design documentation (architecture, protocol, etc.)
├── src/
│   ├── device/             Player tracker firmware (state machine, GPS, fences)
│   └── gateway/            Gateway firmware (mesh server + WebSocket bridge)
├── variants/
│   ├── heltec_v3/          Heltec WiFi LoRa 32 V3 build environments
│   └── t1000-e/            Seeed T1000-E (nRF52840) build environments
├── mock_game/              Go mock game server for development and field testing
├── meshcore/               MeshCore library (git submodule)
└── platformio.ini          Root PlatformIO config and platform base sections
```

## Project boundary

This repository delivers:

- **Device firmware** — state machine, GPS evaluation, geofence storage, LED
  indicators, mesh request/response handling
- **Gateway firmware** — enrollment handler, event forwarder, WebSocket bridge
  client, gateway advertisement broadcast
- **Gateway ↔ bridge interface** — WebSocket JSON envelope schema and HTTP
  endpoint definitions
- **Mock game server** (Go) — a self-contained stand-in for the real game backend
  used during development and field testing

The **game server project** delivers separately:
- Bridge microservice (Go, Postgres, Redis)
- Game API extensions to support sensor enrollment and event intake
- Organiser UI to assign devices to games and monitor device state

The interface between the two is defined in [`doc/gateway-design.md`](doc/gateway-design.md).

## Hardware

| Role    | Board                             | Notes                                    |
|---------|-----------------------------------|------------------------------------------|
| Device  | Seeed T1000-E (nRF52840)          | GPS + LR1110 LoRa built in, compact form factor, low power |
| Gateway | Heltec WiFi LoRa 32 V3 (ESP32-S3) | WiFi for internet connectivity           |

The system uses board-variant configs and can be extended to other MeshCore-supported boards, but these are the two targeted platforms.

## Getting started

### Prerequisites

- [PlatformIO](https://platformio.org/) (CLI or VS Code extension)
- Git with submodule support

### Clone and initialise

```bash
git clone <repo-url> gpsgame-meshclient
cd gpsgame-meshclient
git submodule update --init --recursive
```

### Build

List available environments:

```bash
pio project config --json-output | python -c \
  "import sys,json; [print(e) for e in json.load(sys.stdin)[0][1]]"
```

Build a specific target:

```bash
pio run -e t1000e_gps_device        # player device (T1000-E)
pio run -e heltec_v3_gps_gateway    # gateway (Heltec V3)
```

Upload and monitor:

```bash
pio run -e t1000e_gps_device -t upload
pio device monitor -e t1000e_gps_device
```

### Local overrides

Create `platformio.local.ini` (git-ignored) for per-machine settings:

```ini
[env:heltec_v3_gps_gateway]
upload_port = /dev/ttyUSB0
build_flags =
    ${heltec_v3_gps_gateway.build_flags}
    -D WIFI_SSID='"MyNetwork"'
    -D WIFI_PWD='"MyPassword"'
    -D ADMIN_PASSWORD='"secret"'
```

## Board variants

Board-specific configurations live under `variants/*/platformio.ini`. Each file
defines a board base section and one or more `[env:...]` build environments.

The base sections extend `[esp32_base]`, `[nrf52_base]`, etc. from the root
`platformio.ini`. `MC_VARIANT="<board>"` tells the MeshCore library build script
(`meshcore/build_as_lib.py`) which board HAL to compile from
`meshcore/variants/<board>/`.

To add a new board:

1. Create `variants/<board-name>/platformio.ini`.
2. Define a `[<board-name>]` section extending the appropriate platform base.
3. Set `MC_VARIANT='"<board-name>"'` — this must match a directory under
   `meshcore/variants/`.
4. Add `[env:<board-name>_gps_device]` and/or `[env:<board-name>_gps_gateway]`
   environments with `build_src_filter = +<device>` or `+<gateway>`.

## Documentation

| Document | Contents |
|----------|----------|
| [`doc/architecture.md`](doc/architecture.md) | System overview, component roles, startup sequence, event flow |
| [`doc/protocol.md`](doc/protocol.md) | Packet formats, geofence encoding, message sequences |
| [`doc/device-design.md`](doc/device-design.md) | State machine, LED patterns, NVS persistence, module structure |
| [`doc/gateway-design.md`](doc/gateway-design.md) | Gateway responsibilities, WebSocket bridge, deployment tiers |
| [`doc/bridge-design.md`](doc/bridge-design.md) | Bridge microservice design (Go, Postgres, Redis) |
| [`doc/implementation-plan.md`](doc/implementation-plan.md) | Phased work plan and integration guidance |

## License

Copyright 2026 Richard Corfield.
Licensed under the [Apache License, Version 2.0](LICENSE).

The Apache 2.0 licence was chosen specifically because it includes an explicit
patent grant from contributors. Any entity that contributes to this project
automatically grants users a licence to patents that their contribution
necessarily infringes, and forfeits that grant if they initiate patent
litigation against any project user. This is particularly relevant given the
potential for checkpoint scanning techniques (GPS geofencing, NFC station
reading) to attract patent claims from contributors with large portfolios.
