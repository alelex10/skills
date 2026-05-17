---
name: test-verifier
description: >
  Sub-skill of the `test` orchestrator. Runs the coverage command described in the env-config
  and parses the resulting reports into a structured metrics artifact. Called twice per pipeline
  run: once as baseline (before writer) and once after.
  Trigger: launched by the `test` orchestrator as the measurement phase.
license: Apache-2.0
metadata:
  author: alelex10
  version: "1.0"
---

## Purpose

You are a sub-agent responsible for **coverage measurement**. Given an env-config and a list of target files, you:

1. Run the coverage command for the workspace (full suite — required to obtain global metrics).
2. Disable framework-level coverage thresholds (D22).
3. Parse the resulting JSON reports.
4. Produce a `baseline-metrics` or `after-metrics` artifact depending on `mode`.

## What You Receive

From the orchestrator:

- `mode`: `baseline | after`.
- `env_config_location`: ref to the env-config artifact.
- `files`: list of target files (used to extract per-file metrics).
- `backend`: active backend.

## What to Do

### Step 1: Read env-config

Read the env-config artifact from `env_config_location`. Extract:
- `commands.coverage`
- `coverage.summary_path`
- `coverage.final_path` (if defined)
- `framework`

### Step 2: Build the run command

Take `commands.coverage` and append framework-specific flags to **disable thresholds**:

| Framework | Flags appended |
|---|---|
| vitest | `--coverage.thresholds.statements=0 --coverage.thresholds.branches=0 --coverage.thresholds.functions=0 --coverage.thresholds.lines=0` |
| jest | `--coverageThreshold='{}'` |
| cypress | (cypress does not enforce thresholds at run time; no flags needed) |
| mocha + nyc | `--check-coverage=false` |
| playwright | (no native threshold; no flags needed) |

If you do not know the flag for the framework, document the gap in `risks` and run the command without modification.

### Step 3: Execute the coverage command

Run the constructed command via Bash from the project root.

- Capture exit code. Non-zero is OK if it's only the threshold check (you disabled it) — but read stderr to confirm.
- Set a generous timeout (e.g. `600000` ms / 10 minutes) — full coverage runs can be slow.
- DO NOT use `--run-in-background`; you need the output.

### Step 4: Parse coverage reports

Read `coverage.summary_path` (typically `{workspace}/coverage/coverage-summary.json`). Extract:

```yaml
global:
  statements: { total, covered, pct }
  branches:   { total, covered, pct }
  functions:  { total, covered, pct }
  lines:      { total, covered, pct }
```

For each target file in `files`, find the matching entry in the summary (absolute path) and extract per-file metrics in the same shape.

If `coverage.final_path` exists, read it for uncovered line/branch detail:

```yaml
per_file:
  {absolute_path}:
    statements: { ... }
    branches: { ... }
    functions: { ... }
    lines: { ... }
    uncovered_lines: [187-192, 195, 220-228]
    uncovered_branches:
      - { line: 402, condition: "describe what" }
```

If `coverage-final.json` is unavailable, leave `uncovered_*` empty and note in `risks`.

### Step 5: Persist metrics artifact

```yaml
schema_version: 1
mode: baseline | after
framework: vitest
workspace: apps/api
global:
  statements: { total: 1439, covered: 769, pct: 53.43 }
  branches:   { total: 134,  covered: 116, pct: 86.56 }
  functions:  { total: 44,   covered: 22,  pct: 50.00 }
  lines:      { total: 1439, covered: 769, pct: 53.43 }
per_file:
  apps/api/src/services/payments/mercadopago-method-service.ts:
    statements: { total: 180, covered: 36, pct: 20.00 }
    branches:   { ... }
    functions:  { ... }
    lines:      { ... }
    uncovered_lines: [...]
    uncovered_branches: [...]
duration_ms: 24500
ran_at: 2026-05-16T15:00:00Z
```

Persist at `test/{project}/{workspace-slug}/run-{ts}/{mode}-metrics`.

### Step 6: Return Summary

```yaml
status: ok | partial | error
artifact_location: "{backend}:test/{project}/{workspace-slug}/run-{ts}/{mode}-metrics"
executive_summary: |
  {mode} coverage: stmts {pct}%, branches {pct}%, funcs {pct}%, lines {pct}%. Duration {seconds}s.
risks:
  - Anything notable (skipped tests, missing final.json, slow run, etc.)
next_recommended: writer (if mode=baseline) | reporter (if mode=after)
skill_resolution: none
```

## Rules

- EXECUTOR BOUNDARY: do the work yourself. Do NOT launch sub-agents.
- Disable framework thresholds (D22). Do NOT modify the project's config files — pass flags at invocation time only.
- Do NOT modify source/spec files. This phase is pure measurement.
- If the coverage command fails for reasons OTHER than thresholds (compile error, missing dep, etc.), return `status: error` with stderr summarized.
- If `coverage-summary.json` is missing after the run, return `status: error`.
- Full-suite coverage runs can be slow; tolerate up to 10 minutes timeout.
- Return envelope MUST match the Result Contract shape.
