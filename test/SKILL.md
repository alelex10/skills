---
name: test
description: >
  Test coverage orchestrator. Measures coverage before/after, writes or improves specs,
  and produces a detailed delta report. Multi-framework (vitest, cypress, jest, mocha, playwright, ...).
  Trigger: when user runs `/test <files>`, asks to improve test coverage, write specs, or
  measure how testing changes affected coverage for given files.
license: Apache-2.0
metadata:
  author: alelex10
  version: "1.0"
---

## Purpose

Coordinate a six-phase pipeline to measure and improve test coverage for one or more files:

```
detector → (explorer?) → verifier:baseline → writer → verifier:after → reporter
```

You are an ORCHESTRATOR. You NEVER execute the phases directly — every phase is delegated to its specialized sub-skill (`test-detector`, `test-explorer`, `test-verifier`, `test-writer`, `test-reporter`).

## When to Use This Skill

- User runs `/test <files>` explicitly.
- User asks to "improve coverage of X", "write tests for X", "measure coverage delta".
- User picks one or more source files and wants the full pipeline.

DO NOT use this skill for:
- Production code refactoring (this skill NEVER touches production — see Rules).
- Debugging an already-failing test (the user must fix that first).

## Commands

```
/test [--target=N] [--framework=X] [--refresh-config] <files...>
```

| Argument | Meaning |
|---|---|
| `<files...>` | One or more source files (or specs). Multi-framework input is grouped automatically. |
| `--target=N` | Override target coverage % (defaults to value in saved env-config or 95). |
| `--framework=X` | Force a specific framework. Default: auto-detect from path. |
| `--refresh-config` | Re-learn the env-config for the affected workspace before running. |

### Single entry, resumable

There is **one** command: `/test`. It is intelligent:

- Detects an incomplete prior run and offers to continue or restart.
- Detects first-invocation-of-session and prompts for mode + backend (see below).
- Detects no-spec scenarios and prompts the user (D6).

There are NO `/test-continue`, `/test-status`, `/test-ff` commands.

### First Invocation Prompts

On the first `/test` of a session, ask:

1. **Execution mode**: `automatic` (run all phases back-to-back) vs `interactive` (pause after each phase). Cache for the session.
2. **Backend** (only if no env-config exists yet for this project): `engram | local | hybrid`. Cache for the project.

### Resumption

Before launching any phase, read the active backend for a state artifact:

- If `state.phase != completed` → prompt: "Veo un run anterior en fase `{phase}`. ¿Continuar o reiniciar?"
- Continue → resume from the saved phase using stored location references.
- Restart → wipe state, start fresh.

## Delegation Boundary

You are a COORDINATOR. Your only job is to delegate work to sub-skills and synthesize results.

### Hard Stop Rule (ZERO EXECUTIONS)

Before using Read, Edit, Write, Grep, or Bash on source/spec/config files: STOP and delegate.

The ONLY direct calls allowed:

- Read/write the state artifact through the active backend (`mem_save`/`mem_search` or filesystem on `.atl/test/`).
- Bash for `git status` / `git log` if needed to confirm working-tree cleanliness.
- Conversational prompts to the user (mode, backend, D6, resumption).

| Action | Orchestrator? | Sub-skill? |
|---|---|---|
| Read source / spec files | ❌ | ✅ test-detector / test-explorer / test-writer |
| Run coverage commands | ❌ | ✅ test-verifier |
| Write specs | ❌ | ✅ test-writer |
| Build the report | ❌ | ✅ test-reporter |
| Track state, pass references | ✅ | — |
| Prompt user | ✅ | — |

### Anti-Patterns

- DO NOT read source files to "decide what framework it is" — delegate to test-detector.
- DO NOT parse `coverage-summary.json` inline — delegate to test-verifier.
- DO NOT write a "quick spec" inline because it's small — delegate to test-writer.
- DO NOT inline-render the final report — delegate to test-reporter.

## Dependency Graph

```
test-detector
     │
     ▼  (only if needs_learning === true OR --refresh-config)
test-explorer
     │
     ▼
test-verifier (mode=baseline)
     │
     ▼
test-writer  ── no spec exists ──► STOP (D6), ask user
     │
     ▼
test-verifier (mode=after)
     │
     ▼
test-reporter
```

### Dependencies by Phase

| Phase | Required | Optional | Primary Artifact |
|---|---|---|---|
| `test-detector` | none | none | `env-detection` |
| `test-explorer` | `env-detection` with `needs_learning=true` | — | `env-config` + empty `patterns` seed (D17/D27) |
| `test-verifier:baseline` | `env-config` | — | `baseline-metrics` |
| `test-writer` | `env-config`, `baseline-metrics` | learned `patterns` | `writer-output` |
| `test-verifier:after` | `env-config`, `writer-output` | — | `after-metrics` |
| `test-reporter` | `baseline-metrics`, `after-metrics`, `writer-output`, `env-config` | — | `report` |

### Resolution Rules

