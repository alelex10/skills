---
name: cy-tester
description: >
  Escribe tests Cypress para componentes React aplicando buenas prácticas del proyecto.
  Trigger: Cuando el orchestrator lanza cy-tester para escribir tests de un componente específico.
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "2.0"
---

## Purpose

**IMPORTANT FOR ORCHESTRATOR**: When this skill is launched, YOU (the orchestrator) must act as the sub-agent yourself following these instructions. DO NOT invoke the `skill` tool - that will only return these instructions again. Instead, read the component, write the tests, and return the results directly.

You are a sub-agent responsible for WRITING TESTS. You receive a component path, apply best practices from engram, and produce Cypress test files following project patterns.

## What You Receive

From the orchestrator:
- Component path to test
- Best practices from engram (cypress-best-practices) to apply
- Answers to previous questions (testing type, callbacks, validations, etc.)
- Artifact store mode (`engram | file | hybrid | none`)

## Execution and Persistence Contract

> Follow **Section B** (retrieval) and **Section C** (persistence) from `skills/_shared/cy-phase-common.md`.

- **engram**: Read `cypress-best-practices` from engram. Save test artifact as `cy/{component-name}/test`.
- **file**: Write test file to filesystem only.
- **hybrid**: Follow BOTH conventions — persist to Engram AND write to filesystem.
- **none**: Return result only.

## What to Do

### Step 1: Load Skills
Follow **Section A** from `skills/_shared/cy-phase-common.md`.

### Step 2: Read the Component to Test

```typescript
read_file(file_path: "{component-path}")
```

Analyze:
- Component props
- Callbacks (onSubmit, onClose, onChange, etc.)
- Internal states (loading, error, success)
- User interactions (inputs, buttons, forms)
- DOM structure

### Step 3: Search Best Practices in Engram

```typescript
mem_search({
  query: "cypress-best-practices",
  project: "{project-name}",
  type: "pattern"
})
```

Retrieve full content for each practice:
```typescript
mem_get_observation(id: {observation_id})
```

### Step 4: Determine Test Structure

Based on the component and best practices, decide what tests to write:

**Basic tests (always include):**
- Initial render without errors
- Basic props (title, description, etc.)
- Callbacks if they exist
- Accessibility (visibility)

**Component-specific tests:**
- Forms: validations, onSubmit, errors
- Modals: onClose, conditional render
- Inputs: type, onChange, validations
- Buttons: click, disabled states
- Async states: loading, error, success

### Step 5: Write the Test File

Create `{component}.cy.tsx` alongside the component:

```typescript
import { ComponentName } from "./{component}.tsx";

describe("ComponentName", () => {
    describe("initial render", () => {
        it("renders without errors with basic props", () => {
            cy.mount(<ComponentName {...basicProps} />);
            // Verifications
        });
    });

    describe("prop: {propName}", () => {
        it("renders {propName} correctly", () => {
            // Prop test
        });
    });

    describe("prop: onSubmit", () => {
        it("calls onSubmit with correct values", () => {
            const onSubmit = cy.stub().as("onSubmit");
            cy.mount(<ComponentName onSubmit={onSubmit} />);
            // Interactions
            cy.get("@onSubmit").should("have.been.calledOnce");
        });
    });

    describe("accessibility", () => {
        it("content is visible", () => {
            cy.mount(<ComponentName />);
            cy.get("[data-testid='...']").should("be.visible");
        });
    });
});
```

### Step 6: Apply Best Practices from Engram

**Use data-testid for stable selectors:**
```typescript
// In component: add data-testid
<div data-testid="component-root" ... />
<button data-testid="submit-button" ... />

// In test:
cy.get("[data-testid='component-root']")
cy.get("[data-testid='submit-button']")
```

**Test callbacks with cy.stub():**
```typescript
const onSubmit = cy.stub().as("onSubmit");
cy.mount(<Component onSubmit={onSubmit} />);
cy.get("@onSubmit").should("have.been.calledOnce");
cy.get("@onSubmit").should("have.been.calledWithMatch", {...});
```

**Organized structure with describe/it:**
```typescript
describe("ComponentName", () => {
    describe("initial render", () => { ... });
    describe("prop: title", () => { ... });
    describe("prop: onSubmit", () => { ... });
    describe("DOM structure", () => { ... });
    describe("accessibility", () => { ... });
});
```

**Mock ValidationSchema if needed:**
```typescript
const mockSchema: ValidationSchema<Record<string, unknown>> = () => ({
    isValid: true,
    value: {},
});
```

### Step 7: Save the Test File

```typescript
write_to_file({
  TargetFile: "{component-path}.cy.tsx",
  CodeContent: "{test-content}",
  EmptyFile: false
})
```

### Step 8: Add data-testid to Component (if missing)

If the component lacks data-testid, add them for stable selectors:

```typescript
// Add data-testid to key elements
<div data-testid="{component-name}-root">
  <h1 data-testid="{component-name}-title">{title}</h1>
  <button data-testid="{component-name}-submit">Submit</button>
</div>
```

### Step 9: Persist Test Artifact

**This step is MANDATORY — do NOT skip it.**

Follow **Section C** from `skills/_shared/cy-phase-common.md`.
- artifact: `test`
- topic_key: `cy/{component-name}/test`
- type: `pattern`

### Step 10: Return Summary

Return to the orchestrator:

```markdown
## Test Created

**Component**: {component name}
**File**: {test path}
**Tests written**: {count}
**Best practices applied**: {list of engram practices used}

### Tests Included
- Initial render
- Props: {list}
- Callbacks: {list}
- Accessibility
- {other specific tests}

### Component Modified
- Added data-testid attributes: {yes/no}
```

## Rules

- ALWAYS use data-testid for stable selectors
- ALWAYS test callbacks with cy.stub() and aliases
- ALWAYS organize tests with nested describe/it
- ALWAYS include accessibility tests
- ALWAYS apply best practices from engram (cypress-best-practices)
- DO NOT use fragile CSS selectors (Tailwind classes)
- DO NOT write trivial tests without functional value
- DO NOT omit callback testing if the component has them
- Return envelope per **Section D** from `skills/_shared/cy-phase-common.md`.

## Commands

```bash
# Ejecutar el test creado
yarn cypress run --component --spec "{ruta-del-test}"

# Ejecutar en modo interactivo
yarn cypress open --component
```

## Resources

- **Buenas prácticas**: Engram topic_key `cypress-best-practices`
- **Cypress Docs**: https://docs.cypress.io
