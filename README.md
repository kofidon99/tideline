# Tideline

Draw your boat a route through moving water. The line is drawn *in* the sea, so
the sea carries it. Single HTML file, no build step, no dependencies, no
external network requests.

## Controls

| Action | Touch | Keyboard |
|---|---|---|
| Draw a route | Drag anywhere | — |
| Pause | Pause button | `Esc` or `P` |
| Mute | Speaker button | `M` |

One drag, and it is the same drag on every difficulty. A new stroke always
replaces the old plan and always starts at the boat.

## The hook — the line is in the water

The boat is under power and follows your route exactly. The sea does not push
the boat. What the sea does is move **your plan**: every point of the line you
drew is carried by the current at the same speed as everything else afloat.

That is one rule, and it has a consequence worth knowing, because it is not the
one it looks like:

> **Anything that shares the water shares your plan.**

The lanterns float in the same current as your line, so a line drawn at a
lantern goes on pointing at it while both are carried downstream. Aiming works
*because* the ink drifts — pin the ink in place and the lanterns swim out from
under it. Measured: a line that floats gathers **39% more lanterns** than one
that does not (see below).

What breaks the arrangement is the boat. It moves under power at 205 units a
second, cutting across water that is moving at a fraction of that, so a long
route is a plan made in one frame of reference and executed in another. The
further ahead you draw, the more the two disagree.

The current is painted — every streak on the water is a scrap of sea being
carried by exactly the field that carries your line — and the route carries a
tick for every second of the way ahead, so "where will I be in two seconds"
stays honest as the water moves it.

Floes sit deep and take about three fifths of the surface set, so the ice
slides across your plan rather than riding with it. That is a look, not a
difficulty — the measurements below say so plainly.

## The hull

Every floe you touch costs a plank, knocks out whatever you had planned, and
the voyage ends when the last one goes. **Five lanterns in a row mend a plank**,
up to the ceiling shown by the hollow gauge. That is the only way to gain one,
so the way to earn safety is to keep a chain going, not to sail carefully.

The chain also drives the multiplier, up to ×3.

Draw nothing and the boat is **adrift**: no power, no say, and it goes wherever
the sea puts it, which is usually into the ice.

## Difficulty

One measured number: **drift** — how fast the water moves compared with the
boat.

| | Drift | Set | Floes | Lanterns | Planks (max) | Pays |
|---|---|---|---|---|---|---|
| **Cruise** | 22% | gentle | every 3.4s → 2.1s | every 1.0s | 4 (6) | ×0.7 |
| **Drive** | 34% | steady | every 2.7s → 1.5s | every 1.05s | 3 (5) | ×1.0 |
| **Redline** | 46% | running | every 2.1s → 1.1s | every 1.15s | 2 (4) | ×1.45 |

At 22% a line drawn straight stays roughly straight and you can plan two
seconds ahead. At 46% anything you draw is a visibly different shape by the
time you are halfway along it, and the only way to sail it is to draw into
where the water is going.

**Nothing about the input changes between tiers** — same drag, same boat speed,
same length of line. The channel, the boat and the line budget are **fixed
logical sizes**, not viewport fractions, so a 9:20 phone and a 16:9 desktop
sail identically; extra width only buys more sandbank.

Each tier keeps its own best score, most lanterns and longest chain.

## Playables compliance notes

- **Initial load ~61 KB**, one file. Limit is 30 MB.
- **Zero external requests.** All art drawn procedurally on canvas, all audio
  synthesised with Web Audio. Verified in `test/run.js`.
- **No copyrighted assets** — no image or audio file in the bundle.
- **Scales to 1:1, 16:9 and 9:16.** Screenshots in `test/shots/`.
- **60 fps** at phone and desktop resolutions, measured under load, with every
  draw phase still on.
- **`firstFrameReady()` then `gameReady()`**, in that order.
- **Pause and mute obeyed immediately.** The stroke in progress is dropped on
  the way into a pause, so a held finger cannot keep drawing while you are away.
