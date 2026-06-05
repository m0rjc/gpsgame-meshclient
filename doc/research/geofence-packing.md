# Geofence Packing Research

Analysis of whether protobuf-style varint encoding with delta compression is worth
the added complexity for `GEOFENCE_SEGMENT` packets.

---

## Encoding scheme

The current fixed encoding uses 3 bytes per axis (`latlon_t`), regardless of the
distance between consecutive fence centres.

The proposed alternative: encode fence centres as signed deltas from the previous
entry, using ZigZag + unsigned varint (protobuf `sint32` convention). The first entry
in each packet is absolute (delta from zero). The server sorts fence groups by
nearest-neighbour before encoding; compound fences (same `control_id`) must be kept
together to preserve the compound flag chain.

ZigZag maps signed integers to unsigned before varint encoding, keeping small negative
deltas cheap: 0→0, −1→1, 1→2, −2→3, …

## Byte cost thresholds

Varint byte count is determined by the ZigZag-encoded value:

| ZigZag value | Varint bytes | Raw delta |
|---|---|---|
| 0–127 | 1 byte | ±63 units |
| 128–16,383 | 2 bytes | ±8,191 units |
| 16,384–2,097,151 | 3 bytes | ±1,048,575 units — same as fixed |
| > 2,097,151 | 4+ bytes | worse than fixed |

Units are degrees × 10⁶. Converting to distance (at 54°N):

- **2-byte zone**: steps < **912 m** (lat) or < **537 m** (lon)
  — longitude threshold is tighter because degrees of longitude are compressed by cos(54°) ≈ 0.59
- **Equal to fixed** (3 bytes): steps between those thresholds and ~116 km (lat) / ~69 km (lon)
- **Worse than fixed** (4 bytes): steps > **116 km** (lat) or > **69 km** (lon)
  — not reachable within any realistic single-game area

## Analysis on real game data

Games in the system by fence entry count (`control_locations` joined to `controls`):

| game_id | locations | fixed packets | varint packets | notes |
|---------|-----------|---------------|----------------|-------|
| 12      | 32        | **2**         | 1              | wide Harrogate coverage |
| 13      | 18        | 1             | 1              | |
| 14      | 3         | 1             | 1              | test data |
| 17      | 31        | 1 (exactly)   | 1              | |
| 18      | 25        | 1             | 1              | town centre, 2.3 km × 1.7 km |

The boundary is 31 **fence entries** (not controls) — compound fences count
individually, so a game with 28–29 controls but several compounds can tip over
the limit.

### Game 12 — wide Harrogate coverage (worst case)

32 entries, 25 controls, 3 compound controls. Nearest-neighbour sort applied to
group centroids, compound groups kept intact.

```
Fixed:  224 bytes → 2 packets  (2 bytes over the 222-byte limit)
Varint: 169 bytes → 1 packet   (5.3 bytes/entry avg)

First entry (absolute): 9 bytes  (lat ~54 M sint32 → 4 bytes; lon ~−1.5 M ZigZag → 4 bytes; flags 1 byte)
Subsequent entries avg: 5.2 bytes

Inter-group steps after nearest-neighbour sort:
  min 115 m · median 473 m · max 2,979 m

Varint byte distribution:
  Lat: 2 bytes×29, 3 bytes×2, 4 bytes×1
  Lon: 2 bytes×28, 3 bytes×3, 4 bytes×1
```

91% of deltas encode to 2 bytes despite the wider spread — the 537 m longitude
threshold is just holding at a 473 m median step. The max step of ~3 km (the jump
to/from the geographically isolated controls 22/23/26 in the west) produces the
handful of 3–4 byte values. Game 12 is being penalised a full second packet for
being 2 bytes over the fixed-encoding limit.

### Game 18 — town centre (comparison)

25 entries, 24 controls, 1 compound. Bounding box 2.3 km N-S × 1.7 km E-W.

```
Fixed:  175 bytes → 1 packet  (47 bytes of headroom)
Varint: 132 bytes → 1 packet

Inter-group steps after nearest-neighbour sort:
  min 80 m · median 302 m · max 1,108 m

Varint byte distribution:
  Lat: 2 bytes×23, 3 bytes×1, 4 bytes×1
  Lon: 2 bytes×22, 3 bytes×2, 4 bytes×1
```

Varint doesn't change the outcome here — both encodings fit in one packet. The
compression characteristics are nearly identical to game 12 (92% of deltas in 2
bytes), confirming the encoding is efficient across both dense and wider games. The
single max step of 1,108 m is a jump to one outlying eastward control; even that only
costs 3–4 bytes rather than causing a cascade.

### Nidderdale Rotary Walk — linear hike (comparison)

8 checkpoints, no compounds. Bounding box 11.4 km N-S × 11.0 km E-W. Nearest-neighbour
sort follows the route naturally: 121→122→123→124→125→128→126→127.

```
Fixed:  56 bytes → 1 packet
Varint: 57 bytes → 1 packet  (1 byte worse)

Inter-checkpoint distances:
  121→122: 2,091 m    122→123: 4,083 m    123→124: 3,144 m
  124→125: 1,191 m    125→128: 1,907 m    128→126: 4,577 m
  126→127: 2,328 m

Min 1,191 m · median 2,328 m · max 4,577 m

Varint byte distribution:
  Lat: 2 bytes×1, 3 bytes×6, 4 bytes×1
  Lon: 3 bytes×7, 4 bytes×1
```

Every step exceeds the 2-byte thresholds (912 m lat, 537 m lon), so nearly all
deltas cost 3 bytes — the same as fixed. The first entry's 2-byte absolute overhead
is not recovered, leaving varint 1 byte worse overall. This is the expected result
for a rural linear event with kilometre-scale checkpoint spacing.

## Verdict

Game 12 is already paying the 2-packet cost in production. Game 17 sits at the limit
with no headroom for any future compound additions. With six teams enrolling
simultaneously, game 12 generates twelve LoRa packet exchanges that varint would
reduce to six.

For a sparse linear hike with checkpoints > 1 km apart, most deltas fall into the
3-byte zone and the benefit is negligible.

## Implementation complexity

The primitive encode/decode functions are ~15 lines each (ZigZag + varint loop).
The integration cost is higher:

- **Server (bridge)**: must sort fence groups by nearest-neighbour centroid before
  encoding; compound groups stay together. Packet fill loop must encode speculatively
  and split on overflow rather than computing a fixed entry count.
- **Device**: parser becomes a stateful forward-only byte reader rather than
  fixed-stride random access. A bit error in a varint desyncs the remainder of the
  packet (fixed encoding loses one entry per corrupted record; varint can cascade).

Given that game 12 already requires two packets under fixed encoding, this is worth
implementing rather than deferring.
