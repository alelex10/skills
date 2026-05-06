---
name: airtable-context-loader
description: Carga configuración base y contextual (entities, views, credentials, team) desde Engram para Airtable.
version: "1.0"
archetype: retriever
triggers:
  - Cuando se inicia una operación de Airtable y se necesita contexto del proyecto.
inputs:
  required:
    - project_name
  optional:
    - operation_context (e.g. "search", "mutate", "person_name")
outputs:
  produces: airtable_context
  consumer: airtable-id-resolver | airtable (orchestrator)
---

# Role

Purpose:
Extraer de la memoria persistente toda la información necesaria para que las subsiguientes skills puedan resolver IDs y construir requests.

Owns:

- Consultas a Engram (mem_search, mem_get_observation)
- Estructuración del paquete de contexto base

Avoids:

- Resolver IDs por sí mismo
- Validar credenciales (solo las carga)

Produces:

- Un paquete JSON con: entities, views, credentials, team-mappings.

---

# Execution Flow

1. **Base Search**: Ejecutar `mem_search(query: "skill/airtable", scope: "project")` para cargar IDs de tablas y credenciales base.
2. **Personal Search**: Ejecutar `mem_search(query: "skill/airtable/views", scope: "personal")` para vistas del usuario.
3. **Contextual Enrichment**: Si `operation_context` incluye nombres de personas, buscar en `skill/airtable/team`.
4. **Aggregate**: Combinar resultados en un solo objeto `airtable_context`.
5. **Emit**: Retornar el paquete de contexto.

---

# Constraints

Always:

- Buscar por keywords en `title` y `content` de las observaciones.
- Incluir la fecha de última actualización si está disponible.

Never:

- Devolver un paquete vacío sin advertir al orquestador.
- Buscar datos de registros, tareas o resultados de queries previas en Engram. Solo cargar configuración estática (entities, views, credentials, team).
- Mezclar datos operacionales en el artefacto de contexto.

---

# Output Contract

Artifact: airtable_context

Schema:
entities: { type: array, items: { type: object } }
views: { type: array, items: { type: object } }
credentials: { type: object }
team: { type: array, items: { type: object } }
