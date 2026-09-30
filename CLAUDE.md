# ooerrace: working on it

A night circuit driven against the clock, in the browser: TypeScript, Vite,
and WebGPU through the game path of
[artshape-render](https://github.com/onion2k/artshape-render). `README.md` is
the design log: what was tried, what it measured and why it is the way it is,
and every number is justified there. This file says how the project is
changed. The house rules in `~/.claude/CLAUDE.md` apply on top of it, and where
they disagree this file wins.

*Written from what the code does on 2026-09-30, from commit `b186290`, and cut
to the house length the same day. What came out is in
[docs/known-faults.md](docs/known-faults.md).*

## Commands

    npm run dev          the game at http://localhost:5190 (strict port)
    npm test             the suite: node, no GPU, about ten seconds
    npm run build        tsc --noEmit && vite build; the only typecheck
    npm run calibrate    the pilot laps each class and reports par; minutes, not seconds

**There is no `check` and no pre-commit hook.** The gate before a commit is
`npm test && npm run build`, run by hand, both green. A type error passes
`npm test` on its own, because the typecheck lives only in `build`.

`npm run calibrate` (optionally with class names) is **a report, not a gate**:
it exits 0 whatever it prints, and a person hand-edits the `par` constants in
`vehicles.ts` from it.

## The gates

Every tolerance lives in its own test, with the reason beside it: this names
the gate, and the number is read where it is. All in `src/__tests__`:

| claim | gate |
| --- | --- |
| par predicts the pilot's best lap | `driving.test.ts` *par*; `wild.test.ts` *driven to par* |
| a class turns the circle its spec says | `driving.test.ts` *turns the circle* |
| top speed does not depend on the display's frame rate | `driving.test.ts` *whatever the display runs at* |
| the race is identical at any frame rate, bit for bit | `driving.test.ts` *the race clock* |
| the wheels sit on the drawn road, the body clear of it | `driving.test.ts` *sits on the drawn road* |
| roll damping fits inside the substep | `driving.test.ts` *roll damping* |
| every circuit clears its kind's own limit | `kinds.test.ts` *holds every circuit*; `wild.test.ts` *clear their limits* |
| nothing is placed on the road | `wild.test.ts` *put nothing on the road*; `kinds.test.ts` *width of the road* |
| the racing line stays on the tarmac and bends less than the road | `raceline.test.ts` |
| the grid is never under water, and water is lowered only as far as it demands | `water.test.ts` *the water* |
| disco's lights fit `LIGHT_CAPACITY` | `modes.test.ts` *disco night* |

Two **recordings**, which are snapshots and not invariants, in
`src/__tests__/__snapshots__`:

- `driving.test.ts` *matches the recorded trajectories* hashes the body after
  **every frame** of a scripted drive across classes, biomes, sizes and seeds.
  It catches the vehicle model, the clock, terrain, grip, water drag and every
  collision at once.
- `water.test.ts` *matches the recorded layout* records the arena as built:
  the props, posts, rails and bollards and the water depth.

A change meant to leave the driving alone must leave every hash alone. One
meant to change it runs `npx vitest -u`, **looks at which runs moved**, and says
why in the commit.

### What is not held

- **Performance: nothing.** No test measures a frame; README's *What it costs*
  is prose, re-checked by hand with `measure()`. A change that doubles the frame
  cost passes.
- **Looks: nothing.** No Playwright, no image comparison, no CI. The only check
  is `shoot()` and your own eyes.
- **No lint, no formatter, no fuzzer, no `invariants.ts`.** Match the
  surrounding style by reading it.
- **The renderer is not the version `package.json` pins:** see Verifying.
- **Untested,** among others, `ghost.ts` and `settings.ts`'s load and
  migration: see `docs/known-faults.md`.

## Layout

`src/circuit.ts` is the seam. `useTrack(seed, size, biome, wild, kind)` is
everything the CPU builds when a circuit changes, in a dependency order its own
comment calls non-negotiable, and nothing it draws. It exists so a test can
build exactly what the page builds with no canvas in sight. Anything new that
the circuit needs goes through it.

**Headless: every module but three.** The race (`game.ts`), the vehicle
(`vehicle.ts`), the circuit (`track.ts`) and the rest run in node. **The page is
`src/main.ts`**, the only module that touches the DOM or a device, with
`photo.ts` and `particles.ts`. Logic does leak into it, and that is the main
structural debt (`docs/known-faults.md`). The one that bites: `roadChanged` in
`main.ts` duplicates `useTrack`'s argument list, so **a setting that changes
what is built must be added to it, or the circuit silently fails to rebuild.**

**The import cycle.** `track → biomes → props → scene → track` only evaluates
cleanly when entered from the circuit's side: `./circuit` before `./raceline` in
`main.ts`, `../circuit` first in `sim.ts`. **A new module that imports `track`
inherits this.**

**Content** is a typed table in its own module, and the pickers iterate it, so
adding one is mostly a line there: biomes in `biomes.ts`, vehicle classes in
`vehicles.ts` (bodies in `models.ts`), circuit kinds in `kind.ts`, sizes in
`world.ts`, terrain waves in `terrain.ts`, prop meshes in `props.ts`, the
settings list in `settings.ts`.

**The save** is one key, `'arena.settings'`, the game's old name; renaming it
would throw away everyone's settings. The whole `Settings` object, flat, no
version field: each field is validated on load (keys against the live key
arrays, sliders clamped) and bad JSON falls back to `DEFAULTS`. A save with no
`kind` believes the car, and one with a `kind` replaces the car. **Best laps and
ghosts are not persisted.**

## Model features

Copy the shape of these:

- **A thing in the world: `furniture.ts`.** It places rails, drums, tyre stacks
  and boards from the circuit's own curvature and returns `RAILS`, `BOLLARDS`
  and `SIGNS` with their meshes. `useTrack` rebuilds it, the collision grid takes
  it wholesale, `main.ts` draws it from a static group and the layout recording
  counts it. That is every path a new world thing travels.
- **A tool that measures: `bench.ts`.** It pins the ground flat and the grip
  full (`setBenchFlat`, `setBenchGrip`, both restored in `finally`), drives one
  vehicle in isolation and returns corner, accel and brake, which are pasted into
  `vehicles.ts` beside the numbers they produced. Its file comment is required
  reading before touching vehicle physics.
- **A gate that is a claim: the frame-rate test in `driving.test.ts`.** A
  physical property, driven at several frame rates against a fine-step
  reference.

## The test API

`src/__tests__/sim.ts` is the only helper. Everything else calls production
code directly, with no mocks of game modules and no fixtures.

    useTrack(seed, size, biome, wild?, kind?)   build the circuit, headless
    raceOn(seed, size, biome, vehicle)          a Race with a class in it
    script(t)                                   the standard eyeless input
    drive(race, seconds, hz, input?)            step and fold to one hash
    stateOf(race)                               FNV-1a of body + wheels
    gridUnder(level)                            is the start grid under water

Time is fixed and nested: `STEP` in `game.ts`, `SUBSTEP` in `vehicle.ts`.
`race.advance(dt, input)` takes whole steps and carries the remainder;
`race.step(dt, input)` takes one step of a size the caller picks. There is **no
pause flag and no injected clock**, so reset and step is the equivalent. The
only wall-clock seam is `performance.now()`, which `disco.ts` reads directly
and `modes.test.ts` spies on.

Chance is seeded everywhere that matters: `Math.random` appears once in `src`,
in the random-circuit button in `main.ts`, which no test reaches. Read results
off `race.truck`, `race.shown`, `race.lap`, `race.clock`, and the world-state
globals (`COLUMNS`, `PROPS`, `BOLLARDS`, `RAILS`, `SIGNS`, `WATER_LEVEL`) after
`useTrack`.

**Tests share module singletons** (`TRACK_HALF`, the shape, the terrain waves,
the water level, the prop arrays) and most files never reset them. **A new test
that changes one restores it in a `finally`,** or the next test runs on the
wrong road.

## Edge-case checklist

For anything new, go through every path it can reach. By kind:

**A new biome:** the `BiomeKey` union and `BIOME_KEYS`; the full record, whose
`terrain.ampScale` must have exactly as many entries as `WAVES`; the `BIOMES`
registry; prop meshes if it wants kinds that do not exist. Then the branches:
`water.depth !== null` (four places, including the grid-clearance rule in
`circuit.ts`; the marsh comment in `biomes.ts` is the cautionary tale),
`dust: null`, `water.ice !== null`. `driving.test.ts` names biomes rather than
iterating, so add a row; both recordings move.

**A new vehicle class:** the union and `VEHICLE_KEYS`; the spec, with `rating`
from `bench()` and `par` from `npm run calibrate`; the registry; a body and a
detail mesh. Then **add it to a kind's `vehicles` array in `kind.ts`, or it can
never be selected**, and **add a `VineBand` in `concours.ts`, because
`BANDS[spec.key]` has no fallback and throws when Concours is switched on.**
`kinds.test.ts` hard-codes both kinds' vehicle arrays, and the driving recording
gains a row.

**A new circuit kind:** the union and `KIND_KEYS`; the `KindSpec`; an entry in
`FALLBACK` **that itself satisfies the kind's limits**; a five-band word table in
`BANDS`. Then the ternaries that assume two kinds: seed decorrelation and repair
tries in `track.ts`, the select screen's slowest-corner branch, the wild gate.
Two test files hard-code `['rally','track']`.

**A new size:** one row in `SIZES` and the union; the rest derives. Check the
size-conditional behaviour that is not in the table: the sun-shadow box at
`SIZE <= 1`, fog reach, flora thinning, ground cell growth, the minimap cell and
the light-capacity warning as posts multiply.

**A new mode** (the disco, concours and line family): the field, the default,
the `SliderKey` exclusion **or the panel builds a slider for it**, the boolean
load check, the `restoreDefaults` preserve list, the button table and its apply
branch. If it adds geometry, the dynamic group indices are a hand-kept
positional list: **insert one in the middle and every later group draws the
wrong mesh.**

**A new prop kind:** a mesh factory and an entry in some biome's `flora.kinds`.
Watch `radiusOf` returning 0, which means a wheel goes through it and is
branched in two places that must stay in step, and `wet: true`, the only thing
planted on flooded ground.

**A new slider:** the field, the default, a `CONTROLS` row, and the apply
callback if it is not read live. **If it changes what is built rather than how
it looks, `roadChanged` must learn of it and `useTrack` must take it.**

**A new terrain wave:** every `ampScale` must grow to match, in four biomes and
two kinds. A short kind array is tolerated, and a short *biome* array silently
drops the new wave.

**Always:** what it costs a frame at XL with the lamps on, and what it does to
the two recordings.

## Verifying

Headless for anything measured: `npm test` needs no browser. For anything
**seen** there is no automated path, and the in-app browser pane pauses its
frames when hidden, so drive the dev server on a visible window and use its
console handles:

- `measure(w = 1920, h = 1080, n = 120)` renders into an offscreen texture,
  warms up, then times `n` frames fenced on `onSubmittedWorkDone()`. It returns
  `{ ms, mpx, msPerMpx, lights }`. It is deliberately not rAF-timed, since an
  uncomposited tab stops calling back. **Let the orbit settle before timing.**
- `shoot(w, h, name)` snaps the orbit to its target, draws one frame and POSTs
  it to the dev-only `/__shot` middleware, which writes `docs/<name>.png`. It is
  the renderer's own frame, not a screenshot of a pane.
- Other handles (`newTrack`, `switchVehicle`, `bench`, `track`, `arena`,
  `orbit`, ...) are put on `globalThis` by `Object.assign` in `main()`.

**A console `import('/src/track.ts')` is a different module instance with a
different circuit in it.** Use the `track` global.

Never write over the player's settings: a dev browser's `arena.settings` is
theirs.

**The renderer you run is not the one `package.json` pins:**
`node_modules/artshape-render` is a symlink to `~/projects/artshape-render`.
**`npm install` or `npm ci` replaces it with the pinned release and nothing
re-creates it**, silently, since `npm test` never touches the GPU. After any
install, re-link and look at a frame. A change the renderer needs belongs in
that repo, with a version bump here. Detail in `docs/known-faults.md`.

## Definition of done

The house's nine points, here: tests in `src/__tests__` seen failing first; the
checklist above; `npm test && npm run build` green; a recording updated only
deliberately, every moved run looked at; the frame measured with `measure()`
before and after, since no gate will; a `shoot()` of anything visual, looked at;
new tests mutation-checked; and what was not verified said plainly.

## Commits

The house rules apply. There is no hook, so run `npm test && npm run build`
first.
