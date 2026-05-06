# Template Pattern: What You Receive

> **Invariante** — Obligatorio en toda skill.

Toda skill **debe** incluir una sección `## What You Receive` que declare:

## Con Data Contract

```markdown
## What You Receive

From the orchestrator:
- [input 1 específico de la skill]
- [input 2 específico de la skill]
- Artifact store mode (`engram | openspec | hybrid | none`)
```

## Sin Data Contract

```markdown
## What You Receive

From the orchestrator:
- [input 1 específico de la skill]
- [input 2 específico de la skill]
```

## Reglas

1. **Fuente**: Siempre "From the orchestrator" — las skills no descubren contexto por su cuenta, lo reciben
2. **Inputs específicos**: Solo lo que esa skill necesita para operar, no más
3. **Artifact store mode**: Obligatorio si la skill tiene Data Contract — siempre va al final de la lista. Si no tiene Data Contract, se omite
4. **Formato**: Lista bullet, sin tablas, sin prosa

## Ejemplos

### Con Data Contract

```
code-review:
  From the orchestrator:
  - Repository and PR info
  - Diff content to review
  - Artifact store mode (`engram | openspec | hybrid | none`)

sdd-apply:
  From the orchestrator:
  - Change name
  - The specific task(s) to implement (e.g., "Phase 1, tasks 1.1-1.3")
  - Artifact store mode (`engram | openspec | hybrid | none`)
```

### Sin Data Contract

```
simple-validator:
  From the orchestrator:
  - Input data to validate
  - Validation rules
```

## Anti-patrones

❌ "You read the codebase to understand..." — eso va en "What to Do", no en "What You Receive"
❌ Omitir el artifact store mode si la skill tiene Data Contract
❌ Incluir el artifact store mode si la skill NO tiene Data Contract
❌ Listar dependencias de otras skills que no recibe directamente del orchestrator
❌ Usar formato narrativo en lugar de lista bullet