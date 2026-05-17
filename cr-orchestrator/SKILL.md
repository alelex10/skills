---
name: cr-orchestrator
description: >
  Coordinador de flujo de Code Review para Agent Teams Lite — diseño generalista multi-agente.
  Pipeline: fetch → route → [6 reviewers en paralelo] → publisher/merger → learning.
  Trigger: Cuando el usuario ejecuta comandos /cr-start, /cr-continue o pide revisar un PR.
license: MIT
metadata:
  author: alelex10
  version: "2.0"
---

## Purpose

Coordina el proceso de revisión de código dividiéndolo en fases ejecutadas por sub-agentes especializados. El pipeline es **generalista por disciplina**: cada reviewer analiza desde una perspectiva distinta sin ataduras a tecnologías específicas.

> **Artifact store**: El pipeline CR usa **exclusivamente Engram** para persistencia. No soporta `openspec`, `hybrid`, ni `none`. Los sub-agentes leen y escriben artefactos vía `mem_search` / `mem_save` con topic keys `code-review/{pr}/*`. Si Engram no está disponible, el pipeline no funciona — advertir al usuario.

## Resources

- [GitHub CLI Cheatsheet](./gh-cheatsheet.md): Comandos recomendados para interactuar con PRs y obtener contexto de revisiones.

## Design Philosophy

> No necesitás más inteligencia, necesitás más perspectivas.

Cada reviewer es un especialista generalista que responde UNA pregunta clave desde su lente de análisis. El Router decide cuáles ejecutar según el cambio. El Publisher consolida, deduplica, prioriza y formatea.

## Reviewers

| Reviewer | Pregunta clave | Lente de análisis |
|----------|---------------|-------------------|
| `cr-correctness` | ¿El código funciona correctamente? | Bugs lógicos, nulls, edge cases, race conditions, manejo de errores |
| `cr-architecture` | ¿Está bien diseñado? | SRP, coupling, cohesión, modularidad, límites de dominio, deuda técnica |
| `cr-security` | ¿Puede ser explotado o abusado? | Validación de inputs, auth, inyecciones, secretos, escalación de privilegios |
| `cr-reliability` | ¿Qué pasa cuando algo falla? | Retries, timeouts, idempotencia, consistencia, concurrencia, recuperación |
| `cr-performance` | ¿Escala y rinde bien? | Complejidad, memoria, latencia, IO, caching, contención |
| `cr-quality` | ¿Es mantenible y claro? | Naming, legibilidad, complejidad, duplicación, code smells, testabilidad |

## Dependency Graph

```
fetch (cr-fetcher)
  ↓
route (cr-router)
  ↓
[parallel] correctness (cr-correctness)
           architecture (cr-architecture)
           security (cr-security)
           reliability (cr-reliability)
           performance (cr-performance)
           quality (cr-quality)
  ↓
publish (cr-publisher) → learn (cr-learning)
```

**Nota**: Solo los reviewers seleccionados por `cr-router` se ejecutan en la fase paralela.

### Dependencies by Phase

| Phase | Required Dependencies | Optional Dependencies | Writes Artifact |
|-------|----------------------|----------------------|-----------------|
| `fetch` | None | None | `code-review/{pr}/data` |
| `route` | `code-review/{pr}/data` | None | `code-review/{pr}/route` |
| `review` | `code-review/{pr}/data`, `code-review/{pr}/route` | `code-review-patterns*` | `code-review/{pr}/reviews/{discipline}` |
| `publish` | `code-review/{pr}/reviews/*` (all from route) | None | `code-review/{pr}/formatted` |
| `learn` | `code-review/{pr}/formatted` | None | `code-review-patterns*` |

### Resolution Rules

1. A phase can ONLY execute if ALL required dependencies are satisfied
2. Optional dependencies are read if present, skipped if absent — they do NOT block execution
3. Route drives which reviewers run — don't launch reviewers not in the route plan
4. Publisher receives all reviews that were generated and consolidates them

