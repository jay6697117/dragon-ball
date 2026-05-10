# Godot — Breaking Changes

Last verified: 2026-05-10

Changes between Godot versions, focused on post-LLM-cutoff changes (4.4+).

## 4.6.1 → 4.6.2 (Apr 2026 — POST-CUTOFF, HIGH RISK)

| Subsystem | Change | Details |
|-----------|--------|---------|
| Release | Stable maintenance release | Official notes: 122 fixes from 61 contributors. |
| Compatibility | No known incompatibilities with 4.6.1 | Maintenance release should be safe, but use version control and verify exports. |
| Project Impact | Low within 4.6 line | Still run Web export tests because this project targets browser play and input latency matters. |

## 4.5 → 4.6 (Jan 2026 — POST-CUTOFF, HIGH RISK)

| Subsystem | Change | Details |
|-----------|--------|---------|
| Physics | Jolt is now the DEFAULT 3D physics engine | New projects use Jolt automatically. Existing projects keep their setting. Some HingeJoint3D properties only work with GodotPhysics. 2D physics remains separate. |
| Rendering | Glow processes BEFORE tonemapping | Default glow blend mode changed and glow may appear brighter. Adjust Environment glow settings after upgrading. |
| Rendering | D3D12 default on Windows | Was Vulkan. For better driver compatibility. |
| Rendering | AgX tonemapper new controls | White point and contrast parameters added. |
| Rendering | Volumetric fog blending changed | More physically accurate and often brighter; reduce density/brightness if needed. |
| Core | Quaternion initializes to identity | Was zero. Unlikely to affect most code but technically breaking. |
| Core | TSCN scene format changed | `load_steps` no longer saved; unique node IDs saved. Expect large diffs after resaving older scenes. |
| Core | `FileAccess.create_temp()` parameter type changed | Parameter changed from `int` to `FileAccess.ModeFlags`; GDScript compatible in most cases. |
| Core | `FileAccess.get_as_text()` removed `skip_cr` parameter | Update calls that used the optional parameter. |
| UI | Dual-focus system | Mouse/touch focus now separate from keyboard/gamepad focus. Visual feedback differs by input method. |
| UI | Focus methods gained optional focus-hiding parameters | `Control.grab_focus(hide_focus)`, `Control.has_focus(ignore_hidden_focus)`, `LineEdit.edit(hide_focus)`. |
| Animation | IK system fully restored | CCDIK, FABRIK, Jacobian IK, Spline IK, TwoBoneIK via SkeletonModifier3D nodes. |
| Animation | `AnimationPlayer` String fields now use `StringName` | GDScript mostly compatible; C# binary/source compatibility can be affected. |
| Networking | TCP methods moved to base socket classes | `StreamPeerTCP` methods moved to `StreamPeerSocket`; `TCPServer` methods moved to `SocketServer`. GDScript compatible. |
| Navigation | AStar path behavior changed | AStar2D/AStar3D/AStarGrid2D path methods return empty path when start point is disabled/solid. |
| GUI / Editor | `EditorFileDialog.add_side_menu()` removed | Incompatible for GDScript/C#. Avoid using this editor plugin API. |
| Editor | New "Modern" theme default | Grayscale replaces blue-tint. Restore: Editor Settings → Interface → Theme → Style: Classic. |
| Editor | "Select Mode" keybind changed | New "Select Mode" (v key) prevents accidental transforms. Old mode renamed "Transform Mode" (q key). |
| 2D | TileMapLayer scene tile rotation | Scene tiles can now be rotated like atlas tiles. |
| Localization | CSV plural form support | No longer requires Gettext for plurals. Context columns added. |
| C# | Automatic string extraction | Translation strings auto-extracted from C# code. |
| Plugins | New EditorDock class | Specialized container for plugin docks with layout control. |
| Android | Export template layout changed | Java files moved under `src/main/java/`; manifest/assets moved into `src/main/`. |

## 4.4 → 4.5 (Late 2025 — POST-CUTOFF, HIGH RISK)

