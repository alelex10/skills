# Orchestrator Pattern: Result Contract

> **Invariante** — Obligatorio en todo orchestrator.

El Result Contract define el formato estructurado que **TODOS los sub-agentes deben retornar** al orchestrator. Sin esto, el orchestrator no puede parsear resultados, verificar completitud, ni decidir qué fase sigue.

Es el **lado orchestrator** del Response Format del executor: el executor define qué retorna, el Result Contract define qué el orchestrator **exige** que le retornen.

---

## Template

```markdown
## Result Contract

Every sub-agent MUST return a structured envelope. No free-form text.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | enum | YES | `success`, `partial`, or `blocked` |
| `executive_summary` | string | YES | 1-3 sentence summary of what was done |
| `detailed_report` | string | NO | Full output. Omit if already inline in summary. |
| `artifacts` | list | YES | Artifact keys/paths written. Empty list if none. |
| `next_recommended` | string | YES | Next phase to run, or `"none"` |
| `risks` | list | YES | Risks discovered. Empty list or `"None"` if none. |
| `skill_resolution` | enum | YES | `injected`, `fallback-registry`, `fallback-path`, or `none` |

### Status Values

| Status | Meaning | Orchestrator Action |
|--------|---------|---------------------|
| `success` | Phase completed fully | Update state, advance DAG |
| `partial` | Phase completed with gaps | Update state, warn user, suggest next phase |
| `blocked` | Phase cannot proceed | STOP, report blocker to user |

### Skill Resolution Values

| Value | Meaning | Orchestrator Action |
|-------|---------|---------------------|
| `injected` | Standards were passed correctly | ✅ No action |
| `fallback-registry` | Block was lost, sub-agent found registry | Re-read registry, re-inject in subsequent delegations |
| `fallback-path` | Sub-agent found standards via hardcoded path | Same correction as above |
| `none` | No standards found at all | Same correction as above |

### Example

```markdown
**Status**: success
**Summary**: Proposal created for {change-name}. Defined scope, approach, and rollback plan.
**Artifacts**: Engram `sdd/{change-name}/proposal` | openspec `openspec/changes/{change-name}/proposal.md`
**Next**: sdd-spec or sdd-design
**Risks**: None
**Skill Resolution**: injected — 3 skills (react-19, typescript, tailwind-4)
```
```

---

## Reglas

1. **Todos los sub-agentes retornan el mismo formato**: No importa la fase, el envelope es idéntico. El orchestrator parsea un formato, no N.
2. **`status` es obligatorio y enum**: Solo `success`, `partial`, o `blocked`. No inventar valores.
3. **`skill_resolution` es obligatorio**: Sin este campo, el orchestrator no puede detectar si la inyección de estándares falló (compaction safety).
4. **`artifacts` es una lista, nunca un string**: Incluso si es un solo artifact, es una lista de uno.
5. **`next_recommended` puede ser `"none"`**: Si la fase es la última del DAG, el sub-agente retorna `"none"`.
6. **El orchestrator NO parsea texto libre**: Si un sub-agente retorna texto sin estructurar, el orchestrator no puede verificar completitud ni decidir qué fase sigue.

## Ejemplo real (SDD Return Envelope)

```markdown
## D. Return Envelope

Every phase MUST return a structured envelope to the orchestrator:

- `status`: `success`, `partial`, or `blocked`
- `executive_summary`: 1-3 sentence summary of what was done
- `detailed_report`: (optional) full phase output, or omit if already inline
- `artifacts`: list of artifact keys/paths written
- `next_recommended`: the next SDD phase to run, or "none"
- `risks`: risks discovered, or "None"
- `skill_resolution`: how skills were loaded — `injected`, `fallback-registry`, `fallback-path`, or `none`

Example:

**Status**: success
**Summary**: Spec created for auth-migration. 12 requirements defined, 8 edge case scenarios documented.
**Artifacts**: Engram `sdd/auth-migration/spec` | openspec `openspec/changes/auth-migration/specs/auth/spec.md`
**Next**: sdd-design
**Risks**: Breaking change in API v1 — consider feature flags
**Skill Resolution**: injected — 2 skills (react-19, typescript)
```

## Ejemplo real (cy-orchestrator)

```markdown
## Result Contract

Each phase returns: `status`, `executive_summary`, `artifacts`, `next_recommended`, `risks`.
```

## Diferencia con Response Format (executor)

| Dimensión | Result Contract (orchestrator) | Response Format (executor) |
|-----------|-------------------------------|---------------------------|
| **Quién lo define** | El orchestrator | La skill executor |
| **Quién lo cumple** | Los sub-agentes | La skill misma |
| **Propósito** | Parsear resultados y decidir qué fase sigue | Estructurar el artifact de salida |
| **`skill_resolution`** | Obligatorio (feedback loop) | Obligatorio (parte del envelope) |
| **Ubicación** | En la skill del orchestrator | En la skill del executor |

Ambos usan el mismo formato de envelope — la diferencia es de **perspectiva**: el executor lo define para sí mismo, el orchestrator lo exige para sus sub-agentes.

## Anti-patrones

❌ Permitir sub-agentes que retornan texto libre — el orchestrator no puede parsear ni decidir
❌ Inventar campos ad-hoc por fase — el envelope es idéntico para todas las fases
❌ Omitir `skill_resolution` — sin este campo, el feedback loop de corrección no funciona
❌ Usar `status` como string libre — es un enum con 3 valores, no una descripción
❌ No manejar `partial` — si el sub-agente completó con gaps, el orchestrator necesita saberlo para advertir al usuario
❌ Poner `detailed_report` como obligatorio — es opcional, el summary puede ser suficiente