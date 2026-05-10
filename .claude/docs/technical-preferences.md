# Technical Preferences

## Engine & Language

- **Engine**: Godot 4.6.2
- **Language**: GDScript
- **Rendering**: Godot 2D Canvas，2.5D presentation only
- **Physics**: Godot 2D physics for broad collisions; custom data-driven hitboxes/hurtboxes for fighting logic

## Input & Platform

- **Target Platforms**: Web / Browser
- **Input Methods**: Keyboard
- **Primary Input**: Keyboard
- **Gamepad Support**: Partial
- **Touch Support**: Partial
- **Platform Notes**: Web MVP is keyboard-single-player first. Test browser focus, fullscreen, audio unlock, and input latency early. No hover-only UI. 本地双人、手柄优先和移动触控都延后。

## Naming Conventions

- **Classes**: PascalCase (e.g., `PlayerController`)
- **Variables**: snake_case (e.g., `move_speed`)
- **Signals/Events**: snake_case past tense (e.g., `health_changed`)
- **Files**: snake_case matching class (e.g., `player_controller.gd`)
- **Scenes/Prefabs**: PascalCase matching root node (e.g., `PlayerController.tscn`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `MAX_HEALTH`)

## Performance Budgets

- **Target Framerate**: 60fps
- **Frame Budget**: 16.67ms
- **Draw Calls**: Keep low for Web; minimize layered VFX, oversized sprites, and unnecessary CanvasItem overlap
- **Memory Ceiling**: Keep Web build lightweight; prefer small sprite sheets, compressed textures, and minimal loaded scenes for MVP

## Testing

- **Framework**: GdUnit4
- **Minimum Coverage**: Critical gameplay formulas and deterministic state transitions must be covered before story completion
- **Required Tests**: Input buffering, combat state transitions, hitbox/hurtbox rules, damage/hitstun formulas, guard/block rules, gas/energy meter rules, simple CPU behavior

## Forbidden Patterns

- [None configured yet — add as architectural decisions are made]

## Allowed Libraries / Addons

- [None configured yet — add as dependencies are approved]

## Architecture Decisions Log

- [No ADRs yet — use /architecture-decision to create one]

## Engine Specialists

- **Primary**: godot-specialist
- **Language/Code Specialist**: godot-gdscript-specialist (all .gd files)
- **Shader Specialist**: godot-shader-specialist (.gdshader files, VisualShader resources)
- **UI Specialist**: godot-specialist (no dedicated UI specialist — primary covers all UI)
- **Additional Specialists**: godot-gdextension-specialist (GDExtension / native C++ bindings only)
- **Routing Notes**: Invoke primary for architecture decisions, ADR validation, and cross-cutting code review. Invoke GDScript specialist for code quality, signal architecture, static typing enforcement, and GDScript idioms. Invoke shader specialist for material design and shader code. Invoke GDExtension specialist only when native extensions are involved.

### File Extension Routing

| File Extension / Type | Specialist to Spawn |
|-----------------------|---------------------|
| Game code (.gd files) | godot-gdscript-specialist |
| Shader / material files (.gdshader, VisualShader) | godot-shader-specialist |
| UI / screen files (Control nodes, CanvasLayer) | godot-specialist |
| Scene / prefab / level files (.tscn, .tres) | godot-specialist |
| Native extension / plugin files (.gdextension, C++) | godot-gdextension-specialist |
| General architecture review | godot-specialist |
