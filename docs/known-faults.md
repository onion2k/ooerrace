# Known faults, and where logic has leaked

Moved out of `CLAUDE.md` on 2026-09-30, when that file was cut to the house
length. Nothing here is fixed; it is written down so that nobody trips on it
twice. The facts hold on commit `b186290`. Check one against the code before
building on it.

## Logic that lives in the page

`src/main.ts` is meant to be only the page. These are the places it is not,
and each is a place a change can be missed by every test:

- `preview()` works out the select screen's verdicts inline: the difficulty
  band, the length, the "tighter than the car can turn" warning. The rules for
  what a circuit is *said to be* live in a function that writes the DOM.
- `roadChanged` is a five-field comparison that duplicates `useTrack`'s
  argument list. Add a setting that changes the circuit and this is the line
  that silently fails to rebuild it.
- `newTrack()` is the real change-circuit transaction, and it is neither
  exported nor testable.
- The concours presentation blend, the plinth and the per-lap gem maths are in
  `upload()`, not in `concours.ts`.
- The skid and wheel-effect thresholds (`slide > 0.35 && speed > 300`) are
  driving constants that live in the draw loop.
- `main.ts`'s header comment still describes the arena shooter, with shots,
  enemies and muzzle flashes that no longer exist. `track.ts`, the largest
  logic module, has no header at all.

## Bugs and traps

- **`restoreDefaults` drops `kind` and `line`.** It preserves seed, size,
  vehicle, biome and the mode flags, but resets `kind` to `'rally'` while
  keeping the car. That manufactures the invalid `{kind, vehicle}` pair the
  load-time migration exists to repair, which then silently changes one of them
  on the next load. It also switches the racing line off, which is not what the
  button says it does.
- **`gridUnder` in `src/__tests__/sim.ts` re-implements `lowestOnGrid`** from
  `circuit.ts` with a different sampling step, so the water tests can pass
  while the real flood rule drifts.
- **`filigrees` in `main.ts` is the one cache with no eviction.** It is bounded
  by the four vehicle classes today, and a genuine leak if its key ever becomes
  per-instance.
- **Five of six test files mutate module singletons with no `afterEach`:**
  `TRACK_HALF`, the shape, the terrain waves, the water level and the prop
  arrays. Only `modes.test.ts` cleans up, and `kinds.test.ts` restores
  `setTrackHalf` by hand. Vitest's per-file isolation is all that contains it.

## Untested

`ghost.ts` (the headline feature: nothing asserts that a recorded lap replays
where it was driven), `settings.ts`'s load and migration, the collision
responses in `game.ts`, `Progress.update`'s wrap detection, and the handbrake,
which no input script ever presses.

## The renderer link

`package.json` pins a tagged release of artshape-render, but
`node_modules/artshape-render` is a symlink to `~/projects/artshape-render`,
which is at a much later version. The pin is not what you run, build or
typecheck against. Three lines in `vite.config.ts` exist only for that layout:
`server.fs.allow` (without it the tracer's worker fails to import and a traced
frame never arrives), un-ignoring `node_modules` in the watcher, and excluding
it from `optimizeDeps`.

`npm install` or `npm ci` replaces the link with the pinned release and
nothing re-creates it: there is no `postinstall` and no `prepare`. That is a
silent behaviour change, not a compile error, and `npm test` stays green
through it because the driving recordings never touch the GPU. Re-link after
any install, and look at a frame.

## Two cautionary tales

Both are told in full in `README.md`.

- The vehicle refactor was checked by hand against a worktree, and that check
  only ever drove seed 0 in the forest. That is how fords draining on every
  other circuit got through. It is why the driving recording covers several
  seeds, biomes, sizes and classes.
- A frame figure in the README read 3.64 ms because it was taken three seconds
  after the camera moved, with the frame still easing. Let the orbit settle
  before timing anything.
