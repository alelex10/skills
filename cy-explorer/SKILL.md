---
name: cy-explorer
description: >
  Explora archivos de tests Cypress existentes y sus componentes para aprender patrones y buenas prácticas.
  Busca y guarda buenas prácticas en engram bajo el tópico 'cypress-best-practices'.
  Trigger: Cuando el orchestrator lanza cy-explorer para explorar tests existentes, aprender patrones del proyecto, o analizar componentes para testing.
license: Apache-2.0
metadata:
  author: alelex10
  version: "2.0"
---

## Purpose

**IMPORTANT FOR ORCHESTRATOR**: When this skill is launched, YOU (the orchestrator) must act as the sub-agent yourself following these instructions. DO NOT invoke the `skill` tool - that will only return these instructions again. Instead, explore the codebase, identify patterns, and return the results directly.

You are a sub-agent responsible for EXPLORATION. You investigate existing Cypress tests, identify patterns and best practices, and return a structured analysis. You save discovered patterns to engram for reuse by other Cypress skills.

## What You Receive

From the orchestrator:
- Component paths to explore (optional)
- Artifact store mode (`engram | file | hybrid | none`)

## Execution and Persistence Contract

> Follow **Section B** (retrieval) and **Section C** (persistence) from `skills/_shared/cy-phase-common.md`.

- **engram**: Search for existing `cypress-best-practices` patterns. Save exploration as `cy/{component-name}/explore` (or `cy/explore/{topic-slug}` if standalone). Save new patterns as `cypress-best-practices`.
- **file**: Save exploration to filesystem only.
- **hybrid**: Follow BOTH conventions — persist to Engram AND write to filesystem.
- **none**: Return result only.

## What to Do

### Step 1: Load Skills
Follow **Section A** from `skills/_shared/cy-phase-common.md`.

### Step 2: Search Existing Best Practices

Search engram for existing patterns to avoid duplication:

```typescript
mem_search({
  query: "cypress-best-practices",
  project: "{project-name}",
  type: "pattern"
})
```

If found, retrieve full content:
```typescript
mem_get_observation(id: {observation_id})
```

### Step 3: Explore Cypress Test Files

Find all Cypress test files in the project:

```bash
find apps/web/src -name "*.cy.tsx" -o -name "*.cy.ts"
```

Or use Grep:
```typescript
Grep({
  pattern: "\\.cy\\.(tsx|ts)$",
  path: "/path/to/project",
  glob: "*.cy.*"
})
```

### Step 4: Analyze Each Test and Component

For each test file found:

1. **Read the test file**
   ```typescript
   read_file(file_path: "src/components/example.cy.tsx")
   ```

2. **Identify the component being tested**
   - Find the component mounted with `cy.mount()`
   - Extract the component name
   - Locate the corresponding component file

3. **Read the component**
   ```typescript
   read_file(file_path: "src/components/example.tsx")
   ```

4. **Analyze patterns used**
   - Selectors (data-testid, text, aria-label, classes)
   - Testing strategy (callbacks, validations, async states)
   - Test structure (describe, it, beforeEach)
   - Mocks and stubs used

### Step 5: Identify Best Practices

Look for patterns that represent best practices:

- **Stable selectors**: Use of `data-testid` instead of fragile CSS classes
- **Callback testing**: Verification of `onSubmit`, `onClose`, etc. with `cy.stub()`
- **Error validations**: Tests that verify error messages
- **Async states**: Handling of loading, success, error with `cy.wait()`, `cy.intercept()`
- **Correct types**: Respect of types according to schema (intField → integers)
- **Non-redundant tests**: Avoid trivial tests without functional value

### Step 6: Save New Best Practices to Engram

For each best practice discovered that is not in existing records:

```typescript
mem_save({
  title: "{descriptive practice name}",
  content: """
  **What**: {what practice was found}
  **Why**: {why it's a best practice}
  **Where**: {files where observed}
  **Learned**: {specific implementation details}
  """,
  project: "{project-name}",
  type: "pattern",
  topic_key: "cypress-best-practices",
  scope: "personal"
})
```

### Step 7: Persist Exploration Artifact

**This step is MANDATORY when tied to a named component — do NOT skip it.**

Follow **Section C** from `skills/_shared/cy-phase-common.md`.
- artifact: `explore`
- topic_key: `cy/{component-name}/explore` (or `cy/explore/{topic-slug}` if standalone)
- type: `pattern`

### Step 8: Return Structured Analysis

Return EXACTLY this format to the orchestrator:

```markdown
## Exploration: {topic}

### Current State
{How Cypress testing works today in this project}

### Test Files Found
- `path/to/test.cy.tsx` — {component tested}
- `path/to/other.cy.tsx` — {component tested}

### Patterns Identified
1. **{Pattern name}** — {description}
   - Example: {code snippet}
   - Files: {where observed}

2. **{Pattern name}** — {description}
   - Example: {code snippet}
   - Files: {where observed}

### Best Practices Saved to Engram
- {practice 1}
- {practice 2}

### Inconsistent Patterns
{If any, describe inconsistencies in testing style}

### Best Practices from Engram to Apply
{List of cypress-best-practices relevant for upcoming tests}

### Ready for Testing
{Yes/No — and what the orchestrator should tell the user}
```

## Rules

- The ONLY file you MAY create is the exploration artifact (if in file/hybrid mode)
- DO NOT modify any existing code or test files
- ALWAYS read real test files, never guess about patterns
- Keep your analysis CONCISE - the orchestrator needs a summary, not a novel
- If you can't find enough information, say so clearly
- ALWAYS use `topic_key: "cypress-best-practices"` for saving patterns
- ALWAYS save practices with the structured **What/Why/Where/Learned** format
- DO NOT save trivial or obvious practices
- PRIORITIZE project-specific patterns over generic Cypress patterns
- Return envelope per **Section D** from `skills/_shared/cy-phase-common.md`.

## Commands

```bash
# Buscar archivos de tests Cypress
find apps/web/src -name "*.cy.tsx" -o -name "*.cy.ts"

# Buscar archivos con extensión .cy
find . -type f -name "*.cy.*"

# Contar cantidad de tests Cypress
find apps/web/src -name "*.cy.*" | wc -l
```

## Resources

- **Cypress Docs**: https://docs.cypress.io
- **Engram**: Sistema de memoria persistente para buenas prácticas
