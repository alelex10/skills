# CR Common — Shared Contract for Code Review Skills

## Purpose

This file defines the shared contract across all `cr-*` skills: how to load project standards, how to retrieve and persist artifacts, and what envelope format to return.

All `cr-*` skills reference this file via:
```
> Follow **Section A** (skill resolution), **Section B** (retrieval), **Section C** (persistence), and **Section D** (return envelope) from `skills/common/cr-common.md`.
```

---

## Section A: Skill Resolution (Loading Project Standards)

CR sub-agents receive project standards pre-injected by the orchestrator as:

```
## Project Standards (auto-resolved)
```

If this block is absent from the prompt, the sub-agent should check for a `## Project Standards (auto-resolved)` block. If still absent, proceed without — this is not an error.

**Do NOT** attempt to load SKILL.md files or the skill registry directly.

---

## Section B: Data Retrieval from Engram

CR artifacts use these topic_key patterns:

| Artifact | topic_key | Produced By | Consumed By |
|----------|-----------|-------------|-------------|
| PR Raw Data | `code-review/{pr-number}/data` | cr-fetcher | Router, all reviewers, publisher |
| Route Plan | `code-review/{pr-number}/route` | cr-router | Publisher |
| Lite Review | `code-review/{pr-number}/reviews/lite` | cr-lite | cr-publisher |
| Correctness Review | `code-review/{pr-number}/reviews/correctness` | cr-correctness | cr-publisher |
| Architecture Review | `code-review/{pr-number}/reviews/architecture` | cr-architecture | cr-publisher |
| Security Review | `code-review/{pr-number}/reviews/security` | cr-security | cr-publisher |
| Reliability Review | `code-review/{pr-number}/reviews/reliability` | cr-reliability | cr-publisher |
| Performance Review | `code-review/{pr-number}/reviews/performance` | cr-performance | cr-publisher |
| Quality Review | `code-review/{pr-number}/reviews/quality` | cr-quality | cr-publisher |
| Formatted Review | `code-review/{pr-number}/formatted` | cr-publisher | GitHub (via gh CLI) |
| State | `code-review/{pr-number}/state` | orchestrator | orchestrator (recovery) |
| Patterns | `code-review-patterns/{category}` | cr-learning | All reviewers (optional) |

### Legacy Artifact Keys (backward compatibility)

| Old Key | New Key | Notes |
|---------|---------|-------|
| `code-review/{pr}/categories/security` | `code-review/{pr}/reviews/security` | Renamed for clarity |
| `code-review/{pr}/categories/performance` | `code-review/{pr}/reviews/performance` | Renamed for clarity |
| `code-review/{pr}/categories/architecture` | `code-review/{pr}/reviews/architecture` | Renamed for clarity |
| `code-review/{pr}/categories/react-vercel` | REMOVED | Replaced by generalist reviewers |
| `code-review/{pr}/review` | REMOVED | Each reviewer writes its own review |

### 2-Step Retrieval Protocol

Step 1 — search:
```
mem_search(query: "code-review/{pr-number}/{artifact}", project: "{project}")
```

Step 2 — get full content (REQUIRED — search returns truncated previews):
```
mem_get_observation(id: {id})
```

### Fallback When Engram Fails ⚠️

**CRITICAL**: Task sub-agents (launched via `task` tool) may NOT have access to Engram tools (`mem_search`, `mem_get_observation`, `mem_save`). If the Engram tools are unavailable:

1. **Run `gh` CLI commands directly**:
   - `gh pr diff <pr-number>` → full diff
   - `gh pr view <pr-number> --json number,title,author,body,headRefName,baseRefName,state,files,additions,deletions,changedFiles` → metadata
   - `gh pr view <pr-number> --comments` → existing comments

2. **Read source files** from the local working directory using the `read` tool

The GATE in each skill SHOULD be: "If Engram retrieval fails, fall back to `gh` CLI + file reads. Only return `blocked` if BOTH fail."

---

## Section C: Persistence to Engram

### Naming Convention

```
title:     "code-review/{pr-number}/{artifact-type}"
topic_key: "code-review/{pr-number}/{artifact-type}"
type:      "discovery"
project:   "{detected project name}"
scope:     "project"
```

### Save Format

Step 1 — save artifact:
```
mem_save(
    title: "code-review/{pr-number}/{artifact-type}",
    topic_key: "code-review/{pr-number}/{artifact-type}",
    type: "discovery",
    project: "{project}",
    scope: "project",
    content: "{full artifact content}"
)
```

Step 2 — update state (data producer for this phase):
```
mem_save(
    title: "code-review/{pr-number}/state",
    topic_key: "code-review/{pr-number}/state",
    type: "config",
    project: "{project}",
    content: "status: {phase}\nartifact: code-review/{pr-number}/{artifact-type}"
)
```

### Fallback When Engram Save Fails

If `mem_save` is unavailable (task sub-agent limitation):
- Return the artifact content **inline** in the structured envelope under `detailed_report`
- The orchestrator will persist it to Engram

---

## Section D: Return Envelope (Structured Output)

Every CR sub-agent MUST return this exact structured envelope:

```
**Status**: success | partial | blocked
**Summary**: {1-3 sentence summary}
**Artifacts**: {comma-separated list of artifact keys}
**Next**: {next phase name, or "none"}
**Risks**: {list of risks, or "None"}
**Skill Resolution**: injected | fallback-registry | fallback-path | none
```

### Status Values

| Status    | Meaning                   | Orchestrator Action                         |
|-----------|---------------------------|---------------------------------------------|
| `success` | Phase completed fully     | Update state, advance DAG                   |
| `partial` | Phase completed with gaps | Update state, warn user, suggest next phase |
| `blocked` | Phase cannot proceed      | STOP, report blocker to user                |

### Skill Resolution Values

| Value              | Meaning                                       |
|--------------------|-----------------------------------------------|
| `injected`         | Received `## Project Standards` from orchestrator |
| `fallback-registry`| Self-loaded from skill registry               |
| `fallback-path`    | Self-loaded via SKILL.md path                 |
| `none`             | No skills loaded                              |