| Subsystem | Change | Details |
|-----------|--------|---------|
| GDScript | Variadic arguments added | Functions can accept `...` arbitrary params. |
| GDScript | `@abstract` decorator | Abstract classes and methods now enforceable. |
| GDScript | Script backtracing | Detailed call stacks available even in Release builds. |
| Rendering | Stencil buffer support | New capability for advanced visual effects. |
| Rendering | SMAA 1x antialiasing | New post-processing AA option. |
| Rendering | Shader Baker | Pre-compiles shaders — reportedly much faster startup in some demos. |
| Rendering | Bent normal maps, specular occlusion | New material features. |
| Accessibility | Screen reader support | Control nodes work with accessibility tools via AccessKit. |
| Editor | Live translation preview | Test GUI layouts in different languages in-editor. |
| Physics | 3D interpolation rearchitected | Moved from RenderingServer to SceneTree. API unchanged but internals differ. |
| Animation | BoneConstraint3D | New: AimModifier3D, CopyTransformModifier3D, ConvertTransformModifier3D. |
| Resources | `duplicate_deep()` added | New explicit method for deep duplication of nested resources. |
| Navigation | Dedicated 2D navigation server | No longer a proxy to 3D navigation; smaller export for 2D games. |
| UI | FoldableContainer node | New accordion-style container for collapsible UI sections. |
| UI | Recursive Control behavior | Disable mouse/focus interactions across entire node hierarchies. |
| Platform | visionOS export support | New platform target. |
| Platform | SDL3 gamepad driver | Delegated gamepad handling to SDL library. |
| Platform | Android 16KB page support | Required for Google Play targeting Android 15+. |
| Core/API | `JSONRPC.set_scope` replaced | Use `set_method`. |
| Networking | `Node.get_rpc_config` renamed | Use `get_node_rpc_config`. |
| Editor plugin API | `_get_option_icon` return type changed | Return type changed from `ImageTexture` to `Texture2D`. |
| Resources | `Resource.duplicate(true)` behavior changed | Use explicit deep duplication APIs where needed. |
| RichTextLabel | Image sizing parameters changed | Review code that inserts or sizes images in rich text. |

## 4.3 → 4.4 (Mid 2025 — NEAR CUTOFF, VERIFY)

| Subsystem | Change | Details |
|-----------|--------|---------|
| Core | `FileAccess.store_*` return `bool` | Was `void`. Methods: `store_8`, `store_16`, `store_32`, `store_64`, `store_buffer`, `store_csv_line`, `store_double`, `store_float`, `store_half`, `store_line`, `store_pascal_string`, `store_real`, `store_string`, `store_var`. |
| Core | `OS.execute_with_pipe` | Added optional `blocking` parameter. |
| Core | `RegEx.compile/create_from_string` | Added optional `show_error` parameter. |
| Rendering | `RenderingDevice.draw_list_begin` | Many parameters removed; `breadcrumb` parameter added. |
| Rendering | Shader texture types | Parameter/return types changed from `Texture2D` to `Texture`. |
| Particles | `.restart()` method | Added optional `keep_seed` parameter for CPU/GPU 2D/3D particles. |
| GUI | `RichTextLabel.push_meta` | Added optional `tooltip` parameter. |
| GUI | `GraphEdit.connect_node` | Added optional `keep_alive` parameter. |
| GUI | `GraphEdit.frame_rect_changed` parameter type changed | Check custom GraphEdit tools/plugins. |
| Editor | Editor plugin API changes | Review editor plugins before migrating. |
| Curve | Range enforcement changed | Curves now enforce expected ranges more strictly. |
| CSG | Non-manifold mesh behavior changed | Review CSG meshes if used. |
| Android | Sensor events disabled by default | Enable only if needed. |

## 4.2 → 4.3 (In Training Data — LOW RISK)

| Subsystem | Change | Details |
|-----------|--------|---------|
| Animation | `Skeleton3D.add_bone` returns `int32` | Was `void`. |
| Animation | `bone_pose_updated` signal | Replaced by `skeleton_updated`. |
| TileMap | `TileMapLayer` replaces `TileMap` | One node per layer instead of multi-layer single node. |
| Navigation | `NavigationRegion2D` | Removed `avoidance_layers`, `constrain_avoidance` properties. |
| Editor | `EditorSceneFormatImporterFBX` | Renamed to `EditorSceneFormatImporterFBX2GLTF`. |
| Animation | AnimationMixer base class | AnimationPlayer and AnimationTree now extend AnimationMixer. |
