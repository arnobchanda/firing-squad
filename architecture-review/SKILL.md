---
name: architecture-review
description: Critique an architecture document from 6 distinct expert viewpoints in parallel, then synthesize. Use when the user has an architecture.md (typically with supporting context.md and ADRs) and wants a multi-perspective review board critique. Spawns 6 isolated subagents (Doomer, Premortem, Adversary, Integrator, Performance Engineer, Pragmatist) plus a Chair synthesizer. Triggers on requests like "review this architecture", "critique my design doc", "run the architecture panel", or any time the user references an architecture.md they want challenged from multiple angles.
---

# Architecture Review Panel

You are orchestrating a synthetic architecture review board. Six expert personas each critique the same architecture document from a non-overlapping viewpoint, in parallel. A seventh persona (the Chair) synthesizes their outputs into a ranked action list.

## When to use

The user has written or is iterating on an architecture document — usually `architecture.md` with supporting files like `context.md` and a directory of ADRs (Architecture Decision Records) — and wants it stress-tested from multiple expert angles before committing to the design.

## Inputs

Before dispatching, locate and read the architecture artifacts. Look in this order:

1. If the user specified a path or file, use that.
2. Otherwise, look in the current working directory for:
   - `architecture.md` (the main doc — required)
   - `context.md` (background/motivation — read if present)
   - `adr/` or `adrs/` or `decisions/` directory (read all `.md` files inside if present)
3. If none of these exist, ask the user where the document is. Do not guess.

Read all relevant files into context before dispatching subagents. Each subagent will need to see the full document set, so you'll pass the file paths to them.

## Workflow

### Step 1: Confirm scope

Show the user the files you found and confirm before proceeding:

> Found these documents for review:
> - architecture.md (X lines)
> - context.md (X lines)
> - adr/0001-foo.md, adr/0002-bar.md, ... (N ADRs)
>
> Dispatching 6-persona review panel + Chair synthesizer. This will take a few minutes. Proceed? (yes/no)

Wait for confirmation. If they say no, stop. If they want to adjust the persona lineup, accommodate.

### Step 2: Dispatch personas in parallel

Use the Task tool to dispatch **6 subagents in parallel** (single message, multiple tool calls). Each subagent gets:

- The full content of `architecture.md`, `context.md`, and all ADRs (paste as part of the prompt, do not rely on the subagent re-reading from disk)
- The persona instructions from `personas/<persona-name>.md`
- An instruction to return its output in the exact structure defined in that persona file

The 6 personas are:

| # | Persona | File |
|---|---|---|
| 1 | Doomer (Operational) | `personas/doomer-operational.md` |
| 2 | Premortem (Strategic) | `personas/premortem-strategic.md` |
| 3 | Adversary (Red Team) | `personas/adversary.md` |
| 4 | Integrator | `personas/integrator.md` |
| 5 | Performance Engineer | `personas/performance-engineer.md` |
| 6 | Pragmatist | `personas/pragmatist.md` |

For each subagent, the prompt structure is:

```
You are acting as the following persona in an architecture review panel:

<persona_instructions>
[paste full content of personas/<name>.md here]
</persona_instructions>

Here is the architecture documentation under review:

<architecture_md>
[full content of architecture.md]
</architecture_md>

<context_md>
[full content of context.md if present, else "Not provided"]
</context_md>

<adrs>
[concatenated content of all ADR files, each prefixed with its filename]
</adrs>

Produce your critique in the exact output structure specified in your persona instructions. Do not break character. Do not reference the other personas — you are working independently.
```

### Step 3: Collect outputs

When all 6 subagents return, collect their outputs verbatim. Do not edit or summarize them yet.

### Step 4: Dispatch the Chair synthesizer

Use the Task tool once more, dispatching the Chair subagent. Pass it:

- The original `architecture.md` (full content)
- All 6 persona outputs (verbatim, each labeled by persona name)
- The instructions from `personas/chair.md`

### Step 5: Assemble the final report

Build the final report with this exact structure:

```markdown
# Architecture Review — <date>

**Documents reviewed:** <list of files>

---

## Chair's Synthesis

<output from Chair subagent>

---

## Persona Critiques

### 1. Doomer (Operational)
<verbatim output>

### 2. Premortem (Strategic)
<verbatim output>

### 3. Adversary
<verbatim output>

### 4. Integrator
<verbatim output>

### 5. Performance Engineer
<verbatim output>

### 6. Pragmatist
<verbatim output>
```

The Chair's synthesis goes **first** so the user sees the ranked action list immediately, with the raw persona outputs below for drilling into specifics.

### Step 6: Save and display

1. Create directory `./reviews/` if it doesn't exist.
2. Save the report to `./reviews/YYYY-MM-DD-architecture-review.md` (use today's date; if a file with that name exists, append `-2`, `-3`, etc.).
3. Display the **full report inline** in the chat — do not just say "saved to file". The user wants both.
4. End with a one-line summary: `Saved to ./reviews/<filename>. <N> critical issues, <M> important issues, <K> nits across all personas.`

## Rules

- **Always run all 6 personas in parallel** in a single message with multiple Task tool calls. Sequential dispatch defeats the point.
- **Never let personas contaminate each other.** Each subagent must see only the architecture, not other personas' outputs. Only the Chair sees everything.
- **Do not skip the Chair.** The synthesis is what makes the panel actionable; raw outputs are too much to act on directly.
- **Preserve persona outputs verbatim** in the final report. Do not paraphrase or "clean up" their critiques.
- **The user owns the decision** on what to act on. The Chair ranks issues but does not prescribe; the user picks.
