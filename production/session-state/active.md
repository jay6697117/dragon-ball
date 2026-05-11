# Active Session State

## Current Task

Revising `fixed-logic-runtime` GDD for 《星核斗魂》 after the fifth full design review.

## Status

`design/gdd/fixed-logic-runtime.md` was re-reviewed with full specialist coverage after the prior fourth-review revision. Verdict: MAJOR REVISION NEEDED. The fifth-review blockers have now been revised in the GDD, and `design/registry/entities.yaml` has been synced with the revised catch-up defaults, display-watermark presentation ack semantics, `presented_running_tick_index`, and Web performance budgets.

## Current Section

Fifth-review revision complete — systems index and review log updated; user chose fresh-session re-review next.

## Completed Sections

- Overview
- Player Fantasy
- Detailed Rules
- Formulas
- Edge Cases
- Dependencies
- Tuning Knobs
- Acceptance Criteria
- Visual/Audio Requirements
- UI Requirements
- Open Questions

## Completed This Session

- Ran full `/design-review design/gdd/fixed-logic-runtime.md` after the fourth-review revision.
- Specialist review returned MAJOR REVISION NEEDED, with blockers around `presented_running_tick_index`, presentation ack semantics, CPU/dummy fairness, catch-up preflight, event/packet idempotency, Web budgets, QA ACs, and ADR gates.
- User approved these revision decisions:
  - CPU/dummy fairness is MVP-blocking.
  - `presentation_ack` means display watermark, not proof the player truly saw or understood the fact.
  - Web MVP uses small-step catch-up.
  - Recovery pause, focus restore, and ack wait share a safe “ready to continue” countdown pattern.
- Revised `design/gdd/fixed-logic-runtime.md`:
  - Updated header to fifth-review revision pending fresh re-review.
  - Changed `catch_up_max_ticks` from `6` to `2`.
  - Changed `catch_up_wall_clock_budget_ms` from `4.0` to `3.0` and included preflight cost.
  - Changed `runtime_normal_tick_budget_ms` from `2.0` to `1.0`.
  - Added `presented_running_tick_index` lifecycle formula.
  - Changed presentation ack to display-watermark semantics with `display_watermark_target`, ack result, visual degradation, and no claim of player perception.
  - Promoted AC-FLR-65 to MVP-blocking and expanded it to CPU plus training dummy deterministic QA.
  - Corrected catch-up stop precedence so `none` only applies when no attempt occurs and `completed_backlog` wins once backlog is clear.
  - Added stronger AI observation, visibility watermark, RNG stream/call-counter, stale-command, projectile-threat, event instance ID, packet ordering, UI one-shot, and audio loop cleanup contracts.
  - Updated UI/player-facing recovery labels to avoid exposing raw technical enum names.
- Synced `design/registry/entities.yaml`:
  - Added `presented_running_tick_index` formula.
  - Updated `catch_up_max_ticks = 2`.
  - Updated `catch_up_wall_clock_budget_ms = 3.0`.
  - Updated `runtime_normal_tick_budget_ms = 1.0`.
  - Updated deprecated `catch_up_attempt_budget_ms` alias to `3.0`.
  - Updated `catch_up_stop_reason` expression/notes.

## Files

- `design/art/art-bible.md` — approved visual identity and asset standards.
- `design/gdd/game-concept.md` — source concept.
- `design/gdd/systems-index.md` — current systems index; `fixed-logic-runtime` remains pending fresh re-review.
- `design/gdd/fixed-logic-runtime.md` — revised first MVP Foundation GDD after fifth review; pending fresh re-review.
- `design/registry/entities.yaml` — fixed-runtime formulas/constants synced after fifth-review revisions.
- `design/gdd/reviews/fixed-logic-runtime-review-log.md` — fifth full review summary appended; pending fresh re-review after revisions.

## Next

Recommended next step: start a fresh session and run `/design-review design/gdd/fixed-logic-runtime.md` to validate the fifth-review revisions before moving to `input-buffering`.
