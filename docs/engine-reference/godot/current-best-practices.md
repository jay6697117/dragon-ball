# Godot — Current Best Practices

Last verified: 2026-05-10 | Engine: Godot 4.6.2 stable

Practices that are **new or changed** since the model's training data (~4.3).
This supplements (not replaces) the agent's built-in knowledge.

## Godot 4.6.2 Maintenance Release

- Godot 4.6.2 is an official stable maintenance release from April 1, 2026.
- Official notes state there are no known incompatibilities with Godot 4.6.1.
- Treat it as the pinned project version, but still use Git/version control and
  verify exported Web builds after engine or export-template updates.

## Project-Specific Web Fighting Game Practices

- Build the combat loop around fixed logic timing and explicit state transitions.
- Do not rely on raw sprite edges for combat. Use authored hitbox/hurtbox data.
- Test Web export in the browser during the first implementation week, not at the end.
- Keep sprite sheets small and animation frame counts limited for MVP.
- Keep energy VFX readable and cheap: no heavy particle stacks for MVP.
- Avoid complex input commands for the first Web prototype; use direct keyboard actions.
- Track input buffering and state priority as first-class gameplay systems.

## GDScript (4.5+)

- **Variadic arguments**: Functions can accept arbitrary parameter counts.
  ```gdscript
  func log_values(prefix: String, values: Variant...) -> void:
      for v in values:
          print(prefix, ": ", v)
  ```

- **Abstract classes and methods**: Use `@abstract` to enforce inheritance.
  ```gdscript
  @abstract
  class_name BaseCombatState extends Resource

  @abstract
  func can_enter(context: Dictionary) -> bool:
      return false
  ```

- **Script backtracing**: Detailed call stacks are available even in Release builds.

## Physics (4.6)

- **Jolt Physics is the default 3D engine** for new projects.
  - Some HingeJoint3D properties only work with GodotPhysics.
  - Switch: Project Settings → Physics → 3D → Physics Engine.
  - 2D physics is unchanged.
- For this project, use 2D physics only for broad collisions if useful. Fighting
  hit detection should use deterministic data-driven hitbox/hurtbox checks.

## Rendering (4.6)

- **D3D12 is the default backend on Windows** (was Vulkan) for better driver compatibility.
- **Glow now processes before tonemapping** with changed defaults and screen blending mode.
  Existing glow setups may appear brighter; tune Environment glow carefully.
- **SSR overhauled** with improved realism, stability, and performance.
- **AgX tonemapper** has white point and contrast controls.

## Rendering (4.5)

- **Shader Baker**: Pre-compile shaders to reduce startup hitching.
- **SMAA 1x**: New AA option — sharper than FXAA, cheaper than TAA.
- **Stencil buffer**: Available for advanced masking/portal effects.
- **Bent normal maps**: Directional occlusion in normal map textures.
- **Specular occlusion**: Ambient occlusion now affects reflections.

## Accessibility (4.5+)

- **Screen reader support**: Control nodes integrate with accessibility tools via AccessKit.
- **Live translation preview**: Test GUI layouts in different languages directly in-editor.
- **FoldableContainer**: New accordion-style UI node for collapsible sections.
- **Recursive Control disable**: Disable mouse/focus interactions for entire node hierarchies with a single property.

## Animation (4.5+)

- **BoneConstraint3D**: Bind bones to other bones with modifiers.
  - AimModifier3D, CopyTransformModifier3D, ConvertTransformModifier3D.

## Animation (4.6)

- **IK system fully restored** for 3D.
  - Available modifiers: CCDIK, FABRIK, Jacobian IK, Spline IK, TwoBoneIK.
  - Applied via `SkeletonModifier3D` nodes.
- `AnimationPlayer` now uses `StringName` for several animation fields and signals.
  GDScript is mostly compatible, but be precise with typed code.

## Resources (4.5+)

- **`duplicate_deep()`**: Explicit deep duplication for nested resource trees.
  - Use `duplicate_deep()` when you need per-instance copies of nested resources.
  - Do not assume `duplicate(true)` semantics from older examples are still ideal.

## Navigation (4.5+)

- **Dedicated 2D navigation server**: No longer proxied through 3D NavigationServer.
  - Reduces export binary size for 2D-only games.

## UI (4.6)

- **Dual-focus system**: Mouse/touch focus is now separate from keyboard/gamepad focus.
  - Visual feedback differs depending on input method.
  - Consider this when designing custom focus behavior.
- For this project, Web MVP is keyboard-first. Avoid hover-only UI and test browser focus loss.

## Editor Workflow (4.6)

- Flexible dock drag-and-drop with blue outline preview, including bottom panel.
- Most panels support floating windows except Debugger.
- New keyboard shortcuts: Alt+O (Output), Alt+S (Shader).
- Export variable auto-generation: drag resource from FileSystem into script editor.
- Live preview in Quick Open dialog when "Live Preview" is enabled.
- New "Select Mode" (`v`) prevents accidental transforms; old mode renamed "Transform Mode" (`q`).

## Tooling

- **ripgrep has no `gdscript` type**: `*.gd` is registered under `gap` (GAP programming language).
  `rg --type gdscript` is a hard error — the search never executes.
  Always use `rg --glob "*.gd"` to filter GDScript files.

## Platform (4.5+)

- **visionOS export**: First new platform since open-sourcing (windowed app mode).
- **SDL3 gamepad driver**: Better cross-platform gamepad support.
- **Android**: Edge-to-edge display, camera feed access, 16KB page support (Android 15+).
- **Linux**: Wayland subwindow support for multi-window capability.