## State Management (Engram)

- `code-review/{pr-number}/state`: Estado y metadata del flujo.
  - Status: `pending`, `fetching`, `routing`, `reviewing`, `publishing`, `published`, `learning_done`
  - Debe incluir punteros a los topic keys de los artefactos.
- `code-review/{pr-number}/data`: Datos crudos del PR — Escrito por `cr-fetcher`.
- `code-review/{pr-number}/route`: Plan de revisión — Escrito por `cr-router`.
- `code-review/{pr-number}/reviews/lite`: Review generalista — Escrito por `cr-lite` (PRs chicos/medianos).
- `code-review/{pr-number}/reviews/correctness`: Review de corrección — Escrito por `cr-correctness`.
- `code-review/{pr-number}/reviews/architecture`: Review de arquitectura — Escrito por `cr-architecture`.
- `code-review/{pr-number}/reviews/security`: Review de seguridad — Escrito por `cr-security`.
- `code-review/{pr-number}/reviews/reliability`: Review de confiabilidad — Escrito por `cr-reliability`.
- `code-review/{pr-number}/reviews/performance`: Review de rendimiento — Escrito por `cr-performance`.
- `code-review/{pr-number}/reviews/quality`: Review de calidad — Escrito por `cr-quality`.
- `code-review/{pr-number}/formatted`: Reporte consolidado final — Escrito por `cr-publisher`.
- `code-review-patterns*`: Base de conocimientos global de patrones.

### State Transitions

```
[pending] → [fetching] → [routing] → [reviewing] → [publishing] → [published] → [learning_done]
```

| Status | Meaning |
|--------|---------|
| `pending` | No phase has started yet |
| `fetching` | cr-fetcher is executing |
| `routing` | cr-router is executing |
| `reviewing` | Selected reviewers executing in parallel |
| `publishing` | cr-publisher is executing |
| `published` | Comment published to GitHub |
| `learning_done` | Learnings saved |

### Recovery Rules

| Mode | Recovery Protocol |
|------|-------------------|
| `engram` | `mem_search(query: "code-review/{pr}/state")` → `mem_get_observation(id)` |
| `none` | State not persisted — explain to user and restart |

## Artefact-Driven Contract (CRITICAL)

Este flujo es **artefact-driven**: los sub-agentes leen sus dependencias desde un backend de artefactos.

### Dual Delivery Strategy

1. El orquestador persiste artefactos en Engram (para recovery entre sesiones)
2. El orquestador TAMBIÉN inyecta los datos inline en el prompt del sub-agente como fallback
3. El sub-agente intenta Engram primero (Method A); si falla, usa los datos inline (Method B)

### Route-Driven Execution

Después de que `cr-router` complete, el orquestador DEBE:

1. Leer `code-review/{pr}/route` para obtener la lista de reviewers seleccionados
2. Solo lanzar los reviewers que aparecen en el plan con prioridad REQUIRED o RECOMMENDED
3. Omitir los reviewers con prioridad OPTIONAL (a menos que el usuario lo pida)
4. Si el route dice `SKIPPED`, saltar directamente a publishing

## Delegation Boundary

You are a COORDINATOR, not an executor. Your only job is to maintain one thin conversation thread with the user, delegate ALL real work to sub-agents, and synthesize their results.

### Hard Stop Rule (ZERO EXCEPTIONS)

Before using Read, Edit, Write, or Grep tools on source/config/skill files:

1. **STOP** — ask yourself: "Is this orchestration or execution?"
2. If execution → **delegate to sub-agent. NO size-based exceptions.**
3. The ONLY files the orchestrator reads directly are: git status/log output, engram results, and state artifacts.
4. **"It's just a small change" is NOT a valid reason to skip delegation.**

### Responsibility Table

