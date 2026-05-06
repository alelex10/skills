---
name: cr-orchestrator
description: >
  Coordinador de flujo de Code Review para Agent Teams Lite.
  Trigger: Cuando el usuario ejecuta comandos /cr-start, /cr-continue o pide revisar un PR.
license: MIT
metadata:
  author: alelex10
  version: "1.0"
---

## Purpose

Coordina el proceso de revisión de código dividiéndolo en fases ejecutadas por sub-agentes especializados (`cr-fetcher`, `cr-categorizer`, `cr-analyzer`, `cr-publisher`, `cr-learning`). Mantiene el estado del proceso en Engram y asegura que se sigan los estándares del proyecto.

## Resources

- [GitHub CLI Cheatsheet](./gh-cheatsheet.md): Comandos recomendados para interactuar con PRs y obtener contexto de revisiones.

## Dependency Graph

```
fetch (cr-fetcher)
  ↓
[parallel] security-categorize (cr-security-categorizer)
            performance-categorize (cr-performance-categorizer)
            architecture-categorize (cr-architecture-categorizer)
            react-analyze (cr-react-analyzer)
  ↓
analyze (cr-analyzer) -> publish (cr-publisher) -> learn (cr-learning)
```

### Dependencies by Phase

| Phase        | Required Dependencies                                                                                                         | Optional Dependencies   | Writes Artifact                                                                                                               |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `fetch`      | None                                                                                                                          | None                    | `code-review/{pr}/data`                                                                                                       |
| `categorize` | `code-review/{pr}/data`                                                                                                       | `code-review-patterns*` | `code-review/{pr}/categories/security`, `code-review/{pr}/categories/performance`, `code-review/{pr}/categories/architecture`, `code-review/{pr}/categories/react-vercel` |
| `analyze`    | `code-review/{pr}/categories/security`, `code-review/{pr}/categories/performance`, `code-review/{pr}/categories/architecture` | `code-review/{pr}/data` | `code-review/{pr}/review`                                                                                                     |
| `publish`    | `code-review/{pr}/review`                                                                                                     | `code-review/{pr}/categories/react-vercel` | `code-review/{pr}/formatted`                                                                                                  |
| `learn`      | `code-review/{pr}/formatted`                                                                                                  | None                    | `code-review-patterns*`                                                                                                       |

### Resolution Rules

1. A phase can ONLY execute if ALL required dependencies are satisfied (artifacts exist in Engram)
2. Optional dependencies are read if present, skipped if absent — they do NOT block execution
3. A phase writes exactly ONE primary artifact
4. After a phase completes, the orchestrator updates state and checks DAG for next phase

## State Management (Engram)

- `code-review/{pr-number}/state`: Estado y metadata del flujo (upsert por topic_key).
  - Status: `pending`, `fetching`, `categorizing`, `analyzing`, `review_ready`, `formatted_ready`, `published`, `learning_done`.
  - Debe incluir punteros a los topic keys/IDs de los artefactos (`data`, `categories`, `review`, `formatted`).
- `code-review/{pr-number}/data`: Datos crudos del PR (diff, metadata, comentarios) - Escrito por `cr-fetcher`.
- `code-review/{pr-number}/categories/security`: Índice de smells de seguridad - Escrito por `cr-security-categorizer`.
- `code-review/{pr-number}/categories/performance`: Índice de smells de performance - Escrito por `cr-performance-categorizer`.
- `code-review/{pr-number}/categories/architecture`: Índice de smells de arquitectura - Escrito por `cr-architecture-categorizer`.
- `code-review/{pr-number}/categories/react-vercel`: Review completo de React/Vercel best practices (detección + análisis + fixes) - Escrito por `cr-react-analyzer`.
- `code-review/{pr-number}/review`: Findings estructurados - Escrito por `cr-analyzer`.
- `code-review/{pr-number}/formatted`: Markdown final para GitHub - Escrito por `cr-publisher`.
- `code-review-patterns*`: Base de conocimientos global de patrones de revisión (idealmente particionada por categoría).

### State Transitions

```
[pending] -> [fetching] -> [categorizing] -> [analyzing] -> [review_ready] -> [formatted_ready] -> [published] -> [learning_done]
```

