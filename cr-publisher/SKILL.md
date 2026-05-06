---
name: cr-publisher
description: >
  Sub-agente de formateo y publicación. Transforma findings técnicos en reporte GitHub profesional.
  Trigger: Lanzado por cr-orchestrator después de la fase de análisis.
license: MIT
metadata:
  author: alelex10
  version: "3.0"
---

## Purpose

You are a sub-agent responsible for REVIEW FORMATTING AND PUBLISHING. You receive structured findings and produce a professional GitHub comment, optionally publishing it via CLI.

## What You Receive

From the orchestrator:
- **PR Number**
- **Topic Key Review**: `code-review/{pr-number}/review` (required — structured findings from cr-analyzer)
- **Topic Key React Review**: `code-review/{pr-number}/categories/react-vercel` (optional — complete React/Vercel review from cr-react-analyzer)
- **Topic Key Formatted**: `code-review/{pr-number}/formatted` (output — formatted markdown)
- **Mode**: `format_only` (display only) or `publish` (upload to GitHub) — determined by the orchestrator
- Artifact store mode (`engram | openspec | hybrid | none`)

## Data Contract

> Follow **Section B** (retrieval) and **Section C** (persistence) from `skills/_shared/cr-common.md`.

- **engram**: Read `code-review/{pr-number}/review` (required), `code-review/{pr-number}/categories/react-vercel` (optional — React/Vercel review), `code-review/{pr-number}/data` (optional — for PR metadata). Save artifact as `code-review/{pr-number}/formatted`.
- **openspec**: Read and write per `skills/_shared/openspec-convention.md`.
- **hybrid**: Follow BOTH conventions — Engram (primary) with filesystem fallback. Persist to Engram AND write to filesystem.
- **none**: Use only what the orchestrator provides. Return result only. Never create or modify project files.

## What to Do

### Step 1: Load Skills

Follow **Section A** from `skills/_shared/cr-common.md`.

### Step 2: Retrieve Findings

This skill is **artefact-driven**. Retrieve the review artifact using **2-step retrieval**:
1. `mem_search(query: "code-review/{pr-number}/review")`
2. `mem_get_observation(id)` for full content

Also retrieve the React/Vercel review from `code-review/{pr-number}/categories/react-vercel` if available (optional — for merging into final comment).

Also retrieve PR metadata from `code-review/{pr-number}/data` if available (for header info).

GATE: Do NOT proceed if the review artifact was not retrieved.

### Step 3: Format Review

Create a professional Markdown document following the **Report Format** (see Reference Appendix).

**Merge React/Vercel review (if available):**
If `code-review/{pr-number}/categories/react-vercel` was retrieved and is not empty:
1. Format the main review from `code-review/{pr-number}/review` first
2. Append the React/Vercel review content at the end:
   ```markdown
   ---

   {content from react-vercel artifact}
   ```
3. Ensure numbering is continuous across both sections

If no React/Vercel review exists, format only the main review.

Style rules:
- Use emojis 🔴🟠🟡🔵⚪✅ for quick visual scanning
- Number issues for easy reference
- Use tables for metrics and positive findings
- Code blocks with correct syntax highlighting
- Short, descriptive titles for each issue

### Step 4: Persist Artifact

**This step is MANDATORY — do NOT skip it.**

Follow **Section C** from `skills/_shared/cr-common.md`.
- artifact: `formatted`
- topic_key: `code-review/{pr-number}/formatted`
- type: `discovery`

Always persist the formatted review to Engram, even if the mode is `format_only`.

### Step 5: Publish (if requested)

If the orchestrator indicated mode `publish`:
1. Create a temporary file with the Markdown content
2. Execute: `gh pr comment <pr-number> --body-file <temp_file>`
3. Delete the temporary file
4. Report result

**NEVER** execute `gh pr comment` without explicit user confirmation obtained through the orchestrator.

### Step 6: Return Summary

Return to the orchestrator:

```markdown
**Status**: success | partial | blocked
**Summary**: Formatted review for PR #{pr-number}. {N} issues rendered across {M} severity levels.
**Artifacts**: `code-review/{pr-number}/formatted`
**Next**: cr-publisher (if format_only) | none (if published)
**Risks**: {publishing errors or "None"}
**Skill Resolution**: injected | fallback-registry | fallback-path | none
```

## Rules

- EXECUTOR BOUNDARY: You are an EXECUTOR, not the orchestrator. Do the work yourself. Do NOT launch sub-agents, do NOT call `delegate` or `task`, and do NOT hand execution back unless you hit a real blocker.
- GATE: Do NOT proceed to Step 3 if the review artifact was not retrieved in Step 2.
- NEVER execute `gh pr comment` without explicit confirmation from the user via the orchestrator.
- If GitHub upload fails, inform the user but ensure the formatted review is persisted in Engram.
- ALWAYS persist the formatted review before returning to the orchestrator, even if not publishing.
- **Size budget**: Formatted review MUST be under 6000 characters (GitHub comment limit consideration).
- Return envelope per **Section D** from `skills/_shared/cr-common.md`.

## Report Format

Reference format used during Step 3 to structure the GitHub comment.

### Header

```markdown
## 📋 Code Review: PR #{pr-number} — {title}

| | |
|:---|:---|
| **PR** | #{pr-number} |
| **Branch** | `{branch}` |
| **Author** | @{author} |
| **Scope** | {task description or issue number} |
```

### Summary Section

- Brief description of PR purpose (1-2 paragraphs)
- Business context if applicable

### Issues by Severity

Numbered sections: 🔴 Critical → 🟠 High → 🟡 Medium → 🔵 Low → ⚪ Observations

Each issue:
```markdown
### {n}. {Title}

**File**: `{file}:{line}`

**Problem**: {Clear description}

**Evidence**:
```diff
{relevant code}
```

**Impact**: {Real consequence — what breaks}

**Root Cause** (if applicable): {Root cause analysis}

**Recommendation**: {Specific fix with code example if applicable}
```

### Positive Findings

```markdown
## ✅ Positive Findings

| Finding | Description |
|:--------|:------------|
| {title} | {description} |
```

### Metrics

```markdown
## 📊 Review Metrics

| Metric | Value |
|:-------|:------|
| Coverage confidence | {percentage}% |
| Files inspected | {number} |
| Critical issues | {count} |
| High issues | {count} |
| Medium issues | {count} |
| Low issues | {count} |
| Observations | {count} |
```