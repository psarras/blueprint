# Architecture

This repository is a **game blueprint**, not a sealed engine.

The intended workflow is:

1. keep the reusable plumbing in the `core_*.odin` files,
2. keep game-specific rules in `sauce/game.odin` and `sauce/entity.odin`,
3. change the source directly when the project needs a new feature.

That philosophy shows up everywhere in the codebase: the architecture is deliberately flat, direct, and easy to cut apart.

## 1. High-level structure

### Top-level folders

- `sauce/` — all Odin source.
- `res/` — runtime assets loaded by the game.
- `asset_workbench/` — authoring pipeline inputs (Aseprite, FMOD project, export helpers).
- `sauce/build/` — custom build script.
- `sauce/sokol/`, `sauce/fmod/`, `sauce/steamworks/` — bundled native/runtime integrations.

### Important source files

- `sauce/core_main.odin` — app entrypoint, Sokol callback wiring, frame loop.
- `sauce/game.odin` — main gameplay megafile; most feature work lands here.
- `sauce/entity.odin` — entity allocation, handles, lifetime, lookup.
- `sauce/core_render.odin` — render setup, sprite atlas, quad batching, GPU submission.
- `sauce/core_draw.odin` / `sauce/core_draw_text.odin` — high-level drawing API.
- `sauce/core_input.odin` — input collection and per-frame state.
- `sauce/core_sound.odin` — FMOD init/playback helpers.
- `sauce/game_utils.odin` — game-facing helpers for spaces, timing, actions, UI helpers.
- `sauce/build/build.odin` — generates platform/shader files and performs builds.

## 2. The main runtime loops

The project is organized around a few simple loops layered on top of each other.

## 2.1 Platform loop: Sokol owns the app lifecycle

`main()` in `sauce/core_main.odin` calls `sapp.run(...)` and hands Sokol these callbacks:

- `core_app_init`
- `core_app_frame`
- `core_app_shutdown`
- `event_callback`

That means Sokol is the outermost loop provider. The blueprint plugs its own architecture into those callbacks.

## 2.2 Initialization loop: bootstrap once, then hand control to the game

`core_app_init()` performs startup in this order:

1. restores Odin runtime context,
2. starts the timebase with `utils.seconds_since_init()`,
3. initializes FMOD via `sound_init()`,
4. initializes entity defaults via `entity_init_core()`,
5. allocates the single `Game_State`,
6. installs the window resize callback,
7. initializes rendering via `render_init()`,
8. calls `app_init()` for game-specific startup.

Important consequence: the game layer assumes rendering, sound, and the entity core already exist before `app_init()` runs.

## 2.3 Frame loop: one callback per rendered frame

`core_app_frame()` is the real heartbeat.

Each frame it:

1. measures elapsed real time,
2. clamps `frame_time` to avoid giant catch-up steps,
3. writes `ctx.delta_t`,
4. points `ctx.gs` at the single `Game_State`,
5. points the input helpers at `_actual_input_state`,
6. handles global input like `Alt+Enter` fullscreen toggle,
7. starts the render frame with `core_render_frame_start()`,
8. calls `app_frame()`,
9. flushes rendering with `core_render_frame_end()`,
10. clears transient input flags,
11. frees the temp allocator,
12. increments `app_ticks`.

This is the most important architectural fact in the project:

**everything gameplay-related ultimately hangs off `app_frame()` inside a variable-timestep frame callback.**

## 2.4 Gameplay loop: `app_frame()` coordinates the whole game layer

`app_frame()` in `sauce/game.odin` is the game-facing orchestration point.

Right now it does five major things:

1. draws screen-space UI text,
2. keeps ambient audio alive with `sound_play_continuously(...)`,
3. calls `game_update()`,
4. calls `game_draw()`,
5. updates the listener and master volume with `sound_update(...)`.

The file comment is accurate: this is the place where future game modes would branch.

Typical expansion path:

- title screen,
- save select,
- gameplay,
- pause menu,
- dialogue mode,
- debug tools.

The intended architecture is to turn `app_frame()` into the top-level mode switch, while `game_update()` / `game_draw()` remain the active in-game path.

## 2.5 Entity update loop: scratch list -> animate -> run behavior

`game_update()` is the current world simulation loop.

Per frame it:

1. clears `ctx.gs.scratch`,
2. on the first tick creates the player entity,
3. rebuilds a temporary list of valid entity handles with `rebuild_scratch_helpers()`,
4. iterates all entity handles,
5. advances animation with `update_entity_animation(e)`,
6. calls the entity's `update_proc` if one exists,
7. reacts to click input for a sample sound effect,
8. moves the camera toward the player,
9. increments `game_time_elapsed` and `ticks` in a deferred block.

