# Plan: engine ergonomics

Companion to [ROADMAP.md](ROADMAP.md). Tasks are small enough to commit one
at a time. "Accept" is what must be true before the task is done.

## Phase 0: Reconcile the fork

Status of the facts: 40 files differ between this repo's `engine/src` and
Dreamscape's `engine/src`. I have **not** yet checked which side is newer for
each file; that is task 0.1.

| Task | Do | Accept |
|---|---|---|
| 0.1 | For each differing file, three-way compare using `git log` on both repos. Write the result (newer side, intent, conflicts) to `docs/fork-audit.md`. | Every one of the 40 files has a verdict |
| 0.2 | Merge Dreamscape-side engine changes into this repo on a branch. Split `audio.rs` into `audio/{mod,native,web}.rs`. Keep this repo's Material-as-component change (`38d0ac5`) and `pub use egui` restore. | Workspace builds, all `engine/tests` pass, `sandbox` runs |
| 0.3 | Point Dreamscape's workspace at this repo (git dependency pinned to a rev, or path dependency during development). Delete its `engine/` copy. | Dreamscape builds native and emscripten, its tests pass |
| 0.4 | Do the same for `nn_viz` and `dungeon_oxide`. | Both build against the shared engine |
| 0.5 | Add a CI job that builds a downstream game against the engine on every PR. | Breaking the API fails CI |

Risks: the Material change and Dreamscape's own engine edits may touch the
same code (`ecs/mod.rs`, `renderer/mod.rs`). Resolve by tests, not by taking
one side wholesale. Dreamscape's licence note (MIT for `engine/`, proprietary
for the game) must survive the move.

## Phase 1: Effect primitives

Today: dash ghosts, burst cubes and echoes each track `(Entity, born_at)` in
a `Vec` on the game struct and despawn by hand. STILLNESS and PHASE keep a
`Vec<(Entity, Arc<GpuTexture>)>` to put textures back.

| Task | Do | Accept |
|---|---|---|
| 1.1 | `Lifetime { born, duration }` component and a `tick_lifetimes(world, now)` that despawns expired entities. | Test: entity gone after `duration`, others untouched |
| 1.2 | `Fade` / `ScaleOverLife` curve component read by the same system. Support linear and ease-out. | Test: scale at 0, 50%, 100% of life |
| 1.3 | `TextureOverride { previous }` component. Adding swaps the texture; removing restores it, including when the entity is despawned mid-effect. | Test: texture restored after remove; no leak on despawn |
| 1.4 | Port Dreamscape's dash ghosts, bursts and STILLNESS/PHASE to the new components. Delete `dash_ghosts`, `ghost_timer`, `chilled`, `phased`. | Same visuals; `update_ability_fx` is spawn-only; existing tests green |

Check first: `particles.rs` and `animation.rs` may already cover part of this.
Reuse before adding.

## Phase 2: Audio buses

| Task | Do | Accept |
|---|---|---|
| 2.1 | Define an `AudioBackend` trait covering what `native.rs` and `web.rs` both implement today. | Both backends implement it; no duplicated method lists |
| 2.2 | `Bus` API: `play_loop(name, samples, rate, offset)`, `set_gain(name, g)`, `stop(name)`. Replace the hard-coded music and layer sinks. Music volume scales all buses. | Native test with a null sink: gains compose correctly |
| 2.3 | Shared loop clock so named loops started with the same offset stay in time. | Test: two buses of equal length stay phase-aligned over N loops |
| 2.4 | Port Dreamscape's tension layer to a bus. | Behaviour unchanged; a second layer takes one call |

Risk: web audio and rodio differ in how loops and offsets behave. Verify
alignment on both, not just native.

## Phase 3: Systems and scheduling

| Task | Do | Accept |
|---|---|---|
| 3.1 | Design note: a `Schedule` of named systems `fn(&mut World, &mut Resources, dt)` with `before`/`after` ordering and run conditions (for example "only in Play mode"). Decide how it coexists with the existing `Game` trait (it should wrap, not replace). | Note reviewed; one worked example |
| 3.2 | Implement `Schedule`, topological ordering, and cycle detection with a clear error. | Tests: order respected, cycle rejected, condition skips a system |
| 3.3 | A typed `Resources` map so systems read shared state without a god struct. | Test: insert and get by type |
| 3.4 | Move three Dreamscape features onto it: adaptive music, ability FX, echoes. | Each is a module that registers itself; their fields leave `DreamscapeGame` |
| 3.5 | Measure: record `main.rs` line count and the number of fields on `DreamscapeGame` before and after. | Numbers go in the PR description |

This is the largest phase and the only one that changes how games are
structured. Do it after 0 and 1 so it moves already-clean code. Keep the old
path working so games can migrate one feature at a time.

## Phase 4: Headless test harness

| Task | Do | Accept |
|---|---|---|
| 4.1 | Audit `testkit.rs` for what it already offers. | Short list in the PR |
| 4.2 | A `HeadlessContext` with a stub renderer and texture handles that never touch GL. | Constructs without a window or GL |
| 4.3 | `step(n_frames, dt)` plus input injection (press/release by action). | Test: inject jump, entity rises |
| 4.4 | First real test in Dreamscape: STILLNESS tints enemies and restores them on expiry. | Test passes in CI with no display |

Depends on how tightly GL calls are woven into the game's textures and
meshes. If it is too tight, Phase 3's resources map is what loosens it.

## Phase 5: Tunables and telemetry

| Task | Do | Accept |
|---|---|---|
| 5.1 | `Tunables` resource loaded from RON, watched by `hotreload.rs`. Typed getters with a compiled-in default. | Editing the file changes the value within a frame or two |
| 5.2 | Move Dreamscape's music `NEAR`/`FAR`, form multipliers and `GHOST_LIFE` onto it. | Retune with the game running |
| 5.3 | Lift the balance CSV logger into an engine `Telemetry` hook (env var enables it; zero cost when off). | Dreamscape's `DREAMSCAPE_BALANCE` still works via the hook |

Release builds must compile the defaults in and ignore the file unless a flag
is set, so players cannot retune a shipped game by accident.

## Phase 6: Data-driven unlocks (optional)

Prototype in Dreamscape first: a RON table of `condition -> reward` covering
forms, records and attunement, with one function that explains why something
is locked (for the title-screen hints). Promote to the engine only if a second
game wants it.

## Definition of done (every phase)

- Native and emscripten builds pass.
- New code has tests; existing tests stay green.
- A Dreamscape feature is ported and is smaller afterwards.
- GUIDE.md documents the new API.
- Public API changes carry a migration note.
