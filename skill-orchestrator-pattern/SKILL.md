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
| 3 | State Persistence | Persist state for recovery | `references/orchestrator-pattern-state-persistence.md` |
| 4 | Slash Commands | User-facing commands | `references/orchestrator-pattern-slash-commands.md` |

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

```markdown
## State Management

- `domain/identifier/state`: State and metadata

### State Artifact

```yaml
phase: current-phase
artifacts:
  - name: artifact-a
    topic_key: domain/context/a
    status: complete
  - name: artifact-b
    topic_key: domain/context/b
    status: in_progress
last_updated: 2025-01-15T14:30:00Z
```

### State Transitions

```
pending -> phase-a -> phase-b -> ... -> completed
```

### Recovery Rules

| Mode | Protocol |
|------|----------|
| `engram` | `mem_search` → `mem_get_observation` |
| `openspec` | Read `openspec/changes/.../state.yaml` |
| `hybrid` | Try engram first, fallback |
| `none` | No recovery — explain to user |
```

## Slash Commands Template

```markdown
## Commands

- `/command-start [params]`: Start workflow
- `/command-continue`: Continue with next phase
- `/command-status`: Show current state
- `/command-ff`: Fast-forward to completion
- `/command-phase [params]`: Run specific phase

### Continue Logic

1. Read state to determine current phase
2. Find first incomplete phase
3. Verify required dependencies exist
4. If missing → inform user
5. If OK → delegate sub-agent
```

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

## Example: SDD Orchestrator Commands

```markdown
## Commands

| Command | Action |
|---------|-------|
| `/sdd-init` | Run `sdd-init` |
| `/sdd-explore <topic>` | Run `sdd-explore` |
| `/sdd-continue [change]` | Continue next phase |
| `/sdd-ff [change]` | Execute through tasks |
| `/sdd-apply [change]` | Run `sdd-apply` |
| `/sdd-verify [change]` | Run `sdd-verify` |
| `/sdd-archive [change]` | Run `sdd-archive` |
```

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
  - `orchestrator-pattern-slash-commands.md`
  - `orchestrator-pattern-result-contract.md`

- **Executor pattern**: See `skill-template-pattern` SKILL.md for execution patterns