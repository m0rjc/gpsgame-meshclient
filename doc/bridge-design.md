# Bridge Microservice Design

This document describes the design of the bridge microservice — the Go service that sits between the MeshCore gateway and the game backend.

> **Status: stub.** Sections marked TODO need to be filled in as the design matures.

## Role

The bridge is the authoritative store for the hardware device → game/player/team mapping. It:

- accepts device events from one or more gateways (via WebSocket or HTTP)
- owns the enrollment lifecycle: pending → active → unenrolled
- forwards game-relevant events (arrival, button) to the game backend API
- routes device-bound replies (fence segments, ACKs, UNENROLL) back through the correct gateway
- provides an organiser-facing status API consumed by the game UI

The gateway↔bridge interface (WebSocket JSON envelopes, HTTP endpoints) is defined in `gateway-design.md`.

## Tech Stack

- **Language**: Go
- **Persistent store**: Postgres — authoritative mappings, event log, gateway registry
- **Cache / queue**: Redis — ephemeral state, delivery queue, last-seen gateway, WebSocket presence
- **Deployment**: Kubernetes (production) or `docker-compose` (local dev)

## Data Model

### Postgres

See `gateway-design.md` for the `nodes` and `gateways` table definitions.

TODO: add event log table, enrollment request table, game/team reference tables.

### Redis

| Key pattern | Type | Contents |
|-------------|------|----------|
| `device:{node_id}:gateway` | string | last-seen gateway ID |
| `gateway:{gateway_id}:ws` | string | WebSocket connection status (`connected`/`disconnected`) |
| `deliver:{gateway_id}` | list | pending `deliver` commands queued while gateway is offline |
| `device:{node_id}:state` | hash | cached enrollment state for fast lookup |

TODO: define TTLs and eviction policy for each key type.

## API Endpoints

### Gateway-facing

TODO: define request/response shapes, authentication (per-gateway API key), versioning.

- `POST /v1/events` — receive device event from gateway (INIT_REQ, EVENT_REPORT, SYNC_STATUS)
- `WS /v1/gateway/{gateway_id}` — persistent WebSocket for bridge→gateway delivery commands

### Mock game / game backend-facing

TODO: define the contract the real game backend must implement. The mock game server is the reference implementation.

- `POST /enroll` — bridge calls this when a device sends INIT_REQ; game returns game_id + fence list
- `POST /event` — bridge calls this for arrival (0x00) and button (0x01) events
- `GET /status` — organiser UI polls or subscribes for device state

### Internal / admin

TODO: decide which of these are needed for production vs dev only.

- `GET /admin/devices` — list all known devices and their current state
- `POST /admin/assign/{node_id}` — manually trigger assignment (mirrors mock game manual mode)
- `GET /health` — liveness/readiness probe for Kubernetes

## Enrollment Flow

TODO: write out the full state transitions the bridge manages, mirroring the device state machine.

High-level:
1. Gateway forwards INIT_REQ → bridge records device as `pending`, calls `POST /enroll` on game backend
2. Game backend returns fence list (or defers — TODO: how does deferred assignment work here?)
3. Bridge sends fence segments to gateway via WebSocket `deliver` command
4. Gateway completes SYNC_STATUS handshake with device; reports ready to bridge
5. Bridge updates device state → `active`

## Gateway Connectivity

TODO: detail the WebSocket handler: connection registry, heartbeat, reconnect behaviour, delivery queue drain on reconnect.

See `gateway-design.md` for the deployment model (local LAN / VPN / direct ingress).

## Deployment

### Local dev

TODO: provide a `docker-compose.yml` with bridge, Postgres, and Redis. Bridge connects to a gateway running on the local network (or a gateway stub for unit testing).

### Kubernetes

TODO: define manifests or Helm chart. Expected services:
- bridge `Deployment` + `Service`
- Postgres (or external managed instance)
- Redis (or external managed instance)
- `ConfigMap` / `Secret` for gateway API keys, game backend URL, DB credentials

Environment variables the bridge reads (TODO: finalise):

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | Postgres connection string |
| `REDIS_URL` | Redis connection string |
| `GAME_API_URL` | Base URL of the game backend (or mock game server) |
| `GATEWAY_API_KEY` | Shared key expected from gateways (TODO: per-gateway keys) |
| `LOG_LEVEL` | `slog` level: `debug`, `info`, `warn`, `error` |

## Logging

Use `slog` throughout. Structured fields on every log line: `gateway_id`, `node_id`, `friendly_id` where available. Event intake and delivery commands should log at `INFO`; retries and queue operations at `DEBUG`; errors at `ERROR`.

## TODO Summary

- [ ] Define event log and enrollment request table schemas in Postgres
- [ ] Define Redis TTLs and eviction policy
- [ ] Specify gateway WebSocket envelope schema and version it (coordinate with `gateway-design.md`)
- [ ] Specify game backend API contract (mock game server is the reference)
- [ ] Design deferred assignment flow (game not ready to assign immediately)
- [ ] Define per-gateway API key provisioning
- [ ] Write `docker-compose.yml` for local dev
- [ ] Write Kubernetes manifests / Helm chart
- [ ] Decide on organiser status API: polling vs SSE vs WebSocket
