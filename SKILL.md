---
name: godot-gdscript-to-csharp
description: Port Godot 3 GDScript to Godot 4 (GDScript and/or C#). Use when translating .gd files to .cs in a Godot 4 mono project, when migrated gameplay code compiles but silently does nothing at runtime, or when a C# port misbehaves only in real gameplay or under load. Covers migration order, 3-to-4 GDScript deltas, dual-API bridging, silent-failure traps, and verification.
---

# Godot GDScript Migration (3 to 4, GDScript and/or CSharp)

You are migrating a Godot 3 GDScript project to Godot 4, in three supported paths:

1. **Godot 3 GDScript to Godot 4 GDScript** (new engine, same language), see section A.
2. **Godot 4 GDScript to Godot 4 CSharp** (same engine, new language), see main sections below.
3. **Godot 3 GDScript to Godot 4 CSharp** (both at once): do section A first, then the rest.
   Never debug engine and language changes in the same step.

## A. Godot 3 to 4 GDScript deltas (same language)

Syntax annotations first (`export` becomes `@export`, `onready` becomes `@onready`,
`tool` becomes `@tool`, RPC keywords become `@rpc`), then these behavior changes:

- `yield(obj, "sig")` becomes `await obj.sig`; one-frame yield becomes
  `await get_tree().process_frame`.
- `Tween` node plus `interpolate_property` becomes `create_tween()` plus
  `tween_property()` (tweens are one-shot; parallel and sequence via chaining,
  not node config).
- `KinematicBody2D` becomes `CharacterBody2D`, and `move_and_slide()` takes
  **no arguments** (it consumes the `velocity` property);
  `move_and_collide` keeps its signature.
- `File` and `Directory` become `FileAccess` and `DirAccess`; `JSON.parse`
  becomes `JSON.parse_string`; `OS.get_ticks_msec()` becomes
  `Time.get_ticks_msec()`.
- `VisualServer` becomes `RenderingServer`; shader built-ins are restricted:
  sampling must happen in `fragment()` (custom functions cannot see `TEXTURE`).
- `CPUParticles2D`, `GPUParticles2D` and `AudioServer` buses are largely unchanged;
  `AudioBusLayout` serialization must be complete (`format=3`, all `send` entries written).
- `Camera2D`, `CanvasModulate`, groups, `$Node` and `%UniqueName` paths, and `tr()`
  translations: same concepts, verify per use. `TileMap` becomes `TileMapLayer` on 4.3+.
- Scene files use `format=3`, resources gain `uid` attributes; node names must stay
  untouched (never "normalize" them: animation tracks, connections and `GetNode`
  paths all chain off names).
- Old high-level multiplayer networking was rewritten: out of scope here.
  Isolate and rewrite separately, never inside a language migration.
- After this step the project must run **fully in GDScript on Godot 4** before any
  `.gd` to `.cs` translation begins. That isolates engine bugs from language bugs.

The C# translation below assumes path 2 or a completed section A.

## Order of work

1. **One behavior owner**: only one language may own the same gameplay mutation at a time.
2. **Bottom-up by dependency**: pure rules → reusable components → vertical slices
   (projectiles/characters) → managers/state machines → autoloads last (largest fan-out).
3. **Don't rewrite scenes for the language change**: keep node paths, export names,
   collision layers, animation tracks, and signal connections; any scene edit needs
   a verification record.
4. **Four-piece verification per slice**: `dotnet build` → Godot load/import →
   directed assertions (smoke) → compare against the GD baseline. Missing one means
   the slice is not done. Run the checks **before and after deleting the `.gd`**.

## Bridging conventions (while GD callers still exist)

- Expose **both** PascalCase (C#) and snake_case (GD Callable bridge) entries;
  delete only the snake_case side later.
- `[Export]` defaults must match the `.tscn`; every class with lowercase tscn
  assignments needs either handwritten lowercase bridges or a `_Get`/`_Set`
  reflection fallback — one of the two, never neither.
- `Godot.Call` takes **no default arguments** — arity must be exact.
- Signals via `SignalName`, `await` via `ToSignal` plus alive re-checks.
- `Callable.From(handler)` passed to `Connect()` must be kept alive in a static
  collection, or GC will silently kill the callback; prefer C# events.

## Silent-failure traps (build green, runs wrong)

1. **C# events (`+=`) on long-lived singletons never auto-disconnect.** Godot only
   disconnects native connections. After the subscribing scene is freed, every
   state change still fires into the dead node. Guard every such handler with
   `IsInstanceValid(this)`, use `GetNodeOrNull`, and unsubscribe on tree exit.
2. **Fresh full-screen panels must use `SetAnchorsAndOffsetsPreset`, not
   `SetAnchorsPreset`.** The latter sets anchors but back-computes offsets to
   "preserve" the current (zero) size — offsets become negative parent size and
   the panel ends up (0,0) with collapsed scroll areas, while all content exists.
3. **Bounds-check every index/key that comes from game state.** Burn DOT owners,
   killer indexes and pet owners can be `-1`; missing effect keys must return
   defaults via `TryGetValue`. One blind index aborts the whole frame — and
   `Debug.Assert` in debug builds aborts frames by itself, so keep asserts
   off hot paths.
4. **Resource constructors run before autoloads are ready.** Null-guard singleton
   access there and defer the rest — but the deferred backup is itself unreliable
   under resource-load timing, so **use sites must also tolerate unconverted state**.
   Null-guard every string field fed to hash/lookup functions (unset exports are null).
5. **Put hooks in the override that actually runs.** If the character class overrides
   the whole damage pipeline, a hook in the base class never fires. Confirm the live
   path from the call stack, never from the inheritance diagram.
6. **Closing a UI page must go back through the same state machine that opened it.**
   A `Hide()`-only close leaves the menu state on the hidden page — visually stuck.
7. **When reusing an engine path, close the loop at the new call site.**
   If the original deducts currency in the button handler while `buy_item` only
   fills the inventory, the new caller must do check → deduct → grant itself;
   if the original only merges into full containers, the new caller must pre-check
   capacity, or users pay for nothing.
8. **Don't mix quantities with different units in one display.** Currency earned,
   materials saved, and damage dealt are three different things — merging them
   into one ranking is unreadable. Keep one unit per view, merge duplicates,
   label the unit in the header.
9. **`GD.Randi()`'s return type has changed across GodotSharp versions** — check your
   version's signature and cast explicitly before `%` or int arithmetic. Enums need
   `(int)` casts. `Godot.Collections.Array` has no `Sort_custom` (use LINQ or loops).
   Array literals are read-only — `.duplicate()` before mutating.

## Porting layered content packs (data + scripts)

A content pack (new entities/items/weapons plus scripts that extend core
classes) ports in layers: **data resources first, effect scripts second,
extension scripts third, registration last.**

- Data rewrite rules (mechanical, scriptable): source prefix becomes the
  in-project content dir; core script refs keep their path but become
  PascalCase files (`effect.gd` → `Effect.cs`); pack-owned scripts point at
  their future homes (build those next); texture types updated to the engine's
  current names; normalize directory case and keep filenames otherwise;
  copy audio too; never hand-write import metadata — the engine generates it
  on boot.
- After conversion, scan every resource/scene for script refs and check each
  target exists on disk (one dangling script ref kills the referencing scene
  at load, cascading into thousands of log errors from a single bad path).
- Extension scripts (add-ons to core classes): if the target C# class is
  `partial`, merge directly; on member-name collision rename the incoming side
  and add a one-line hook call in the base method (pre/post/passthrough
  decided explicitly in the hook). Keep a call-point list at the top of each
  merged file so the wiring is auditable.
- API facts worth re-verifying on each engine version: packed array types may
  not exist in C# bindings — use `List<T>`/`T[]` (`.ToArray()` at engine
  boundaries); particle/sprite member names change case between versions
  (`Hframes`, fill-mode enum names); collection types may lack
  `Erase/PopBack/Front/Back/Sort_custom` (use `Remove/RemoveAt`/index);
  `TopLevel` property vs setter; shape `Size` vs `Extents`;
  `GetPhysicsFrames()` returns `ulong`; random-range returns double
  (cast to float); `Play(name)` takes no bool flag.
- Preserve file line endings in bulk-fix scripts (CRLF vs LF per file), or the
  diff drowns the fix.

## Performance: measure in-game, never headless

- Headless uses a dummy renderer: **zero sprites/particles/text drawn**. A scene that
  costs 0.05ms script + 0.46ms physics headless can still run 117ms+ frames live —
  98% of frame time in rendering.
- Add an in-game overlay (FPS, frame ms, entity/object/node counts, script vs
  physics ms) and read it under real load.
- Proven wins, in order: pool spawned labels (cap concurrency, configurable);
  drive animations from `_Process` instead of allocating Tweens per item;
  batch custom drawing into a single node `_Draw` instead of one node per entity;
  cap concurrent particle effects (hit effects are large translucent quads —
  classic fill-rate killers); cap on-screen spawn counts when spawners can
  grow without bound.

## Probe discipline

- Probes must be **autoloads**, never the main scene (`ChangeSceneToFile` frees them).
- New `.cs` needs `--import` before running.
- `QueueFree` takes effect end-of-frame — wait before asserting cleanup.
- Run probes against an **isolated user dir** (temp `custom_user_dir_name` + copied
  saves), never the live profile. Delete probes afterward; `git status` must be clean.
- Save files: "temp file + rename" fails silently on freshly created user dirs —
  fall back to direct write. Cheat/numeric overrides must remember the original
  value and recompute from it, or lowering them never restores.
