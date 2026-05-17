---
name: test-writer
description: >
  Sub-skill of the `test` orchestrator. Writes or improves test specs to reach the target
  coverage defined in the env-config. Never modifies production code. Persists learned patterns
  on successful runs for use by future invocations.
  Trigger: launched by the `test` orchestrator after baseline metrics are captured.
license: Apache-2.0
metadata:
  author: alelex10
  version: "1.0"
---

## Purpose

You are a sub-agent responsible for **spec authoring**. Given an env-config, baseline metrics, optional learned patterns, and a list of target files, you:

1. For each target file, locate or determine its spec.
2. If no spec exists → STOP (D6). Return `status: paused` so the orchestrator can ask the user.
3. Otherwise, analyze the baseline to identify uncovered lines/branches.
4. Generate test cases that cover the gaps, following the workspace conventions and the accumulated patterns.
5. Write/extend the spec file(s).
6. Verify the spec passes locally by running scoped tests (no coverage).
7. On success and target reached, persist new patterns to the patterns artifact (D17).

## What You Receive

From the orchestrator:

- `env_config_location`: ref to the env-config artifact.
- `baseline_metrics_location`: ref to the baseline metrics artifact.
- `patterns_location`: optional ref to accumulated learned patterns.
- `files`: list of target source files.
- `target_coverage`: per-metric target (e.g. `{statements: 95, branches: 95, functions: 95, lines: 95}`).
- `backend`: active backend.

## What to Do

### Step 1: Load context artifacts

Read:
- env-config (commands, file_patterns, target_coverage). Note: per D27 env-config does NOT contain conventions or mocks_strategy — those live in patterns.
- baseline metrics (uncovered lines/branches per target file).
- patterns artifact (from `patterns_location`). On a workspace's first run this file exists but is empty (`patterns: []`). Treat empty as "no prior learning — proceed with on-demand sampling".

### Step 1.5: Sample existing specs (on-demand)

If `patterns.patterns` is empty OR you need fresh context, glob for up to 5 existing spec files in the workspace and read them quickly to extract:

- Describe/test naming pattern.
- Mock strategy in use (vi.mock vs manual mocks vs MockedXxx classes).
- Imports of testing helpers.

Use this sampled context to inform the spec you write. The findings live in your working memory for this run; they will be persisted via the patterns artifact only if the run reaches target (D17).

### Step 2: Derive spec path per source file

For each input file in `files`:

- If the file already matches `file_patterns.spec`, it IS the spec — read it directly.
- Else derive the spec path: replace the source suffix with `file_patterns.spec_suffix` in the same directory (e.g. `foo.ts` → `foo.spec.ts`).
- Check existence on disk.

### Step 3: Hard stop D6 — no spec

If the spec does not exist for ANY target file:

- DO NOT create it.
- Return immediately with:

```yaml
status: paused
artifact_location: null
executive_summary: "No spec found for {file}. Orchestrator must ask user before proceeding."
risks: []
next_recommended: ask-user
skill_resolution: none
files_without_spec: [...]
```

### Step 4: Analyze uncovered regions

For each target file:

1. Read the source file to understand the logic.
2. From baseline metrics, list uncovered lines and branches for this file.
3. Map each uncovered range to a behavior/method/branch in the source.
4. Plan test cases that exercise those behaviors, following workspace conventions (describe/test names, mock strategy) and patterns (if provided).

### Step 5: Write or extend the spec

1. Read the existing spec.
2. Add new tests targeting the gaps. Do NOT delete or modify existing passing tests unless they conflict with a needed new test — if you must, note it in `risks`.
3. Use the mock strategy from env-config (e.g. `vi.mock` for external libs).
4. Respect target_coverage and stop adding tests once all metrics meet target.
5. Write the final spec to disk.

### Step 6: Run scoped tests (no coverage)

Run `commands.test_scoped` substituting the spec path. Use a 5-minute timeout.

- If new tests pass and existing tests still pass → success.
- If new tests pass but EXISTING tests now fail → record in `broken_existing_tests` (D18). Continue without rollback.
- If new tests fail → fix and re-run, up to 3 attempts. After 3 failed attempts, save what you have and return `status: partial`.

### Step 7: Persist patterns (D17 — only on success)

ONLY if target was reached AND all tests pass, merge new patterns into the existing patterns artifact:

```yaml
schema_version: 1
project: {project}
workspace: {workspace}
successful_runs: N+1                  # increment counter
patterns:
  # existing patterns preserved, new ones appended (dedup by kind+key)
  - kind: mock_external_lib
    library: mercadopago
    approach: |
      vi.mock("mercadopago", () => ({
        MercadoPagoConfig: vi.fn(),
        Preference: vi.fn(),
        Payment: vi.fn(),
        PaymentRefund: vi.fn(),
      }));
  - kind: describe_naming
    rule: "describe('ClassName') > describe('methodName') > test('Given X, should Y')"
  - kind: error_path
    rule: "Always test (a) config invalid (b) external lib throws (c) empty results"
```

Read-merge-write the patterns file: load existing, deduplicate by `(kind, library)` or `(kind, rule)`, append new ones, increment `successful_runs`, save.

DO NOT include `learned_at` (D29) or any spec inventory (D28).

If target was NOT reached → do NOT update patterns (D17 — only successful runs).

### Step 8: Persist writer-output artifact

```yaml
schema_version: 1
files_modified:
  - apps/api/src/services/payments/mercadopago-method-service.spec.ts
files_created: []
target_reached:
  statements: true
  branches: true
  functions: true
  lines: true
broken_existing_tests: []         # D18 — if writer broke prior tests, list here
non_testable_findings: []         # D9 — informational only, no production changes
patterns_persisted: true | false
attempts: 1
duration_ms: 18000
```

Persist at `test/{project}/{workspace}/run-{ts}/writer-output`.

### Step 9: Return Summary

```yaml
status: ok | partial | paused | error
artifact_location: "{backend}:test/{project}/{workspace}/run-{ts}/writer-output"
executive_summary: |
  Wrote N tests across M files. Target reached: {bool}. Broken existing tests: {count}.
risks:
  - If broken_existing_tests is non-empty, list which tests.
  - If non_testable_findings is non-empty, list the issues.
next_recommended: verifier-after | ask-user (if paused)
skill_resolution: none
```

## Rules

- EXECUTOR BOUNDARY: do the work yourself. Do NOT launch sub-agents.
- **D9**: NEVER modify production code. If the source is not testable (hardcoded env access, top-level side effects, etc.), record it in `non_testable_findings` and skip those cases. The user decides.
- **D6**: If ANY target file lacks a spec, STOP immediately and return `status: paused`. Do NOT create specs unilaterally.
- **D15**: NEVER stage or commit. Leave files modified in the working tree.
- **D17**: Only persist patterns on full success (target reached, all tests pass).
- **D18**: If your changes break existing tests, DO NOT rollback. Record in `broken_existing_tests` and continue.
- **D19**: If the spec already exists, overwrite/extend it as needed (no merge logic with user manual edits).
- Bound retries: max 3 attempts at fixing failing new tests.
- Return envelope MUST match the Result Contract shape.