That temporary helper list is important: the game does **not** iterate the raw fixed array directly in most gameplay code. Instead it builds a compact per-frame working set and loops that.

## 2.6 Draw submission loop: gameplay code queues quads, renderer flushes once

The draw path is split into two phases.

### Phase A: gameplay submits draw commands

`game_draw()` and helper functions like `draw_sprite()` / `draw_rect()` do not render immediately.
They append quads into `draw_frame.quads[z_layer]`.

Relevant ideas:

- `push_coord_space(...)` switches between world space, screen space, or clip space.
- `push_z_layer(...)` controls draw ordering.
- draw helpers convert high-level calls into quad vertices.
- sprites and text both end up as batched quad submissions.

### Phase B: renderer flushes the batch

`core_render_frame_end()`:

1. merges all `ZLayer` arrays into one flat quad buffer,
2. uploads that buffer to the GPU,
3. binds the sprite atlas and font texture,
4. applies the shader pipeline,
5. uploads shader uniforms from `draw_frame.shader_data`,
6. issues one indexed draw call for all queued quads,
7. commits the pass.

Architecturally, this means the game layer only worries about **what** to draw and **which layer/space** to draw it in. The renderer owns **when** GPU submission happens.

## 2.7 Input loop: events accumulate, gameplay consumes

`event_callback()` in `sauce/core_input.odin` receives Sokol events and mutates `_actual_input_state`.

The frame loop then points the global `state` pointer at that struct before gameplay runs.

The pattern is:

- OS/window system emits events,
- Sokol translates them,
- `event_callback()` stores them,
- gameplay queries them with helpers like `key_pressed`, `key_down`, `is_action_down`,
- gameplay may consume them with `consume_key_pressed`.

This is why input feels immediate while still allowing one-frame events like presses/releases.

## 2.8 Audio loop: fire events, then update listener each frame

Audio is centered around FMOD helpers in `sauce/core_sound.odin`.

Startup:

- create the FMOD Studio system,
- load `Master.bank` and `Master.strings.bank`,
- cache the core system and master channel group.

Runtime:

- `sound_play(...)` fires one-shot events,
- `sound_play_continuously(...)` keeps a looping event alive by unique id,
- `sound_update(...)` runs FMOD update, applies volume, and updates listener position.

There is also an `update_sound_emitters()` helper for auto-cleaning continuous emitters, but it is not currently wired into `app_frame()`. If you expand continuous world audio heavily, that function is an obvious integration point.

## 2.9 Build loop: source generation happens before compile

The project has a custom build step in `sauce/build/build.odin`.

Before compiling, it:

1. writes `sauce/generated.odin` with platform/game-kind constants,
2. runs `sokol-shdc-*` on `sauce/shader.glsl`,
3. produces `sauce/generated_shader.odin`,
4. builds the `sauce` package,
5. copies platform runtime libraries,
6. copies `res/` for release builds.

This matters because rendering changes often involve both:

- editing `shader.glsl`, and
- letting the build step regenerate shader bindings.

## 3. Core data model

## 3.1 One main game state

The project currently uses one heap-allocated `Game_State` and exposes it through `ctx.gs`.

That state contains:

- frame/game counters,
- camera position,
- entity storage,
- a few gameplay handles like `player_handle`,
- per-frame scratch data.

This is intentionally centralized. New features generally add state directly into `Game_State` unless there is a strong reason not to.

## 3.2 A simple entity megastruct

The entity system is intentionally small.

Each `Entity` holds:

- an `Entity_Handle`,
- a `kind`,
- `update_proc` and `draw_proc`,
- generic transform/animation/render fields,
- a small per-frame scratch block.

This is not a full archetype ECS. It is a pragmatic hybrid:

- fixed backing array for storage,
- handle validation via unique ids,
- free-list reuse,
- function-pointer based behavior dispatch,
- one broad gameplay struct where project-specific fields can accumulate.

That is why the README calls it a “dead simple entity gameplay programming ECS”.

## 3.3 Temporary frame scratch is a first-class pattern

Two scratch mechanisms are used repeatedly:

- `ctx.gs.scratch` for transient gameplay data,
- `context.temp_allocator` for per-frame allocations.

At the end of every frame, the temp allocator is freed wholesale.

That means feature code can lean on frame-local dynamic arrays and temporary views without building long-lived memory management around them.

## 4. Rendering architecture

## 4.1 Asset loading model

At startup, `render_init()` loads every sprite declared in `Sprite_Name` from `res/images/<name>.png`.

Then it:

- decodes each PNG,
- packs them into a single atlas,
- stores each sprite's atlas texture coordinates in `sprites[sprite].atlas_uvs`.

