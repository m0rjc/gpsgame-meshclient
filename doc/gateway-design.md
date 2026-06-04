# Gateway and Bridge Design

This document describes the gateway responsibilities, discovery model, and bridge microservice callback design.

## Components

- **Device**: battery-powered player device running device firmware.
- **Gateway**: a board with both LoRa and internet connectivity (e.g. ESP32 with WiFi) running `simple_room_server`-derived firmware. Reachable by both the mesh and the internet.
- **Bridge microservice**: a cloud-hosted service with a database and a public API consumed by game backends.

## Responsibilities

**Gateway:**
- Receive mesh messages from devices (`INIT_REQ`, `SYNC_STATUS`, `EVENT_REPORT`)
- Forward them to the bridge microservice
- Receive delivery commands from the bridge and forward them to devices as mesh packets
- Broadcast its presence so devices can discover it (see Gateway Discovery below)
- Maintain a persistent connection to the bridge for low-latency callbacks

**Bridge microservice:**
- Own the authoritative hardware ID → game/player/team mapping
- Store the last-seen gateway for each device
- Accept event and status notifications from gateways
- Forward game events to the game backend API
- Route device-bound replies through the last-seen gateway

## Gateway Discovery

The gateway broadcasts its presence on the mesh so devices can find it without manual configuration. This removes the need for per-device setup and allows devices to be powered up anywhere within range.

**Discovery model:**
- The gateway broadcasts a small advertisement frame on power-on, on organiser button press, and at infrequent intervals (e.g. every few minutes) while operational.
- A device in the Uninitialised state listens for an advertisement. On hearing one, it records the gateway's node ID and transitions to the Unenrolled state.
- The device then directs all `INIT_REQ` and `EVENT_REPORT` packets at that gateway.

**Scope:** A MeshCore `RegionMap` should restrict advertisement flooding to the game-area mesh. This also naturally separates game traffic from the public MeshCore network.

**Authentication:** A rogue gateway could intercept game packets. For a single-organiser deployment the simplest mitigation is a pre-shared key baked into both gateway and device firmware at build time. The advertisement and all device→gateway exchanges are authenticated with this key.

**TODO:** The advertisement packet format and key exchange mechanism are not yet specified. See the TODO in `protocol.md`.

**LED/display:** A device with no gateway should display a distinct "searching" state (LED colour or flash pattern) so the organiser can confirm devices have found the gateway before the game starts.

## Callback Mechanism: Gateway ↔ Bridge

The bridge needs to push replies and fence data to devices through the gateway. Because the gateway is typically behind NAT, the gateway initiates the connection.

**Recommended: WebSocket over TLS, with Cloudflare bypassed for this endpoint**

The gateway is a trusted device, not a browser from the open internet. Cloudflare's HTTP proxy (orange cloud) imposes a 100-second idle timeout and can silently drop long-lived connections — particularly problematic when the gateway is on a mobile hotspot or weak WiFi link.

Solution: set the gateway's subdomain to **DNS-only (grey cloud)** in Cloudflare. The gateway connects directly to the Go server; Cloudflare never touches the WebSocket. The public API (organiser UI, game backend callbacks) stays behind the orange cloud where CF protection is useful.

With that in place:
- Gateway opens a `wss://` connection to the bridge at startup.
- Bridge sends typed JSON command frames to the gateway when it needs to reach a device.
- Gateway sends device events upstream over the same connection.
- Bidirectional, low-latency, NAT-friendly, and fully supported by Go's standard library.

**Fallback if the gateway endpoint must go through Cloudflare:**
Use HTTP POST upstream (gateway → bridge, stateless, trivially retried) and short-interval polling downstream (gateway polls bridge every 2–3 s for pending commands). At GPS game event cadence (checkpoint visits every few minutes) this latency is imperceptible and the implementation is simpler to make resilient. SSE offers a middle ground but still requires HTTP POST for the upstream direction, gaining little over polling at this event rate.

### WebSocket message shapes

**Gateway → Bridge (device event):**
```json
{
  "type": "device_event",
  "gateway_id": "gw-001",
  "node_id": 1234,
  "friendly_id": 42,
  "event": "init_req",
  "lat": 51234567,
  "lon": -12345678,
  "timestamp": 1710000000
}
```

**Bridge → Gateway (deliver to device):**
```json
{
  "type": "deliver",
  "node_id": 1234,
  "payload_hex": "060042000007",
  "delivery_mode": "direct"
}
```

## Message Flow

### Enrollment

