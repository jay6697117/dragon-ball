# Active Session State

## Current Task

Paused during fresh full `/design-review design/gdd/input-buffering.md` re-review for `input-buffering` GDD in 《星核斗魂》.

## Status

User explicitly chose to skip `/design-review design/gdd/fixed-logic-runtime.md` earlier and proceed directly to `input-buffering`. `fixed-logic-runtime` remains revised after ninth fresh review and pending fresh re-review, so `input-buffering` treats it as the current working dependency but not as final approved implementation authority.

`design/gdd/input-buffering.md` received multiple full `/design-review design/gdd/input-buffering.md` passes on 2026-05-13, each returning MAJOR REVISION NEEDED before revision. The latest full re-review again returned MAJOR REVISION NEEDED because runtime/input authority, catch-up semantics, UI routing, keyboard-only Web shell, accessibility scope, Burst/guard trust rules, performance measurement, trace budgets and AC ownership were still unresolved.

The user selected “Revise now,” made these design decisions, and approved writing them to the GDD:

- Public/player-facing MVP requires free keyboard remapping plus player-facing key-test flow; fixed profiles alone are valid only for internal prototype evidence.
- Same-snapshot Burst plus attack resolves as Burst wins, no attack fallback; Burst also suppresses ordinary guard for the vulnerability window.
- Web focus/audio/fullscreen/browser shortcut handling requires a Web shell/JS bridge or equivalent ADR gate.
- Input performance budgets use non-authoritative microsecond/high-resolution profiling excluded from deterministic hashes.

Current status: `input-buffering` revised after latest full re-review — pending fresh re-review.

## Current Section

`input-buffering` draft is fully revised after the latest full re-review. A fresh `/design-review design/gdd/input-buffering.md` re-review was started after `/clear`, but paused before specialist results returned. No new verdict was produced.

## Latest Fresh Re-review Attempt — Paused

- Started full `/design-review design/gdd/input-buffering.md` after `/clear`.
- Loaded `production/session-state/active.md`, `design/gdd/input-buffering.md`, `design/gdd/systems-index.md`, `design/gdd/reviews/input-buffering-review-log.md`, `design/gdd/game-concept.md`, and project `CLAUDE.md`.
- Direct full read of `design/gdd/fixed-logic-runtime.md` exceeded token limit, so dependency validation used targeted grep/read excerpts instead.
- Verified declared dependency files exist: `design/gdd/fixed-logic-runtime.md` and `design/registry/entities.yaml`.
- Verified current `design/gdd/` contains only two per-system GDDs: `fixed-logic-runtime.md` and `input-buffering.md` plus index/concept files.
- Confirmed fixed runtime bidirectionally references input-buffering and exposes relevant contracts: `combat_input_policy`, `direction_pre_read_only`, `catch_up_max_ticks`, `recovery_pause`, `presentation_ack_wait`, hitstop handoff, trace budgets and runtime state payloads.
- Began mandatory full-mode specialist batch for game-designer, systems-designer, qa-lead, godot-specialist, performance-analyst, gameplay-programmer, ux-designer, ui-programmer and accessibility-specialist.
- User interrupted during tool use before any specialist findings returned. Creative-director synthesis was not started. No Phase 4 verdict exists for this attempt.

## Completed Sections

- Overview
- Player Fantasy
- Detailed Rules
- Formulas
- Edge Cases
- Dependencies
- Tuning Knobs
- Acceptance Criteria

## Completed This Session

- Restored active state after compaction.
- Completed full re-review reporting for `design/gdd/input-buffering.md`.
- Specialists consulted: game-designer, systems-designer, qa-lead, godot-specialist, performance-analyst, gameplay-programmer, ux-designer, ui-programmer, accessibility-specialist, creative-director.
- Latest full re-review verdict: MAJOR REVISION NEEDED.
- Senior synthesis scope signal: XL.
- Revised `design/gdd/input-buffering.md` after latest full re-review:
  - Aligned `combat_input_policy` terminology with fixed runtime by using `direction_pre_read_only` and making guard pre-read an input-owned exception flag.
  - Clarified capture/seal/commit snapshot fields and avoided claiming future post-commit runtime state at capture time.
  - Added candidate combat command handoff schema aligned to fixed-runtime command concepts.
  - Added `input_generation_id` and `interruption_epoch` to `InputSample`.
  - Hardened UI context ordering so modals outrank non-modals, UI overrides cannot loosen runtime policy, and control-state records are replayable/hashable.
  - Added deterministic request ordering, UI held-repeat records, text-entry/remap contexts and reason catalog ownership.
  - Required Web shell/JS bridge or equivalent ADR for keyboard-only first focus, audio/fullscreen fallback, prevent-default, reserved shortcuts and key reconciliation.
  - Made public/player-facing MVP require free keyboard remapping plus player-facing key-test flow; fixed profiles remain acceptable only for internal prototype evidence.
  - Changed same-snapshot Burst+attack from reject-all to Burst-wins/no-fallback and tied Burst to ordinary guard suppression.
  - Made hitstop realtime cap a hard lifetime cap checked at each evaluation so it does not stack with post-hitstop running buffer.
  - Replaced the impossible 4-delayed-tick catch-up AC with `catch_up_max_ticks` default 2 plus recovery-pause evidence for larger backlog.
  - Switched input performance p95/p99 measurement to non-authoritative microsecond profiling and defined nearest-rank percentile.
  - Aligned input trace window/memory with fixed-runtime trace budgets and added byte caps for all authoritative input records.
  - Rewrote weak ACs with fixture/stub ownership, concrete matrices, browser-reserved shortcut limits, reason catalog validation and pass/fail gate evidence.
- Updated `design/gdd/systems-index.md` to mark input-buffering revised after latest full re-review and pending fresh re-review.
- Appended the latest full re-review/revision summary to `design/gdd/reviews/input-buffering-review-log.md`.

## Files

- `design/art/art-bible.md` — approved visual identity and asset standards.
- `design/gdd/game-concept.md` — source concept.
- `design/gdd/systems-index.md` — marks `fixed-logic-runtime` as revised after ninth fresh review pending fresh re-review; marks `input-buffering` as revised after latest full re-review pending fresh re-review.
- `design/gdd/fixed-logic-runtime.md` — current dependency; revised after ninth fresh re-review; pending fresh re-review.
- `design/gdd/input-buffering.md` — revised after latest full re-review; pending fresh re-review.
- `design/registry/entities.yaml` — fixed-runtime constants/formula notes synced after ninth-review revisions; input-buffering registry sync deferred until approval.
- `design/gdd/reviews/fixed-logic-runtime-review-log.md` — ninth fresh review summary appended; earlier eighth summary remains unlogged.
- `design/gdd/reviews/input-buffering-review-log.md` — latest full re-review summary appended after revision.
- `production/session-state/active.md` — this session state.

## Next

Recommended next step: start a fresh session and run `/design-review design/gdd/input-buffering.md` again from the beginning. Because this attempt was interrupted before specialist findings returned, do not reuse it as review evidence and do not append a review-log entry for it. Fixed runtime also remains pending fresh re-review, but the user explicitly deferred it before starting `input-buffering`.
