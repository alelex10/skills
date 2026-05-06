# Phase A — Context Loading

## Purpose

Cargar configuración base y contextual (entities, views, credentials, team) desde Engram para que las fases posteriores puedan resolver IDs y construir requests.

---

## Execution Procedure

1. **Search Entities**: Ejecutar `mem_search(query: "skill/airtable/entities", scope: "project")` para obtener IDs de tablas.
2. **Search Views**: Ejecutar `mem_search(query: "skill/airtable/views", scope: "project")` para obtener IDs de vistas.
3. **Search Credentials**: Ejecutar `mem_search(query: "skill/airtable/credentials", scope: "personal")` para obtener token y base_id.
4. **Search Team**: Ejecutar `mem_search(query: "skill/airtable/team", scope: "project")` para obtener mapeo de personas.
5. **Search Phase Configs**: Ejecutar `mem_search(query: "skill/airtable/config/*", scope: "project")` para obtener configuraciones por fase.
6. **Get Full Observations**: Para cada resultado, llamar `mem_get_observation(id)` para obtener el contenido completo (no usar previews truncados).
7. **Parse Content**: Extraer los datos estructurados de cada observación según el formato aprendido.
8. **Validate**: Verificar que se obtuvieron al menos entities y credentials.
9. **Produce Artifact**: Generar el artefacto `airtable_context` con todos los datos.

---

## Constraints

Always:

- Buscar por topic keys específicos, no por keywords genéricas.
- Obtener el contenido completo de cada observación antes de usarlo.
- Si falta alguna categoría, reportar error con `execution_status: failed`.

Never:

- Usar datos de previews truncados.
- Proceder sin credentials válidas.

---

## Output Artifact

```yaml
airtable_context:
  entities:
    - { name: "Tareas AMIA", id: "tbl6eFGsNzS6PaB2m" }
    - { name: "Formar Web", id: "tbliBTcxSBQXntm2b" }
  views:
    - { name: "Vista Alex", id: "viws8uOr7j4zz3hZG" }
    - { name: "Todas las tareas", id: "viwnRb2O58AxPANzb" }
  credentials:
    token: "patwR2DIpQzgSYQ9q..."
    base_id: "app1dRELIjLmNCLLd"
  team:
    - { name: "Alex Pumari", id: "recUCXUc1vwsrvth0" }
    - { name: "Valeria Medina", id: "recXYZ..." }
  configs:
    phase_c:
      active_statuses: ["Todo", "In progress", "en revision"]
      exclude_patterns: ["deploy"]
  execution_status: complete | failed
```

---

## Failure Modes

- **No entities found**: `execution_status: failed`, error: "No se encontraron entities en skill/airtable/entities"
- **No credentials found**: `execution_status: failed`, error: "No se encontraron credentials en skill/airtable/credentials"
- **Invalid observation format**: `execution_status: failed`, error: "Formato de observación inválido"