- **Progress saved through `saveData` / `loadData`**, localStorage as fallback.
- **No ads wired up yet.**

## Repo layout

```
index.html               the whole game
.nojekyll                serve files as-is
tideline-playables.zip   bundle for the developer portal
src/body.html            source of truth
build.js                 wraps src/body.html into index.html
test/driver.js           the auto-helm the other tests share
test/run.js              aspect ratios, external requests, drawing, pause, perf
test/sdk.js              integration against a mocked ytgame SDK
test/gameplay.js         hull ledger, drifting-ink and lagging-ice differentials
test/probe.js            measures how long a voyage actually lasts
test/bisect.js           disables one draw phase at a time and measures
test/shot.js             screenshot capture
```

`node build.js` rebuilds. `node test/gameplay.js 2` runs one section.

### Testing a drawing game without debug hooks

The shipped build has no test affordances, so the tests sail it the way a
person does: `test/driver.js` reads the canvas at a coarse step, finds the boat
by its red hull and the lanterns by their golden glass, works out which lights
are not behind ice, and dispatches real pointer events to draw the route. It
steers straight at where a lantern *is*, which is exactly the mistake the game
is about.

The correctness backbone is an identity. A plank is lost to exactly one floe
and mended only by a chain of five, and the voyage ends when the last one goes:

```
floes hit  ===  the tier's starting planks  +  planks mended
```

### What the differentials actually found

Both of the central measurements came back the opposite way from the guess,
which is the reason for making them.

| Claim tested | Result |
|---|---|
| a drifting line makes aiming *harder* | **wrong** — pinning the ink costs 28% of the lanterns gathered and drops the best chain from 10 to 6 |
| floes lagging the water is what holes you | **wrong** — 22.0 holes against 21.7 over nine minutes of sailing; no effect at all |

The first is now the heart of the design and it is stated at the top of this
file. The second was a finding for about twenty minutes: a forty-second pass
showed 9.3 holes against 7.3 and looked convincing. Running the same comparison
for ninety seconds an arm made it vanish. The test that replaced it asserts the
*non*-effect, so that if the drag rate ever starts mattering, something else
has changed.

The lesson is the one every noisy measurement teaches: an effect measured over
a window shorter than the thing being measured is a coin toss with a decimal
point. The differential windows here are ninety seconds, three runs an arm.

### Measuring the fun

`test/probe.js` answers a design question rather than a correctness one: **how
long is a voyage, and does anything in it ever pay off?**

Its first run said no. Two helms, three tiers, and across every combination
the plank-mending chain — a whole system of the game — fired exactly zero
times: at eight lanterns in a row it needed thirty seconds of clean sailing,
and a voyage lasted twenty-two. The chain is five now, the ice is thinner, and
the reward exists:

| | Afloat | Lanterns | Best chain | Planks mended |
|---|---|---|---|---|
| **PLANNER**, cruise | 50s | 12 | 4 | 0 |
| **PLANNER**, drive | 47s | 10 | 5 | 1 |
| **PLANNER**, redline | 49s | 11 | 7 | 1 |
| **TWITCH**, cruise | 77s | 28 | 16 | 4 |
| **TWITCH**, drive | 53s | 17 | 9 | 2 |

Read those as a floor rather than an average. Neither helm leads the current,
which is the entire skill, and neither has any idea that the chain is worth
protecting — the twitchy one does better simply because it redraws often enough
to stay pointed at something.

## Performance

60 fps at every resolution on the first measurement, and `test/bisect.js`
reports 60 fps with each individual draw phase *still on* — the whole frame has
headroom, not just the total. The lessons from the earlier games were already
in place: the coast is baked once per layout at real canvas resolution and
blitted 1:1, the lantern glow is authored at exactly the size it is drawn
rather than stretched, and no gradient is built per frame.
