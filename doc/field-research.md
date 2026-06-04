# Field Research Notes

Empirical findings from walk tests and experiments, recorded as they arise.
These feed back into design decisions and the future-directions backlog.

---

## 2026-06-04 — Walk test, 4.5 dBi antenna, frequency retuning

### Setup

- Antenna: commercial 4.5 dBi at 6 m on a portable mast
- Repeater Node mounted at mast top (short feeder, negligible feedline loss)
- Initial frequency: 869.618 MHz (project default, main MeshCore channel)
- Retuned to: 869.525 MHz mid-test

### Findings

**869.618 MHz (default, 42:26 uptime)**

| Direction | Time on air | Fraction |
| --------- | ----------- | -------- |
| TX        | 1:59        | 4.7%     |
| RX        | 4:43        | 11.1%    |

| Packets   | Total | Flood | Direct | Duplicates | Errors |
| --------- | ----- | ----- | ------ | ---------- | ------ |
| Received  | 430   | 401   | 29     | 234        | 75     |
| Sent      | 180   | 171   | 9      | —          | —      |

The channel is dominated by ambient MeshCore mesh traffic. 93% of received
packets were flood; 54% of those were duplicates. The 4.7% TX duty cycle is
almost entirely this node re-broadcasting flood traffic rather than application
messages.

**869.525 MHz (retuned, 42:15 uptime)**

| Direction | Time on air | Fraction |
| --------- | ----------- | -------- |
| TX        | 0:09        | 0.36%    |
| RX        | 0:08        | 0.32%    |

| Packets   | Total | Flood | Direct | Duplicates | Errors |
| --------- | ----- | ----- | ------ | ---------- | ------ |
| Received  | 24    | 0     | 24     | 0          | 5      |
| Sent      | 20    | 4     | 16     | —          | —      |

All received traffic was direct; zero duplicates. The 5 errors are attributed
to the device being in a pocket or passing through a temporary dead spot.
Note: 869.525 MHz is also used by Meshtastic, so a small number of those
packets may have been Meshtastic traffic rather than this node's peer.

The TX figure is what matters for regulatory duty-cycle limits; 0.36% is well
under the 10% sub-band limit for this frequency range.

**Link reliability was good** and tracked the Meshtastic antenna coverage
prediction tool closely, giving confidence that the tool can be used for
pre-event site planning. Signal could often be recovered in a dead spot by holding
the T1000E above the head, advice that could be given to players if needed.

### Implications

- The default frequency in `platformio.ini` (869.618 MHz) needs revisiting for
  game use. At 4.7% TX duty cycle the node is spending almost all of its airtime
  relaying other people's mesh traffic. On 869.525 MHz the same session consumed
  0.36% TX — a 13× reduction. A quieter channel also reduces the duplicate and
  error rate substantially. This was experienced as a much more reliable remote
  repeater admin experience.
- 869.525 MHz is shared with Meshtastic; it is less congested than 869.618 MHz
  in the test area but is not interference-free. Worth monitoring across more
  test locations before committing to it as the game default.
- The 4.5 dBi antenna at 869.525 MHz is a viable baseline for further testing.
- Coverage prediction tools (Meshtastic's in particular) appeared reliable enough
  to use for checkpoint placement planning ahead of an event.
