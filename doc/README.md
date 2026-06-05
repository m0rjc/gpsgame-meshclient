# GPS Game Example

This folder contains planning and design documentation for a MeshCore-based GPS game example.

Scenario: A Scout Troop wishes to run a wide area navigation task. Examples could
include an urban wide-game, or hiking challenge with checkpoints. They would prefer not to use the Scout's own mobile phones, but instead have the Scouts carry an inexpensive device. 

The design is built around three components:

- **Device**: based on `examples/simple_sensor`
  - runs GPS; knows its Mesh ID, a human-readable label ID, and its geofence set
  - downloads fences from the game server at enrollment
  - reports geofence arrivals, button-press check-ins, and periodic location pings
- **Server**: based on `examples/simple_room_server`
  - receives device startup and event reports
  - sends geofence segments to enrolled devices
  - forwards confirmed events to a bridge microservice over HTTP
- **Bridge microservice**
  - stores the device-to-team/player mapping in a database
  - receives ready and event notifications from gateway servers
  - translates node IDs into player tokens and calls the game API

## Documents

- `architecture.md` — system architecture, roles, and data flow
- `protocol.md` — message formats, fencing encoding, and packet protocols
- `device-design.md` — device state machine, LED indicators, firmware module structure
- `gateway-design.md` — gateway responsibilities, discovery model, bridge microservice and callback design
- `bridge-design.md` — bridge microservice design (Go, Postgres, Redis, Kubernetes) — stub
- `implementation-plan.md` — phased work plan and integration guidance
- `future-directions.md` — backlog of well-understood stories not yet committed to design

## Research

Empirical findings and analysis supporting design decisions:

- `research/field-research.md` — walk test results and radio experiments
- `research/geofence-packing.md` — varint delta encoding analysis for `GEOFENCE_SEGMENT` packets
