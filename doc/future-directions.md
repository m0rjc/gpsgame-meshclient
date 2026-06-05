# Future Directions

Ideas and architectural observations captured here as they arise, before they
are needed. None of this is committed design — treat it as a backlog of
well-understood stories that have already been thought through to the point
where implementation would not require revisiting the fundamentals.

---

## NFC checkpoint stations

### Motivation

The GPS tracker model depends on handheld devices maintaining reliable LoRa
connectivity while carried in pockets or rucksacks across a wide area. If field
testing shows that link reliability or GPS accuracy is insufficient to give
players a consistent game experience, a fixed-station model is the natural
fallback: the network infrastructure stays in known, well-sited locations, and
the player interaction becomes an explicit physical tap rather than an automatic
detection.

The two models are kept protocol-compatible so that the fallback does not require
a redesign. 

### Architecture

The **mesh device is fixed at the checkpoint**; the **player carries a passive
NFC tag** (card, wristband, or keyring fob). No battery, no firmware, no setup
required on the player side.

When a player visits a checkpoint:

1. The player presents their NFC tag to the station reader.
2. The station reads the tag UID and sends an `EVENT_REPORT` to the gateway:
   new `event_type = 0x03` (NFC visit), carrying `tag_uid` (suggest `uint32_t`
   or `uint64_t` depending on tag family) alongside the station's own `latlon_t`
   and `time_t`.
3. The bridge maps `tag_uid` to a player token and records the checkpoint visit.

The station device is a new firmware target (`src/station/`) — a fixed,
mains- or battery-powered MeshCore node with an NFC reader and no need for GPS
(its location is configured at installation). It has no player state machine; it
is a simple event reporter. Commodity NFC tags (MIFARE Ultralight, NTAG21x, or
similar) and standard reader ICs (PN532, RC522) keep hardware costs low.

## NFC-based self-service enrollment

For NFC station games the pre-game setup problem is registering which NFC tag
belongs to which player. A natural solution: designate one station (or a
handheld unit carried by the organiser) as an **enrollment station**. Each player
taps their tag to it before the game starts. The station sends an enrollment
event carrying the `tag_uid`; This can appear to the game system in the same way
as the GPS device design.

This is primarily a bridge and game-server concern. The station firmware reports
every tag read identically; the bridge decides whether a read is an enrollment
event or a checkpoint visit based on the station's registered role.

---

## Large Charity Walks

A charity walk has a different requirement to a wide game. It's no longer important who reached a checkpoint first, but it is important who
has reached each checkpoint and approximately when. RAYNET provide radio communications for such events and often pass this information over
radio using voice FM. They will pass competitor numbers and times, often bucketing to 5 minute intervals. Messages are batched so that reports
are not too delayed (especially for the first entrants and the sweep walker) but the amount of transmissions is kept small.

Such a system would require a different packet protocol and a different backend server, but it has a lot in common with this system and could
share a lot of the codebase. The checkpoint stations would batch arrival information and maintain a delivery queue, delivering packets when full or
when enough idle time has passed. A large linear event such as the Nidderdale Rotary Walk could consider a forwarding approach where checkpoints
further down the valley relay packets for more remote stations, or use a hilltop repeater network.

### Encoding

Records are transmitted in arrival order as a flat stream of protobuf-style varint pairs: `(uint64 time_delta, sint32 competitor_delta)`. The first record encodes absolute values (delta from zero); each subsequent record encodes the delta from the previous. Time is in whole minutes since the POSIX epoch (POSIX timestamp ÷ 60). No bucketing configuration is needed — 1-minute grain is sufficient, and the protocol is simpler for it.

**Cost of the first record:** POSIX minutes (~29 million) requires 4 bytes; a competitor number up to ~16,000 requires 2 bytes — 6 bytes total. Each subsequent record averages ~2 bytes. In a ~200-byte body this yields roughly 97 arrivals per packet, with no pre-configured field widths and no server-side configuration of bucket sizes.

