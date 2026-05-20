# firing-squad

> Seven expert critics. One architecture doc. Simultaneous fire.

A Claude Code skill that runs a 7-persona architecture review panel + a Chair synthesizer on an architecture document. Each persona is dispatched as an isolated parallel subagent, so they critique your doc independently without contaminating each other. The Chair then reads all seven verdicts, arbitrates conflicts with named flip-triggers, and produces a ranked action list plus a machine-readable findings appendix.

## The squad

| Persona | Question they own |
|---|---|
| Doomer (Operational) | What breaks in production in the first 90 days? |
| Premortem (Strategic) | It is 18 months from now and the project was scrapped — why? |
| Adversary | Where is the highest-leverage attack? |
| Integrator | Where does this touch something it doesn't own, and what could the other side do? |
| Performance Engineer | Where are the budgets, bottlenecks, and missing numbers? |
| Pragmatist | What can be cut, simplified, or deferred without compromising shipping? |
| Defender | Which critiques are correctly identified problems at the wrong scale? |
| **Chair** (synthesizer) | Given everyone's findings, what's the ranked action list with explicit conflict resolution? |

Each critic persona has a strict output schema with two-axis severity ratings (**structural** — independent of scale; **contextual** — at the stated scale) plus a convergence counter, so the Chair can rank findings mechanically. Each persona's findings include a "Strengths to preserve" section so the synthesis can flag load-bearing strengths the author shouldn't accidentally break in response to critiques.

## What's new in v0.2

| Change | Why |
|---|---|
| **Required scale context input** | Personas without scale context invent their own threat models and produce miscalibrated reviews. The skill now elicits a 4–8 sentence scale paragraph (stage, threat model, compliance, device class, named ceilings) and injects it verbatim into every persona's prompt. |
| **Accepted-risks input** | Stops personas from re-flagging risks the architect has already deliberately accepted. Personas may reinforce (note missed implications) or escalate (note that scale has changed), but not re-raise as new. |
| **Defender persona** | A seventh persona reads the same architecture and rebuts critique overreaches — critiques that are technically correct in the abstract but contextually wrong for the stated scale. Not a yes-man; defends with named flip-triggers. |
| **Two-axis severity** | Every finding gets `structural` and `contextual` severity ratings. A genuinely structural problem doesn't move with scale; a contextual one does. The two-axis grid makes "defer with named trigger" a first-class outcome. |
| **Mechanical convergence ranking** | Every finding has a `convergence: N/7` field. The Chair fills in the numerator after seeing all outputs, then sorts findings by `(structural severity, convergence, contextual severity)` in that priority order. No more eyeballed P0 rankings. |
| **Per-finding confidence** (v0.2.1) | Every CRITICAL/IMPORTANT finding includes a `persona-confidence` bucket (high/medium/low) with explicit rubrics. Lets the architect triage faster — a `high` confidence CRITICAL is meaningfully different from a `low` one. Unified scheme across persona findings and Chair arbitrations. |
| **Chair arbitrates with confidence** | For every conflict, the Chair now produces three fields: `recommended-default` with `arbitration-confidence` bucket (high/medium/low), `dissenting-view` preserved verbatim from the persona output, and named `flip-triggers`. Conflicts are no longer parked for the user to resolve. |
| **Structured YAML findings appendix** | Every report ends with a machine-readable YAML block. One record per finding with severity, convergence, action, and flip-triggers. Use it to diff future reviews against the prior, and to scaffold ADRs from individual records. |

## Why this works

Single-reviewer setups (even strong ones) fail when the reviewer shares a blind spot with the author. A panel fixes this *only if* the reviewers are forced into non-overlapping blind spots — otherwise they converge on the same critique. Every persona file has an explicit `What you ignore` section that defers other concerns to their owning persona. This is the main anti-overlap mechanism.

