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
