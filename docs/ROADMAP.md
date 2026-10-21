# Jame Engine roadmap

Goal: make games on Jame faster to build and safer to change. The pressure
comes from Dreamscape, the largest game on the engine (~34k lines, an 8k-line
`main.rs`, one `DreamscapeGame` struct holding most state). Each phase below
ships on its own and is judged by a concrete game change it makes smaller.

The detailed plan, with tasks and acceptance tests, is in
[PLAN-engine-ergonomics.md](PLAN-engine-ergonomics.md).

## Where things stand

- The engine is ~9.4k lines (standalone repo) with ECS, renderer, shader
  profiles, editor panels, physics, pathfinding, AI, rigs, scripting, net,
  audio, hot reload and a profiler.
- **The engine has forked.** Dreamscape carries its own copy under
  `engine/` that differs from this repo in 40 files (~1,765 changed lines),
  including audio split into `audio/{native,web}.rs` while this repo still has
  a single `audio.rs`. Features have been landing in the copy, not here.
  Everything below depends on fixing that first.
- The `Game` trait is `update(ctx, dt)` + `render(ctx)`. The engine offers no
  structure beyond that, so every game hand-orders its own per-frame work.

## Phases

| # | Phase | Why | Size | Depends on |
|---|---|---|---|---|
| 0 | Reconcile the fork | One engine, one source of truth | M | none |
| 1 | Effect primitives | `Lifetime`/`Fade` and `TextureOverride` replace three hand-rolled spawn/fade/despawn patterns | S | 0 |
| 2 | Audio buses | N named looping tracks sharing a clock; native and web from one trait | M | 0 |
| 3 | Systems and scheduling | Features plug in; `main.rs` can be split | L | 0, 1 |
| 4 | Headless test harness | Test game behaviour without a GL context | M | 3 helps, not required |
| 5 | Tunables and telemetry | Retune constants without a rebuild; shared balance logging | S | 0 |
| 6 | Data-driven unlocks (optional, game-side first) | Adding a form/record becomes a data edit | M | none |

Suggested order: **0 → 1 → 2 → 5**, then **3 → 4**. Phases 1, 2 and 5 are
small and independent once 0 is done, so they can run in parallel.

## Phase summaries

**0. Reconcile the fork.** Work out, file by file, which side is newer, merge
into this repo, then make Dreamscape (and `nn_viz`, `dungeon_oxide`) consume
the engine as a dependency instead of a copy. Exit: `diff` between the
Dreamscape build's engine and this repo is empty because there is only one.

**1. Effect primitives.** Dash ghosts, burst cubes and echoes each spawn an
entity, record a birth time, shrink it, then despawn it. Phase and Stillness
swap a texture and remember the old one to restore it. Make both components
and ticking systems in the engine. Exit: Dreamscape's `update_ability_fx` and
burst code shrink to spawning the component.

**2. Audio buses.** The recent adaptive-music change added a second
hard-coded sink (`layer`, `layer_gain`) and had to be written twice, for
native and web. Replace it with named tracks that have a gain and share one
loop clock. Exit: a third layer is one line of game code and works on both
backends.

**3. Systems and scheduling.** A registry of systems with explicit ordering
and run conditions, so a feature adds a system instead of editing the main
loop. Exit: at least three Dreamscape features (adaptive music, ability FX,
echoes) run as registered systems and `main.rs` loses their fields.

**4. Headless test harness.** A world plus a stub context that skips GL so
tests can advance N frames and assert on entities. Exit: a test proves
Stillness restores enemy textures when it ends.

**5. Tunables and telemetry.** Load named constants (music `NEAR`/`FAR`, form
multipliers, `GHOST_LIFE`) from a RON file watched by hot reload, and turn the
balance CSV logger into an engine hook. Exit: edit a value, see it in the
running game.

**6. Data-driven unlocks.** A condition → reward table in RON for forms,
records and attunements. Prove it in Dreamscape before deciding whether it
belongs in the engine at all.

## Non-goals

- A new renderer, editor rewrite, or scripting-language changes.
- Breaking the public API without a migration note in the PR.
- Anything that stops existing seeds from replaying in Dreamscape. New
  randomness uses its own seeded stream.

## Working rules

- One PR per phase, tests first, commit per green task.
- Each phase is verified by porting a real Dreamscape feature, not by a toy.
- Native and web (emscripten) must both build before a phase is done.
