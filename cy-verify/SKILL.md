---
name: cy-verify
description: >
  Verifica coverage de tests Cypress ejecutando tests y generando reportes de coverage.
  Trigger: Cuando el orchestrator lanza cy-verify para verificar coverage de componentes testeados.
license: Apache-2.0
metadata:
  author: alelex10
  version: "2.0"
---

## Purpose

**IMPORTANT FOR ORCHESTRATOR**: When this skill is launched, YOU (the orchestrator) must act as the sub-agent yourself following these instructions. DO NOT invoke the `skill` tool - that will only return these instructions again. Instead, execute the tests, verify coverage, and return the results directly.

You are a sub-agent responsible for VERIFICATION. You execute Cypress tests, analyze coverage reports, and return a structured verification report with coverage metrics and recommendations.

## What You Receive

From the orchestrator:
- Component paths tested
- Test file paths created
- List of components to verify
- Artifact store mode (`engram | file | hybrid | none`)

## Execution and Persistence Contract

> Follow **Section B** (retrieval) and **Section C** (persistence) from `skills/_shared/cy-phase-common.md`.

- **engram**: Read test artifacts if available. Save verification report as `cy/{component-name}/verify`.
- **file**: Save verification report to filesystem only.
- **hybrid**: Follow BOTH conventions — persist to Engram AND write to filesystem.
- **none**: Return result only.

## What to Do

### Step 1: Load Skills
Follow **Section A** from `skills/_shared/cy-phase-common.md`.

### Step 2: Execute Cypress Tests

Execute tests for the specified components:

```bash
# Execute tests with coverage enabled
npx cypress run --component --spec "{test-path}" --env codeCoverage=true
```

If there are multiple components, execute all:
```bash
# Execute all tests with coverage
npx cypress run --component --env codeCoverage=true
```

### Step 3: Read Coverage Report

**Option A (recommended)**: Read coverage-summary.json for component-specific coverage:

```bash
# Read coverage-summary.json
cat coverage/coverage-summary.json
```

This file contains detailed coverage for all components executed in the test.

**Option B**: View coverage summary with nyc:
```bash
# View global coverage summary
npx nyc report --reporter=text-summary
```

This option only shows a global summary, not component-specific.

### Step 4: Extract Component-Specific Coverage

From coverage-summary.json, extract coverage for each component:

```bash
# Search for specific component coverage
cat coverage/coverage-summary.json | grep -A 1 "{component-path}"

# Example:
cat coverage/coverage-summary.json | grep -A 1 "approve.tsx"
```

Component coverage format:
```json
{
  "/full/path/to/component.tsx": {
    "statements": { "total": 50, "covered": 45, "skipped": 0, "pct": 90 },
    "branches": { "total": 20, "covered": 18, "skipped": 0, "pct": 90 },
    "functions": { "total": 10, "covered": 9, "skipped": 0, "pct": 90 },
    "lines": { "total": 50, "covered": 45, "skipped": 0, "pct": 90 }
  }
}
```

### Step 5: Analyze Results

For each component, analyze:

- **Statements coverage**: Percentage of statements covered
- **Branches coverage**: Percentage of branches covered
- **Functions coverage**: Percentage of functions covered
- **Lines coverage**: Percentage of lines covered

Categorize coverage:
- **Excellent**: 90%+
- **Good**: 75-89%
- **Acceptable**: 60-74%
- **Needs improvement**: <60%

### Step 6: Identify External Components Without Tests

Analyze the tested component to identify external dependencies affecting coverage:

```typescript
// Read component to identify imports
read_file(file_path: "{component-path}")

// Identify external components:
// - Modals (ApproveModal, RejectModal, etc.)
// - External library components (@for-it/web-lib, etc.)
// - Custom hooks
// - External services or utilities
```

Classify external components by coverage impact:
- **High impact**: Modals with callbacks, components with complex logic
- **Medium impact**: UI components with interactions
- **Low impact**: External library components (likely already tested)

### Step 7: Ask User About External Component Exploration

If coverage is not 100% (less than 90% in any metric) and external components were identified:

```markdown
## Coverage Not Complete

Component coverage is {percentage}% but not at 100% due to external components without tests.

### External Components Identified

| Component | Type | Location | Impact |
|-----------|------|----------|--------|
| {name} | {type} | {path} | {high/medium/low} |
| {name} | {type} | {path} | {high/medium/low} |

Do you want to explore external components to get their coverage report?
```

If user responds "yes", proceed to Step 8. If "no", go to Step 9.

### Step 8: Explore External Components (optional)

If user confirms, explore external components:

```bash
# Search for external component test files
find {project-path} -name "{component}.cy.tsx" -o -name "{component}.cy.ts"

# Search for external component coverage in JSON
cat coverage/coverage-summary.json | grep -A 1 "{component-name}"
```

Read coverage-summary.json to get external component coverage:
```bash
# Already read in Step 3, use grep to filter specific components
cat coverage/coverage-summary.json | grep "{component-name}"
```

Generate external component coverage report:
```markdown
### External Component Coverage

| Component | Statements | Branches | Functions | Lines | Status | Has Tests |
|-----------|------------|----------|-----------|-------|--------|----------|
| {name} | {pct}% | {pct}% | {pct}% | {pct}% | {status} | {yes/no} |
```

### Step 9: Ask About Creating Tests for External Components (optional)

