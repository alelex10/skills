# Orchestrator Pattern: DAG / Dependency Graph

> **Invariante** — Obligatorio en todo orchestrator multi-fase.

El DAG define el orden de ejecución con dependencias requeridas vs. opcionales. Es la **columna vertebral** del orchestrator — sin DAG, no hay `/continue` ni `/ff`.

Incluye el **Reference-Passing Contract**: el orchestrator pasa referencias (topic keys, file paths), NO contenido inline. Cada sub-agente hace 2-step retrieval por su cuenta.

---

## Template

```markdown
## Dependency Graph

```
[phase-a] -> [phase-b] -> [phase-c] -> [phase-d]
                 \                    /
                  -> [phase-e] ------
```

### Dependencies by Phase

| Phase | Required Dependencies | Optional Dependencies | Writes Artifact |
|-------|----------------------|----------------------|-----------------|
| `[phase-a]` | None | None | `[domain]/[context]/[artifact-a]` |
| `[phase-b]` | `[artifact-a]` | None | `[domain]/[context]/[artifact-b]` |
| `[phase-c]` | `[artifact-b]` | `[artifact-a]` | `[domain]/[context]/[artifact-c]` |
| `[phase-d]` | `[artifact-b]`, `[artifact-c]` | `[artifact-a]` | `[domain]/[context]/[artifact-d]` |
| `[phase-e]` | `[artifact-d]` | All previous | `[domain]/[context]/[artifact-e]` |

### Resolution Rules

1. A phase can ONLY execute if ALL required dependencies are satisfied (artifacts exist in backend)
2. Optional dependencies are read if present, skipped if absent — they do NOT block execution
3. A phase writes exactly ONE primary artifact to the backend
4. After a phase completes, the orchestrator updates state and checks DAG for next phase

### Reference-Passing Contract

The orchestrator passes **references** (topic keys, file paths), NOT the full content of artifacts. Sub-agents retrieve content themselves.

**Why**: Injecting a full PR diff or design document into a sub-agent prompt can be 50K+ tokens. Under compaction, that content is lost. With reference-passing, sub-agents read only what they need, when they need it.

**Orchestrator prompt**:
```
Read artifact from: sdd/auth-migration/proposal
Pass as reference: topic_key `sdd/auth-migration/proposal`
DO NOT paste the full proposal content in the prompt.
```

**Sub-agent retrieval** (2-step protocol):
1. `mem_search(query: "{topic_key}", project: "{project}")` → get observation ID
2. `mem_get_observation(id: {id})` → full content (REQUIRED — search results are truncated)

**Artifact verification**: Before launching the next phase, the orchestrator verifies the previous artifact exists in the backend. If missing → inform user, do not proceed.
```

---

## Reglas

1. **DAG es inmutable**: El grafo no cambia en runtime. Las fases y dependencias se definen una vez en la skill.
2. **Required vs. Optional es explícito**: Cada fase declara qué artifacts necesita obligatoriamente y cuáles son nice-to-have. Si un required falta, la fase NO se ejecuta.
3. **Un artifact por fase**: Cada fase produce exactamente un artifact primary. El orchestrator verifica que existe antes de avanzar.
4. **`/continue` usa el DAG**: Busca el primer artifact faltante cuyas dependencias requeridas estén satisfechas y lanza esa fase.
5. **`/ff` sigue el DAG en secuencia**: Ejecuta fases en orden topológico hasta completar o hasta que una falle.
6. **Reference-Passing es obligatorio**: El orchestrator pasa topic keys/paths, no contenido. Los sub-agentes hacen 2-step retrieval.
7. **Artifact verification antes de avanzar**: Antes de lanzar la siguiente fase, verificar que el artifact de la fase anterior existe en el backend.

## Ejemplo real (SDD DAG)

```markdown
## Dependency Graph

```
explore -> propose -> spec -> design -> tasks -> apply -> verify -> archive
                       ^                   ^
                       |                   |
                     design -------------+
```

### Dependencies by Phase

| Phase | Required Dependencies | Optional Dependencies | Writes Artifact |
|-------|----------------------|----------------------|-----------------|
| `sdd-explore` | None | None | `sdd/{change}/explore` |
| `sdd-propose` | None | `explore` | `sdd/{change}/proposal` |
| `sdd-spec` | `proposal` | `explore` | `sdd/{change}/spec` |
| `sdd-design` | `proposal` | `spec` | `sdd/{change}/design` |
| `sdd-tasks` | `spec`, `design` | `proposal` | `sdd/{change}/tasks` |
| `sdd-apply` | `tasks`, `spec`, `design` | `proposal` | `sdd/{change}/apply-progress` |
| `sdd-verify` | `spec`, `tasks` | `design`, `apply-progress` | `sdd/{change}/verify-report` |
| `sdd-archive` | All previous | None | `sdd/{change}/archive-report` |

### Resolution Rules

1. A phase can ONLY execute if ALL required dependencies are satisfied
2. Optional dependencies are read if present, skipped if absent
3. A phase writes exactly ONE primary artifact
4. After a phase completes, the orchestrator updates state and checks DAG for next phase
```

## Anti-patrones

❌ Definir el DAG como lista de fases sin tabla de dependencias — sin declared dependencies, `/continue` no puede verificar qué puede ejecutarse
❌ Marcar todo como "required" — si una fase puede trabajar sin un artifact, declararlo optional
❌ No declarar qué artifact escribe cada fase — el orchestrator necesita saber qué verificar después de cada fase
❌ Inyectar contenido de artifacts en el prompt del sub-agente — pasar referencias (topic keys), no payloads
❌ DAG circular — por definición, un grafo de dependencias debe ser acíclico
❌ Fases que escriben más de un artifact primary — si una fase produce dos cosas, son dos fases