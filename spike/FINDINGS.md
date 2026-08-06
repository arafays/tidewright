# Netcode spike — can an authoritative sand sim be streamed?

**Verdict: yes, comfortably. Bandwidth is not the risk. The risk is elsewhere.**

Measured against the real TIDEWRIGHT simulation with a real encoder and real
gzip, not estimated. Reproduce with `spike/netstream-probe.js` (dev tool, not
shipped, not referenced by `index.html`).

---

## Why this had to be answered first

A GPU sand sim **cannot be lockstepped**. GPU floating-point is not
bit-identical across vendors and drivers, so two machines running the same
shader diverge within seconds, and every frame of an 8-neighbour relaxation
compounds it. That rules out the peer-to-peer deterministic model most indie
RTS and TD games use.

So the architecture is forced: **one authority simulates, everyone else receives
heightfield state.** That is only viable if the state fits down a domestic
uplink during the worst case — a flood, when the sea is taking a castle apart.

## The wire format tested

Field downsampled to a coarse authoritative grid, tiled into 8×8 blocks. A block
that has not moved costs one bit. A block that has moved sends 64 deltas against
the state the client last acknowledged, quantised to 4 mm — far below what is
visible on sand. One byte per delta, with a two-byte escape. Then gzip.

Quantisation error accumulates in the probe exactly as it would on the wire:
the baseline advances only by what was actually sent.

## Results — 192×192 authority, 20 Hz

| Scenario | mean | p95 | peak |
|---|---|---|---|
| Idle, calm | 22.5 kbit/s | 32.3 | 34.4 |
| Building (pail, dragging) | 26.7 kbit/s | 35.0 | 35.4 |
| Digging | 30.5 kbit/s | 41.6 | 45.0 |
| Flood, tide IX, castle eroding | 23.5 kbit/s | 29.3 | 33.6 |
| Flood, erosion forced to max | 30.8 kbit/s | 40.3 | 43.2 |
| Large mould stamp (sharpest burst found) | 52.2 kbit/s | 84.0 | **84.0** |
| Soaking a keep until it slumps | 18.4 kbit/s | 27.5 | 29.3 |

**Worst burst measured across every scenario: 84 kbit/s — about 10 KB/s.**

Naive comparison: shipping the whole field as 16-bit every tick would be
73,728 bytes/tick, **11.8 Mbit/s**. The encoder is a 140–800× saving.

Joining a match costs one keyframe: **16.5 KB** at 128², **32.6 KB** at 192²,
**50.1 KB** at 256².

## Why it is so cheap: sand barely moves per tick

This is the finding that decides it. At maximum erosion:

```
mean cell change      0.013 mm / tick
cells moving > 4 mm   22 out of 36,864  (0.06 %)
```

Sand is dramatic **cumulatively** — a castle dissolves over a minute — but per
20 Hz tick almost nothing happens. Delta encoding is close to a perfect fit,
and the cost tracks *how much moved*, not how big the field is. Hence:

| Authority | cell size | join | mean | peak |
|---|---|---|---|---|
| 128² | 37.5 cm | 16.5 KB | 14.8 kbit/s | 20.3 |
| 192² | 25.0 cm | 32.6 KB | 21.2 kbit/s | 29.1 |
| 256² | 18.8 cm | 50.1 KB | 23.3 kbit/s | 29.1 |

Going from 128² to 256² — four times the cells — costs 57 % more bandwidth, not
400 %. **Pick the resolution on gameplay grounds, not network grounds.**

## The cost that is real: host readback

Per network tick, on the host:

| stage | ms |
|---|---|
| GPU readback | **10.48** |
| downsample (CPU) | 1.71 |
| encode | 0.23 |
| gzip | 0.39 |
| **total** | **12.81** (26 % of a 50 ms tick) |

Encoding is free. The readback dominates, and as written it is a **synchronous
stall** that blocks the pipeline — worse in practice than the number suggests.

Two fixes, both known-good:

1. **Downsample on the GPU** with a reduction pass, then read back 192² instead
   of 512². Seven times less data across the bus, and the CPU downsample cost
   disappears.
2. **Read back asynchronously** — PBO plus `fenceSync`, which this codebase
   already does for its metrics pass. One tick of latency, no stall.

Together these should put the host cost in low single-digit milliseconds.

## What this does NOT prove

- **Nothing about feel.** Bandwidth is settled; responsiveness at 150 ms is not.
  Building must be locally predicted — the player's own brush strokes applied
  immediately client-side and reconciled against authority — or it will feel
  like dragging a tool through treacle. That needs a build to evaluate.
- **The numbers are pessimistic.** The probe downsamples the 512² sim rather
  than running a genuinely coarse authority, so the field carries
  high-frequency detail a real coarse sim would never produce. Real deltas land
  under these.
- **Physics divergence is not addressed.** Clients render an interpolated
  heightfield; they do not simulate. Any client-side cosmetic detail must be
  strictly decorative, or it becomes a second source of truth.

## Recommendation

Bandwidth is not a reason to avoid multiplayer. Design for:

- **192² authority** — 25 cm cells, ~25 kbit/s per client, 33 KB to join.
- **20 Hz** state, client-side interpolation between ticks.
- **Client-side prediction on your own tools**, authority reconciliation on everyone else's.
- **GPU downsample + async readback** on the host from day one.
- Visual detail decoupled from authority: clients may add high-frequency
  cosmetic relief locally, but it must never feed back into gameplay.

The next unknown worth spending a week on is **latency feel**, not throughput.
