# ooerrace: working on it

A night circuit driven against the clock, in the browser: TypeScript, Vite,
and WebGPU through the game path of
[artshape-render](https://github.com/onion2k/artshape-render). `README.md` is
the design log — 2,300 lines saying what was tried, what it measured and why
it is the way it is, and every number in this file is justified there. This
file says how the project is made. The house rules in `~/.claude/CLAUDE.md`
apply on top of it, and where they disagree with this file, this file wins.

This game predates the house line in `artshape-game-template`. It has a real
test suite and a great many enforced numbers, and it has no lint, no
formatter, no pre-commit hook, no performance gate and no look test. The
section **What is not held** says exactly what that costs, so that nobody
reads the green tick on `npm test` as meaning more than it does.

## Commands

    npm run dev          the game at http://localhost:5190 (strict port)
    npm test             the suite: 6 files, 32 tests, ~11 s, node, no GPU
    npm run test:watch   the same, watching
    npm run build        tsc --noEmit && vite build — the only typecheck, ~1.5 s for the types
    npm run calibrate    the pilot laps every class and reports par; ~93 s a class, ~6 min for four
    npm run preview      the built game

**There is no `check` and no pre-commit hook.** `.git/hooks` holds nothing
but the samples git ships. The gate before a commit is `npm test && npm run
build`, run by hand, both green. Run it. A type error passes `npm test` on
its own, because the typecheck lives only in `build`.

`npm run calibrate` takes arguments — `npm run calibrate technical rally`
fits only those classes.

## What holds the project to a number

The suite is not a smoke test; it is a set of physical claims, and most of
the design's real constraints are in it. In `src/__tests__`:

| claim | held to | where |
| --- | --- | --- |
| par predicts the pilot's best lap | within 6% | `driving.test.ts`, `wild.test.ts` |
| a class's measured turning circle matches its spec | within 2% | `driving.test.ts` |
| top speed is the same at 20/30/60/120/144/240 Hz | within 1% of a 1000 Hz reference | `driving.test.ts` |
| the race is bit-for-bit identical at any frame rate | one hash, 1680 steps | `driving.test.ts` |
| cross-axle load imbalance flat out | < 0.02 | `driving.test.ts` |
| every wheel sits on the drawn road, body clear of it | within 3 mm | `driving.test.ts` |
| `rollDamping(spec) * SUBSTEP` | < 1 | `driving.test.ts` |
| every circuit clears its kind's corner limit | within 2% | `kinds.test.ts` |
| nothing is placed within `TRACK_HALF + 40` of the road | exact | `kinds.test.ts`, `wild.test.ts` |
| wild shapes' corner, length and radius | `WILD_LIMITS` | `wild.test.ts` |
| the racing line stays on the tarmac and bends less than the road | 0.9× | `raceline.test.ts` |
| the grid is never under water, and water is lowered only when the grid demands | exact | `water.test.ts` |
| disco's light count fits `LIGHT_CAPACITY` | 256 | `modes.test.ts` |

Two **recordings** — snapshots, not invariants:

- `driving.test.ts` hashes `x, y, z, yaw, vx, vy` after **every one of 2,040
  frames** of a 34-second scripted drive, for 9 runs across four classes,
  four biomes, three sizes and two seeds. It pins the whole trajectory, so it
  catches the vehicle model, the substepping, the clock's carry, terrain,
  grip, water drag and every collision, all at once.
- `water.test.ts` records the arena as built: prop, post, rail and bollard
  counts and the water depth, for 2 seeds × 4 sizes × 4 biomes.

A change meant to leave the driving alone must leave every hash alone. One
meant to change it runs `npx vitest -u`, **looks at which runs moved**, and
says why in the commit. The cautionary tale is in `README.md`: the vehicle
refactor was checked by hand against a worktree, that check only ever drove
seed 0 in the forest, and that is how fords draining on every other circuit
got through.

`npm run calibrate` is **a report, not a gate**. It asserts nothing and exits
0 whatever it prints. A human reads it and hand-edits the `par` constants in
`vehicles.ts`. The standing gate on the par model is the fast 6% test above.

### What is not held

Say so plainly rather than implying otherwise:

- **No performance gate at all.** No test measures a frame. Every figure in
  README's *What it costs* — the 4.92 ms night frame, 0.62 ms noon, the
  1.46/1.06/1.76 split — is prose, re-checked only by calling `measure()` in
  the console by hand. A change that doubles the frame cost passes.
- **No look test.** No Playwright, no image comparison, no CI. `docs/*.png`
  are committed but nothing compares against them. The only visual check is
  `shoot()` and your own eyes.
- **No lint and no formatter.** Match the surrounding style by reading it.
- **No fuzzer and no `invariants.ts`.** The invariant tests are the closest
  thing, and they are hand-written per area.
- **The renderer version is not pinned in practice.** See below.

## Layout

`src/circuit.ts` is the seam. `useTrack(seed, size, biome, wild, kind)` is
everything the CPU builds when a circuit changes, in a dependency order its
own comment calls non-negotiable — and nothing it draws. It exists so a test
can build exactly what the page builds with no canvas in sight. Anything new
that the circuit needs goes through it.

**Headless — every module but three.** `game.ts` (the race, `Progress`,
`FixedStep`, collision), `vehicle.ts` (a rigid body on four suspension
springs), `track.ts` (the circuit, `where`, the speed plan, `rateTrack`),
`terrain.ts`, `water.ts`, `scene.ts`, `flora.ts`, `furniture.ts`,
`raceline.ts`, `ghost.ts`, `lighting.ts`, `spatial.ts` and the content
modules all run in node.

**The page is `src/main.ts`**, 1,636 lines, the only module that touches the
DOM or a device — with `photo.ts` (the still-life renderer for photo mode)
and `particles.ts` (device-coupled in practice). Its `main()` is one closure
holding the renderer, the pools, the select screen, the panel, the camera and
the frame loop.

**Logic does leak into the page, and it is the main structural debt.** Know
about these before adding to them:

- `preview()` computes the select screen's verdicts inline — the difficulty
  band, the length, the "tighter than the car can turn" warning. The rules
  for what a circuit is *said to be* live in a DOM-writing function.
- `roadChanged` is a five-field comparison duplicating `useTrack`'s argument
  list. **Add a circuit-affecting setting and this is the line that silently
  fails to rebuild.**
- `newTrack()` is the real change-circuit transaction and is neither exported
  nor testable.
- The concours presentation blend, the plinth and the per-lap gem maths are
  in `upload()`, not in `concours.ts`.
- Skid and wheel-effect thresholds (`slide > 0.35 && speed > 300`) are
  driving constants living in the draw loop.

### The import cycle

`track → biomes → props → scene → track` is a real cycle. It only evaluates
cleanly when entered from the circuit's side. Two places carry the ordering
as a load-bearing comment: [main.ts:37](src/main.ts:37) (`./circuit` before
`./raceline`) and `src/__tests__/sim.ts` (`../circuit` first). Enter from the
wrong end and `scene` stands lamp posts along a circuit that does not exist
yet. **A new module that imports `track` inherits this.**

### Content

Each kind of thing is a typed table in its own module, and the pickers
iterate the tables, so adding one is mostly a line there and none in the
page: biomes in `biomes.ts`, vehicle classes in `vehicles.ts` (bodies in
`models.ts`), circuit kinds in `kind.ts`, sizes in `world.ts`, terrain waves
in `terrain.ts`, prop meshes in `props.ts`, the settings list in
`settings.ts`.

### The save

One key, `'arena.settings'` — still the old name, and renaming it would throw
away everyone's settings, so leave it. The whole `Settings` object, flat, no
wrapper and **no version field**. Compatibility is per-field validation on
load: bad JSON falls back to `DEFAULTS` wholesale; `size`/`vehicle`/`biome`/
`kind` are checked against the live key arrays, so removing one auto-migrates
old saves; sliders are clamped to each control's range. The one real
migration is the car/kind reconciliation — a save with no `kind` believes the
car and derives the kind; a save with one replaces the car.

**Best laps and ghosts are not persisted.** A reload loses every lap time.

## Model features

Copy the shape of these:

- **A thing in the world: the trackside furniture.** `furniture.ts` places
  rails, drums, tyre stacks, boards and chevrons from the circuit's own
  curvature, returns `RAILS`/`BOLLARDS`/`SIGNS` plus their meshes and
  buffers, is rebuilt by `useTrack`, goes into the collision grid wholesale,
  is drawn from a static group in `main.ts`, and is recorded by the layout
  snapshot. That is every path a new world thing has to travel.
- **A tool that measures: `bench.ts`.** It pins the ground flat and the grip
  full (`setBenchFlat`, `setBenchGrip`, both restored in `finally`), drives
  one vehicle in isolation, and returns corner, accel and brake. Its results
  are pasted into `vehicles.ts` as comments beside the numbers they produced.
  Its file comment is required reading before touching vehicle physics.
- **A gate that is a claim, not a smoke test:** the frame-rate-independence
  test in `driving.test.ts`. It states a physical property, picks six frame
  rates and a fine-step reference, and holds the difference to 1%.

## The test API

`src/__tests__/sim.ts`, 95 lines, the only helper. Everything else calls
production code directly — no mocks of game modules, no fixtures.

    useTrack(seed, size, biome, wild?, kind?)   build the circuit, headless
    raceOn(seed, size, biome, vehicle)          a Race with a class in it
    script(t)                                   the standard eyeless input
    drive(race, seconds, hz, input?)            step and fold to one hash
    stateOf(race)                               FNV-1a of body + wheels
    gridUnder(level)                            is the start grid under water

Time is fixed and nested: `STEP = 1/120` in `game.ts`, `SUBSTEP = 1/480` in
`vehicle.ts`. `race.advance(dt, input)` takes whole 120ths and carries the
remainder; `race.step(dt, input)` takes one step at a caller-chosen size.
There is **no pause flag and no injected clock** — reset and step is the
practical equivalent. The only wall-clock seam is `performance.now()`, which
`disco.ts` reads directly and `modes.test.ts` has to spy on.

Chance is seeded everywhere that matters: `Math.random` appears **once** in
all of `src`, at [main.ts:711](src/main.ts:711), the random-circuit button,
which no test can reach. Terrain is a local mulberry32 over wave phase and
heading only, and seed 0 restores the shipped ground exactly.

Read results off `race.truck` (the physics body and its four wheels),
`race.shown` (the drawn, interpolated pose), `race.lap`, `race.clock`, and
the world-state globals `COLUMNS`, `PROPS`, `BOLLARDS`, `RAILS`, `SIGNS`,
`WATER_LEVEL` after `useTrack`.

**Five of six test files mutate module singletons with no `afterEach`.**
`TRACK_HALF`, the shape, the terrain waves, the water level and the prop
arrays are all module-level. Only `modes.test.ts` cleans up; `kinds.test.ts`
restores `setTrackHalf` by hand. Vitest's per-file isolation is the only
thing containing this, so **a new test in an existing file that forgets to
restore leaves the next test on the wrong road.** Restore in a `finally`.

## Edge-case checklist

For anything new, go through every path it can reach. By kind:

**A new biome** — the `BiomeKey` union and `BIOME_KEYS`; the full record,
whose `terrain.ampScale` must have exactly as many entries as `WAVES`; the
`BIOMES` registry; prop meshes if it wants kinds that do not exist. Then the
branches: `water.depth !== null` (four places, including the grid-clearance
rule in `circuit.ts` — the marsh comment in `biomes.ts` is the cautionary
tale), `dust: null`, `water.ice !== null`. `driving.test.ts` names biomes
explicitly rather than iterating, so add a row; both snapshots move.

**A new vehicle class** — the union and `VEHICLE_KEYS`; the spec, with
`rating` from `bench()` and `par` from `npm run calibrate`; the registry; a
body and a detail mesh. Then: **add it to a kind's `vehicles` array in
`kind.ts` or it can never be selected**, and **add a `VineBand` in
`concours.ts` — `BANDS[spec.key]` has no fallback and throws the moment
Concours is switched on.** `kinds.test.ts` hard-codes both kinds' vehicle
arrays; the driving snapshot gains a row.

**A new circuit kind** — the union and `KIND_KEYS`; the `KindSpec`; an entry
in `FALLBACK` **which must itself satisfy the kind's own limits**; a
five-band word table in `BANDS`. Then the ternaries that assume two kinds:
seed decorrelation and repair tries in `track.ts`, the select screen's
slowest-corner-versus-difficulty branch, the wild gate. Tests hard-code
`['rally','track']` in two files.

**A new size** — one row in `SIZES` and the union; everything else derives.
But check the size-conditional behaviour that is *not* in the table: the
sun-shadow box switching at `SIZE <= 1`, fog reach, flora thinning, ground
cell growth, minimap cell, and the light-capacity warning as posts multiply.

**A new mode** (disco/concours/line family) — the field, the default, the
`SliderKey` exclusion **or the panel builds a slider for it**, the boolean
load check, the `restoreDefaults` preserve list, the button table and its
apply branch. If it adds geometry: the dynamic group indices are a
hand-maintained positional list — **insert one in the middle and every later
group draws the wrong mesh.**

**A new prop kind** — a mesh factory and an entry in some biome's
`flora.kinds`. Watch `radiusOf` returning 0, which means a wheel goes through
it and is branched in two places that must stay in step; and `wet: true`,
which is the only thing planted on flooded ground.

**A new slider** — field, default, `CONTROLS` row, and the apply callback if
it is not read live. **If it changes what is built rather than how it looks,
`roadChanged` must learn about it and `useTrack` must take it.**

**A new terrain wave** — and then every `ampScale` must grow to match: four
biomes and two kinds. A short kind array is tolerated; a short *biome* array
silently drops the new wave.

**Always** — what it costs a frame at XL with the lamps on, and what it does
to the two recordings.

## Verifying

Headless for anything measured: `npm test` is node and needs no browser. For
anything **seen**, there is no automated path — the in-app browser pane
pauses its frames when hidden and its screenshots go stale, so drive the dev
server and use its own console handles, on a visible window:

- `measure(w = 1920, h = 1080, n = 120)` — renders into an offscreen texture,
  10 warm-up frames, then `n` fenced on `onSubmittedWorkDone()`. Returns
  `{ ms, mpx, msPerMpx, lights }`. Deliberately not rAF-timed: an
  uncomposited tab stops calling back, which reads as a scene that
  mysteriously got four times slower.
- `shoot(w, h, name)` — snaps the orbit to its target, draws one frame and
  POSTs it to the dev-only `/__shot` middleware, which writes
  `docs/<name>.png`. The renderer's own frame, not a screenshot of a pane.
  This made every image in `docs/` and every A/B in the README.
- `newTrack`, `switchVehicle`, `bench`, `vehicles`, `arena`, `renderer`,
  `orbit`, `lights`, `skids`, `timings`, `track`, `furniture`, `world`,
  `biomes`.

**Let the orbit settle before timing anything.** A figure in the README was
3.64 ms until someone noticed it had been taken three seconds after moving
the camera, with the frame still easing.

**A console `import('/src/track.ts')` is a different module instance with a
different circuit in it.** Use the `track` global, which is why it exists.

Never write over the player's settings: a dev browser's `arena.settings` is
theirs. Reset what a test script changed.

### The renderer

`package.json` pins `github:onion2k/artshape-render#v0.13.0`, but
`node_modules/artshape-render` **is a symlink to `~/projects/artshape-render`,
which is at v0.22.0** — nine minor versions of drift between what the
lockfile says and what you run, build and typecheck against. Three lines in
`vite.config.ts` exist only for that layout: `server.fs.allow` (without it the
tracer's worker fails to import and a traced frame never arrives),
un-ignoring `node_modules` in the watcher, and excluding it from
`optimizeDeps`.

**`npm install` or `npm ci` replaces the link with 0.13.0 and nothing
re-creates it.** No `postinstall`, no `prepare`. That downgrade is a silent
behaviour change, not a compile error — the driving snapshots never touch the
GPU, so `npm test` stays green through it. After any install, re-link and
look at a frame. A change the renderer needs belongs in that repo, held to
its own bar there, with a version bump here.

## Definition of done

The house's nine points. Here they mean: acceptance criteria as tests in
`src/__tests__` seen failing first; the checklist above for the kinds the
change touches; `npm test && npm run build` both green; the recordings
updated only deliberately, with every moved run looked at and explained; any
new always-true rule written as an invariant test beside its neighbours; the
frame measured with `measure()` before and after in the scenes it touches,
since no gate will do it for you; a `shoot()` of anything visual, looked at;
and every new test mutation-checked — put the bug back, watch it fail,
restore it.

Anything not verified is said plainly in the report.

## Sharp edges

Known, unfixed, and worth knowing before you trip:

- **`restoreDefaults` drops `kind` and `line`.** It preserves seed, size,
  vehicle, biome and the mode flags, but resets `kind` to `'rally'` while
  keeping the car — manufacturing exactly the invalid `{kind, vehicle}` pair
  that the load-time migration exists to repair, which then silently changes
  one of them on the next load. It also switches the racing line off, which
  is not what the button says it does.
- **`main.ts`'s header comment is stale** — it still describes the arena
  shooter, with shots, enemies and muzzle flashes that no longer exist.
  `track.ts`, the largest logic module, has no header at all.
- **`gridUnder` in the test helper re-implements `lowestOnGrid` from
  `circuit.ts`** with a different sampling step. The water tests can pass
  while the real flood rule drifts.
- **Untested:** `ghost.ts` — the headline feature, and nothing asserts a
  recorded lap replays where it was driven — plus `settings.ts`'s load and
  migration, the collision responses in `game.ts`, `Progress.update`'s wrap
  detection, and the handbrake, which no input script ever presses.
- `filigrees` in `main.ts` is the one cache with no eviction. Bounded by the
  four classes today; a genuine leak if its key ever becomes per-instance.

## Commits

Commit only when asked. A sentence summary in the house voice, a body saying
what changed and why, and two commits when a refactor and a feature land
together. There is no hook, so run `npm test && npm run build` yourself
first.