After showing external component coverage, ask the user:

```markdown
## Creating Tests for External Components

The following external components have low coverage and affect the main component's coverage:

- {component-1}: {coverage}% - {impact}
- {component-2}: {coverage}% - {impact}

Do you want to create Cypress tests for any of these components?
```

If user responds with a component name, proceed to Step 10. If "no", go to Step 11.

### Step 10: Launch cy-orchestrator for External Component Tests (optional)

If user specifies an external component to test, launch cy-orchestrator:

```typescript
// Launch cy-orchestrator as skill to create tests for external component
skill({ SkillName: "cy-orchestrator" })
```

cy-orchestrator will receive:
- External component path to test
- Best practices from engram (cypress-best-practices)
- Context that this is an external component to improve another component's coverage

cy-orchestrator will execute:
1. cy-explorer to learn patterns
2. User questions about testing type
3. cy-tester to write tests
4. cy-verify to verify new component coverage
5. Save progress to engram
6. Report results

### Step 11: Persist Verification Report

**This step is MANDATORY — do NOT skip it.**

Follow **Section C** from `skills/_shared/cy-phase-common.md`.
- artifact: `verify`
- topic_key: `cy/{component-name}/verify`
- type: `pattern`

### Step 12: Return Summary

Return to the orchestrator:

```markdown
## Coverage Summary

**Components verified**: {count}
**Tests executed**: {count}

### Coverage by Component

| Component | Statements | Branches | Functions | Lines | Status |
|-----------|------------|----------|-----------|-------|--------|
| {name} | 90% | 85% | 95% | 90% | ✅ Excellent |
| {name} | 70% | 65% | 75% | 70% | ⚠️ Acceptable |

{IF COVERAGE NOT 100% AND EXTERNAL COMPONENTS EXIST:}

### External Components Affecting Coverage

| Component | Type | Location | Coverage Impact | Test Status |
|-----------|------|----------|-----------------|------------|
| {name} | {type} | {path} | {high/medium/low} | {yes/no} |
| {name} | {type} | {path} | {high/medium/low} | {yes/no} |

{IF EXTERNAL COMPONENTS EXPLORED:}

### External Component/Utility Coverage

| Component | Statements | Branches | Functions | Lines | Status | Has Tests |
|-----------|------------|----------|-----------|-------|--------|----------|
| {name} | {pct}% | {pct}% | {pct}% | {pct}% | {status} | {yes/no} |

### Recommendations

- **{component}**: {specific recommendations}
- **{component}**: {specific recommendations}

{IF EXTERNAL COMPONENTS WITHOUT TESTS:}

To achieve 100% coverage in {main-component}:
1. Test {external-component-1}: Create Cypress tests for this component
2. Test {external-component-2}: Create Cypress tests for this component
3. Mock external components in {main-component} tests: As alternative

### Detailed Report Files

- **Summary**: `coverage/coverage-summary.json`
- **HTML**: `coverage/lcov-report/index.html`
- **Specific component**: `coverage/lcov-report/{path/component}.tsx.html`
```

## Rules

- ALWAYS execute tests with `--env codeCoverage=true` to generate coverage
- ALWAYS read coverage from `coverage/coverage-summary.json` for component-specific data (Option A)
- ALWAYS analyze coverage by component, not just globally
- ALWAYS categorize coverage (Excellent/Good/Acceptable/Needs improvement)
- ALWAYS identify external components affecting coverage
- ALWAYS ask user if they want to explore external components when coverage < 90%
- ALWAYS ask user if they want to create tests for external components with low coverage
- ALWAYS launch cy-orchestrator if user wants to create tests for an external component
- ALWAYS provide specific recommendations for external components without tests
- DO NOT use `npx nyc report --reporter=text-summary` for component-specific coverage (only gives global summary, use Option A for specific)
- DO NOT assume high coverage means quality tests
- DO NOT omit branch and function analysis
- DO NOT explore external components without user confirmation
- DO NOT launch cy-orchestrator without explicit user confirmation
- Return envelope per **Section D** from `skills/_shared/cy-phase-common.md`.

## Commands

```bash
# Ejecutar tests con coverage
npx cypress run --component --env codeCoverage=true

# Opción A: Ver resumen de coverage desde JSON (recomendado para específico por componente)
cat coverage/coverage-summary.json

# Opción B: Ver resumen de coverage global con nyc
npx nyc report --reporter=text-summary

# Ver coverage de un archivo específico
cat coverage/coverage-summary.json | grep -A 1 "{ruta/archivo}"

# Ver coverage de componentes externos
cat coverage/coverage-summary.json | grep -A 1 "{nombre-componente}"

# Abrir reporte HTML detallado
open coverage/lcov-report/index.html
```

## Coverage Thresholds

| Categoría | Statements | Branches | Functions | Lines |
|-----------|------------|----------|-----------|-------|
| Excelente | 90%+ | 85%+ | 90%+ | 90%+ |
| Bueno | 75-89% | 70-84% | 75-89% | 75-89% |
| Aceptable | 60-74% | 55-69% | 60-74% | 60-74% |
| Mejorar | <60% | <55% | <60% | <60% |

## Resources

- **Cypress Coverage**: https://docs.cypress.io/guides/tooling/code-coverage
- **Istanbul/NYC**: https://istanbul.js.org/
