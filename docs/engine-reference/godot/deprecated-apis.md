# Godot — Deprecated APIs

Last verified: 2026-05-10

If an agent suggests any API in the "Deprecated" column, it MUST be replaced
with the "Use Instead" column.

## Nodes & Classes

| Deprecated | Use Instead | Since | Notes |
|------------|-------------|-------|-------|
| `TileMap` | `TileMapLayer` | 4.3 | One node per layer instead of multi-layer node. |
| `VisibilityNotifier2D` | `VisibleOnScreenNotifier2D` | 4.0 | Renamed for clarity. |
| `VisibilityNotifier3D` | `VisibleOnScreenNotifier3D` | 4.0 | Renamed for clarity. |
| `YSort` | `Node2D.y_sort_enabled` | 4.0 | Property on Node2D, not a separate node. |
| `Navigation2D` / `Navigation3D` | `NavigationServer2D` / `NavigationServer3D` | 4.0 | Server-based API. |
| `EditorSceneFormatImporterFBX` | `EditorSceneFormatImporterFBX2GLTF` | 4.3 | Renamed. |

## Methods & Properties

| Deprecated / Changed | Use Instead | Since | Notes |
|------------|-------------|-------|-------|
| `yield()` | `await signal` | 4.0 | GDScript 2.0 coroutine syntax. |
| `connect("signal", obj, "method")` | `signal.connect(callable)` | 4.0 | Callable-based connections. |
| `instance()` | `instantiate()` | 4.0 | Renamed. |
| `PackedScene.instance()` | `PackedScene.instantiate()` | 4.0 | Renamed. |
| `get_world()` | `get_world_3d()` | 4.0 | Explicit 2D/3D split. |
| `OS.get_ticks_msec()` | `Time.get_ticks_msec()` | 4.0 | Time singleton preferred. |
| `duplicate()` for nested resources | `duplicate_deep()` | 4.5 | Use explicit deep copy control when nested subresources must be unique. |
| `Skeleton3D` signal `bone_pose_updated` | `skeleton_updated` | 4.3 | Renamed. |
| `AnimationPlayer.method_call_mode` | `AnimationMixer.callback_mode_method` | 4.3 | Moved to base class. |
| `AnimationPlayer.playback_active` | `AnimationMixer.active` | 4.3 | Moved to base class. |
| `JSONRPC.set_scope` | `JSONRPC.set_method` | 4.5 | Official 4.5 migration notes replacement. |
| `Node.get_rpc_config` | `Node.get_node_rpc_config` | 4.5 | Renamed in 4.5 migration notes. |
| `FileAccess.get_as_text(skip_cr)` | `FileAccess.get_as_text()` then normalize line endings if needed | 4.6 | Optional `skip_cr` parameter removed. |
| `EditorFileDialog.add_side_menu()` | Custom editor UI / FileDialog-based controls | 4.6 | Method removed. Editor plugin API only. |

## Patterns (Not Just APIs)

| Deprecated Pattern | Use Instead | Why |
|--------------------|-------------|-----|
| String-based `connect()` | Typed signal connections | Type-safe, refactor-friendly. |
| `$NodePath` in `_process()` | `@onready var` cached reference | Avoid repeated path lookup every frame. |
| Untyped `Array` / `Dictionary` | `Array[Type]`, typed variables | GDScript compiler optimizations and clearer contracts. |
| `Texture2D` in shader parameters | `Texture` base type | Changed in 4.4. |
| Manual post-process viewport chains | `Compositor` + `CompositorEffect` | Structured post-processing (4.3+). |
| GodotPhysics3D for new 3D projects | Jolt Physics 3D | Default since 4.6; better stability for 3D. Not relevant to this project's 2D fight logic unless 3D physics is added later. |
| Visual sprite edge as hitbox truth | Data-driven hitbox/hurtbox resources | Generated sprites may vary frame-to-frame; fighting logic needs stable authored boxes. |
| Variable-frame combat timing | Fixed logic step + explicit frame/state data | Fighting game input windows and hitstun must be reproducible. |
