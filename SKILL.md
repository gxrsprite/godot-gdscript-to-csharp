---
name: godot-gdscript-to-csharp
description: Port Godot 3 GDScript to Godot 4 C# slice by slice. Use when translating .gd files to .cs in a Godot 4 mono project, when migrated gameplay code compiles but silently does nothing at runtime, or when a C# port misbehaves only in real gameplay or under load. Covers migration order, dual-API bridging, silent-failure traps, and verification.
---

# Godot GDScript → C# Slice Migration

You are migrating a Godot 3 GDScript project to Godot 4 C# (mono), one slice at a time.

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