Six critics with no defender bias the synthesis toward "fix everything." The Defender persona (v0.2) rebuts overreaches calibrated to the stated scale — not by defending bad architecture, but by identifying critiques that demand solutions disproportionate to the realistic threat model and named ceilings.

Two-axis severity (v0.2) separates "this is wrong at any scale" from "this is wrong at the scale you stated." Convergence weighting (v0.2) is mechanical, not eyeballed. Chair arbitration (v0.2) replaces parked conflicts with recommended defaults plus named flip-triggers, making disagreement actionable without compressing it.

## Install

```bash
git clone https://github.com/arnobchanda/firing-squad.git
cp -r firing-squad/architecture-review ~/.claude/skills/
```

The skill installs as `architecture-review/` inside `~/.claude/skills/` — the descriptive name helps Claude auto-trigger it on natural-language requests like "critique my architecture" or "review this design doc."

Restart your Claude Code session (or `/reload`).

### Symlink for easier updates

If you plan to iterate on the skill, symlink the directory instead of copying:

```bash
# Linux/Mac
rm -rf ~/.claude/skills/architecture-review
ln -s /full/path/to/firing-squad/architecture-review ~/.claude/skills/architecture-review

# Windows PowerShell (Admin or Developer Mode)
Remove-Item -Recurse -Force $env:USERPROFILE\.claude\skills\architecture-review
New-Item -ItemType SymbolicLink -Path $env:USERPROFILE\.claude\skills\architecture-review -Target F:\path\to\firing-squad\architecture-review
```

Then `git pull` updates the skill in place — just `/reload` to pick up changes.

## Use

In a directory containing `architecture.md` (and optionally `context.md`, `adr/`):

```
Run the architecture review panel on this doc.
```

Or just:

```
Use the architecture-review skill.
```

The skill will:
1. Locate and read your architecture artifacts
2. **Elicit scale context and accepted risks** (this is non-negotiable in v0.2)
3. Show you what it found and ask to proceed
4. Dispatch 7 personas in parallel
5. Dispatch the Chair to synthesize with arbitration
6. Display the full report inline (Chair synthesis first, then raw critiques, then YAML appendix)
7. Save a copy to `./reviews/YYYY-MM-DD-architecture-review.md`

### Optional pre-population

If you'd rather not be prompted, drop these files in your project root before running:

- `scale.md` — your scale and stage paragraph (4–8 sentences covering stage, threat model, compliance posture, device class, named ceilings)
- `accepted-risks.md` — your list of risks already deliberately accepted

The skill will read them and skip the interactive prompt.

## Customizing the squad

The squad composition is set in `architecture-review/SKILL.md`. To swap personas:
1. Add or remove files in `personas/`
2. Update the persona table in `SKILL.md`'s "Step 3: Dispatch personas" section
3. Update the final report assembly section to match
4. Update the convergence denominator references (`N/7` → `N/<new-count>`) in the Chair and any critic files

Suggested alternates (not currently included):

| Persona | Question they own |
|---|---|
| Optimist | What goes right if this works? What's the upside case? |
| Maintainer (Future You) | I just inherited this. Can I understand it? Can I change it safely? |
| Standards Auditor | Does this satisfy the spec/standard we have to comply with? Where's the gap? |
| Cost Accountant | What does this cost in BOM, license fees, certification fees, engineering hours, opex? |
| Skeptic of Novelty | What's actually new here vs. what's standard? Why aren't we using the boring proven thing? |
| Devil's Advocate on Assumptions | List your top 5 load-bearing assumptions. What if each is wrong? |

## Notes

- All 7 personas run in parallel via the Task tool, so context isolation is guaranteed — personas don't see each other's critiques. Only the Chair does.
- Each persona's output is preserved verbatim in the final report.
- The Chair's synthesis appears **first** in the report so the action list is immediately visible; raw persona critiques follow below for drilling; the YAML appendix sits at the end for tooling.

## License

MIT — modify freely.
