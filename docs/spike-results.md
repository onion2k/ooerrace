# The spike that decided the renderer

Before this game existed, a throwaway project measured what a game built on
`artshape-render` would cost, so the architecture would be settled by numbers
rather than by argument. It asked five questions — what a pixel costs through
the library's still-life shader, whether keeping the static half of the frame
pays, what moving thousands of things costs, what overdraw costs, and how many
dynamic lights a forward loop carries — answered all five, and was then
deleted. This is its record, kept verbatim, because it is why the library has
a game path at all and why that path is shaped as it is.

What it bought, in the library rather than here: a lean material path instead
of the still life's 11 ms a megapixel, one batched `writeBuffer` of a group's
matrices a frame rather than one a thing, and `CULL_BY_RADIUS` on the point
lights. This game consumes all three through `artshape-render/game`. The
table reflection became the still life's second economy rung for the same
reason, which is recorded in artshape's ROADMAP.

Two conclusions the game has since overtaken. The kept static frame of E2 is
not used here, for the reason E4b found — the lamps move light across the
ground, so the arena is redrawn every frame anyway. And the per-light figures
were taken with no shadows, where the sixteen spots here each carry a shadow
map; what the game measures now is under
[What it costs](../README.md#what-it-costs).

The last section of most experiments below is what went wrong measuring it.
Six measurement faults in this document produced confident, wrong numbers, and
three had no symptom except a plausible figure. They are the kind that recur.

---

# What a game renderer on this library would cost

Measured 8 September 2026 on the Mac mini (`apple/metal-3`, load average
~1.9), Chrome with timestamp queries granted. A screen-filling slab, drawn
by `artshape-render` and by a deliberately lean shader, at 1280×720 and
1920×1080, medians of 24 fenced frames with an empty submit subtracted.

## The number

**The library's shader costs about 11 ms per megapixel on the pixels it
covers.** Three runs of the same measurement gave 11.45, 11.46 and 10.77,
and the seven materials sit close together — 9.8 to 13.6 ms/Mpx, so no one
model is the problem and the branch is not what makes it expensive.

| material | 720p | 1080p | marginal ms/Mpx |
| --- | --- | --- | --- |
| metal, polished | 11.35 ms | 23.30 ms | 10.4 |
| metal, hammered | 12.10 | 25.50 | 11.6 |
| nacre | 10.95 | 23.55 | 10.9 |
| gem | 12.55 | 25.90 | 11.6 |
| plastic | 11.00 | 23.20 | 10.6 |
| wood | 10.95 | 23.00 | 10.5 |
| light | 10.40 | 21.70 | 9.8 |
| *no geometry (film chain alone)* | 1.40 | 2.40 | 0.9 |
| **lean shader** | — | — | **< 0.1 total, by timestamp** |

Marginal cost is the slope between the two resolutions, which separates
what a pixel costs from what a frame costs before any pixel. The fixed
part is 0.9–1.9 ms, most of it the film chain.

**At 1080p, fully covered, that is 22 ms in a 16.7 ms frame.** The library
cannot hold 60 fps on a screen filled with its own material on a fast
desktop GPU, with nothing moving, no entities and no effects.

## The lean shader

One material model, GGX for one directional light, one prefiltered cube
tap and the split-sum lookup, and a tonemap. Its whole frame is **below
the fence's floor** (an empty submit costs 0.2–0.4 ms), and its scene pass
is at or under the timestamp resolution — readings of 0.000 and 0.066 ms
at both resolutions. So the honest statement is that it is unmeasurably
cheap here, not that it is exactly 0.03 ms/Mpx. The gap is **at least
200×**, and what matters is that it is comfortably inside budget where the
library is comfortably outside it.

## Why this does not contradict the earlier numbers

`artshape-render`'s own calibration measures 5–6 ms/Mpx for the rosette
and 8–17 for the chess set. Those are whole frames of a small piece on a
large table: the piece covers perhaps a tenth of the frame and the table
is far cheaper per pixel. A frame that is 90% table at ~1 ms/Mpx and 10%
gold at ~11 averages about 2. Coverage is the whole difference, and a game
is the case where the expensive material covers everything.

## P — where the 11 ms goes

The follow-up to E1: cut parts out of the library's shader, rebuild the one
pipeline that draws the piece, and measure. Cumulative, gold polished,
1080p, two runs agreeing to within 0.2 ms/Mpx.

| variant | ms/Mpx | saved |
| --- | --- | --- |
| P0 stock | 11.53 | — |
| P1 no table reflection (`seen`) | 7.60 | **3.9** |
| P2 + no key soft shadow (`keyShadowAt`) | 2.89 | **4.7** |
| P3 + no local lights (`localLights`) | 2.68 | 0.2 |
| P4 + no contact read (`contactAt`) | 2.70 | ~0 |

**Two features are three quarters of the cost.**

- **The table reflection, ~3.9 ms/Mpx.** `seen()` runs a ray from every
  glossy pixel down to the ground plane, samples the cushion's height field
  where there is one, and then *shades the table* at the point it lands.
- **The key's soft shadow, ~4.7 ms/Mpx.** A blocker search and a filter,
  thirty-six texture reads between them, per pixel.

Everything else together — the material model and its branch, the
environment, the probe, wear, micro-variation, the tangent frame — is
**2.7 ms/Mpx**, which is *inside* the 4.8 budget.

### A uniform branch buys nothing; removing the code buys everything

The first attempt at the reflection saving put a flag in the frame uniform
and branched on it. Measured interleaved against itself, in one thermal
state: **34.65 ms with the reflection, 35.50 ms without it** — that is
noise, not a saving. Cutting the same code out of the shader saved **5.38
ms/Mpx** in the same run. A branch the compiler cannot fold leaves the code
resident, and on this GPU residency is most of what it costs.

A module-scope `const` folds like the stub does — 6.12 ms/Mpx against the
stub's 6.05 — so the mechanism is a shader permutation with a `const`
prelude, not a uniform and not string surgery.

That also corrects the earlier note here. The guess had been that the
engraving, pattern and lettering machinery was evaluated on every pixel and
could be compiled out; reading the shader appeared to kill it, since all
three are guarded on a material field and never run for a plain slab. But
guards are branches too: **compiling those three out saved 1.11 ms/Mpx on
a material that never executes them.** Smaller than the two big items, and
real.

A corollary was drawn here that turned out to be wrong, and is left in as
a correction. The `contact` rung looked like a uniform flag of the same
worthless kind, since P4 measured ~0 when `contactAt` was cut. **P4 was
void**: the renderer had `setContact(0)` throughout E1 and P, so
`frame.aoOn` was already 0 and the stub removed a function that was
already returning immediately.

Measured properly, with the contact passes running and `aoOn` at 1,
compiling `contactAt` out saves **-0.22 ms/Mpx** — nothing, and rightly.
It is one bilinear read of a half-resolution r8 with no loop and nothing
held live across it, so unlike `seen` there is no residency to recover.

The contact rung is not a shader flag. It removes the depth prepass over
every triangle and the occlusion passes after it, which is real work:
**0.75 ms a frame on a slab of 1,944 triangles, 1.50 ms on a piece of
437,760.** It scales with geometry where the reflection rung scales with
pixels. It should be left as it is.

The general rule this leaves: a rung is worth making a permutation when
what it skips is *big* — loops, dependent texture reads, live registers.
A rung that skips one cheap instruction is fine as a uniform, and a rung
that removes whole passes was never a shader question at all.

## What it decides

E1's exit criterion was that a cut-down shader being **2× cheaper** would
send the game renderer to its own material path. On E1 alone the gap was
at least 200× and the answer was forced. P complicates it usefully: most
of the library's cost is two features, not the material model, so the
choice is now a real trade.

- **Sharing the library's shader is viable if two features come out.** At
  2.7 ms/Mpx a full-screen 1080p background costs 5.6 ms of a 16.7 ms
  frame. That leaves room for entities and effects, and both projects would
  gain from the features being switchable.
- **Writing a lean shader is still about 27× cheaper** — under 0.1 ms/Mpx
  against 2.7 — and buys back that 5.6 ms for gameplay. For an arena with
  hundreds of entities and heavy overdraw from effects, that headroom is
  probably worth more than the shared code.
- **The recommendation is unchanged but for a different reason.** Write the
  game's own material path — not because the library's is unusable, but
  because a Smash TV screen is mostly overdrawn effects, and 5.6 ms of
  background is a poor trade for code reuse. Revisit if the first game
  turns out to be quieter than that.
- **The mesh generators, the device layer, the post chain and the
  calibration ladder are all still reusable** — none is implicated.

### Acted on

The reflection became the ladder's second rung in `artshape-render` v0.2.0,
implemented as a permutation: `pbrSource({ reflectTable })` puts a
`const` in front of the shader, the renderer builds both variants at
startup alongside every other pipeline, and `setEconomy` swaps them.
Through the library's own API it saves **4.6 ms/Mpx**, and a GPU test
draws the piece both ways — `seen()` is where a NaN once turned every
frame black, and the fallback through it had nothing exercising it.

### The finding that prompted it

The ladder's rungs are supersample, shadow taps, the contact pass, and
detail. The shadow rung is well chosen — the key shadow is the single
largest cost here, and quartering its taps should recover most of 4.7
ms/Mpx, which matches yesterday's measurement of 14.9 ms/Mpx with the key
against 8.5 without.

**But there is no rung for the table reflection, and it is worth almost as
much: ~3.9 ms/Mpx, a third of the frame.** On the Windows laptop that is
the largest single saving still on the table, and giving it up costs a
still life less than giving up its shadows does. It belongs in the ladder,
probably between the supersample and the shadow taps.

## What went wrong on the way, and would again

Four measurement faults, each of which produced confident, wrong numbers
before it was caught. Three of the four were in the checking, not the
timing, which is the lesson: an instrument that cannot fail is worth as
much as the measurement.

1. **The fill check certified empty frames.** It asked only that a corner
   be brighter than almost-black; the lean renderer's tonemapped clear
   colour reads [46, 43, 43], which passed. It now compares each frame
   against the same frame drawn with no geometry, which is the only
   version that can tell "drew the slab" from "cleared to grey".
2. **The slab was not centred on the origin.** A `plate` builds its
   outline from a corner, so a 4000 mm slab spans y = 0…4000. The camera
   was aimed at the origin, off its edge, and most of what filled the
   frame was the *table* — so the first library numbers were substantially
   the table's material, not the slab's. Fixed by aiming at the assembly's
   own bounds.
3. **The reference frame for the fill check was the slab.** In the
   permutation run it was captured after drawing the piece rather than
   before, so every variant was compared against the thing it was supposed
   to differ from, and all five were reported as drawing nothing. The
   timings were sound throughout — P0 reproduced E1's independent figure
   for the stock shader, which is what showed the checker rather than the
   pipeline swap was at fault.
4. **The fence has a floor.** An empty submit costs 0.2–0.4 ms, which is
   larger than the entire lean frame. Whole-frame fencing cannot measure
   anything that cheap, and dividing by it produced a "factor" of a
   hundred million. Timestamp queries are the only usable instrument at
   that end, and E2 will need them throughout.

## Open

- **Answered by P**, above: not by compiling out unused features, but by
  making the table reflection and the shadow filter switchable.
- **What does the remaining 2.7 ms/Mpx consist of?** It was not broken down
  further. If most of it is the probe and environment sampling, a game
  could cut that too and land nearer the lean figure while keeping the
  library's material model.
- The library numbers were taken with all 720p runs before all 1080p ones.
  Interleaving would rule out clock drift between the halves; the three
  runs agreeing to within 7% suggests it is not a large effect.
- None of this has been run on the Windows laptop, where it matters most.

---

# E2 — does keeping the static half of the frame pay?

A fixed camera means the arena is the same pixels every frame. Three ways
to use that, in the lean renderer, at 1080p with 500 movers, per-pass GPU
time by timestamp, strategies interleaved so drift cannot favour whichever
went first:

- **redraw** — draw the arena and the movers every frame. The baseline.
- **copy** — keep the arena's colour and depth, copy both back each frame,
  draw the movers over them.
- **compose** — copy only the depth, which the movers need to be occluded
  correctly, put them in their own colour, and let the composite read both.
  Trades a 16 MB colour copy for one texture read in a pass already running.

| arena | static triangles | redraw | copy | compose |
| --- | --- | --- | --- | --- |
| light | 57,928 | 0.43 ms | 0.33 | 0.33 |
| medium | 1,097,608 | 1.28 | **0.33** | 0.39 |
| heavy | 4,563,208 | 5.90 | **0.46** | 0.52 |

**All three produce pixel-identical frames** — maximum channel difference 0
across the image, which is what makes the timings worth reading.

## What it says

**It works, and it is worth doing when the arena is heavy.** At 4.5M static
triangles, keeping the frame turns 5.9 ms into 0.46 — the static half stops
existing as a per-frame cost, exactly as hoped. At 1.1M it turns 1.28 into
0.33. Below a few hundred thousand it disappears into the noise.

**`copy` beats `compose`, which is the opposite of the prediction.** The
plan expected avoiding a 16 MB colour copy to win. On unified memory the
copy is nearly free and the composite's second texture read costs more than
it saves. The simpler strategy is also the faster one.

**But the whole frame is under 1.5 ms at plausible weights.** That is the
number that should change the plan more than the ranking does. An arena at
a realistic weight plus 500 movers, through the lean shader, costs about a
millisecond of a 16.7 ms budget. The frame structure is not where a game of
this shape spends its time, and architecting around the static split before
there is something to save would be premature. Build it when the arena
turns out heavy; measure first.

The 4.5M-triangle arena is a pathological case, made to get the passes above
the timer's resolution rather than because a game would have one.

## What went wrong on the way

**Chrome quantises timestamp queries to about 65.5 µs.** The first run used
8k–295k triangles, every pass landed on two to six quanta, and one-quantum
differences were being reported as "20% faster". The arena had to be scaled
to millions of triangles before the quantum became noise rather than the
signal. Anything measured this way needs passes of milliseconds, not
microseconds.

**Pass attribution between scene and composite is not reliable.** The
composite's measured time rose with the *static* weight, which is
impossible — it is the same fullscreen pass in every case. The GPU overlaps
passes and the boundary timestamp is recorded before the previous pass has
drained. Only the totals are trustworthy, and only they are quoted above.

---

# E3 — what does moving a lot of things cost?

Lean renderer, `copy` strategy, 1080p, an arena of 58k static triangles,
drones of 608 triangles each. GPU by timestamp, CPU on the main thread.

| entities | triangles | GPU scene | CPU matrices | CPU upload |
| --- | --- | --- | --- | --- |
| 100 | 60,800 | 0.07 ms | <0.1 | <0.1 |
| 500 | 304,000 | 0.13 | <0.1 | <0.1 |
| 2,000 | 1,216,000 | 0.59 | 0.2 | <0.1 |
| 8,000 | 4,864,000 | **1.90** | **0.8** | 0.1 |

**Entity count is not the constraint.** Eight thousand movers — nearly five
million triangles — cost 1.9 ms on the GPU and 0.9 ms on the CPU. Under
three milliseconds of a 16.7 ms frame, together. A Smash TV arena wants
perhaps a few hundred entities; there is an order of magnitude of room
above that.

Most of the CPU figure is not the renderer. `droneMatrices` is a JavaScript
loop building the transforms — simulation, which a real game would spend
differently — and the upload itself is a tenth of a millisecond for half a
megabyte.

## How the matrices get there, at 8,000

| | |
| --- | --- |
| one `writeBuffer` of the whole pool | **0.10 ms** |
| one `writeBuffer` per entity | 1.80 ms |
| mapped staging buffer, then a copy | 0.40 ms |

**Batch the upload.** Writing per entity is eighteen times worse, which is
the shape of cliff worth knowing about before someone writes the obvious
loop. And the mapped staging buffer — the clever option — is four times
worse than the plain `writeBuffer`, because waiting for the map costs more
than the copy saves. The dull answer wins again, as it did in E2.

## What the library's API would cost a game

`moveAll` is the method the chess game contributed. After a move it marks
the scene's bounds, lights, probe and shadows stale, which is right for a
still life and wasted sixty times a second.

| entities | `moveAll` | raw `writeBuffer` |
| --- | --- | --- |
| 100 | 0.05 ms | <0.1 |
| 500 | 0.10 | <0.1 |
| 2,000 | 0.40 | <0.1 |
| 8,000 | **1.40** | **0.10** |

Fourteen times the cost at eight thousand, and linear in the count — it is
the bounds being re-measured over every placement. Not fatal: 1.4 ms is
affordable. But it is 8% of a frame spent re-deriving things that have not
changed, and it confirms what the plan assumed — **the game renderer needs
a path that writes matrices and touches nothing else.**

## What it decides

- **The entity budget is generous.** Design for hundreds and there is room
  for thousands. Whatever eventually limits an arena game of this shape, it
  is not the number of things moving.
- **One batched `writeBuffer` a frame**, and no staging-buffer cleverness.
- **A `moveFast` that only writes matrices**, as the plan called for —
  worth about 1.3 ms a frame at eight thousand entities, and free to write.
- **What has not been measured is overdraw.** The drones here are spread
  over an arena floor; explosions and bullets in a real game overlap
  heavily, and fill is the one cost that E1 showed to be sensitive. That,
  not entity count, is where the next measurement should go.

## The instrument, again

`performance.now()` is clamped to 0.1 ms in Chrome without cross-origin
isolation, so every CPU figure below a tenth of a millisecond in these
tables means "under the clock's resolution" rather than a value. The
8,000-entity row is the only one where the CPU numbers are several ticks
wide and can be read literally.

---

# E4a — overdraw

E1 found fill to be the cost the shader is sensitive to and E3 found entity
count is not a constraint, so this is the measurement that was left. Every
layer is a screen-filling additive quad with no depth write, so nothing
rejects anything and the overdraw factor is exactly the layer count. Over an
arena and 500 movers, at 1080p.

| layers | effect shader | material shader |
| --- | --- | --- |
| 0 | 0.16 ms | 0.16 |
| 1 | 0.20 | 0.20 |
| 4 | 0.26 | 0.33 |
| 8 | 0.33 | 0.59 |
| 16 | 0.52 | 1.18 |
| 32 | 1.11 | 2.95 |
| 64 | **3.02** | **9.60** |

Per screen-filling layer: **0.045 ms** through a shader that is a falloff and
a colour, **0.147 ms** through one doing a GGX lobe and an environment tap —
about **3.3×** for drawing effects as though they were lit surfaces. Layers
were confirmed to be accumulating: mean brightness rises 91 → 94 → 104 →
121 as the count goes 0 → 1 → 8 → 32.

**Overdraw is not a constraint either, on this machine.** A screen covered
sixteen times over in effects costs half a millisecond with a proper effect
shader. Even sixty-four times over — far past anything a game would draw —
is three.

## The whole spike, in one budget

A plausible arena frame at 1080p on the Mac mini, with the lean path:

| | ms |
| --- | --- |
| arena, kept rather than redrawn | ~0.2 |
| 500 movers | ~0.13 |
| 16× overdraw of effects | ~0.5 |
| composite | ~0.13 |
| **total** | **~1 of 16.7** |

Against which: **the library's material shader, on a screen it fills, is 22
ms.** That is the whole finding of the spike. Everything a game does —
entities, effects, overdraw, frame structure — is cheap on this hardware.
The only thing that busts the budget is shading pixels with a still-life
shader, and E1 settled that the game renderer writes its own.

## The caveat that matters most

**All of this is a Mac mini.** Every figure above is fill-bound, and fill is
exactly what a laptop with integrated graphics is worse at — the calibration
work in `artshape-render` measured that class of machine at several times
the desktop's cost per pixel. Scale this table by four and a plausible frame
is still comfortable; scale the 64-layer case by four and it is over budget
on its own.

So the ordering of what to worry about, on the hardware that actually
matters, is: **fill first, then overdraw, then everything else a long way
behind.** An effects-density rung — fewer or smaller layers when the machine
measures slow — belongs in a game renderer's ladder for the same reason the
table reflection belongs in the library's.

None of it has been run on the Windows laptop. That remains the largest
open question in this document, as it was in the last one.

---

# E4b — how many dynamic lights a forward loop carries

Point lights, no shadows, over an arena and 500 movers at 1080p. Every light
is tested against every pixel; a light whose radius does not reach the pixel
is skipped after one distance test. `tight` is a muzzle flash, `broad` washes
a third of the arena. Both permutations of the shader draw an identical
picture — the cull is exact, not an approximation.

| lights | tight, culled | tight, naive | broad, culled | broad, naive |
| --- | --- | --- | --- | --- |
| 8 | 0.33 ms | 0.52 | 0.39 | 0.52 |
| 32 | 0.66 | 1.57 | 0.98 | 1.57 |
| 128 | **2.23** | 5.93 | **3.47** | 5.90 |
| 512 | 8.55 | 23.99 | 13.66 | 23.79 |
| 2048 | 35.85 | 99.12 | 60.10 | 102.76 |

Linear in the light count, as a per-pixel loop must be: **0.0175 ms per
tight light, 0.029 per broad one**, culled.

## What it says

**Hundreds of dynamic lights are affordable, and that is the answer to the
question.** Against a 10 ms scene budget:

- **128 lights: 2.2–3.5 ms.** Comfortable, with most of the frame still free.
- **512 lights: 8.6–13.7 ms.** The whole budget, and nothing left for
  anything else.
- **2048: impossible** — 36 to 60 ms.

So the ceiling for a naive forward loop on this machine is somewhere around
**300 to 500 lights**, and the comfortable working range is a couple of
hundred. For a game that wants flashes on every muzzle, glow on every bullet
and a light inside every explosion, that is a lot of room.

**Always cull by radius.** It is worth 1.7× at broad radii and 2.8× at tight
ones, it costs one distance test, and it is exact — the culled and naive
images are identical to the pixel. There is no argument for the naive loop;
it is here only to show the shape of the cost without it.

**Past about five hundred, the loop has to become something else** — tiles or
clusters, so the cost is the lights that reach a pixel rather than all of
them. That is the point at which this stops being a simple renderer, and it
is a long way past what an arena needs.

## The interaction with E2 that neither experiment saw alone

**Moving lights and a kept static frame are alternatives, not companions.**
E2 found that keeping the arena's colour and depth saves almost all of its
cost. But a light that moves changes how the arena is lit, so a kept frame
is stale the moment the lights are dynamic — the arena has to be redrawn
anyway.

The first two runs of E4b were measured with the kept frame, and the light
loop only ran over the movers' few percent of the screen: the cost looked
flat from 32 lights to 2048, which is what sent me looking. The figures above
redraw everything, which is what a game with moving lights must do.

So the two findings have to be spent together, not added: either the arena is
lit by static light and kept, or it is lit by moving lights and redrawn. A
game that wants both needs its dynamic lights to touch only dynamic
geometry — which is a real constraint on the design, and worth knowing before
rather than after.

## What went wrong on the way

**The light count never reached the shader, and the first two runs were
void.** The edit that should have put the live count into the frame uniform
landed in an older method with an identical line, leaving the real one
writing a literal zero. The timings looked plausible — a smooth curve, a
sensible ordering — and were entirely of a loop that ran no iterations. What
caught it was not the timings but the picture: mean brightness was 91.1 at
zero lights and 91.1 at two thousand and forty-eight.

That is the sixth measurement fault in this document and the third whose only
symptom was a plausible number. The check that has caught every one of them
is the same: render it and look at whether the image changed.
