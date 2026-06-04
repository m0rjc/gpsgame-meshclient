# GPS Game Architecture

## Overview

The GPS game design uses MeshCore as the transport layer and adds a lightweight application layer for game management.

There are three primary components:

1. **Device firmware** (sensor-style)
   - based on `examples/simple_sensor`
   - knows its Mesh ID, a human-readable `friendly_id` (matching the label on the hardware), and its current geofence set
   - manages GPS, local fence storage, event detection, and startup requests
   - sends mesh `REQ` packets to the server node
   - receives fence downloads and confirmation replies
   - stores fences locally and triggers player events (geofence entry, button press, periodic ping)
   - does not need to know team assignment; that mapping lives in the bridge

2. **Server firmware** (room-server-style)
   - based on `examples/simple_room_server`
   - receives device `I\'m here` messages and event reports
   - forwards location reports and event payloads to the bridge microservice for validation and assignment

3. **Bridge microservice**
   - receives HTTP events from one or more gateway servers
   - determines if the remote device is mapped to a current game player token
   - either starts the process to authenticate/connect the device to the game, or
   - calls the game player API on behalf of the player.
   - stores and owns the hardware ID → player token mapping and lifecycle
   - manages the authoritative mapping in a database such as Postgres/Redis
   - remembers the last gateway that reported for each device so it can route replies if needed
   - Handles game-over, either triggered by the game or by detecting invalidity
     of the expired player token next time the device is used.
   - Future extensions may include game-wide broadcast. This would see the bridge
     maintain knowledge of game membership and implement a callback mechanism.
   
4. **The existing game server**
   - The existing game server already has mechanisms to authenticate players and handle player events.
   - Currently joining a game is explicit (a player enters a join code, directly or via QR Code). We cannot
     easily do this on a headless sensor.
   - The game needs new API to support the sensor enrolement process, and new game 
   organiser UI flow to allow the organiser to assign sensors to teams.

## Key concepts

### Mesh routing vs application data

MeshCore provides:
- multi-hop direct and flood routing
- encrypted request/response packets
- contact and path management

The GPS game design adds:
- a compact geofence encoding
- client startup and event messaging
- server-side game assignment and backend bridging
- optional text replies later

### Regions and routing

The existing `RegionMap` system is used for scoping flood traffic among routers, but it is not the same as a geographic fence.

The game should:
- use region/transport scope to restrict routing to managed game-area routers
- use the geofence payload data to evaluate actual lat/lon arrival events

### Why request/reply instead of broadcast

For initial fence delivery, the recommended approach is:
- client sends a request for geofence data
- server replies with one or more segments
- client confirms receipt

This is more reliable than broadcast and supports occasional retries.

## Startup sequence

1. Device powers up and joins the mesh.
2. Device sends an `INIT_REQ` to the server node.
3. Server determines game context using:
   - node ID and friendly label ID
   - latitude/longitude
   - current time

4. If the server cannot immediately assign the device (organiser approval or manual assignment required), the server records the registration but does not reply. The device retries `INIT_REQ` on each button press. When the organiser assigns the device, the server responds to the next `INIT_REQ` with geofence segments.

5. Server replies with:
   - geofence segments (zero segments for a button-press-only game)
5. Device stores the fences and signals ready.
6. Server sends an HTTP-ready notification to the bridge microservice.
7. The bridge microservice records device assignment and forwards the ready event to the game backend.

## Gateway discovery

The gateway advertises its presence on the mesh so that devices can find it without manual configuration. The advertisement mechanism is not yet fully designed (see `protocol.md` TODO), but the intended model is:

- The gateway broadcasts on power-on and on organiser button press, at infrequent intervals thereafter.
- A device in the unenrolled state listens for a gateway advertisement and, once heard, directs its `INIT_REQ` packets at that gateway.
- Authentication against a rogue gateway is an open question; a pre-shared key baked into firmware is the likely approach for a single-organiser deployment.

### Gateway mapping

The microservice should own the hardware ID → team/game membership mapping.

- it has the database and compute resources
- it can track the last gateway that reported from each node
- it can route replies through the appropriate gateway if needed

If the node remembers the last gateway, it increases device complexity and state requirements.
Therefore the cleanest design is:

- devices find a gateway by broadcast or pre-start discovery
- servers forward events to the bridge microservice
- the bridge stores the mapping and associates devices with the gateway that reported last
- if replies are required, the bridge can ask the correct gateway to deliver them back to the node

## Game event flow

1. Device evaluates arrival at a checkpoint using the latest fences.
2. If the device enters a geofence, it sends an `EVENT_REPORT` with `event_type = 0x00` (auto).
3. If the player presses the button, the device sends an `EVENT_REPORT` with `event_type = 0x01` (button press).
4. If the device has not reported for a while, it sends an `EVENT_REPORT` with `event_type = 0x02` (location ping).
5. Server acknowledges and forwards game events (types 0x00 and 0x01) to the HTTP microservice; pings may be used for organiser tracking only.
6. Microservice translates node ID into game player token and calls game API.

The server may respond to any `EVENT_REPORT` with `UNENROLL` if the device is no longer part of an active game. The device clears its fences and returns to the unenrolled state on receipt. This covers three cases: the game system ending the game, a user (days later) powering up the device and pressing the button to join a new game, or a forgotten device still sending pings after game-over.

The server may also push a fresh `GEOFENCE_SEGMENT` set to the device at any time (e.g. to adjust checkpoints mid-game). The device replaces its current fences and completes the normal sync handshake.

This flow is entirely synchronous and can take a couple of hundred milliseconds (in the very busy load test games I've run) or tens of milliseconds in a typical game running on a small embedded server.

## Future extensions

- text reply support for messages like `Score +10 at The Cinema`
- broadcast channel/room messages for game-wide announcements
- full game-state sync in the mesh using group datagrams
- richer checkpoint metadata beyond simple circular fences

## Gateway callbacks and microservice mapping

The system includes a bridge microservice that stores the authoritative mapping between hardware/node IDs and game/team assignments. The bridge also remembers the last gateway that reported for each node. This enables the bridge to request replies be sent back to specific nodes by routing via the appropriate gateway.

Callback options:

- **WebSocket (recommended)**: gateway opens a TLS `wss://` connection to the bridge; the bridge sends delivery commands over this connection and the gateway forwards them to nodes on the mesh.
- **SSE (Server-Sent Events)**: gateway opens an SSE stream to the bridge for server->gateway push; gateway still uses HTTP POST upstream for node events.
- **Polling / HTTP**: gateway polls the bridge for pending commands; simplest but higher-latency.

Operational model:

- Gateways forward node events to the bridge via HTTP. The bridge persists the node→team mapping in its database (Postgres/Redis).
- When the bridge needs to reply to a node, it looks up the node's `last_seen_gateway_id`. If that gateway maintains a live socket, the bridge sends a `deliver` command for that node. Gateway then converts the command into a mesh packet and transmits.
- If the gateway is not connected, the bridge can queue the command for a short window and retry when the gateway reconnects.

This arrangement centralises the heavy lifting in the microservice while keeping the embedded nodes simple and stateless beyond their own ID and fence data.
