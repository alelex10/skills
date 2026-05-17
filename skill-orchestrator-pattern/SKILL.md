---
name: skill-orchestrator-pattern
description: >
  Orchestrator pattern following 4 invariants and 1 optional. Coordinates sub-agents, never executes directly.
  Trigger: When creating or editing an orchestrator skill that coordinates workflows.
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

## When to Use This Skill

- Creating an orchestrator skill that coordinates multiple sub-agents
- Editing an existing orchestrator workflow
- Understanding coordination patterns vs. execution patterns
- Setting up multi-phase workflows with state persistence

## Orchestrator vs. Executor

| Dimension | Executor | Orchestrator |
|-----------|----------|-------------|
| **Boundary** | "Do the work yourself, don't delegate" | "NEVER execute, ONLY coordinate" |
| **Flow** | Sequential steps | DAG with dependencies |
| **State** | Stateless per invocation | Stateful — persists for `/continue` |
| **Commands** | Invoked by orchestrator | Exposes commands to user |
| **Sub-agents** | Prohibited | Coordinates them |
| **Artifacts** | Reads/writes its own | Passes references |
| **Result format** | Response Format | Result Contract |

## 4 Invariants

Every orchestrator MUST have:

| # | Pattern | Defines | File |
|---|---------|-----------|------|
| 1 | Delegation Boundary | "NEVER execute" + responsibility table | `references/orchestrator-pattern-delegation-boundary.md` |
| 2 | DAG / Dependency Graph | Phase order + dependencies | `references/orchestrator-pattern-dag.md` |
| 3 | State Persistence | Persist state for recovery (backend-agnostic) | `references/orchestrator-pattern-state-persistence.md` |
| 4 | Entry Commands | At least one user-facing command; shape depends on the skill | (see `## Commands` section below) |

## Optional

Include when orchestrator launches sub-agents with structured results:

| # | Pattern | Include when... | File |
|---|---------|------------------|------|
| 5 | Result Contract | Sub-agents return structured data | `references/orchestrator-pattern-result-contract.md` |

## Delegation Boundary Template

```markdown
## Delegation Boundary

You are a COORDINATOR, not an executor. Your only job is to delegate ALL work to sub-agents and synthesize results.

### Hard Stop Rule (ZERO EXECUTIONS)

Before using Read, Edit, Write, or Grep on source/config/skill files:

1. STOP — ask: "Is this orchestration or execution?"
2. If execution → delegate to sub-agent
3. The ONLY files you read: git status/log, engram results, state artifacts
4. "It's just a small change" is NOT a valid reason

### Responsibility Table

| Action | Orchestrator? | Sub-agent? |
|--------|------------|-----------|
| Read/write code | NO | YES |
| Analyze code | NO | YES |
| Write specs/proposals | NO | YES |
| Short answers | YES | — |
| Coordinate phases | YES | — |
| Track state | YES | — |

### Anti-Patterns

- DO NOT read source code to "understand" — delegate
- DO NOT write code — delegate
- DO NOT write specs or proposals — delegate
- DO NOT do "quick" analysis — it bloats context
```

## DAG / Dependency Graph Template

```markdown
## Dependency Graph

[phase-a] -> [phase-b] -> [phase-c]
                  \
                   -> [phase-d]

### Dependencies by Phase

| Phase | Required Dependencies | Optional Dependencies | Artifact |
|-------|------------------|--------------------|----------|
| `phase-a` | None | None | `domain/context/a` |
| `phase-b` | `a` | None | `domain/context/b` |
| `phase-c` | `b` | `a` | `domain/context/c` |
| `phase-d` | `b`, `c` | `a` | `domain/context/d` |

### Resolution Rules

1. A phase executes ONLY if ALL required dependencies exist
2. Optional dependencies are read if present, skipped if absent
3. Each phase writes exactly ONE primary artifact
4. Verify artifact exists before advancing

### Reference-Passing Contract

Orchestrator passes topic keys/paths, NOT content. Sub-agents retrieve themselves:
- `mem_search(query: "{topic_key}", project: "{project}")`
- `mem_get_observation(id: {id})`
```

