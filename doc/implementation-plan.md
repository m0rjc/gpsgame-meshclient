# GPS Game Implementation Plan

## Goal

Create a MeshCore-based GPS game example with:
- device state machine (uninitialised → unenrolled → syncing → active)
- geofence delivery and sync handshake
- player arrival, button check-in, and periodic location ping
- unenrollment and mid-game fence updates
- server-side HTTP bridge to a game backend

## Recommended bases

- **Device firmware**: `examples/simple_sensor`
- **Server firmware**: `examples/simple_room_server`
- **Bridge microservice**: small Go HTTP service with Postgres/Redis and game backend integration

## Project boundary

This MeshCore project delivers:
- device firmware (state machine, GPS, geofence evaluation, LED indicators)
- server/gateway firmware (enrollment handler, event forwarder, advertisement broadcast)
- the gateway↔bridge interface specification (WebSocket JSON envelopes, HTTP endpoints)
- **mock game server** (Go) — a self-contained stand-in for the real game backend (see below)

The **game development project** delivers:
- bridge microservice (Go) — implements the gateway↔bridge interface
- game API extensions — new endpoints to support sensor enrollment and event intake
- organiser UI — assigns devices to games, monitors device state

The interface between the two projects is defined in `gateway-design.md`. Agree and freeze the JSON envelope schema before either side begins implementation.

## Mock game server

A small Go service included in this project that stands in for the real game backend during development and field testing. It removes the dependency on the game project being ready and serves as an executable specification of the API contract the real game must implement.

**Reads on startup:**
```json
{
  "game_id": 1,
  "name": "Scout Wide Game",
  "fences": [
    { "id": 1, "name": "Start", "lat": 51.5074, "lon": -0.1278, "radius_m": 50 },
    { "id": 2, "name": "The Old Mill", "lat": 51.5080, "lon": -0.1290, "radius_m": 30 }
  ],
  "devices": [
    { "node_id": 1234, "friendly_id": 1, "player": "Alice", "team": "Red" }
  ]
}
```

**Device assignment modes** (configurable via flag or config file):

| Mode | Flag | Behaviour |
|------|------|-----------|
| Immediate | `--assign-delay 0` | Device assigned as soon as `POST /enroll` is received; never leaves Unenrolled. Fast regression testing. |
| Timed | `--assign-delay 10s` | Assignment sent after a fixed delay. Delay should be visible on the device LED (≥5 s) but not patience-testing; keep it short in CI. |
| Manual | `--assign-manual` | Assignment held until `POST /assign/{node_id}` is called or a key is pressed. The only mode that genuinely exercises the organiser-device interaction and is the best choice for demos and realistic field testing. |

The delay in timed mode corresponds to the human operational window in the enrollment sequence — the time between the organiser seeing the device appear as unassigned and acting on it. In a real game this can be seconds to minutes; the mock should not assume a value.

**Exposes:**
- `POST /enroll` — accepts device registration from bridge; returns game_id and fence list
- `POST /assign/{node_id}` — manually triggers assignment in manual mode
- `POST /event` — accepts arrival, button, and ping events from bridge
- `GET /status` — dumps current device states for the organiser

**Logs every event with `slog`:**
```
time=2026-06-03T10:15:23Z level=INFO msg=event node_id=1234 friendly_id=1 player=Alice team=Red event_type=arrival fence="The Old Mill" lat=51.5081 lon=-0.1289 ts=1717409723
```

Structured output means event streams can be filtered with `jq` during field tests or redirected to a file for post-game analysis.

Lives at `examples/gps_game/mock_game/` alongside the device and gateway code.

## Gateway advertisement: prerequisite for testing

The gateway advertisement packet format (currently a TODO in `protocol.md`) must be resolved before end-to-end testing is possible — devices cannot find the gateway without it.

**Unblocking development**: hardcode the gateway node ID as a compile-time constant (`GATEWAY_NODE_ID`) in device firmware for initial testing. This skips Uninitialised state entirely and allows Phase 1 to proceed without the advertisement design being complete. Replace with the real advertisement mechanism once designed.

The advertisement design should be completed alongside Phase 1, not deferred to Phase 3.

---

## Phase 1: Device state machine and enrollment

### Device tasks

- Implement the state machine from `device-design.md`: **Uninitialised → Unenrolled → Syncing → Active**
- On button press in Unenrolled: send `INIT_REQ` (node_id, friendly_id, latlon_t, time_t)
- Receive and accumulate `GEOFENCE_SEGMENT` packets; store to NVS
- Send `SYNC_STATUS(0x00)` when all segments received; send `SYNC_STATUS(0x01, expected_index)` for a missing segment
- Transition to Active on `READY_ACK`
- Handle zero-segment enrollment (button-only game): `READY_ACK` arrives with no prior segments
- Handle unsolicited `GEOFENCE_SEGMENT[index=0]` in Active state: discard current fences and re-enter Syncing
- Handle `UNENROLL` in Active state: clear NVS fences and return to Unenrolled
- Persist gateway node ID, fence set, and enrolled flag to NVS; restore state on power-on
- Implement LED patterns from `device-design.md`; add `has_rgb` build flag for RGB colour support

### Server tasks

