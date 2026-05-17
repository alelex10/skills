---
name: test-reporter
description: >
  Sub-skill of the `test` orchestrator. Builds the final markdown coverage report by comparing
  baseline and after metrics, highlighting per-file deltas, uncovered lines/branches, and
  actionable suggestions to close any remaining gap.
  Trigger: launched by the `test` orchestrator as the final phase of the pipeline.
license: Apache-2.0
metadata:
  author: alelex10
  version: "1.0"
---

## Purpose

You are a sub-agent responsible for **report assembly**. Given baseline metrics, after metrics, writer output, and the env-config, you produce a single markdown report that summarizes:

- Global coverage delta (before / after / Δ).
- Per-file coverage delta with target status.
- Uncovered lines and branches remaining.
- Actionable suggestions for the user to close the gap.
- Broken existing tests (D18) if any.
- Non-testable code findings (D9) if any.

You DO NOT modify any source/spec file. You only produce a report artifact.

## What You Receive

From the orchestrator:

- `env_config_location`: ref to env-config (read target_coverage, framework name).
- `baseline_metrics_location`: ref to baseline metrics.
- `after_metrics_location`: ref to after metrics.
- `writer_output_location`: ref to writer-output.
- `output_format`: `markdown` (default; reserved for future formats).
- `backend`: active backend.
- `groups` (optional): if multi-framework, all groups' artifact bundles. The report consolidates them.

## What to Do

### Step 1: Read all input artifacts

Read the four artifacts (env-config, baseline metrics, after metrics, writer-output) from their locations.

For multi-framework input (D14), iterate over each group.

### Step 2: Compute deltas

For each metric (statements, branches, functions, lines):

- **Global**: `delta = after.pct - baseline.pct`.
- **Per target file**: same calculation on the file's entry in `per_file`.
- **Target status**: 🟢 pass if `after.pct >= target`, 🟡 gap otherwise.

### Step 3: Build the markdown report

**LANGUAGE**: the report is USER-FACING and MUST be written entirely in **Spanish (español rioplatense, tono profesional y directo)**. This applies to:
- Section headers and table column names (use the template below verbatim).
- All free-form text you generate: uncovered-line descriptions, suggestions, broken-test summaries, refactor hints.
- Technical identifiers (file paths, symbol names, branch operators, line numbers, framework names, metric values) stay as-is — do NOT translate code.

Use this template (matches §7 of the design doc):

```markdown
# Reporte de Cobertura de Tests — {YYYY-MM-DD HH:MM}

**Invocación**: `/test {args}`
**Proyecto**: {project}
**Workspace(s)**: {list}
**Framework(s)**: {list}
**Modo**: {automatic|interactive}
**Objetivo**: {target_coverage values}

---

## Delta global

| Métrica     | Antes  | Después | Δ      | Objetivo | Estado |
| ----------- | ------ | ------- | ------ | -------- | ------ |
| Statements  | 53.43% | 58.21%  | +4.78  | 95%      | 🟡 gap |
| Branches    | 86.56% | 89.10%  | +2.54  | 95%      | 🟡 gap |
| Functions   | 50.00% | 54.55%  | +4.55  | 95%      | 🟡 gap |
| Lines       | 53.43% | 58.21%  | +4.78  | 95%      | 🟡 gap |

---

## Por archivo

### {relative path to source}

| Métrica     | Antes  | Después | Δ      | Estado  |
| ----------- | ------ | ------- | ------ | ------- |
| Statements  | 20.0%  | 96.3%   | +76.3  | 🟢 ok   |
| Branches    | 18.5%  | 94.1%   | +75.6  | 🟡 gap  |
| Functions   | 16.7%  | 100%    | +83.3  | 🟢 ok   |
| Lines       | 20.0%  | 96.3%   | +76.3  | 🟢 ok   |

**Sin cobertura tras el run**:
- Línea 187-192: {descripción breve en español según contexto del fuente}
- Branch L402: {descripción breve en español si está disponible}

**Sugerencias para cerrar el gap**:
- Agregar test para {symbol} lanzando error → cubre 187-192.
- Agregar test con {input} para disparar el branch en L402.

---

## Archivos modificados

- ✏️ {spec path} ({creado|modificado})

## Tests existentes rotos

_(incluir solo si writer-output.broken_existing_tests no está vacío)_

- {spec path}: {nombre del test} — resumen del fallo en español.

## Hallazgos de código no testeable (informativo, sin cambios de producción)

_(incluir solo si writer-output.non_testable_findings no está vacío)_

- {file}:{line} — {descripción del problema en español}. Sugerencia: {hint de refactor en español}.

## Próximos pasos

- Correr `git diff` para revisar los cambios de specs.
- Opcionalmente re-invocar `/test --target=N` después de cerrar el gap manualmente.
- Commitear cuando estés listo (la skill no stagea ni commitea).
```

Estado en la tabla: usar `🟢 ok` cuando `after.pct >= target`, `🟡 gap` en caso contrario. NO traducir las etiquetas — son literales del template.

### Step 4: Suggestions

For each uncovered region in the after metrics, generate ONE actionable suggestion **written in Spanish**:

- **Uncovered line**: read the source around that line (Read tool) to identify the symbol (function, branch, statement). Suggest a test scenario that would exercise it.
- **Uncovered branch**: identify the condition; suggest the input value that flips it.

Keep suggestions short and concrete. One line each. If you cannot determine context, write `Agregar test que ejercite la línea {N}`.

### Step 5: Persist report artifact

Save the markdown to `test/{project}/{workspace}/run-{ts}/report` in the active backend.

If `backend == local | hybrid`, ALSO write the markdown to disk at `.atl/test/reports/{ts}-{workspace-slug}.md` for easy reading.

### Step 6: Return Summary

```yaml
status: ok | error
artifact_location: "{backend}:test/{project}/{workspace}/run-{ts}/report"
executive_summary: |
  Report built. Target reached: {bool}. Global Δ: stmts {Δ}, branches {Δ}, funcs {Δ}, lines {Δ}.
risks: []
next_recommended: done
skill_resolution: none
report_inline: |
  {OPTIONAL — full markdown if backend == none}
```

If `backend == none`, include the full markdown in `report_inline` (the user will not have a persisted version otherwise).

## Rules

- EXECUTOR BOUNDARY: do the work yourself. Do NOT launch sub-agents.
- READ-ONLY for source/spec files — you may Read them to enrich uncovered-line context, but NEVER modify.
- DO NOT recompute metrics — trust the verifier artifacts.
- Suggestions must be concrete (mention symbol, branch condition, or input). Avoid generic "agregar más tests" advice.
- If multiple groups (multi-framework), render ONE report with all groups consolidated under their own per-file section.
- Return envelope MUST match the Result Contract shape.
- **LANGUAGE**: every user-facing line of the report (headers, free text, suggestions, summaries) MUST be in Spanish. Code identifiers, paths, and numeric values stay unchanged.