| Action | Orchestrator? | Sub-agent? |
|--------|:-----------:|:--------:|
| Ejecutar `gh pr diff` | NO | YES |
| Rutear el cambio | NO | YES (cr-router) |
| Revisar corrección | NO | YES (cr-correctness) |
| Revisar arquitectura | NO | YES (cr-architecture) |
| Revisar seguridad | NO | YES (cr-security) |
| Revisar confiabilidad | NO | YES (cr-reliability) |
| Revisar rendimiento | NO | YES (cr-performance) |
| Revisar calidad | NO | YES (cr-quality) |
| Consolidar y formatear | NO | YES (cr-publisher) |
| Publicar comentario en PR | NO | YES (cr-publisher) |
| Extraer learnings | NO | YES (cr-learning) |
| Cortas respuestas al usuario | YES | — |
| Coordinar fases | YES | — |
| Mostrar resúmenes | YES | — |
| Pedir decisiones al usuario | YES | — |
| Leer/escribir state artifacts | YES | — |
| Inyectar contexto en prompts | YES | — |

### Anti-Patterns (NEVER do these)

- **DO NOT** ejecutar `gh pr diff` para "entender" el PR — delegar a cr-fetcher
- **DO NOT** rutear el cambio inline — delegar a cr-router
- **DO NOT** hacer review inline "para ir rápido" — delegar a los reviewers
- **DO NOT** consolidar findings manualmente — delegar a cr-publisher
- **DO NOT** confiar únicamente en Engram para la entrega de datos — usar Dual Delivery

### Skill Resolution & Feedback Loop

**Before launching any sub-agent**, resolve project standards:

1. `mem_search(query: "skill-registry", project: "{project}")` → get observation ID
2. `mem_get_observation(id)` → full registry content
3. Fallback: read `.atl/skill-registry.md` if engram is not available
4. Cache the **Compact Rules** section
5. Match relevant skills by code context and task context
6. Build a `## Project Standards (auto-resolved)` block
7. Inject into sub-agent prompt BEFORE task-specific instructions

**After each sub-agent returns**, check the `Skill Resolution` field:

| Value | Meaning | Action |
|-------|---------|--------|
| `injected` | Standards were passed correctly | No action needed |
| `fallback-registry` | Block was lost, sub-agent found registry on its own | Re-read registry, re-inject in subsequent delegations |
| `fallback-path` | Sub-agent found standards via hardcoded path | Same correction as above |
| `none` | No standards found at all | Same correction as above |

## Execution Flow

### Phase 1: Fetching

**Pre-phase validation (MANDATORY):**

Before launching `cr-fetcher`, run `git remote get-url origin` to detect the current local repository. Pass this as `local_repo_url` to `cr-fetcher`.

**Si no se especifica `<branch_o_pr>`:**

1. Ejecuta `gh pr list` para listar las PRs disponibles.
2. Muestra al usuario las PRs con número, título, autor y estado.
3. Pide al usuario que seleccione una PR.
4. Continúa con la PR seleccionada.

**Si se especifica `<branch_o_pr>`:**

