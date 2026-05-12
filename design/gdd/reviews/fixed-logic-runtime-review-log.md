# Review Log: fixed-logic-runtime

## Review — 2026-05-11 — Verdict: NEEDS REVISION

Scope signal: XL
Specialists: game-designer, systems-designer, qa-lead, godot-specialist, performance-analyst, gameplay-programmer, ux-designer, ui-programmer, audio-director, creative-director
Blocking items: 8 | Recommended: 7
Summary: The GDD direction is correct and protects the core pillars: fixed tick authority, presentation read-only behavior, and non-authoritative Godot physics all support “读招定胜负” and “我可以练会”. The review found implementation-blocking gaps in tick terminology, hitstop tick semantics, catch-up fairness/performance, schema contracts, UI focus recovery, and QA instrumentation. These blockers were revised in the same session; a fresh re-review is required before using this GDD as the foundation for downstream systems.
Prior verdict resolved: First review
Revision status: Revised 2026-05-11; pending fresh re-review

## Review — 2026-05-11 — Verdict: NEEDS REVISION

Scope signal: XL
Specialists: game-designer, systems-designer, qa-lead, godot-specialist, performance-analyst, gameplay-programmer, ux-designer, ui-programmer, audio-director, creative-director
Blocking items: 8 | Recommended: 0
Summary: The second full re-review confirmed that the runtime foundation is directionally sound but still needed sharper implementation contracts around recovery pause, catch-up stop reasons, CPU/training-dummy observation fairness, event/state ordering, UI/audio delivery, trace bounds, and acceptance-criteria triage. The GDD was revised the same day to add safe resume countdown semantics, strict AI observation snapshots, namespaced runtime event ordering, separate runtime state sequencing, Web-safe catch-up guards, bounded trace contracts, and MVP/ADR/hardening AC tiers.
Prior verdict resolved: Yes — first review blockers were narrowed, but second review found remaining contract gaps.
Revision status: Revised 2026-05-11; pending fresh re-review

## Review — 2026-05-11 — Verdict: NEEDS REVISION

Scope signal: XL
Specialists: game-designer, systems-designer, qa-lead, godot-specialist, performance-analyst, gameplay-programmer, ux-designer, ui-programmer, audio-director, creative-director
Blocking items: 7 | Recommended: 0
Summary: The third full re-review found that the runtime foundation remained directionally sound but still needed implementation-ready contracts for AC tiering, total ordering, catch-up/recovery precedence, command/AI fairness, QA schemas, Web budgets, and presentation delivery idempotency. The GDD was revised the same day to add AC-FLR-66 through AC-FLR-74, deterministic catch-up stop precedence, AI decision-age requirements, atomic RuntimePresentationPacket delivery, stale generation invalidation, Web performance/memory budgets, and transition-matrix QA coverage.
Prior verdict resolved: Partially — second review blockers were narrowed, but the third review found remaining contract gaps.
Revision status: Revised 2026-05-11; pending fresh re-review

## Review — 2026-05-11 — Verdict: MAJOR REVISION NEEDED

Scope signal: XL
Specialists: game-designer, systems-designer, qa-lead, godot-specialist, performance-analyst, gameplay-programmer, ux-designer, ui-programmer, audio-director, ai-programmer, creative-director
Blocking items: 12 | Recommended: 4
Summary: The fourth full re-review found the runtime direction still creatively sound but structurally unsafe for implementation: catch-up could hide player-readable causes, presentation delivery was not strong enough to prove the player saw key facts, AI/dummy fairness was not enforceable, and QA/Web budgets were not measurable enough. The GDD was revised the same day to add pre-result catch-up fairness gates, ack-gated RuntimePresentationPacket delivery, presented-running-tick AI reaction age, batch-staged command validation, versioned QA schemas, measurable Web profiling, ADR implementation gates, and registry/index consistency fixes.
Prior verdict resolved: No — third-review blockers were narrowed but escalated into a structural MAJOR REVISION verdict.
Revision status: Revised 2026-05-11; pending fresh re-review

## Review — 2026-05-11 — Verdict: MAJOR REVISION NEEDED

Scope signal: XL
Specialists: game-designer, systems-designer, qa-lead, godot-specialist, performance-analyst, gameplay-programmer, ux-designer, ui-programmer, audio-director, ai-programmer, creative-director
Blocking items: 8 | Recommended: 4
Summary: The fifth full re-review found the runtime foundation still creatively sound but not yet implementation-safe: `presented_running_tick_index`, presentation display-watermark semantics, CPU/training-dummy fairness, catch-up preflight/precedence, event and packet idempotency, Web budgets, QA criteria, and ADR gates needed stronger contracts. The GDD was revised the same day to define display-watermark ack, MVP CPU/dummy deterministic fairness, small-step Web catch-up, corrected catch-up stop precedence, idempotent packet/audio/UI contracts, measurable Web budgets, QA schemas, ADR gates, and registry consistency.
Prior verdict resolved: No — fourth-review blockers were narrowed, but fifth review still found structural implementation gaps.
Revision status: Revised 2026-05-11; pending fresh re-review

## Review — 2026-05-12 — Verdict: MAJOR REVISION NEEDED

Scope signal: XL
Specialists: game-designer, systems-designer, qa-lead, godot-specialist, performance-analyst, gameplay-programmer, ux-designer, ui-programmer, audio-director, ai-programmer, creative-director
Blocking items: 10 | Recommended: 5
Summary: The sixth full re-review confirmed the fixed-runtime direction is still aligned with readable, fair combat, but implementation remained unsafe because AC-FLR-73 contradicted the small-step catch-up policy, `presentation_ack_wait` and `presented_running_tick_index` were still ambiguous, key readability phases could be compressed, AI/dummy fairness needed mode separation, and Godot/Web ADR gates and profiling criteria were not yet concrete enough. The GDD was revised to resolve the 5-tick catch-up contradiction, fully protect startup/active/recovery/punish/control-return facts, canonicalize ack wait under `runtime_state = paused`, add normal-tick presented-watermark auto-advance, split dummy fairness modes, tier ACs, strengthen Web profiling and payload-budget contracts, and sync registry terminology.
Prior verdict resolved: Partially — fifth-review blockers were narrowed, but sixth review still found structural implementation gaps before this revision.
Revision status: Revised 2026-05-12; pending fresh re-review

## Review — 2026-05-12 — Verdict: MAJOR REVISION NEEDED

Scope signal: XL
Specialists: game-designer, systems-designer, qa-lead, godot-specialist, performance-analyst, gameplay-programmer, ux-designer, ui-programmer, audio-director, ai-programmer, technical-artist, creative-director
Blocking items: 12 | Recommended: 5
Summary: The seventh fresh re-review confirmed the fixed-runtime direction is still creatively aligned, but implementation remained unsafe because Web profiling methodology, presentation ack lifecycle, display-watermark readability proof, catch-up tier consistency, recovery UX, AI/dummy visibility enforcement, ADR gates, audio degradation, and trace/payload budgets still lacked enforceable contracts. The GDD was revised to separate real browser profiling from synthetic backlog conformance, add conservative visual ack/degradation, map must-show facts to minimum visual cues, harden presented-watermark and decision-age validation, split audio degradation from focus suspension, add explicit ack/watermark/audio ACs, and sync registry constants.
Prior verdict resolved: Partially — sixth-review blockers were narrowed, but seventh fresh re-review still found contract gaps before this revision.
Revision status: Revised 2026-05-12; pending fresh re-review
