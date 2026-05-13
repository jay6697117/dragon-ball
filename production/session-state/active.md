# Active Session State

## Current Task

Completing the fresh design-review revision workflow for `input-buffering` GDD in 《星核斗魂》.

## Status

User explicitly chose to skip `/design-review design/gdd/fixed-logic-runtime.md` earlier and proceed directly to `input-buffering`. `fixed-logic-runtime` remains revised after ninth fresh review and pending fresh re-review, so `input-buffering` treats it as the current working dependency but not as final approved implementation authority.

`design/gdd/input-buffering.md` received its first full `/design-review design/gdd/input-buffering.md` on 2026-05-13 with verdict MAJOR REVISION NEEDED. The user selected “Revise now,” and the GDD was revised. A fresh full re-review was then run on 2026-05-13 and again returned MAJOR REVISION NEEDED. The user selected “Revise now” again, and the GDD has been revised to address the fresh re-review blockers. Current status: Revised after fresh re-review — Pending Fresh Re-review.

`design/gdd/systems-index.md` marks system #2 as Revised after fresh re-review — Pending Fresh Re-review. `design/gdd/reviews/input-buffering-review-log.md` records both the first full review and the fresh re-review summary.

## Current Section

`input-buffering` draft is fully revised after fresh re-review. No section is currently being authored. Next formal step is a fresh `/design-review design/gdd/input-buffering.md` when ready.

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

- Continued from the first full review revision state for `input-buffering`.
- Ran fresh full `/design-review design/gdd/input-buffering.md`.
- Specialists consulted: game-designer, systems-designer, qa-lead, godot-specialist, performance-analyst, gameplay-programmer, ux-designer, ui-programmer, accessibility-specialist, creative-director.
- Fresh re-review verdict: MAJOR REVISION NEEDED.
- Senior synthesis scope signal: XL.
- Main fresh re-review blockers:
  - Input authority and sealing were not yet one implementable Godot/Web contract.
  - One-shot buffer lifecycle and cross-tick arbitration were underspecified.
  - Continuous direction/guard behavior conflicted with player expectation, accessibility, and fixed-runtime command boundaries.
  - UI, pause, focus, menu navigation, quick restart/exit, and presentation ack routing were not fully defined.
  - Accessibility support for keyboard MVP was below bar.
  - Performance budgets and trace caps were not enforceable enough.
  - Acceptance criteria lacked fixture ownership and reproducible evidence boundaries.
- Revised `design/gdd/input-buffering.md` after fresh re-review:
  - Updated status and review history metadata.
  - Defined central input router as the sole authority, including Godot/Web capture behavior, Control consumption constraints, SceneTree pause/process expectations, Web prevent-default policy, and polling-only-for-reconciliation limits.
  - Added Standard, Alternate, and Accessibility keyboard profiles with hash-visible deterministic validation.
  - Added menu navigation actions and context-separated key reuse.
  - Added canonical serialization/hashing rules and stable schema requirements.
  - Added actor/player ownership to snapshots and stable UI context stack records.
  - Added required `InputRequestRecord`, `InputBufferEntry`, and `InputDecisionRecord` schema fields.
  - Unified request/combat sealing under one deterministic seal boundary.
  - Defined catch-up `no_new_physical_input` safe continuous inheritance rules.
  - Reworked one-shot lifecycle into explicit states and results.
  - Changed cross-tick one-shot arbitration to newest valid intent first, with same-snapshot priority only inside one seal.
  - Added hitstop real-time cap to prevent overly old hitstop inputs from firing.
  - Resolved `input_generation_id` and `interruption_epoch` semantics; unsafe boundaries now use both with distinct purposes.
  - Added physical-key `pending_release` state machine, reconciliation cleanup, no-deadlock fallback, and accessible recovery prompts.
  - Changed round-start countdown to allow direction and guard pre-read while still blocking one-shot attack/burst pre-buffering.
  - Kept resume countdown conservative for old held guard after unsafe interruption.
  - Split automatic `presentation_ack_wait` from player recovery prompt; confirm cannot satisfy presentation watermark.
  - Added quick restart/exit as menu-owned semantic requests.
  - Added debug overlay routing rules.
  - Added trace total memory cap, byte overflow formulas, p95/p99 measurement protocol, catch-up performance coverage, and input-owned heap measurement.
  - Reworked Acceptance Criteria into 53 fixture-owned ACs grouped by authority/schema, lifecycle, recovery, UI/Web, accessibility, performance, and review/integration gates.
- Synced `design/gdd/systems-index.md` to show input-buffering revised after fresh re-review and added a fresh re-review next-step checkbox.
- Appended the fresh re-review summary to `design/gdd/reviews/input-buffering-review-log.md`.

## Files

- `design/art/art-bible.md` — approved visual identity and asset standards.
- `design/gdd/game-concept.md` — source concept.
- `design/gdd/systems-index.md` — marks `fixed-logic-runtime` as revised after ninth fresh review pending fresh re-review; marks `input-buffering` as revised after fresh re-review pending fresh re-review.
- `design/gdd/fixed-logic-runtime.md` — current dependency; revised after ninth fresh re-review; pending fresh re-review.
- `design/gdd/input-buffering.md` — revised after fresh re-review; pending fresh re-review.
- `design/registry/entities.yaml` — fixed-runtime constants/formula notes synced after ninth-review revisions; input-buffering registry sync deferred until approval.
- `design/gdd/reviews/fixed-logic-runtime-review-log.md` — ninth fresh review summary appended; earlier eighth summary remains unlogged.
- `design/gdd/reviews/input-buffering-review-log.md` — first full review summary and fresh re-review summary recorded.
- `production/session-state/active.md` — this session state.

## Next

Recommended next step: run `/design-review design/gdd/input-buffering.md` in a fresh session to verify the fresh re-review blockers are resolved. Fixed runtime also remains pending fresh re-review, but the user explicitly deferred it before starting `input-buffering`.