1. Device sends `INIT_REQ` over mesh → Gateway receives it.
2. Gateway forwards to Bridge via WebSocket (`device_event`, `event: "init_req"`).
3. Bridge stores device registration; notifies organiser UI.
4. Organiser assigns device to game.
5. Bridge sends `deliver` command to last-seen gateway with encoded `GEOFENCE_SEGMENT` payloads.
6. Gateway transmits segments to device over mesh.
7. Device completes `SYNC_STATUS` handshake; Gateway forwards confirmation to Bridge.
8. Bridge updates device state → Active; sends `deliver` with `READY_ACK`.

### Game Event

1. Device sends `EVENT_REPORT` over mesh → Gateway receives it.
2. Gateway forwards to Bridge via WebSocket.
3. Bridge looks up player token and calls game API.
4. Bridge sends `deliver` with `EVENT_ACK` (or `UNENROLL` if the game has ended).

## Data Model (Postgres)

### `nodes`
| Column | Type | Notes |
|--------|------|-------|
| `node_id` | bigint PK | MeshCore Mesh ID |
| `friendly_id` | smallint | physical label number |
| `assigned_game_id` | bigint FK | null if unenrolled |
| `team_id` | bigint FK | null if unassigned |
| `last_seen_gateway_id` | text FK | routes replies |
| `last_seen_at` | timestamptz | |
| `enrolled_at` | timestamptz | |
| `metadata` | jsonb | |

### `gateways`
| Column | Type | Notes |
|--------|------|-------|
| `gateway_id` | text PK | |
| `last_seen_at` | timestamptz | |
| `ws_connected` | bool | live socket present |
| `metadata` | jsonb | |

## Reliability

- Gateway retries HTTP/WS forwarding with exponential backoff on transient bridge failures.
- Bridge queues outbound `deliver` commands while a gateway is offline; delivers on reconnect.
- Commands expire after a configurable window (suggested 30 s–2 min) to prevent stale delivery after a device has moved on.
- Bridge deduplicates events using `(node_id, event_timestamp)` in case a gateway retries.

## Security

- All gateway↔bridge traffic uses TLS (`wss://`, `https://`).
- Gateways authenticate to the bridge with a per-gateway API key provisioned at setup.
- Mesh traffic is encrypted using MeshCore shared secrets.
- Device→gateway authenticity relies on the pre-shared key described in Gateway Discovery.

## Operations

- Admin UI: view node→gateway mapping, last-seen times, enrollment state.
- Gateway diagnostics endpoint: trigger advertisement broadcast, force WebSocket reconnect, tail logs.
- Bridge health endpoint for monitoring.

## Deployment Tiers

The bridge microservice does not need to be cloud-hosted. Three practical configurations, in increasing complexity:

### 1. Local LAN (development / home games)

Gateway and bridge are on the same network. WebSocket connects to a LAN IP; TLS is optional. No router configuration, no Cloudflare, no VPN needed. Use this for development and for games played close to home.

### 2. VPN (recommended for field games)

Gateway in the field connects home over a WireGuard (or similar) VPN before talking to the bridge. The bridge sees it as a local connection; no port-forwarding or public ingress required. If a VPN is already in place this is the lowest-friction production option and keeps the bridge entirely off the public internet.

### 3. Direct internet ingress (fallback / multi-organiser)

For gateways that cannot use the VPN:

- **IPv6 (preferred)**: assign the bridge host a public IPv6 address from your range. No NAT, no port-forwarding on the router; the host firewall is the only gate. Most modern hardware and mobile hotspots support IPv6.
- **IPv4 with a dedicated port**: open a specific port on the router, point it at the bridge. A non-standard port reduces casual scanning exposure.

In both cases: TLS is mandatory; the per-gateway API key means an attacker reaching the port cannot do anything useful. Add OS-level rate-limiting (`nftables`/`ufw`) to absorb noise.

**DNS**: use a Cloudflare grey-cloud (DNS-only) subdomain pointing at the static IP or IPv6 address. This gives a stable hostname independent of IP changes and is compatible with Let's Encrypt or a Cloudflare origin certificate — without routing traffic through CF's proxy.

## Next Steps

1. Design the gateway advertisement packet format (the `protocol.md` TODO).
2. Define the WebSocket JSON envelope schema and version it.
3. Add a WebSocket client to `examples/simple_room_server`.
4. Implement bridge endpoints, DB schema, and game API connector.
5. Test end-to-end with one gateway and two or three devices (or a local emulator).