## State Persistence Template

State and artifacts use a **uniform `location`** field that is backend-agnostic. The orchestrator resolves it according to the active backend.

```markdown
## State Management

- State artifact location: backend-dependent (see Resolution table below)

### State Artifact (shape)

```yaml
phase: current-phase
backend: engram | local | hybrid | none
artifacts:
  - name: artifact-a
    location: "engram:domain/context/a"   # or "file:.atl/domain/a.yaml"
    status: complete
  - name: artifact-b
    location: "engram:domain/context/b"
    status: in_progress
last_updated: 2025-01-15T14:30:00Z
```

### Location resolution

| Backend | `location` prefix | Read | Write |
|---------|-------------------|------|-------|
| `engram` | `engram:{topic_key}` | `mem_search` → `mem_get_observation` | `mem_save` |
| `local` | `file:{relative_path}` | filesystem read | filesystem write (create parents) |
| `hybrid` | both prefixes — write to both, read engram first | engram → fallback file | both |
| `none` | n/a | inline, no recovery possible | inline, warn user |

### Recovery Rules

1. Read state via the active backend.
2. If state exists and `phase != completed` → resume from that phase (prompt user if applicable).
3. If state missing → start from scratch.
4. Each sub-skill receives the **location reference**, not the content.

### State Transitions

```
pending -> phase-a -> phase-b -> ... -> completed
```
```

## Commands

Slash commands are **not** prescribed by this pattern. They depend on the specific skill or family of skills being generated:

- Single-command skills: one slash command does everything (`/foo <args>`).
- Multi-phase skills: separate commands per phase (`/foo-explore`, `/foo-apply`).
- Hybrid: one entry command that detects state and offers continuation (recommended for skills with interruptible flows).

If your skill persists state and supports resumption, prefer the **hybrid** pattern: the single entry command checks for an incomplete prior run and prompts the user to continue or restart. Avoid proliferating `*-continue`, `*-status`, `*-ff` commands unless the skill genuinely benefits from explicit phase invocation.

Define your skill's commands in its own `## Commands` section. The orchestrator pattern only requires that **at least one** command exists as the user-facing entry point.

## Inline Execution Fallback

When `skill` tool cannot pass required parameters to sub-agent:

1. Load the skill's SKILL.md directly
2. Execute inline following those instructions
3. Inject compact rules as project standards
4. Persist artifacts to backend
5. Return structured envelope

## Skill Resolution

Before launching any sub-agent:

1. `mem_search(query: "skill-registry", project: "{project}")` → observation ID
2. `mem_get_observation(id)` → registry content
3. Fallback: read `.atl/skill-registry.md`
4. Cache compact rules section
5. Inject into sub-agent prompt BEFORE task instructions

After each sub-agent returns, check `skill_resolution` field:

| Value | Meaning | Action |
|-------|--------|--------|
| `injected` | Standards passed | ✅ No action |
| `fallback-registry` | Block lost | Re-inject in next delegations |
| `fallback-path` | Found via path | Re-inject |
| `none` | No standards | Re-inject |

## Combined with Executor Patterns

An orchestrator skill COMPLETE uses:
- All 4 orchestrator patterns (from this skill)
- Executor patterns that apply (from skill-template-pattern):
  - Quality Gates (if classifying findings)
  - Response Format (if complex return)
  - Reference Appendix (if needing quick ref)

## Resources

- **References**: See [references/](references/) for full pattern files
  - `orchestrator-pattern-delegation-boundary.md`
  - `orchestrator-pattern-dag.md`
  - `orchestrator-pattern-state-persistence.md`
  - `orchestrator-pattern-result-contract.md`

- **Executor pattern**: See `skill-template-pattern` SKILL.md for execution patterns