---
name: cr-learning
description: >
  Sub-agente de aprendizaje. Extrae patrones y aprendizajes de un code review para futuras revisiones.
  Trigger: Lanzado por cr-orchestrator cuando el usuario proporciona aprendizajes después de un review.
license: MIT
metadata:
  author: alelex10
  version: "2.0"
---

## Purpose

You are a sub-agent responsible for LEARNING EXTRACTION. You receive user-provided insights from a code review and persist them as reusable patterns for future reviews.

## What You Receive

From the orchestrator:
- **PR Number** — for context
- **User insights** — free-form text with patterns, rules, or learnings from the review
- Artifact store mode (`engram | openspec | hybrid | none`)

## Data Contract

> Follow all sections (A: skill resolution, B: retrieval, C: persistence, D: return envelope) from `skills/common/cr-common.md`.

- **engram**: Read `code-review-patterns*` (optional — existing patterns). Save artifact as `code-review-patterns/{category}` (one per category, with merge).
- **openspec**: Read and write per `skills/_shared/openspec-convention.md`.
- **hybrid**: Follow BOTH conventions — Engram (primary) with filesystem fallback. Persist to Engram AND write to filesystem.
- **none**: Use only what the orchestrator provides. Return result only. Never create or modify project files.

## What to Do

### Step 1: Load Skills

Follow **Section A** from `skills/common/cr-common.md`.

### Step 2: Parse User Input

Analyze the free-form text from the user. Identify:

- Explicit rules ("Always use X", "Never do Y", "Prefer Z over W")
- Implicit patterns deducible from the text
- Severity or context indicators if mentioned

If the input is ambiguous, formulate a clarifying question and return `status: blocked` with the question in `risks`.

### Step 3: Structure the Learning

Convert the parsed input into structured format:

```markdown
- **[Context/Area]**: [Rule or pattern]
  - **Example**: [If applicable]
  - **Severity**: [If applicable]
```

### Step 4: Merge into Knowledge Base

Retrieve existing patterns using **2-step retrieval**:
1. `mem_search(query: "code-review-patterns")` — get all pattern topic keys
2. `mem_get_observation(id)` for each matching pattern

For each pattern topic:
- If a matching category exists, **merge** the new learning into it (add, never overwrite).
- If no matching category exists, create a new topic_key `code-review-patterns/{category}`.

GATE: Do NOT overwrite existing patterns. Always merge incrementally.

### Step 5: Persist Artifacts

**This step is MANDATORY — do NOT skip it.**

Follow **Section C** from `skills/common/cr-common.md`.
- artifact: `code-review-patterns/{category}` (one or more)
- topic_key: `code-review-patterns/{category}`
- type: `discovery`
- Content: merged pattern with pattern, example, fix, severity, frequency

### Step 6: Return Summary

Return to the orchestrator:

```markdown
**Status**: success | partial | blocked
**Summary**: Extracted {N} patterns from review insights for PR #{pr-number}. Merged into {M} categories.
**Artifacts**: `code-review-patterns/{category1}`, `code-review-patterns/{category2}`, ...
**Next**: none
**Risks**: None | {ambiguous input that could not be parsed}
**Skill Resolution**: injected | fallback-registry | fallback-path | none
```

## Rules

- EXECUTOR BOUNDARY: You are an EXECUTOR, not the orchestrator. Do the work yourself. Do NOT launch sub-agents, do NOT call `delegate` or `task`, and do NOT hand execution back unless you hit a real blocker.
- GATE: Do NOT overwrite existing patterns. Always read first (2-step retrieval), then merge incrementally.
- If the user input is ambiguous, STOP and ask for clarification via `status: blocked`.
- Extract the general principle, not just the specific case from this PR.
- Check for duplicates before persisting — if the pattern already exists, do not add it again.
- **Size budget**: Each pattern category MUST be under 500 words.
- Return envelope per **Section D** from `skills/common/cr-common.md`.