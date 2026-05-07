# CR Common — Shared Contract for Code Review Skills

## Purpose

This file defines the shared contract across all `cr-*` skills: how to load project standards, how to retrieve and persist artifacts, and what envelope format to return.

All `cr-*` skills reference this file via:
```
> Follow **Section A** (skill resolution), **Section B** (retrieval), **Section C** (persistence), and **Section D** (return envelope) from `skills/_shared/cr-common.md`.
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

| Artifact | topic_key | Required by |
|----------|-----------|-------------|
| PR Raw Data | `code-review/{pr-number}/data` | fetcher, all categorizers, analyzer, publisher |
| Security Categories | `code-review/{pr-number}/categories/security` | analyzer |
| Performance Categories | `code-review/{pr-number}/categories/performance` | analyzer |
| Architecture Categories | `code-review/{pr-number}/categories/architecture` | analyzer |
| React Review | `code-review/{pr-number}/categories/react-vercel` | publisher (optional) |
| Analyzed Review | `code-review/{pr-number}/review` | publisher |
| Formatted Review | `code-review/{pr-number}/formatted` | publisher (output) |
| State | `code-review/{pr-number}/state` | orchestrator |

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

Step 2 — update state (orchestrator only):
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

---

## Artifact Type Reference

| Artifact | Produced By | Consumed By | Content |
|----------|-------------|-------------|---------|
| `data` | cr-fetcher | All categorizers, analyzer, publisher | Full diff + PR metadata + comments |
| `categories/security` | cr-security-categorizer | cr-analyzer | Security smell index |
| `categories/performance` | cr-performance-categorizer | cr-analyzer | Performance smell index |
| `categories/architecture` | cr-architecture-categorizer | cr-analyzer | Architecture smell index |
| `categories/react-vercel` | cr-react-analyzer | cr-publisher (optional) | React/Vercel complete review |
| `review` | cr-analyzer | cr-publisher | Structured findings |
| `formatted` | cr-publisher | GitHub (via gh CLI) | Formatted markdown |
| `state` | orchestrator | orchestrator (recovery) | DAG state |
