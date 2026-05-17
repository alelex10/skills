---
name: test-explorer
description: >
  Sub-skill of the `test` orchestrator. Learns the testing conventions, commands, and coverage
  setup for a workspace by inspecting its framework config and existing specs. Produces
  the `env-config` artifact that downstream phases (verifier, writer) consume.
  Trigger: launched by the `test` orchestrator when env-detection reports `needs_learning: true`.
license: Apache-2.0
metadata:
  author: alelex10
  version: "1.0"
---

## Purpose

You are a sub-agent responsible for **environment config learning**. Given a workspace and its framework, you produce TWO artifacts (per D27):

1. `env-config` — pure environment data:
   - Commands (test, coverage, scoped variants).
   - Coverage reporter + output paths + excludes.
   - File patterns (source, spec, spec suffix).
   - Target coverage values.
   - Runtime metadata (package manager, workspace name).
2. `patterns` seed — empty placeholder for D17:
   - `successful_runs: 0`
   - `patterns: []`

You do NOT scan existing specs. Conventions, mock strategies, helpers — all of that is learned by `test-writer` on successful runs (D17), NOT here. Inferring patterns from a 5-spec sample is unreliable; real patterns emerge from runs that actually reach target.

## What You Receive

From the orchestrator:

- `workspace_path`: relative path from project root (e.g. `apps/api`).
- `framework`: framework name from detector (`vitest`, `cypress`, ...).
- `env_detection_location`: ref to the detector artifact (read for context).
- `backend`: active backend.
- `target_override`: optional `--target=N` flag value.

## What to Do

### Step 1: Read env-detection artifact

Read the env-detection artifact from `env_detection_location` to get the framework config files identified by the detector.

### Step 2: Read framework config

Read the framework's config file (e.g. `apps/api/vitest.config.ts`). Extract:

- Coverage `provider` (v8, istanbul, ...).
- Coverage `reporter` list.
- Coverage `reportsDirectory` (default `./coverage` if not set).
- Coverage `exclude` patterns.
- Any other coverage-related settings.

### Step 3: Read workspace `package.json`

Extract the `scripts` section. Identify scripts that invoke the framework. Map them:

- `scripts.test` → likely the base test command.
- `scripts["test:coverage"]` / `scripts.coverage` → coverage variants.

If a yarn/npm workspace, prefer the workspace-prefixed form: `yarn workspace {pkg} test`.

### Step 4: Build commands

Construct the four commands (substitute `{file}` placeholder for scoped variants):

| Command | Template |
|---|---|
| `test` | Full test suite — non-coverage |
| `test_scoped` | Run a single spec file |
| `coverage` | Full suite + coverage |
| `coverage_scoped` | Single spec + coverage |

Examples (vitest in yarn workspace):
- `test`: `yarn workspace {pkg} test --run`
- `test_scoped`: `yarn workspace {pkg} test --run {file}`
- `coverage`: `yarn workspace {pkg} test --coverage --run`
- `coverage_scoped`: `yarn workspace {pkg} test --coverage --run {file}`

### Step 5: Target coverage

If `target_override` is provided, use it for all metrics. Otherwise default to 95 for each of `statements/branches/functions/lines`.

### Step 6: Persist env-config artifact (D27)

Produce ONLY environment data — no conventions, no mock strategy, no helpers, no spec inventory (D28), no learned_at metadata (D29):

```yaml
schema_version: 1
project: {project}
workspace: {workspace_path}
framework: {framework}
commands:
  test: "..."
  test_scoped: "..."
  coverage: "..."
  coverage_scoped: "..."
coverage:
  reporter: v8
  reporters: [text, html, json-summary, lcov]
  reports_directory: {workspace}/coverage
  summary_path:      {workspace}/coverage/coverage-summary.json
  final_path:        {workspace}/coverage/coverage-final.json
  html_path:         {workspace}/coverage/index.html
  lcov_path:         {workspace}/coverage/lcov.info
  excludes: [...]
target_coverage:
  statements: 95
  branches: 95
  functions: 95
  lines: 95
file_patterns:
  source: "src/**/*.ts"
  spec: "src/**/*.spec.ts"
  spec_suffix: ".spec.ts"
runtime:
  package_manager: yarn | npm | pnpm
  workspace_package_name: "..."
```

Persist at `test/{project}/{workspace-slug}/config` in the active backend.

### Step 7: Seed patterns file (D17 — empty seed)

Create an empty patterns artifact at `test/{project}/{workspace-slug}/patterns`:

```yaml
schema_version: 1
project: {project}
workspace: {workspace_path}
successful_runs: 0
patterns: []
```

Only the seed — DO NOT populate patterns from inferences. test-writer fills this on successful runs (D17).

### Step 8: Return Summary

```yaml
status: ok | error
artifact_location: "{backend}:test/{project}/{workspace-slug}/config"
secondary_artifacts:
  - "{backend}:test/{project}/{workspace-slug}/patterns"  # empty seed
executive_summary: |
  Learned env-config for {workspace} ({framework}). Patterns seeded empty.
risks:
  - List anything ambiguous (missing coverage config, multiple scripts, etc.)
next_recommended: verifier-baseline
skill_resolution: none
```

## Rules

- EXECUTOR BOUNDARY: do the work yourself. Do NOT launch sub-agents.
- READ-ONLY: this phase only reads files. Do NOT modify any source/spec/config.
- Sample up to 5 specs in step 4; more wastes context. If the workspace has fewer, sample all.
- If the framework config does not declare coverage settings, fall back to framework defaults and note it in `risks`.
- If `backend == none`, return the full artifact inline in the envelope.
- Return envelope MUST match the Result Contract shape.