**Packet identity and deduplication** — the first `time_delta` value (the absolute arrival time of the first competitor in the batch) is already present in the stream and serves as the packet identifier. The deduplication key is `(node_id, first_arrival_time)`, directly parallel to `(node_id, event_type, ts)` in the GPS event protocol. The station retransmits until it receives an ACK; the server drops duplicates on the same key.

The 1-minute grain is sufficient for this key because a collision requires two distinct packets from the same station to start within the same minute — meaning the first packet must have filled (~100 arrivals) before the minute turns. That throughput implies a large commercial event with multiple parallel timing readers. RAYNET does support events at that scale, but in a safety and welfare role: the timing infrastructure at a mass-participation event is already a commercial system. This protocol targets the community event where a volunteer is currently logging arrivals on a clipboard; at that scale, filling a packet in under a minute at a single checkpoint is not a realistic scenario. Less so if multiple readers at a checkpoint that competitors must interact with are all independent mesh clients with their own node_id.

### Header

| field   | content          |
| ------- | ---------------- |
| 8 bits  | type enum        |
| 8 bits  | version          |

### Body

A stream of varint pairs, read until end of packet:

| field              | encoding | content                                      |
| ------------------ | -------- | -------------------------------------------- |
| `time_delta`       | uint64   | minutes since POSIX epoch (first record), or minutes since previous arrival |
| `competitor_delta` | sint32   | competitor number (first record), or signed delta from previous |


---

## Text reply messages

After an `EVENT_ACK` the server could append a short text string for display on
the device — for example `Score +10 at The Old Mill` or `Checkpoint already
visited`. Candidates:

- Extend `EVENT_ACK` with an optional `uint8_t msg_len` and `char msg[]` tail.
  Zero length means no message; backward compatible.
- Or use a separate `PAYLOAD_TYPE_TXT_MSG` push packet, decoupled from the ACK.
  Simpler for server-initiated messages (e.g. organiser broadcast).

Device-side: if a display is present, show the message for a configurable dwell
time then return to the status screen. On a display-less device, the message is
silently discarded — the event was still recorded.

---

## Game-wide broadcast

A one-to-many channel for organiser announcements: `Game starts in 5 minutes`,
`Course closed`, final scores. Options:

- MeshCore group or room broadcast, separate from the reliable point-to-point
  event path. Lower reliability is acceptable for announcements.
- Gateway pushes a broadcast payload; devices in **Active** state display it
  briefly and resume normal operation.

Keep this decoupled from the enrollment and event path so a broadcast storm
cannot disrupt checkpoint reporting.

---

## Multiple gateways in a single game

A wide-area game with many players may exceed what a single gateway can cover or
handle. Multiple gateways distributed across the playing area would extend range
and share the load, but require player devices to know which gateway to target —
and to re-target if they move or a gateway becomes unreachable.

The current design uses directed (unicast) traffic for event reports rather than
mesh floods, to keep channel airtime low. Whether that holds at scale is an open
question. Two broad approaches are worth exploring:

- **Flood fallback on failure** — keep directed traffic as the primary path; fall
  back to a limited flood if the directed attempt fails. Simpler, and the flood
  cost is bounded by the retry rate.
- **Gateway advertising** — gateways periodically broadcast their presence; devices
  track reachability and select the best gateway. More structured, but adds
  protocol complexity and baseline traffic.

A key factor is that check-in events are inherently infrequent: a player takes
minutes to tens of minutes to travel between checkpoints in a wide game. Per-device
airtime is therefore low regardless of approach. That may make occasional floods
acceptable in a way they would not be in a high-frequency telemetry application,
and is worth weighing when comparing the two options.

---

## Considerations for future protocol versions

The `version` field present in `INIT_REQ` and `EVENT_REPORT` is the hook for
all of the above. When adding new fields or event types:

- Increment `version` in the packet that changes.
- Servers must handle older versions gracefully (ignore unknown event types,
  treat missing optional fields as absent).
- The mock game server is the reference implementation for any new contract.
