# Phantom Jam Lab

**▶ Run it: https://patzoul.github.io/traffic-sim/**

An interactive simulation of how motorway traffic jams form. Three lanes and cars
that follow the Intelligent Driver Model with MOBIL-style lane changing, on either
a 500 m closed loop or an open road with an on-ramp. On the loop, past a certain
density, the flow breaks down into stop-and-go waves on its own, with no accident,
no bottleneck and nobody braking without reason. On the open road, the merge does
it for you.

Single self-contained `index.html` — no build step, no dependencies. Served from
GitHub Pages off `main`, or just open the file in a browser.

## What it shows

- **Live carriageway** — three lanes, cars coloured by speed against the fleet's
  mean desired speed, wrapping at the right edge.
- **Time–space diagram** — position across, time downward, colour is speed. Jams
  appear as dark bands tilted backwards, because the wave travels upstream against
  the traffic at roughly 15 km/h while every car inside it moves forwards.
- **Fundamental diagram** — flow against density, accumulating across parameter
  changes. Flow rises with density, peaks near 1900 veh/h/lane, then collapses.

## Controls

| Control | Range | What it does |
|---|---|---|
| Road | loop / open | Closed 500 m loop, or an open road with an on-ramp at 150 m |
| Cars on the loop | 10–150 | Loop only. On a fixed 500 m loop this *is* density (veh/km/lane) |
| Mainline demand | 500–6500 veh/h | Open only. Traffic wanting to join at the start, all three lanes |
| Ramp demand | 0–1800 veh/h | Open only. Traffic joining from the slip road at 65 km/h |
| Average desired speed | 40–130 km/h | Speed each driver holds on an empty road |
| Spread of desired speeds | 0–26 km/h | Standard deviation across drivers |
| Share of HGVs | 0–40% | Limited to 90 km/h, 16.5 m long, sluggish, barred from lane 3 |
| Following headway | 0.6–2.4 s | Target time gap. The biggest single lever on capacity |
| Reaction time | 0–1.4 s | Delay before a driver acts on what the car in front did |
| Driver imprecision | 0–0.7 m/s² | Random error in accelerating and braking |
| Lane courtesy | 0–1 | MOBIL politeness factor |
| Lane 3 closure | toggle | Cones from 300 m to 360 m, pinning the jam in place |
| Simulation rate | 6×–48× | How fast road time runs |

## Model

Intelligent Driver Model for car following:

```
dv/dt = a·[ 1 − (v/v0)⁴ − (s*/s)² ]
s*    = s0 + max(0, v·T + v·Δv / (2·√(a·b)))
```

MOBIL for lane changing: a change happens only if there is physical room, if it
improves the mover's own acceleration, and if the cost imposed on the new follower
— weighted by the courtesy factor — does not outweigh the gain. A keep-left bias
reserves lane 3 for overtaking; HGVs are barred from it outright and pushed left
four times harder.

Vehicle parameters:

| | Car | HGV |
|---|---|---|
| Length | 4.6 m | 16.5 m |
| Max acceleration | 0.95 m/s² | 0.42 m/s² |
| Comfortable braking | 1.8 m/s² | 1.4 m/s² |
| Standstill gap | 2.2 m | 3.2 m |
| Desired speed | normal(mean, σ) | min(limiter ±1%, normal(mean, σ)) |

### Open road

Vehicles arrive as a Bernoulli process at the demanded rate and join at the first
point where there is a full following gap for the speed they would enter at
(`s0 + T·v` plus a margin), choosing whichever lane has most room. Anything that
cannot find room waits in an entry queue rather than vanishing, so demand above
capacity shows up as a growing queue instead of being silently discarded. Vehicles
are removed at the far end.

The on-ramp is a fourth lane that exists only between 150 m and 285 m. Ramp
vehicles arrive at 65 km/h and must merge into lane 1 before the acceleration lane
runs out, using the same forced-merge machinery as the lane closure — take the
first safe gap, with the acceptable braking imposed on the new follower rising as
the road runs out. The end of the acceleration lane is a hard barrier.

The fundamental diagram switches to a fixed 100 m detector section between 40 m and
140 m, upstream of the merge, because on an open road density varies along the
carriageway and a whole-road average would be meaningless.

Integration is ballistic at a fixed 50 ms step, computed synchronously — every
car's next state is derived from the current state before any of them is committed.
Two hard guards exist for numerical safety: cars never advance past the rear bumper
of the car in front, and the cones are a physical barrier. Both were measured to be
inert in ordinary traffic and only bite during a forced merge.

## Measured results

All at 65 cars (43 veh/km/lane), time-averaged over four minutes of steady state
after warm-up.

Human factors, no HGVs:

| Reaction lag | Imprecision | Flow (veh/h/lane) | Speed spread | Peak crawling |
|---|---|---|---|---|
| 0 | 0 | 1875 | 5.1 km/h | 0% |
| 0.75 s | 0.26 m/s² | 1553 | 12.8 km/h | 35% |

Being human costs about a fifth of the road's throughput.

Vehicle mix, human factors at default:

| HGV share | Flow (veh/h/lane) | Speed spread | Peak crawling |
|---|---|---|---|
| 0% | 1553 | 12.8 km/h | 35% |
| 15% | 1413 (−9%) | 16.9 km/h | 42% |
| 30% | 965 (−38%) | 24.8 km/h | 69% |

Strongly non-linear. For comparison, winding the spread of *car* speeds from zero
to its widest costs about 9%, so a 15% lorry share costs roughly twice what the
widest possible spread of car speeds does.

Density sweep, no HGVs: free flow up to about 25 veh/km/lane, capacity around
1900 veh/h/lane, stop-and-go from roughly 35 upwards, gridlock by 100.

### Open road, and what the on-ramp costs

Throughput measured at the exit over five minutes, after a three-minute warm-up,
no HGVs.

| Mainline | Ramp | Total demand | Throughput | Mean speed | Entry queue |
|---|---|---|---|---|---|
| 6500 | 0 | 6500 | 6372 veh/h (2124/lane) | 75 km/h | none |
| 4500 | 0 | 4500 | 4452 veh/h | 92 km/h | none |
| 4500 | 700 | 5200 | 4080 veh/h | 21 km/h | growing |
| 4500 | 900 | 5400 | 4248 veh/h | 23 km/h | growing |
| 4500 | 1800 | 6300 | 4152 veh/h | 19 km/h | growing fast |

Without the ramp the road carries over 2100 veh/h/lane. Open the ramp and total
throughput caps out around 4200–4400 veh/h however much demand is thrown at it —
the merge, not the road, becomes the constraint, and *more ramp demand does not
produce more throughput*.

The breakdown is metastable, which is the interesting part. At the default 4500 +
700 the road runs freely for roughly 140 seconds and then collapses, and once
collapsed it stays collapsed. That is the capacity drop: recovering free flow needs
demand to fall well below the level that broke it.

## Known limits

- With the lane closure on at high density in loop mode, the queue eventually wraps
  the whole ring and everything gridlocks. That is honest ring-road behaviour, not
  a bug — use open-road mode for anything involving a queue that needs somewhere to
  go.
- The on-ramp is deliberately a harsh one: 135 m of acceleration lane and a 65 km/h
  entry speed. A longer slip road would cost far less capacity, and the ~30% drop
  measured here should not be read as typical of motorway merges in general.
- The open road is only 500 m long, so the upstream queue has about 300 m before it
  reaches the entry and starts backing up into the queue counter rather than being
  visible on the map.
- Headways are heterogeneous only through a small multiplier on a single slider.
  Real drivers differ far more in following distance than in top speed, and
  following distance is what sets capacity.
