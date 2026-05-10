# Godot Engine — Version Reference

| Field | Value |
|-------|-------|
| **Engine Version** | Godot 4.6.2 stable |
| **Release Date** | April 1, 2026 |
| **Project Pinned** | 2026-05-10 |
| **Last Docs Verified** | 2026-05-10 |
| **LLM Knowledge Cutoff** | May 2025 |
| **Risk Level** | HIGH — Godot 4.4, 4.5, and 4.6 are beyond the model's reliable training coverage |

## Knowledge Gap Warning

The model's training data likely covers Godot up to ~4.3. Versions 4.4, 4.5,
and 4.6 introduced significant changes that the model does NOT reliably know.
Always cross-reference this directory before suggesting Godot API calls.

Godot 4.6.2 is a maintenance release in the 4.6 line. The official release notes
state there are no known incompatibilities with Godot 4.6.1, but projects should
still use version control and test exported builds after upgrading.

## Post-Cutoff Version Timeline

| Version | Release | Risk Level | Key Theme |
|---------|---------|------------|-----------|
| 4.4 | ~Mid 2025 | MEDIUM | Jolt physics option, FileAccess return types, shader texture type changes |
| 4.5 | ~Late 2025 | HIGH | Accessibility (AccessKit), variadic args, @abstract, shader baker, SMAA |
| 4.6 | Jan 2026 | HIGH | Jolt default, glow rework, D3D12 default on Windows, IK restored |
| 4.6.2 | Apr 1 2026 | HIGH | Stable maintenance release; 122 fixes; no known incompatibilities with 4.6.1 |

## Project-Specific Notes

- Project target: Web / Browser.
- Primary language: GDScript.
- Rendering approach: Godot 2D Canvas with 2.5D presentation.
- Gameplay-critical systems should use fixed logic timing, explicit state machines,
  and data-driven hitbox/hurtbox definitions rather than relying on visual sprite
  bounds or uncontrolled physics behavior.

## Verified Sources

- Official docs: https://docs.godotengine.org/en/stable/
- 4.3→4.4 migration: https://docs.godotengine.org/en/4.4/tutorials/migrating/upgrading_to_godot_4.4.html
- 4.4→4.5 migration: https://docs.godotengine.org/en/4.5/tutorials/migrating/upgrading_to_godot_4.5.html
- 4.5→4.6 migration: https://docs.godotengine.org/en/4.6/tutorials/migrating/upgrading_to_godot_4.6.html
- 4.6.2 maintenance release: https://godotengine.org/article/maintenance-release-godot-4-6-2/
- 4.6.2 changelog: https://github.com/godotengine/godot/blob/4.6.2-stable/CHANGELOG.md
- Download archive: https://godotengine.org/download/archive/
