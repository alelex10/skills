---
name: test-detector
description: >
  Sub-skill of the `test` orchestrator. Detects the testing framework and resolves the
  workspace for input files, and checks whether a saved env-config exists in the active backend.
  Trigger: launched by the `test` orchestrator as the first phase of the pipeline.
license: Apache-2.0
metadata:
  author: alelex10
  version: "1.0"
---

## Purpose

You are a sub-agent responsible for **environment detection**. Given a set of input files and a project root, you identify:

1. The testing framework in use (vitest, cypress, jest, mocha, playwright, ...).
2. The workspace each file belongs to (monorepo-aware).
3. Whether a saved env-config already exists for that workspace.
4. Whether the env-config must be re-learned (missing or `refresh=true`).

You produce a single artifact: `env-detection`.

## What You Receive

From the orchestrator:

- `project_root`: absolute path to the repository root.
- `files`: list of input paths (source files or specs).
- `backend`: active backend (`engram | local | hybrid | none`).
- `force_framework`: optional override.
- `refresh`: bool — if true, mark `needs_learning: true` regardless of saved state.

## What to Do

### Step 1: Resolve workspace per file

For each input file:

1. Walk up from the file's directory until you find a `package.json`. That directory is the workspace.
2. If no `package.json` is found before the project root, use the project root as the workspace.
3. Record `workspace_path` (relative to project root).

### Step 2: Detect framework

For each unique workspace:

1. If `force_framework` is set, use it.
2. Otherwise, read the workspace `package.json` and inspect `devDependencies` + `dependencies`. Map to framework:

   | Dependency | Framework |
   |---|---|
   | `vitest` | vitest |
   | `cypress` | cypress |
   | `jest`, `@jest/core`, `ts-jest` | jest |
   | `mocha` | mocha |
   | `playwright`, `@playwright/test` | playwright |

3. Confirm by searching for the framework's config file (`vitest.config.*`, `cypress.config.*`, `jest.config.*`, `.mocharc.*`, `playwright.config.*`).
4. If multiple frameworks are present, prefer the one whose config file exists at the workspace root.
5. If no framework detected, return `status: error` with a clear executive_summary.

### Step 3: Check for saved env-config

For each (project, workspace) pair, look up the saved env-config:

| Backend | Lookup |
|---|---|
| `engram` | `mem_search(query: "test/{project}/{workspace-slug}/config")` |
| `local` | check `.atl/test-configs/{workspace-slug}.yaml` exists |
| `hybrid` | engram first, fallback file |
| `none` | always `config_exists: false` |

Set:
- `config_exists: true | false`
- `needs_learning: !config_exists || refresh`

### Step 4: Persist env-detection artifact

Produce the artifact:

```yaml
schema_version: 1
project: {project-slug-from-root}
groups:
  - workspace: apps/api
    framework: vitest
    files: [apps/api/src/services/foo.ts, ...]
    detected_via:
      - apps/api/package.json:devDependencies.vitest
      - apps/api/vitest.config.ts
    config_exists: true | false
    config_location: "{location_ref or null}"
    needs_learning: true | false
generated_at: 2026-05-16T15:00:00Z
```

Persist to the active backend at location `test/{project}/run-{ts}/env-detection`. Return the location reference in the envelope.

### Step 5: Return Summary

Return the Result Contract envelope:

```yaml
status: ok | error
artifact_location: "{backend}:test/{project}/run-{ts}/env-detection"
executive_summary: |
  Detected N frameworks across M workspaces. {needs_learning} workspaces require env-config learning.
risks: []
next_recommended: explorer | verifier-baseline
skill_resolution: none
```

`next_recommended`:
- `explorer` if ANY group has `needs_learning: true`.
- `verifier-baseline` otherwise.

## Rules

- EXECUTOR BOUNDARY: You are an EXECUTOR. Do the work yourself with Read/Glob/Grep/Bash. Do NOT launch sub-agents.
- Read `package.json` files only; do NOT read source files.
- Do NOT modify any file. This phase is purely read + persist artifact.
- If you cannot detect a framework for any group, return `status: error` with the offending paths in the summary.
- If `backend == none`, persist the artifact inline (return content in the envelope) and warn the user that resumption is disabled.
- Return envelope MUST match the Result Contract shape above.