So the renderer's contract is explicit:

- if a sprite is not in `Sprite_Name`, it is invisible to the atlas loader,
- if the PNG does not exist, startup asserts.

## 4.2 Draw API surface

Most gameplay rendering should go through:

- `draw_sprite(...)`
- `draw_rect(...)`
- `draw_text(...)`
- `draw_sprite_entity(...)`

These are the stable high-level hooks for features.

## 4.3 Coordinate spaces

The renderer uses `Coord_Space` to decide projection and camera behavior.

Common spaces:

- `get_world_space()` — gameplay world, camera-following.
- `get_screen_space()` — UI/screen overlay.
- `Coord_Space{proj=Matrix4(1), camera=Matrix4(1)}` — raw clip space, used for full-screen passes/backgrounds.

This split is key when adding features: most bugs around rendering placement come from using the wrong space, not the wrong draw call.

## 4.4 Z layers are the draw-order backbone

`ZLayer` is the project's main sorting model.

Current layers include:

- `background`
- `shadow`
- `playspace`
- `vfx`
- `ui`
- `tooltip`
- `pause_menu`
- `top`

The renderer preserves layer order while keeping within-layer order mostly based on submission order.

If you add a feature that needs consistent stacking, start by deciding its layer.

## 4.5 Shader extensibility

Two extension hooks already exist for shader-driven features:

- `Quad_Flags`
- `params: Vec4`

Those values are stored per vertex and passed through the render pipeline. That makes them the natural path for custom materials, screen-space effects, palette tricks, dissolve effects, distortion, special particles, and similar rendering features.

## 5. How new features are expected to be added

The architecture is optimized for direct source edits, not plugin-style extension points.

## 5.1 Adding a new gameplay entity

Use this when the feature is something world-like: enemies, pickups, projectiles, breakables, interactables.

Typical path:

1. Add a new value to `Entity_Kind`.
2. Add any persistent data fields to `Entity`.
3. Create `setup_<feature>(e: ^Entity)`.
4. Register it in `entity_setup()`.
5. Spawn it with `entity_create(.your_kind)`.
6. Put per-frame behavior in `update_proc`.
7. Put rendering behavior in `draw_proc`.

Use this path when the feature naturally has:

- a position,
- lifetime,
- animation,
- per-frame behavior,
- optional sound hooks.

## 5.2 Adding a non-entity gameplay system

Use this when the feature is global rather than per-entity: quests, weather, screen shake, combat director, dialogue state, save state.

Typical path:

1. Add persistent state to `Game_State`.
2. Add helper procs in `game.odin` or `game_utils.odin`.
3. Call update logic from `game_update()`.
4. Call draw/UI logic from `game_draw()` or `app_frame()`.

Rule of thumb:

- if the feature is many instances of similar world objects, prefer entities,
- if the feature is singleton/global mode state, prefer `Game_State` fields plus plain procs.

## 5.3 Adding a new input action

Typical path:

1. add a new value to `Input_Action`,
2. map it in `action_map`,
3. query it with `is_action_pressed/down/released`,
4. consume it if needed.

This keeps gameplay code independent from raw key codes.

## 5.4 Adding a new sprite or animation

Typical path:

1. add a `.png` to `res/images/`,
2. add the sprite name to `Sprite_Name`,
3. add `sprite_data` if it needs multiple frames, offset, or pivot,
4. use `draw_sprite(...)` or `entity_set_animation(...)`.

For strip animations, `sprite_data[...].frame_count` is the key piece. The renderer already shifts atlas UVs based on `anim_index`.

## 5.5 Adding particles or VFX

There is no dedicated particle subsystem yet, but the current architecture makes three viable approaches obvious.

### Option A: lightweight transient particle array in `Game_State`

Best for:

- sparks,
- dust puffs,
- hit flashes,
- floating numbers,
- short-lived local VFX.

Recommended shape:

- add `particles: [dynamic]Particle` or fixed-array storage to `Game_State`,
- define a `Particle` struct with position, velocity, lifetime, sprite, color, size, rotation,
- update it once per frame in `game_update()`,
- draw it in `game_draw()` on `ZLayer.vfx`.

Why this fits the blueprint:

- uses the existing variable-timestep loop,
- uses the existing draw APIs,
- avoids forcing particles into the entity megastruct if they are too short-lived,
- keeps feature-specific data in the game layer.

### Option B: particle entities

Best for:

- projectiles that also look like particles,
- VFX that need collisions or interaction,
- VFX that should reuse entity animation/draw behavior.

Recommended path:

- add an entity kind like `.particle` or `.projectile`,
- store per-instance state on `Entity`,
- use `update_proc` / `draw_proc` like any other entity.

