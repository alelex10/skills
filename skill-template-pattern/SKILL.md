---
name: skill-template-pattern
description: >
  Skill executor pattern following 4 invariants and 4 optional sections.
  Trigger: When creating or editing a skill that executes work (not coordinates).
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

## When to Use This Skill

- Creating a new skill that executes work (not coordinates)
- Editing an existing executor skill
- Understanding required sections in any skill
- Following consistent skill structure across the project

## Skill Structure: 4 Invariants

Every executor skill MUST have these sections:

| # | Section | Defines | File |
|---|---------|---------|------|
| 1 | Purpose | Identity and flow | `references/template-pattern-purpose.md` |
| 2 | What You Receive | Orchestrator inputs | `references/template-pattern-what-you-receive.md` |
| 3 | What to Do | Execution steps | `references/template-pattern-what-to-do.md` |
| 4 | Rules | Domain rules + executor boundary | `references/template-pattern-rules.md` |

## Optional Sections

Include these based on need:

| # | Section | Include when... | File |
|---|---------|------------------|------|
| 5 | Data Contract | Skill reads/writes external context | `references/template-pattern-data-contract.md` |
| 6 | Response Format | Complex return summary format | `references/template-pattern-response-format.md` |
| 7 | Quality Gates | Skill classifies findings by severity | `references/template-pattern-quality-gates.md` |
| 8 | Reference Appendix | Quick reference material | `references/template-pattern-reference-appendix.md` |

## Purpose Template

```markdown
## Purpose

You are a sub-agent responsible for [ROLE]. You [input] → [output/action].
```

## What You Receive Template

With Data Contract:
```markdown
## What You Receive

From the orchestrator:
- [input 1]
- [input 2]
- Artifact store mode (`engram | openspec | hybrid | none`)
```

Without Data Contract:
```markdown
## What You Receive

From the orchestrator:
- [input 1]
- [input 2]
```

## What to Do Template

With Data Contract:
```markdown
## What to Do

### Step 1: Load Skills
Follow **Section A** from `[skill-common-file]`.

### Step 2: [Preparation]
[Load artifacts, read code, etc.]

### Step 3: [Main Work]
[Skill-specific logic]

### Step 4: Persist Artifact
This step is MANDATORY — do NOT skip it.
Follow **Section C** from `[skill-common-file]`.

### Step 5: Return Summary
Return to the orchestrator:

[Structured markdown summary]
```

Without Data Contract:
```markdown
## What to Do

### Step 1: Load Skills
Follow **Section A** from `[skill-common-file]`.

### Step 2: [Main Work]
[Skill-specific logic]

### Step 3: Return Summary
Return to the orchestrator:

[Structured markdown summary]
```

## Rules Template

```markdown
## Rules

- EXECUTOR BOUNDARY: You are an EXECUTOR, not the orchestrator. Do the work yourself. Do NOT launch sub-agents, do NOT call `delegate` or `task`, and do NOT hand execution back unless you hit a real blocker.
- [Domain-specific rule]
- [Another rule]
- **Size budget**: [Word limit if applies]
- Return envelope per **Section D** from `[skill-common-file]`.
```

## Guard Clauses & Hard Gates

Use these patterns within Rules:

| Pattern | Format | Use Case |
|---------|--------|---------|
| Guard Clause | `Before [action], check [condition]` | Pre-condition between steps |
| Hard Gate | `GATE: Do NOT proceed until [condition]` | Absolute blocking condition |
| STOP | `If [blocking condition], STOP and report` | Abort execution |

## Data Contract Coupling Rules

- **Data Contract includes reading AND writing together** — never one without the other
- If you include Data Contract, `Artifact Store Mode` is required in "What You Receive"
- If you DON'T include Data Contract, omit Artifact Store Mode
- If skill emits findings classified (CRITICAL/WARNING/SUGGESTION), Quality Gates and Response Format go together — Response Format must include the `severity` field

## Examples

Simple executor skill:
```markdown
## Purpose

You are a sub-agent responsible for CODE VALIDATION. You receive code and produce validation findings.

## What You Receive

From the orchestrator:
- Code content to validate
- Validation rules

## What to Do

### Step 1: Load Skills
Follow Section A from `skills/_shared/common.md`.

### Step 2: Validate Code
Apply validation rules to the code content.

### Step 3: Return Summary
Return validation findings.
```

Skill with Data Contract:
```markdown
## Purpose

You are a sub-agent responsible for CODE REVIEW. You receive a diff and produce a structured review report.

## What You Receive

From the orchestrator:
- Repository and PR info
- Diff content to review
- Artifact store mode (`engram`)

## What to Do

### Step 1: Load Skills
Follow Section A from `skills/_shared/common.md`.

### Step 2: Analyze Diff
Run review analysis on the diff.

### Step 3: Persist Artifact
This step is MANDATORY.
Follow Section C from `skills/_shared/common.md`.

### Step 4: Return Summary
Return structured review findings.
```

## Resources

- **Templates**: See [references/](references/) for full pattern files
  - `template-pattern-purpose.md`
  - `template-pattern-what-you-receive.md`
  - `template-pattern-what-to-do.md`
  - `template-pattern-rules.md`
  - `template-pattern-data-contract.md`
  - `template-pattern-response-format.md`
  - `template-pattern-quality-gates.md`
  - `template-pattern-reference-appendix.md`

- **Orchestrator pattern**: See `skill-orchestrator-pattern` SKILL.md for coordination patterns