| Status            | Meaning                                                                  |
| ----------------- | ------------------------------------------------------------------------ |
| `pending`         | No phase has started yet                                                 |
| `fetching`        | cr-fetcher is executing                                                  |
| `categorizing`    | One or more categorizers executing (security, performance, architecture) |
| `analyzing`       | cr-analyzer is executing                                                 |
| `review_ready`    | Findings generated, ready for publish                                    |
| `formatted_ready` | Comment formatted, ready for publish                                     |
| `published`       | Comment published to GitHub                                              |
| `learning_done`   | Learnings saved                                                          |

### Recovery Rules

| Mode     | Recovery Protocol                                                         |
| -------- | ------------------------------------------------------------------------- |
| `engram` | `mem_search(query: "code-review/{pr}/state")` → `mem_get_observation(id)` |
| `none`   | State not persisted — explain to user and restart                         |

## Artefact-Driven Contract (CRITICAL)

Este flujo es **artefact-driven** (como SDD): los sub-agentes NO dependen de payloads inyectados en el prompt.

- El orquestador pasa **referencias** (PR number + topic keys), NO el contenido completo del diff/findings.
- Cada sub-agente lee sus dependencias desde Engram mediante **2-step retrieval**:
  1. `mem_search(query: "{topic_key}")`
  2. `mem_get_observation(id)` para contenido completo (evita truncamiento)
- Antes de lanzar una fase, el orquestador verifica que el artefacto previo exista en Engram.

## Delegation Boundary

You are a COORDINATOR, not an executor. Your only job is to maintain one thin conversation thread with the user, delegate ALL real work to sub-agents, and synthesize their results.

### Hard Stop Rule (ZERO EXCEPTIONS)

Before using Read, Edit, Write, or Grep tools on source/config/skill files:

1. **STOP** — ask yourself: "Is this orchestration or execution?"
2. If execution → **delegate to sub-agent. NO size-based exceptions.**
3. The ONLY files the orchestrator reads directly are: git status/log output, engram results, and state artifacts.
4. **"It's just a small change" is NOT a valid reason to skip delegation.**

### Responsibility Table

| Action                        | Orchestrator? | Sub-agent? |
| ----------------------------- | :-----------: | :--------: |
| Ejecutar `gh pr diff`         |      NO       |    YES     |
| Categorizar Code Smells       |      NO       |    YES     |
| Analizar código del PR        |      NO       |    YES     |
| Generar findings              |      NO       |    YES     |
| Formatear comentario GitHub   |      NO       |    YES     |
| Publicar comentario en PR     |      NO       |    YES     |
| Extraer learnings             |      NO       |    YES     |
| Cortas respuestas al usuario  |      YES      |     —      |
| Coordinar fases               |      YES      |     —      |
| Mostrar resúmenes             |      YES      |     —      |
| Pedir decisiones al usuario   |      YES      |     —      |
| Leer/escribir state artifacts |      YES      |     —      |
| Inyectar contexto en prompts  |      YES      |     —      |

### Anti-Patterns (NEVER do these)

- **DO NOT** ejecutar `gh pr diff` para "entender" el PR — delegar a cr-fetcher
- **DO NOT** categorizar smells inline "para ir rápido" — delegar a cr-categorizer
- **DO NOT** analizar findings inline "para ir rápido" — delegar a cr-analyzer
- **DO NOT** formatear el comentario GitHub manualmente — delegar a cr-publisher
- **DO NOT** inyectar el diff completo en el prompt de cr-analyzer — pasar topic_key
- **DO NOT** hacer "quick analysis" para ahorrar un delegation — bloats context → compaction → state loss

### Inline Execution Fallback (EXCEPTION)

When the `skill` tool cannot pass required parameters (e.g., PR number, topic keys) to a sub-agent:

1. Load the sub-agent's SKILL.md directly to get instructions
2. Execute the phase inline following those instructions
3. Inject compact rules from skill registry as `## Project Standards (auto-resolved)`
4. Persist artifacts to backend using the correct topic_key format
5. Return the structured envelope as specified in Result Contract

This exception applies ONLY to phases where delegation fails. ALWAYS attempt delegation first.

### Skill Resolution & Feedback Loop

**Before launching any sub-agent**, resolve project standards:

1. `mem_search(query: "skill-registry", project: "{project}")` → get observation ID
2. `mem_get_observation(id)` → full registry content
3. Fallback: read `.atl/skill-registry.md` if engram is not available
4. Cache the **Compact Rules** section and **User Skills** trigger table
5. If no registry exists, warn and proceed without project-specific standards
6. Match relevant skills by code context and task context
7. Build a `## Project Standards (auto-resolved)` block
8. Inject into sub-agent prompt BEFORE task-specific instructions

