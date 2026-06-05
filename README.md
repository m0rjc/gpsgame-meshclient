# GPS Game MeshCore Client

Embedded firmware and gateway code for running GPS-based wide-area games over a
[MeshCore](https://github.com/meshcore-dev/MeshCore) LoRa mesh network.

Designed for events such as Scout wide games and orienteering-style navigation
challenges. Players carry inexpensive, GPS-equipped LoRa devices. A gateway
bridges the mesh to a backend game server over the internet. The firmware and
gateway code in this repository are fully open source; the game server that
awards scores and manages players is a separate project and can remain
proprietary.

> **Status: design phase.** Architecture and protocol are documented; firmware
> source has not yet been written. See [`doc/implementation-plan.md`](doc/implementation-plan.md)
> for the phased work plan.

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

## Hardware

| Role    | Board                             | Notes                                    |
|---------|-----------------------------------|------------------------------------------|
| Device  | Seeed T1000-E (nRF52840)          | GPS + LR1110 LoRa built in, compact form factor, low power |
| Gateway | Heltec WiFi LoRa 32 V3 (ESP32-S3) | WiFi for internet connectivity           |

## Repository layout

```
gpsgame-meshclient/
├── doc/                    Design documentation (see below)
├── src/
│   ├── device/             Player tracker firmware (not yet implemented)
│   └── gateway/            Gateway firmware (not yet implemented)
├── variants/
│   ├── heltec_v3/          Heltec WiFi LoRa 32 V3 build environments
│   └── t1000-e/            Seeed T1000-E (nRF52840) build environments
├── mock_game/              Go mock game server (not yet implemented)
├── meshcore/               MeshCore library (git submodule)
└── platformio.ini          Root PlatformIO config and platform base sections
```

## Documentation

All design decisions are captured in [`doc/`](doc/). Read the doc README for an
overview; the key documents are:

| Document | Contents |
|----------|----------|
| [`doc/architecture.md`](doc/architecture.md) | System overview, component roles, startup sequence, event flow |
| [`doc/protocol.md`](doc/protocol.md) | Packet formats, geofence encoding, message sequences |
| [`doc/device-design.md`](doc/device-design.md) | State machine, LED patterns, NVS persistence, module structure |
| [`doc/gateway-design.md`](doc/gateway-design.md) | Gateway responsibilities, WebSocket bridge, deployment tiers |
| [`doc/bridge-design.md`](doc/bridge-design.md) | Bridge microservice design (Go, Postgres, Redis) |
| [`doc/implementation-plan.md`](doc/implementation-plan.md) | Phased work plan and integration guidance |
| [`doc/future-directions.md`](doc/future-directions.md) | Backlog of well-understood stories not yet committed to design |
| [`doc/research/field-research.md`](doc/research/field-research.md) | Walk test results and radio experiments |
| [`doc/research/geofence-packing.md`](doc/research/geofence-packing.md) | Varint delta encoding analysis for `GEOFENCE_SEGMENT` packets |

## Building

This is a [PlatformIO](https://platformio.org/) project. MeshCore is a git
submodule and must be initialised before any build attempt:

```bash
git clone <repo-url> gpsgame-meshclient
cd gpsgame-meshclient
git submodule update --init --recursive
```

Build environments (once firmware is implemented):

```bash
pio run -e t1000e_gps_device        # player device (T1000-E)
pio run -e heltec_v3_gps_gateway    # gateway (Heltec V3)
```

WiFi credentials and other machine-specific settings go in
`platformio.local.ini` (git-ignored). See `CLAUDE.md` for build and variant
details.

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

The **game server project** delivers separately: bridge microservice, game API
extensions, and organiser UI. The interface between the two is defined in
[`doc/gateway-design.md`](doc/gateway-design.md).

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
