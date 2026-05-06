# Template Pattern: What to Do

> **Invariante** — Obligatorio en toda skill.

Toda skill **debe** incluir una sección `## What to Do` con los pasos en este orden:

## Con Data Contract

```markdown
## What to Do

### Step 1: Load Skills
Follow **Section A** from `[skill-common-file]`.

### Step 2: [Skill específica - preparación/contexto]
[Leer artifacts necesarios, código existente, etc.]

### Step [N]: [Skill específica - trabajo principal]
[La lógica específica de esta skill: revisar, generar, implementar, etc.]

### Step [N+1]: Persist Artifact
**This step is MANDATORY — do NOT skip it.**

Follow **Section C** from `[skill-common-file]`.
- artifact: `[nombre-del-artifact]`
- topic_key: `[dominio]/[contexto]/[artifact-type]`
- type: `[type]`

### Step [N+2]: Return Summary
Return to the orchestrator:

[Markdown estructurado con resumen de lo hecho]
```

## Sin Data Contract (mínimo)

```markdown
## What to Do

### Step 1: Load Skills
Follow **Section A** from `[skill-common-file]`.

### Step 2: [Skill específica - trabajo principal]
[La lógica específica de esta skill]

### Step [N]: Return Summary
Return to the orchestrator:

[Markdown estructurado con resumen de lo hecho]
```

## Reglas

1. **Step 1 siempre es Load Skills**: Nunca se omite, siempre referencia Section A del common
2. **Persist Artifact solo si hay Data Contract**: Con la frase exacta "This step is MANDATORY — do NOT skip it." Si la skill no tiene Data Contract, este paso se omite
3. **Pasos intermedios**: Varían según la skill — es donde va la lógica específica del dominio
4. **Numeración secuencial**: 1, 2, 3... sin saltos
5. **Return Summary**: Siempre al final, con formato markdown estructurado

## Ejemplos

```
code-review:
  Step 1: Load Skills
  Step 2: Fetch PR Diff
  Step 3: Read Project Conventions
  Step 4: Run Review Analysis
  Step 5: Generate Findings
  Step 6: Persist Artifact (MANDATORY)
  Step 7: Return Summary

sdd-propose:
  Step 1: Load Skills
  Step 2: Create Change Directory
  Step 3: Read Existing Specs
  Step 4: Write proposal.md
  Step 5: Persist Artifact (MANDATORY)
  Step 6: Return Summary
```

### Ejemplo mínimo (sin Data Contract)

```
simple-validator:
  Step 1: Load Skills
  Step 2: Read Input Data
  Step 3: Validate Against Rules
  Step 4: Return Summary
```

## Anti-patrones

❌ Saltarse Step 1 (Load Skills) — siempre debe estar
❌ Omitir frase "This step is MANDATORY" si la skill tiene Data Contract
❌ No seguir Section C del common para persistencia si la skill tiene Data Contract
❌ Mezclar la lógica de negocio con el boilerplate de persistencia/retorno
❌ Pasos sin numeración o con numeración inconsistente