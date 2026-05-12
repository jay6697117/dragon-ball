# Active Session State

## Current Task

Revising `fixed-logic-runtime` GDD for 《星核斗魂》 after the eighth fresh full design re-review.

## Status

`design/gdd/fixed-logic-runtime.md` was freshly re-reviewed after the seventh-review revision. Verdict: MAJOR REVISION NEEDED. The user chose to revise now and approved these eighth-review decisions:

- Recovery pauses during key live combat exchanges are compromised exchanges, not normal fair continuation.
- `presentation_must_show` gates only critical hidden/catch-up/recovery causality, not every normal realtime phase transition.
- Normal-combat spatial facts cannot be satisfied by HUD-only display watermark proof.
- `min_ai_decision_age_ticks = 6` remains a technical floor; first-round/simple CPU profiles must use at least 12 presented running ticks.
- Combat-critical visual ack uses next-host-frame / 50ms fail-degrade behavior; 250ms remains for non-combat UI/recovery prompts.

The eighth-review blockers have been revised in the GDD, and `design/registry/entities.yaml` has been synced with `combat_critical_ack_deadline_ms`, updated ack-deadline ownership, and revised AI decision-age notes. Consistency checks are pending.

## Current Section

Eighth-review revision complete — consistency checks pending; fresh re-review recommended after checks.

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

- Continued the fresh full `/design-review design/gdd/fixed-logic-runtime.md` after context compaction.
- Completed missing `game-designer` specialist review after retry.
- Collected senior `creative-director` synthesis.
- Final fresh re-review verdict: MAJOR REVISION NEEDED.
- User approved revision choices:
  - Recovery pause strategy: mark key live exchange interruptions as compromised exchanges.
  - Must-show scope: only gate critical hidden causality.
  - Visual proof: disallow HUD-only display watermark proof for normal-combat spatial facts.
  - Defaults: conservative player-first CPU/ack behavior.
  - Write scope: `fixed-logic-runtime.md`, `design/registry/entities.yaml`, and `production/session-state/active.md`.
- Revised `design/gdd/fixed-logic-runtime.md`:
  - Updated header to eighth fresh re-review revision pending fresh re-review.
  - Added first-round zero visible recovery-pause target and compromised-exchange framing.
  - Defined `catch_up_preflight_descriptor` fields, forbidden behavior, fail-closed behavior, and descriptor diagnostics.
  - Narrowed `presentation_must_show` to critical hidden/catch-up/recovery causality.
  - Added combat-layer visual proof requirements and disallowed HUD-only proof for normal-combat spatial facts.
  - Added per-fact ack requirement items, consumer policy, terminal failure state, and separate ack response lifecycle.
  - Added `combat_critical_ack_deadline_ms = 50` and kept `presentation_ack_deadline_ms = 250` for non-combat UI/recovery prompts.
  - Reframed `min_ai_decision_age_ticks = 6` as a technical floor and required first-round/simple CPU profiles to use at least 12 presented running ticks.
  - Added committed-to-presented-running tick mapping requirements.
  - Added AI observation unknown buckets, interruption epoch validation, and all-failed validation diagnostics.
  - Made ADR gates concrete with required deliverables per ADR.
  - Rewrote Web profiling acceptance language with browser run aggregation, warm-up, cold first-round readiness, GC unavailable handling, and hard single-stall failure.
  - Added AC-FLR-78 and AC-FLR-79 for first-round recovery integrity and player-facing causality trust.
  - Updated Visual/Audio and UI requirements for per-fact ack items, combat-layer proof, countdown visual ack, and exchange integrity.
- Synced `design/registry/entities.yaml`:
  - Added constant `combat_critical_ack_deadline_ms`.
  - Updated `presentation_ack_deadline_ms` notes and revised date.
  - Updated `min_ai_decision_age_ticks`, `presented_running_tick_index`, and `decision_age_ticks` notes.

## Files

- `design/art/art-bible.md` — approved visual identity and asset standards.
- `design/gdd/game-concept.md` — source concept.
- `design/gdd/systems-index.md` — still marks `fixed-logic-runtime` as revised after seventh fresh review and pending fresh re-review; pending optional update.
- `design/gdd/fixed-logic-runtime.md` — revised after eighth fresh re-review; pending consistency check and fresh re-review.
- `design/registry/entities.yaml` — fixed-runtime constants/formula notes synced after eighth-review revisions.
- `design/gdd/reviews/fixed-logic-runtime-review-log.md` — seventh fresh review summary exists; eighth fresh review summary pending optional append.

## Next

Recommended next step: run targeted consistency checks, then update `design/gdd/systems-index.md` and append the eighth fresh review summary if approved. After that, run `/clear`, then `/design-review design/gdd/fixed-logic-runtime.md` in a fresh session before moving to `input-buffering`.