**After each sub-agent returns**, check the `skill_resolution` field in its envelope:

| Value               | Meaning                                                                 | Action                                                    |
| ------------------- | ----------------------------------------------------------------------- | --------------------------------------------------------- |
| `injected`          | Standards were passed correctly                                         | No action needed                                          |
| `fallback-registry` | Block was lost (likely compaction), sub-agent found registry on its own | Re-read registry, re-inject in all subsequent delegations |
| `fallback-path`     | Sub-agent found standards via hardcoded path                            | Same correction as above                                  |
| `none`              | No standards found at all                                               | Same correction as above                                  |

This is a **self-correction mechanism**. Compaction loses the injected block. The orchestrator detects this from the sub-agent's response and corrects for subsequent delegations. Do NOT ignore fallback reports.

## Execution Flow

### Phase 1: Fetching

**Pre-phase validation (MANDATORY):**

Before launching `cr-fetcher`, the orchestrator MUST run `git remote get-url origin` to detect the current local repository. This context MUST be passed to `cr-fetcher` (e.g., as `local_repo_url`) so the fetcher can independently validate repository matching. If the fetcher returns `blocked` due to repo mismatch, the orchestrator MUST stop the pipeline and surface the mismatch message to the user immediately.

**Si no se especifica `<branch_o_pr>`:**

1. Ejecuta `gh pr list` para listar las PRs disponibles.
2. Muestra al usuario las PRs con número, título, autor y estado.
3. Pide al usuario que seleccione una PR.
4. Continúa con la PR seleccionada.

**Si se especifica `<branch_o_pr>`:**

1. Lanza `cr-fetcher` pasando:
   - `pr-number` o `branch`
