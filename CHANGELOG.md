# Changelog

## v0.2.1 — 2026-05-21

Confidence calibration improvements based on followup feedback.

### Added

- **`persona-confidence` field on every CRITICAL/IMPORTANT finding** — high/medium/low bucket scheme with explicit rubrics. Lets the architect triage faster than re-reading prose to infer how strong the persona's conviction actually was. NIT items skip this (already low-stakes).
- **`persona-confidence` on Defender STRONG/MODERATE defenses** — same scheme; WEAK items skip.
- **`arbitration-confidence`** — replaces v0.2's percentage-based arbitration confidence. Now uses the same high/medium/low scheme as personas. Different semantics (finding correctness vs resolution correctness), same vocabulary.
- **`max-persona-confidence`** on cluster records in the YAML appendix — surfaces the strongest conviction within a convergent finding cluster.
- **`persona-confidence-recommended` / `persona-confidence-dissenting`** on arbitration records — preserves the asymmetry of conviction between sides for diffing.

### Changed

- Convergent findings table in Chair output now includes a `Max persona-confidence` column.
- Arbitration tables now show `persona-confidence` for both recommended and dissenting sides.
- Chair instructions explicitly note that asymmetric persona-confidence between sides should inform arbitration-confidence (high-vs-low → easier call; high-vs-high → genuine substantive disagreement, arbitration-confidence should be medium at most).

### Rationale

LLMs are bad at calibrated percentages — they cluster around 0.7–0.9 and don't reliably distinguish 0.6 from 0.8. Discrete buckets with explicit rubrics force meaningful distinction and resist anchoring. Unifying the scheme across persona findings and arbitration outcomes gives readers one mental model for the whole report.

## v0.2 — 2026-05-21

Structural improvements based on feedback from a real-world run of v0.1 on a multi-tenant cloud-managed BESS architecture. The biggest weakness identified was structural, not persona-quality: personas had no scale context, the panel had no defender, severity was single-axis, and conflicts were parked rather than arbitrated.

### Added

- **Defender persona** — seventh critic that rebuts critique overreaches calibrated to the stated scale. Not a yes-man; defends with named flip-triggers. Identifies critiques that are technically correct in the abstract but contextually wrong for the system's stage and threat model.
- **Required scale context input** — skill now elicits a 4–8 sentence scale paragraph (stage, threat model, compliance posture, device class, named ceilings) before dispatching. Inject verbatim into every persona's prompt. Eliminates the single biggest source of v0.1 calibration noise.
- **Accepted-risks input** — skill elicits a list of risks the architect has already deliberately accepted. Personas may reinforce (note missed implications) or escalate (note scale change invalidates acceptance), but not re-raise as new findings.
- **Two-axis severity** — every finding gets `structural` (independent of scale) and `contextual` (at stated scale) severity ratings. Distinguishes "this is wrong at any scale" from "this is wrong at the scale you stated."
- **Mechanical convergence ranking** — every finding has `convergence: N/7`. Chair fills in numerators and applies a deterministic sort rule: `(structural severity, convergence, contextual severity)` in that priority order.
- **Chair arbitration** — for every conflict, Chair now produces `recommended-default` with confidence (e.g. 70%), `dissenting-view` preserved verbatim from persona output, and named `flip-triggers`. Conflicts are no longer parked for the user.
- **Defender outcomes** — Chair labels each Defender finding as `Defender wins` / `Critic wins` / `See A<n>`.
- **Structured YAML findings appendix** — every report ends with machine-readable YAML. One record per finding with severity, convergence, action, and flip-triggers. Enables diffing future reviews and scaffolding ADRs.
- **Optional `scale.md` and `accepted-risks.md`** — drop these in the project root to skip interactive prompts.

### Changed

- Report structure now: scale context block → Chair synthesis (with arbitration) → 7 persona critiques → YAML appendix. v0.1 had: Chair synthesis → 6 persona critiques.
- Defer with trigger is now a first-class action tier alongside P0/P1/P2.
- Persona summaries split into structural and contextual top-concerns.
- Chair synthesis includes a `Calibration assessment` section judging whether the architecture is appropriately scoped for its stated scale.

### Migration notes

If you're upgrading an existing install, the changes are backward-compatible at the directory level — just sync the updated files. There is one new persona file (`personas/defender.md`) and modifications to all others plus `SKILL.md` and `README.md`. Existing review outputs are not affected; new reviews will use the v0.2 format.

## v0.1.1 — 2026-05-20

- Reformatted persona output schemas to use 2-column tables instead of nested bullets. Improved skimmability without losing information density.
- Added stable finding IDs (C1, C2, I1, I2, N1, N2) so the Chair can reference findings precisely.

## v0.1 — 2026-05-20

Initial release.

- 6-persona panel: Doomer, Premortem, Adversary, Integrator, Performance Engineer, Pragmatist.
- Chair synthesizer producing ranked action list (P0/P1/P2/defer).
- Strict severity ratings (CRITICAL/IMPORTANT/NIT).
- "Strengths to preserve" section per persona to flag load-bearing strengths.
- Parallel dispatch via Task tool for context isolation between personas.