Why to choose this:

- you get handles, reuse, and world iteration for free,
- the effect behaves like a gameplay object, not just a visual.

### Option C: shader-driven VFX layer

Best for:

- palette swaps,
- heat distortion,
- dissolve,
- wind, water, glow masks,
- material-like per-quad effects.

Recommended path:

1. extend `Quad_Flags`,
2. pack effect controls into `params`,
3. edit `sauce/shader.glsl`,
4. rebuild so `generated_shader.odin` refreshes,
5. submit quads/sprites with the right flags.

Why this fits:

- the render path already forwards flags and params to the shader,
- `ZLayer.vfx` already exists,
- full-screen and world-space quad drawing already exists.

### Practical recommendation for this repo

If you want to add “particles” in the style this blueprint encourages, start with **Option A**.

It matches the rest of the architecture best:

- simple data structure,
- updated from `game_update()`,
- rendered from `game_draw()`,
- no extra abstraction burden.

Move to entity-backed or shader-backed effects only when the particles need gameplay interaction or advanced GPU behavior.

## 5.6 Adding a new screen or mode

The repo hints at this directly in `app_frame()`.

Recommended path:

1. add a high-level mode enum to `Game_State`,
2. split mode-specific update/draw functions,
3. let `app_frame()` switch between them,
4. keep `game_update()` / `game_draw()` as the in-game path only.

This preserves the existing layering:

- `core_app_frame()` remains the engine frame,
- `app_frame()` becomes the game mode router,
- mode functions stay game-specific.

## 5.7 Adding audio features

Typical path:

- author event in FMOD project,
- rebuild banks into `res/fmod/`,
- call `sound_play(...)` for one-shots,
- call `sound_play_continuously(...)` for loops,
- use `emit_sound_from_entity(...)` when the sound should follow an entity.

If you add many looping emitters, wire `update_sound_emitters()` into the frame flow so stale emitters auto-stop.

## 5.8 Adding renderer features

Typical path:

- if it is just content, stay in `game.odin` and draw helpers,
- if it needs batching/shader/atlas behavior, touch `core_render.odin`, `core_draw.odin`, and `shader.glsl`,
- if it needs per-draw metadata, extend `Vertex`, `Quad_Flags`, `Shader_Data`, or `params` usage.

A good mental model is:

- **game layer chooses what to submit**,
- **render layer decides how submissions become GPU work**.

## 6. Architectural conventions worth preserving

## 6.1 `core_` files are reusable-ish, not sacred

The `core_` prefix means “shared plumbing”, not “never touch”.

You can and should modify core files when the game needs it.

## 6.2 The game megafile is intentional

`game.odin` is supposed to be the cozy place where most shipping feature work happens.

This repository does not try to avoid a large gameplay file. It optimizes for iteration speed over textbook separation.

## 6.3 Add data first, abstractions later

Most feature additions should start by adding:

- state to `Game_State` or `Entity`,
- update logic in `game_update()`,
- draw logic in `game_draw()`.

Only extract a subsystem once the pattern is clearly repeating.

## 6.4 Frame-local memory is part of the architecture

If a feature only needs data for one frame or one update pass, prefer temp allocations or scratch fields over permanent ownership structures.

## 6.5 Rendering is submission-based, not immediate-mode GPU execution

Draw calls are cheap gameplay-side submissions into a batch. Heavy render work should continue to respect that batching model.

## 7. A concrete feature-addition checklist

When adding almost any new feature, the usual decision tree is:

1. **Is it global or many-instanced?**
   - global -> `Game_State`
   - many-instanced -> entity or custom array
2. **Does it need raw input?**
   - add/extend `Input_Action`
3. **Does it need art?**
   - add PNG to `res/images/`
   - add enum entry in `Sprite_Name`
4. **Does it animate?**
   - define `frame_count`
   - drive `entity_set_animation()` or your own frame logic
5. **Does it render above/below something?**
   - choose a `ZLayer`
6. **Does it need a special visual rule?**
   - first try sprite/color/params
   - then extend shader flags if needed
7. **Does it make sound?**
   - add an FMOD event and call a sound helper
8. **Does it fit the current mode flow?**
   - add to `game_update()` / `game_draw()` or branch in `app_frame()`

## 8. The shortest summary

This blueprint is built around a small number of strong ideas:

- one Sokol-driven frame callback,
- one central game state,
- one simple entity storage model,
- one batched quad renderer,
- one gameplay megafile where most features are added directly,
- one expectation that you will modify the architecture instead of working around it.

If you follow those rules, new features — including sprites, particles, UI, audio hooks, enemies, and screen states — fit into the project naturally.