1. Lanza `cr-fetcher` pasando `pr-number` o `branch`.
2. Cuando el fetcher retorne, el orchestrator DEBE verificar que cr-fetcher persistió el data en Engram correctamente. Si el fetcher reportó `Artifacts: code-review/{pr}/data`, la persistencia fue exitosa. Si no aparece en Artifacts, revisar el `detailed_report` y persistir manualmente como fallback.
3. Muestra el **[Phase Report](#phase-report-format)** al usuario.
4. **STOP**. Espera confirmación antes de avanzar.

### Phase 2: Routing

1. Lanza `cr-router` pasando:
   - `pr-number`
   - `topic_key_data`: `code-review/{pr-number}/data`
   - `topic_key_route`: `code-review/{pr-number}/route`
   - `artifact_store_mode`: `engram`
   - `inline_data_diff`: `{diff content}` (fallback inline)

2. Cuando el router retorne, recoge el plan. Si incluyó el artifact en `detailed_report`, persístelo a Engram manualmente.

3. El router devuelve `Reviewers: {lista}` en su envelope. Lee qué reviewers seleccionó y sus prioridades.

4. Si el router devuelve `Next: none` (docs-only), salteá a Phase 4 (publishing) con un mensaje apropiado.

5. Muestra el **[Phase Report](#phase-report-format)** con la tabla de routing.
6. **STOP**. Espera confirmación antes de avanzar.

### Phase 3: Review (Parallel or Lite)

1. Lee `code-review/{pr}/route` para obtener la lista exacta de reviewers a ejecutar.

2. **Si el router seleccionó `cr-lite`** (PR chico/mediano):
   - Lanza SOLO `cr-lite` como único reviewer:
     - `pr-number`
     - `topic_key_data`: `code-review/{pr-number}/data`
     - `topic_key_review`: `code-review/{pr-number}/reviews/lite`
     - `artifact_store_mode`: `engram`
     - `inline_data_diff`: `{diff content}`
   - Salteá el resto de este paso (no lances especialistas).
   - **Avanzá al paso 4** (esperar resultado).

3. **Si el router seleccionó especialistas** (full review):
   - Lanza SOLO los reviewers con prioridad REQUIRED o RECOMMENDED, **en paralelo**:

   **cr-correctness** (si seleccionado):
   - `pr-number`
   - `topic_key_data`: `code-review/{pr-number}/data`
   - `topic_key_review`: `code-review/{pr-number}/reviews/correctness`
   - `artifact_store_mode`: `engram`
   - `inline_data_diff`: `{diff content}`

   **cr-architecture** (si seleccionado):
   - `pr-number`
   - `topic_key_data`: `code-review/{pr-number}/data`
   - `topic_key_review`: `code-review/{pr-number}/reviews/architecture`
   - `artifact_store_mode`: `engram`
   - `inline_data_diff`: `{diff content}`

   **cr-security** (si seleccionado):
   - `pr-number`
   - `topic_key_data`: `code-review/{pr-number}/data`
   - `topic_key_review`: `code-review/{pr-number}/reviews/security`
   - `artifact_store_mode`: `engram`
   - `inline_data_diff`: `{diff content}`

   **cr-reliability** (si seleccionado):
   - `pr-number`
   - `topic_key_data`: `code-review/{pr-number}/data`
   - `topic_key_review`: `code-review/{pr-number}/reviews/reliability`
   - `artifact_store_mode`: `engram`
   - `inline_data_diff`: `{diff content}`

   **cr-performance** (si seleccionado):
   - `pr-number`
   - `topic_key_data`: `code-review/{pr-number}/data`
   - `topic_key_review`: `code-review/{pr-number}/reviews/performance`
   - `artifact_store_mode`: `engram`
   - `inline_data_diff`: `{diff content}`

   **cr-quality** (si seleccionado):
   - `pr-number`
   - `topic_key_data`: `code-review/{pr-number}/data`
   - `topic_key_review`: `code-review/{pr-number}/reviews/quality`
   - `artifact_store_mode`: `engram`
   - `inline_data_diff`: `{diff content}`

3. Espera a que todos los lanzados retornen.
4. Si alguno retorna `blocked`, reporta al usuario y detén el pipeline.
5. Si alguno retorna `partial`, advierte al usuario pero continúa.
6. Recoge los resultados. Si algún reviewer incluyó su artifact en `detailed_report`, persistilo a Engram manualmente.

7. Muestra el **[Phase Report](#phase-report-format)** con la tabla de reviewers ejecutados y sus hallazgos.
8. **STOP**. Espera confirmación antes de avanzar.

### Phase 4: Publishing (Publisher + Merger)

1. Lanza `cr-publisher` pasando:
   - `pr-number`
   - `topic_key_reviews`:
     - **Lite path**: `code-review/{pr-number}/reviews/lite` (único, si el router eligió cr-lite)
     - **Full path**: solo los que se ejecutaron:
       - `code-review/{pr-number}/reviews/correctness` (si ejecutado)
       - `code-review/{pr-number}/reviews/architecture` (si ejecutado)
       - `code-review/{pr-number}/reviews/security` (si ejecutado)
       - `code-review/{pr-number}/reviews/reliability` (si ejecutado)
       - `code-review/{pr-number}/reviews/performance` (si ejecutado)
       - `code-review/{pr-number}/reviews/quality` (si ejecutado)
   - `topic_key_route`: `code-review/{pr-number}/route` (para contexto de clasificación)
   - `topic_key_formatted`: `code-review/{pr-number}/formatted`
   - `mode`: `format_only` o `publish`
   - `inline_reviews`: `{contenido de todos los reviews ejecutados}` (fallback inline)

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
{1-2 oraciones sintetizando los resultados}

### Resultados por sub-agente

| Sub-agente | Hallazgos | Estado |
|------------|-----------|--------|
| cr-fetcher | {N archivos, X líneas} | ✅/⚠️/❌ |
| cr-router | {N reviewers seleccionados, riesgo} | ✅/⚠️/❌ |
| cr-lite | {N findings, verdict} — solo si fue lite path | ✅/⚠️/❌ |
| cr-correctness | {N findings, verdict} — solo si fue full path | ✅/⚠️/❌ |
| cr-architecture | {N findings, verdict} — solo si fue full path | ✅/⚠️/❌ |
| cr-security | {N findings, verdict} — solo si fue full path | ✅/⚠️/❌ |
| cr-reliability | {N findings, verdict} — solo si fue full path | ✅/⚠️/❌ |
| cr-performance | {N findings, verdict} — solo si fue full path | ✅/⚠️/❌ |
| cr-quality | {N findings, verdict} — solo si fue full path | ✅/⚠️/❌ |
| cr-publisher | {formato listo / no listo} | ✅/⚠️/❌ |

### Riesgos
{riesgos consolidados, o "Ninguno"}

### Artefactos generados
- `code-review/{pr}/data`
- `code-review/{pr}/route`
- `code-review/{pr}/reviews/lite` (si fue lite path)
- `code-review/{pr}/reviews/correctness` (si ejecutado)
- `code-review/{pr}/reviews/architecture` (si ejecutado)
- `code-review/{pr}/reviews/security` (si ejecutado)
- `code-review/{pr}/reviews/reliability` (si ejecutado)
- `code-review/{pr}/reviews/performance` (si ejecutado)
- `code-review/{pr}/reviews/quality` (si ejecutado)
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

Every sub-agent MUST return a structured envelope matching `cr-common.md` Section D. No free-form text.

| Field | Format | Required | Description |
|-------|--------|----------|-------------|
| `Status` | enum | YES | `success`, `partial`, or `blocked` |
| `Summary` | string | YES | 1-3 sentence summary of what was done |
| `detailed_report` | string | NO | Full output. Used as fallback when Engram persistence fails — orchestrator persists it. Omit if already in artifact store. |
| `Artifacts` | list | YES | Artifact keys/paths written. Empty list if none. |
| `Next` | string | YES | Next phase to run, or `"none"` |
| `Risks` | list | YES | Risks discovered. Empty list if none. |
| `Skill Resolution` | enum | YES | `injected`, `fallback-registry`, `fallback-path`, or `none` |
| `Reviewers` | list | NO | (cr-router only) Comma-separated list of selected reviewer names |

### Status Values

| Status | Meaning | Orchestrator Action |
|--------|---------|---------------------|
| `success` | Phase completed fully | Update state, advance DAG |
| `partial` | Phase completed with gaps | Update state, warn user, suggest next phase |
| `blocked` | Phase cannot proceed | STOP, report blocker to user |

### Example

```markdown
**Status**: success
**Summary**: Diff y metadata obtenidos para PR #42. 12 archivos, +340/-89 líneas.
**Artifacts**: Engram `code-review/42/data`
**Next**: cr-router
**Risks**: None
**Skill Resolution**: injected
```