1. A phase executes ONLY if all required dependencies exist and are readable.
2. Each phase writes exactly ONE primary artifact to the active backend.
3. Verify artifact exists before advancing to the next phase. If missing → STOP and report.
4. Sub-skills receive `location:` references (D23), NEVER inline content.

### Multi-Framework Grouping (D14)

If `<files...>` spans frameworks, group files by framework BEFORE running the DAG. Each group runs the full pipeline independently. test-reporter receives all groups and produces a single consolidated report.

## State Management

State artifact location depends on the active backend:

| Backend | Location |
|---|---|
| `engram` | topic_key `test/{project}/run-state` |
| `local` | `.atl/test/run-state.yaml` |
| `hybrid` | both |
| `none` | not persisted; resumption disabled; warn user |

### State Artifact Shape

```yaml
schema_version: 1
project: amia-servicio-empleo
mode: interactive | automatic
backend: engram | local | hybrid | none
overrides:
  target: 95           # only if --target flag was passed
  framework: vitest    # only if --framework flag was passed
  refresh_config: true # only if --refresh-config was passed
groups:
  - framework: vitest
    workspace: apps/api
    files:
      - apps/api/src/services/payments/mercadopago-method-service.ts
    phase: writer        # current phase for this group
    artifacts:
      - name: env-detection
        location: "engram:test/amia/apps-api/env-detection"
        status: complete
      - name: env-config
        location: "engram:test/amia/apps-api/config"
        status: complete
      - name: baseline-metrics
        location: "engram:test/amia/apps-api/run-2026-05-16T15:00/baseline"
        status: complete
      - name: writer-output
        location: "engram:test/amia/apps-api/run-2026-05-16T15:00/writer"
        status: in_progress
last_updated: 2026-05-16T15:00:00Z
```

### State Transitions

```
pending → detector → explorer? → baseline → writer → after → reporter → completed
                                              │
                                              └── (no spec) → paused-d6
                                              │
                                              └── (writer broke existing tests, D18) → continues, reports later
```

### Recovery Rules

| Backend | Read | Write |
|---|---|---|
| `engram` | `mem_search` → `mem_get_observation` | `mem_save` with stable topic_key |
| `local` | filesystem read of `.atl/test/...` | filesystem write, create parents |
| `hybrid` | engram first; fallback file | write both |
| `none` | not supported | not supported |

## What Each Sub-Skill Receives (Reference-passing — D23)

The orchestrator launches each sub-skill via the Skill tool, passing only references and small parameters. The sub-skill reads larger artifacts from the active backend itself.

### test-detector

- `project_root`: absolute path
- `files`: list of input paths
- `backend`: active backend
- `force_framework`: optional
- `refresh`: bool

### test-explorer

- `workspace_path`: relative path
- `framework`: detected framework
- `env_detection_location`: ref to detector output
- `backend`

### test-verifier

- `mode`: `baseline | after`
- `env_config_location`: ref
- `files`: list of target paths
- `backend`

### test-writer

- `env_config_location`: ref
- `baseline_metrics_location`: ref
- `patterns_location`: optional ref (D17)
- `files`: list of target paths
- `target_coverage`: from env-config or `--target` override
- `backend`

### test-reporter

- `env_config_location`: ref
- `baseline_metrics_location`: ref
- `after_metrics_location`: ref
- `writer_output_location`: ref
- `output_format`: `markdown` (default)
- `backend`

## Result Contract

Every sub-skill must return:

```yaml
status: ok | partial | paused | error
artifact_location: "{backend-prefixed location}"
executive_summary: |
  One-sentence summary of what happened.
risks: []
next_recommended: detector | explorer | verifier-baseline | writer | verifier-after | reporter | done | ask-user
skill_resolution: injected | fallback-registry | fallback-path | none
```

Orchestrator MUST validate `artifact_location` is readable before advancing.

## Hard Stops

- **D6 — no spec exists**: test-writer returns `status: paused`, `next_recommended: ask-user`. Orchestrator prompts: "No spec found for `{file}`. Create one from scratch or report and exit?"
- **D18 — existing tests broken**: test-writer reports the breakage in its envelope (no rollback). Orchestrator advances to verifier-after and reporter — the breakage appears in the final report.
- **Backend `none`**: warn user, run inline without persistence; resumption disabled.

## Rules

- ORCHESTRATOR BOUNDARY: never execute work directly. Delegate.
- **D9**: NEVER modify production code. Sub-skills only write specs.
- **D15**: NEVER stage or commit. The user owns git.
- **D17**: After a successful run (target reached), test-writer persists learned patterns to `test/{project}/{workspace}/patterns`.
- **D22**: When invoking test-verifier, instruct it to disable framework-level coverage thresholds. The skill's target is the only judge.
- **D24**: Do NOT consult any `skill-registry`. Conventions are learned by test-explorer.
- Every sub-skill envelope must match the Result Contract; missing fields → treat as `status: error`.

## References

- Pattern followed: `skill-orchestrator-pattern`.
- Design doc: `/home/alelex10/Documentos/Obsidian Vault/skills-doc/test-skill-design.md`.
- Sub-skills: `test-detector`, `test-explorer`, `test-verifier`, `test-writer`, `test-reporter`.
