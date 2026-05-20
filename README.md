# firing-squad

> Six expert critics. One architecture doc. Simultaneous fire.

A Claude Code skill that runs a 6-persona architecture review panel + a Chair synthesizer on an architecture document. Each persona is dispatched as an isolated parallel subagent, so they critique your doc independently without contaminating each other. The Chair then reads all six verdicts and produces a ranked action list.

## The squad

| Persona | Question they own |
|---|---|
| Doomer (Operational) | What breaks in production in the first 90 days? |
| Premortem (Strategic) | It is 18 months from now and the project was scrapped — why? |
| Adversary | Where is the highest-leverage attack? |
| Integrator | Where does this touch something it doesn't own, and what could the other side do? |
| Performance Engineer | Where are the budgets, bottlenecks, and missing numbers? |
| Pragmatist | What can be cut, simplified, or deferred without compromising shipping? |
| **Chair** (synthesizer) | Given everyone's findings, what's the ranked action list? |

Each persona has a strict output schema with severity ratings (CRITICAL / IMPORTANT / NIT) and a "Strengths to preserve" section so the synthesis can flag load-bearing strengths the author shouldn't accidentally break in response to critiques.

## Why this works

Single-reviewer setups (even strong ones) fail when the reviewer shares a blind spot with the author. A panel fixes this *only if* the reviewers are forced into non-overlapping blind spots — otherwise they converge on the same critique. Every persona file has an explicit `What you ignore` section that defers other concerns to their owning persona. This is the main anti-overlap mechanism. Without it, every persona drifts toward generic critique.

Severity is normalized across personas (CRITICAL/IMPORTANT/NIT) so the Chair can rank cross-persona. The Chair clusters findings flagged by multiple personas (high-confidence issues), surfaces explicit conflicts (where personas disagree — these need the author to resolve), and lists load-bearing strengths to protect.

## Install

```bash
git clone https://github.com/arnobchanda/firing-squad.git
cp -r firing-squad/architecture-review ~/.claude/skills/
```

The skill installs as `architecture-review/` inside `~/.claude/skills/` — the descriptive name helps Claude auto-trigger it on natural-language requests like "critique my architecture" or "review this design doc."

Restart your Claude Code session (or `/reload`).

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
2. Show you what it found and ask to proceed
3. Dispatch 6 personas in parallel
4. Dispatch the Chair to synthesize
5. Display the full report inline
6. Save a copy to `./reviews/YYYY-MM-DD-architecture-review.md`

## Customizing the squad

The squad composition is set in `architecture-review/SKILL.md`. To swap personas:
1. Add or remove files in `personas/`
2. Update the persona table in `SKILL.md`'s "Step 2: Dispatch personas" section
3. Update the final report assembly section to match

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

- All 6 personas run in parallel via the Task tool, so context isolation is guaranteed — personas don't see each other's critiques. Only the Chair does.
- Each persona's output is preserved verbatim in the final report.
- The Chair's synthesis appears **first** in the report so the action list is immediately visible; raw persona critiques follow below for drilling.

## License

MIT — modify freely.
