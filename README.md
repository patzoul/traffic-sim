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
| Road | loop / open | Closed 500 m loop, or an open road with an on-ramp at 90 m |
| Cars on the loop | 10–150 | Loop only. On a fixed 500 m loop this *is* density (veh/km/lane) |
| Mainline demand | 500–6500 veh/h | Open only. Traffic wanting to join at the start, all three lanes |
| Ramp demand | 0–1800 veh/h | Open only. Traffic joining from the slip road at 75 km/h |
| Toll plaza | 0–10 s | Barrier across all lanes at 420 m; average dwell per vehicle, 0 = off |
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

The on-ramp is a fourth lane that exists only between 90 m and 310 m — 220 m of
acceleration lane. Ramp vehicles arrive at 75 km/h and must merge into lane 1
before it runs out, using the same forced-merge machinery as the lane closure:
take the first safe gap, with the acceptable braking imposed on the new follower
rising as the road runs out. The end of the acceleration lane is a hard barrier.

Approaching the slip road, mainline drivers in lane 1 get an incentive to move out
and those in lane 2 are discouraged from moving in, which is what drivers actually
do and which turns out to matter more than the length of the slip road.

The toll plaza puts a barrier across all three running lanes at 420 m, one booth
per lane. A vehicle must come to a genuine stop within a car's length of the line,
dwell for an exponentially distributed service time, and only then is the barrier
lifted for it. Lorries take 1.6× as long. On a loop, vehicles pay again each lap.

The fundamental diagram switches to a fixed 60 m detector section between 20 m and
80 m, upstream of the merge, because on an open road density varies along the
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

Throughput measured at the exit over four minutes, after a three-minute warm-up,
no HGVs, no toll.

| Mainline | Ramp | Total demand | Throughput | Mean speed | Entry queue |
|---|---|---|---|---|---|
| 6500 | 0 | 6500 | 6285 veh/h (2095/lane) | 77 km/h | negligible |
| 4500 | 0 | 4500 | 4215 veh/h | 96 km/h | none |
| 4500 | 700 | 5200 | 4665 veh/h | 30 km/h | growing |
| 4500 | 900 | 5400 | 5025 veh/h | 80 km/h | none |
| 4500 | 1800 | 6300 | 4275 veh/h | 18 km/h | growing |

Note rows three and four: *less* ramp demand broke down while *more* did not. That
is not noise in the measurement, it is the metastability. Near capacity whether the
merge collapses depends on whether a large enough disturbance happens to arrive, so
two runs at the same demand can end up in different states and stay there. Once
collapsed it stays collapsed, which is the capacity drop — recovering free flow
needs demand to fall well below the level that broke it.

With 220 m of acceleration lane and mainline drivers who vacate lane 1, the merge
costs very little until demand approaches capacity. An earlier version used a 135 m
slip road, a 65 km/h entry speed and no courtesy behaviour, and capped throughput at
around 4250 veh/h no matter what — a 32% capacity loss that was an artefact of an
unrealistically harsh merge rather than a property of merging.

### What a toll plaza costs

Demand 6000 veh/h, no ramp, one booth per running lane.

| Dwell at the booth | Per lane | Cycle per vehicle | Overhead beyond the dwell |
|---|---|---|---|
| 0.5 s | 465 veh/h | 7.7 s | 7.2 s |
| 2 s | 380 veh/h | 9.5 s | 7.5 s |
| 5 s | 270 veh/h | 13.3 s | 8.3 s |
| 9 s | 205 veh/h | 17.6 s | 8.6 s |

The transaction is not the expensive part. Braking to a halt, pulling away again and
letting the next vehicle roll forward and stop costs a roughly constant 7–9 seconds
per vehicle whatever the dwell, so even an instantaneous transaction leaves a lane
carrying about 465 vehicles an hour against more than 2,000 flowing freely. That
constant is the entire argument for free-flow tolling.

For reference, real cash toll lanes handle roughly 250–400 vehicles per hour, which
is where the 5 s dwell lands.

## Known limits

- The toll plaza has one booth per running lane. Real plazas fan out to far more
  booths than the road has lanes, which is precisely because a booth-per-lane
  barrier collapses a motorway; the numbers above are correct for what is modelled
  but should not be read as the capacity of a real toll plaza.
- With the lane closure on at high density in loop mode, the queue eventually wraps
  the whole ring and everything gridlocks. That is honest ring-road behaviour, not
  a bug — use open-road mode for anything involving a queue that needs somewhere to
  go.
- The open road is only 500 m long, so the upstream queue has about 90 m before it
  reaches the entry and starts backing up into the queue counter rather than being
  visible on the map.
- Headways are heterogeneous only through a small multiplier on a single slider.
  Real drivers differ far more in following distance than in top speed, and
  following distance is what sets capacity.