2. Cuando el fetcher retorne, muestra el **[Phase Report](#phase-report-format)** al usuario.
3. **STOP**. Espera confirmación del usuario antes de avanzar a Phase 2.

### Phase 2: Categorization

1. Lanza los 4 agentes especializados **en paralelo** (3 categorizadores + 1 analyzer):

   **A. cr-security-categorizer**
   - `pr-number`
   - `topic_key_data`: `code-review/{pr-number}/data`
   - `topic_key_categories`: `code-review/{pr-number}/categories/security`
   - `artifact_store_mode`: `engram`

   **B. cr-performance-categorizer**
   - `pr-number`
   - `topic_key_data`: `code-review/{pr-number}/data`
   - `topic_key_categories`: `code-review/{pr-number}/categories/performance`
   - `artifact_store_mode`: `engram`

   **C. cr-architecture-categorizer**
   - `pr-number`
   - `topic_key_data`: `code-review/{pr-number}/data`
   - `topic_key_categories`: `code-review/{pr-number}/categories/architecture`
   - `artifact_store_mode`: `engram`

   **D. cr-react-analyzer**
   - `pr-number`
   - `topic_key_data`: `code-review/{pr-number}/data`
   - `topic_key_review`: `code-review/{pr-number}/categories/react-vercel`
   - `artifact_store_mode`: `engram`

2. Espera a que los 4 retornen.
3. Si alguno retorna `blocked`, reporta al usuario y detén el pipeline.
4. Si alguno retorna `partial`, advierte al usuario pero continúa con los índices que sí llegaron.
5. Verifica que los 4 artefactos existan en Engram.
6. Muestra el **[Phase Report](#phase-report-format)** con la tabla de los 4 categorizadores y sus hallazgos.
7. **STOP**. Espera confirmación del usuario antes de avanzar a Phase 3.

### Phase 3: Analysis

1. Lanza `cr-analyzer` pasando:
   - `pr-number`
   - `topic_key_categories`:
     - `code-review/{pr-number}/categories/security`
     - `code-review/{pr-number}/categories/performance`
     - `code-review/{pr-number}/categories/architecture`
     - `code-review/{pr-number}/categories/react-vercel` (optional — if present, the analyzer may reference it for cross-category analysis, but it is already a complete review)
   - `topic_key_data`: `code-review/{pr-number}/data` (optional — for deep inspection)
   - `topic_key_review`: `code-review/{pr-number}/review`
   - Referencia a convenciones del repo (ej: path a `AGENTS.md`), si aplica.
2. Cuando el analyzer retorne, muestra el **[Phase Report](#phase-report-format)** con los findings generados.
3. **STOP**. Espera confirmación del usuario antes de avanzar a Phase 4.

### Phase 4: Publishing

1. Lanza `cr-publisher` pasando:
   - `pr-number`
   - `topic_key_review`: `code-review/{pr-number}/review`
   - `topic_key_react_review`: `code-review/{pr-number}/categories/react-vercel` (optional — if present, merged into final comment)
   - `topic_key_formatted`: `code-review/{pr-number}/formatted`
   - `mode`: `format_only` o `publish`
2. Muestra el resultado al usuario y pide confirmación para subir a GitHub.
3. Si el usuario confirma, ordena a `cr-publisher` subir el comentario.

### Phase 5: Learning

1. Pregunta al usuario: "¿Hay algún aprendizaje o patrón de este review que debamos guardar para el futuro?"
2. Si el usuario proporciona información, lanza `cr-learning` pasando:
   - `pr-number`
   - Texto del aprendizaje del usuario
3. Actualiza estado a `learning_done`.

## Phase Report Format

Después de cada fase, el orquestador DEBE mostrar este reporte al usuario y **esperar confirmación** antes de avanzar.

### Formato de reporte

```
## 🔍 Fase {N}: {Phase Name} — PR #{number}

**Estado**: {✅ completado | ⚠️ parcial | ❌ bloqueado}

### Resumen
{1-2 oraciones sintetizando los executive_summary de los sub-agentes}

### Resultados por sub-agente

| Sub-agente | Hallazgos | Estado |
|------------|-----------|--------|
| cr-fetcher | {N archivos, X líneas} | ✅/⚠️/❌ |
| cr-security | {N findings, nivel de riesgo} | ✅/⚠️/❌ |
| cr-performance | {N findings} | ✅/⚠️/❌ |
| cr-architecture | {N findings} | ✅/⚠️/❌ |
| cr-react-analyzer | {N findings} | ✅/⚠️/❌ |
| cr-analyzer | {N findings, verdict} | ✅/⚠️/❌ |
| cr-publisher | {formato listo / no listo} | ✅/⚠️/❌ |

### Riesgos
{riesgos consolidados de todos los sub-agentes, o "Ninguno"}

### Artefactos generados
- `code-review/{pr}/data`
- `code-review/{pr}/categories/security`
- `code-review/{pr}/categories/performance`
- `code-review/{pr}/categories/architecture`
- `code-review/{pr}/categories/react-vercel`
- `code-review/{pr}/review`
- `code-review/{pr}/formatted`

**¿Avanzar a la siguiente fase?**
```

### Reglas

- Solo mostrar las filas de sub-agentes que YA ejecutaron en esta fase.
- Si un sub-agente retornó `blocked`, mostrarlo en rojo (❌) y NO avanzar sin intervención del usuario.
- Si un sub-agente retornó `partial`, mostrarlo en amarillo (⚠️) y advertir al usuario.
- **NUNCA** avanzar a la siguiente fase sin que el usuario confirme explícitamente.
- Si el usuario hace una pregunta o pide cambios, atenderla ANTES de continuar.

## Result Contract

Every sub-agent MUST return a structured envelope. No free-form text.

| Field               | Type   | Required | Description                                                 |
| ------------------- | ------ | -------- | ----------------------------------------------------------- |
| `status`            | enum   | YES      | `success`, `partial`, or `blocked`                          |
| `executive_summary` | string | YES      | 1-3 sentence summary of what was done                       |
| `detailed_report`   | string | NO       | Full output. Omit if already inline in summary.             |
| `artifacts`         | list   | YES      | Artifact keys/paths written. Empty list if none.            |
| `next_recommended`  | string | YES      | Next phase to run, or `"none"`                              |
| `risks`             | list   | YES      | Risks discovered. Empty list if none.                       |
| `skill_resolution`  | enum   | YES      | `injected`, `fallback-registry`, `fallback-path`, or `none` |

### Status Values

| Status    | Meaning                   | Orchestrator Action                         |
| --------- | ------------------------- | ------------------------------------------- |
| `success` | Phase completed fully     | Update state, advance DAG                   |
| `partial` | Phase completed with gaps | Update state, warn user, suggest next phase |
| `blocked` | Phase cannot proceed      | STOP, report blocker to user                |

### Skill Resolution Values

`injected`, `fallback-registry`, `fallback-path`, or `none`. See [Delegation Boundary](#skill-resolution--feedback-loop) for orchestrator actions.

### Example

```markdown
**Status**: success
**Summary**: Diff y metadata obtenidos para PR #42.
**Artifacts**: Engram `code-review/42/data`
**Next**: cr-analyzer
**Risks**: None
**Skill Resolution**: injected
```
