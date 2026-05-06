# Orchestrator Pattern: State Persistence for Recovery

> **Invariante** — Obligatorio en todo orchestrator multi-fase.

El orchestrator es **stateful** por necesidad. Pierde contexto en cada compaction, así que persiste el estado del workflow completo para poder retomar con `/continue`.

---

## Template

```markdown
## State Management

- `[domain]/{identifier}/state`: Estado y metadata del workflow (upsert por topic_key).
    - `phase`: `[current-phase]`
    - `artifacts`: lista de artifacts completados con sus topic_keys
    - `last_updated`: ISO timestamp

Artifacts por fase:

| Artifact       | Topic Key                            | Written by                           |
| -------------- | ------------------------------------ | ------------------------------------ |
| `[artifact-a]` | `[domain]/{identifier}/[artifact-a]` | `[phase-a]`                          |
| `[artifact-b]` | `[domain]/{identifier}/[artifact-b]` | `[phase-b]`                          |
| `[artifact-c]` | `[domain]/{identifier}/[artifact-c]` | `[domain]/{identifier}/[artifact-c]` |

### State Transitions
```

[pending] -> [phase-a] -> [phase-b] -> ... -> [completed]

```

| Status | Meaning |
|--------|---------|
| `pending` | No phase has started yet |
| `[phase-a]` | Phase A is executing |
| `[phase-a]_done` | Phase A completed, ready for next |
| `completed` | All phases done |

### Recovery Rules

| Mode | Recovery Protocol |
|------|-------------------|
| `engram` | `mem_search(query: "{topic_key}", project: "{project}")` → `mem_get_observation(id)` for full content |
| `openspec` | Read `openspec/changes/{identifier}/state.yaml` |
| `hybrid` | Try engram first, fallback to filesystem |
| `none` | State not persisted — explain to user and restart |
```

---

## Reglas

1. **State se persiste DESPUÉS de cada fase**: El orchestrator actualiza el state artifact después de que cada sub-agente retorna, no durante.
2. **State es un artefacto más**: Se persiste con el mismo mecanismo que el resto de los artifacts (engram, openspec, hybrid).
3. **`/continue` lee state para reanudar**: Busca el state artifact, identifica la última fase completada, y lanza la siguiente.
4. **Recovery de 2 pasos en engram**: `mem_search` devuelve previews truncados (300 chars). SIEMPRE llamar `mem_get_observation(id)` para contenido completo.
5. **Modo `none` no tiene recovery**: Si el usuario eligió `none`, el estado no se persiste. Decirle que reinicie.

## Ejemplo real (cr-orchestrator)

```markdown
## State Management (Engram)

- `code-review/{pr-number}/state`: Estado y metadata del flujo.

    - Status: `pending`, `fetching`, `analyzing`, `review_ready`, `formatted_ready`, `published`, `learning_done`
    - Debe incluir punteros a los topic keys/IDs de los artifacts (`data`, `review`, `formatted`).

- `code-review/{pr-number}/data`: Datos crudos del PR — escrito por `cr-fetcher`.
- `code-review/{pr-number}/review`: Findings estructurados — escrito por `cr-analyzer`.
- `code-review/{pr-number}/formatted`: Markdown final para GitHub — escrito por `cr-publisher`.

### State Transitions
```

pending -> fetching -> analyzing -> review_ready -> formatted_ready -> published -> learning_done

```

```

## Ejemplo real (SDD DAG, YAML state)

````markdown
## State Management

- `sdd/{change-name}/state`: YAML con estado completo del workflow.

```yaml
phase: design
artifacts:
    - name: proposal
      topic_key: sdd/auth-migration/proposal
      status: complete
    - name: spec
      topic_key: sdd/auth-migration/spec
      status: complete
    - name: design
      topic_key: sdd/auth-migration/design
      status: in_progress
tasks_progress:
    completed: []
    pending: [1.1, 1.2, 1.3, 2.1, 2.2, 3.1]
last_updated: 2025-01-15T14:30:00Z
```
````

### Recovery Rules

| Mode       | Recovery Protocol                                                          |
| ---------- | -------------------------------------------------------------------------- |
| `engram`   | `mem_search(query: "sdd/{change-name}/state")` → `mem_get_observation(id)` |
| `openspec` | Read `openspec/changes/{change-name}/state.yaml`                           |
| `hybrid`   | Try engram first, fallback to filesystem                                   |
| `none`     | State not persisted — explain to user                                      |

```

```

## Ejemplo real (cy-orchestrator, progress tracking)

```markdown
## State Management (Engram)

- `cy-testing-progress`: Progreso de testing por componente.
    - Componentes testeados con resultados
    - Fase actual del workflow

### Recovery Rule

| Mode     | Recovery                                       |
| -------- | ---------------------------------------------- |
| `engram` | `mem_search(...)` → `mem_get_observation(...)` |
| `file`   | Read `cy-testing/{component}/` directory       |
| `none`   | State not persisted — explain to user          |
```

## Anti-patrones

❌ No persistir state — sin state, `/continue` no sabe dónde reanudar
❌ Persistir state ANTES de que el sub-agente confirme — si la fase falla, el state dice que completó
❌ Incluir contenido completo en el state — el state es un índice (topic_keys + status), el contenido va en artifacts separados
❌ Usar `mem_search` sin `mem_get_observation` — los previews están truncados, el contenido completo requiere el segundo paso
❌ No declarar recovery rules para todos los modos — si el usuario usa `openspec`, necesita saber cómo recuperar
❌ State circular o sin terminal — todo state debe tener estados terminales (`completed`, `failed`, `escalated`)