- Handle `INIT_REQ`: extract node_id, friendly_id, location; record device as pending if not yet assigned
- When organiser assigns device: respond to next `INIT_REQ` with `GEOFENCE_SEGMENT` packets
- Segment fences to fit within the SF7/SF8 LoRa payload limit (max 31 FenceEntry per packet)
- Handle `SYNC_STATUS(0x01)`: retransmit the requested segment locally (no bridge call needed)
- Send `READY_ACK` on `SYNC_STATUS(0x00)`
- Forward device-ready notification to bridge microservice via HTTP POST

### Notes

- No server acknowledgement of `INIT_REQ` when device is unassigned — the server stays silent; the human loop (organiser assigns, player presses button) is the retry mechanism
- `friendly_id` is the number printed on the device label; include it in all bridge payloads for organiser UI

---

## Phase 2: Game events

### Device tasks

- Poll GPS continuously in Active state; evaluate each fix against stored geofences
- On geofence entry: send `EVENT_REPORT(event_type=0x00)` with location and timestamp
- On button press: send `EVENT_REPORT(event_type=0x01)`
- On `PING_INTERVAL` elapsed since last report: send `EVENT_REPORT(event_type=0x02)`
- Wait for `EVENT_ACK`; retry up to `EVENT_RETRY_COUNT` times at `EVENT_RETRY_DELAY` intervals
- On receiving `UNENROLL` in response to any event: clear fences, return to Unenrolled

### Server tasks

- Handle `EVENT_REPORT`; send `EVENT_ACK` with matching event_type and timestamp
- Forward game events (types 0x00 and 0x01) to bridge microservice; pings (0x02) may be organiser-only
- Send `UNENROLL` instead of `EVENT_ACK` when the device is no longer part of an active game

### Notes

- Server does not re-validate location against fences; device reports what it observed
- Deduplication (e.g. repeated geofence entry) should be handled server-side or in the game backend, not in device firmware

---

## Phase 3: Gateway discovery

The gateway advertisement packet format is not yet specified (see TODO in `protocol.md`). This phase designs and implements it.

### Tasks

- Define the advertisement packet: gateway node ID, protocol version, optional game-scope identifier
- Implement gateway broadcast on power-on, organiser button press, and at infrequent periodic intervals
- Scope advertisement flooding using MeshCore `RegionMap` to the game-area mesh
- Device in Uninitialised state listens for advertisement; on receipt, store gateway node ID in NVS and transition to Unenrolled
- Define pre-shared key authentication approach to prevent rogue gateway interception

---

## Phase 4: Bridge microservice and gateway connectivity

### Bridge tasks

- `/v1/events` HTTP endpoint: accept device events and ready notifications from gateways
- Persist node → game/team mapping in Postgres; store `last_seen_gateway_id` and `last_seen_at`
- Forward game events to game backend API with player token substituted for node ID
- Route device-bound replies (fence segments, ACKs, UNENROLL) back through the last-seen gateway

### Gateway / bridge integration tasks

- Implement WebSocket client in gateway firmware connecting to bridge on startup
- Define typed JSON envelope schema for gateway→bridge events and bridge→gateway `deliver` commands
- Queue outbound `deliver` commands in bridge while gateway is offline; expire after configurable window
- Deduplicate inbound events using `(node_id, event_timestamp)`

### Deployment

See `gateway-design.md` for deployment tiers:
- **Local LAN**: gateway and bridge on same network; plain WebSocket, no TLS required
- **VPN**: gateway in field connects home over WireGuard; bridge not exposed publicly (recommended for field games)
- **Direct ingress**: static IPv4/IPv6, TLS mandatory, per-gateway API key; Cloudflare grey-cloud DNS for stable hostname

---

## Phase 5: Text reply support (future)

- Device displays or logs a short message received from the server after an event
- Server sends a text reply such as `Score +10 at The Cinema` using `PAYLOAD_TYPE_TXT_MSG` or a custom packet
- Bridge constructs reply text from game backend response

## Phase 6: Game-wide broadcast (future)

- Broadcast channel for organiser announcements to all active devices
- Uses MeshCore group or room broadcast, separate from the reliable startup and event path

---

## Integration points in MeshCore

### Device side

- `examples/simple_sensor/SensorMesh.h` / `.cpp` — request send/receive callbacks
- GPS handling and NVS storage code
- New: `state_machine`, `fence_store`, `led_indicator` modules (see `device-design.md`)

### Server side

- `examples/simple_room_server/MyMesh.h` / `.cpp`
- `onAnonDataRecv` / `handleRequest` for packet dispatch
- HTTP integration to bridge microservice

---

## Practical development order

1. Device state machine and LED patterns — testable without radio
2. `INIT_REQ` send and `GEOFENCE_SEGMENT` receive end-to-end with a local server
3. `SYNC_STATUS` / `READY_ACK` handshake; verify NVS persistence across power cycles
4. `EVENT_REPORT` (all three types) and `EVENT_ACK` / `UNENROLL` handling
5. Bridge microservice HTTP endpoints and game API connector
6. WebSocket gateway↔bridge link
7. Gateway advertisement (Phase 3)
8. Text replies and broadcast when core is stable

---

## Success criteria

- Device moves through the state machine correctly and persists state across power cycles
- Fence delivery is reliable over multiple hops; missing segments are retried and recovered
- All three event types are delivered and acknowledged
- `UNENROLL` cleanly returns the device to unenrolled state from any event response
- Server-pushed fence update mid-game replaces the fence set without losing the enrolled state
- Bridge maps node IDs to player tokens and calls the game API correctly
- Gateway routes replies back to the correct device via `last_seen_gateway_id`
