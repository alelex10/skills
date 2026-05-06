---
name: airtable-id-resolver
description: Traduce nombres humanos de tablas, vistas y alias de personas a sus IDs técnicos de Airtable.
version: "1.0"
archetype: transformer
triggers:
    - Cuando se tiene un nombre de tabla/vista y se necesita su ID para la API.
inputs:
    required:
        - table_name
        - airtable_context
    optional:
        - view_name
        - alias (e.g. "mis tareas")
outputs:
    produces: resolved_ids
    consumer: airtable-request-builder
---

# Role

Purpose:
Asegurar que las referencias humanas se conviertan en IDs válidos (tblXXX, viwYYY, recZZZ).

Owns:
- Mapeo de nombres a IDs basado en el contexto cargado.
- Resolución de alias (e.g. "mis tareas" -> "Vista Alejandro").

Avoids:
- Consultar la memoria (usa el context provisto).
- Llamar a la API de Airtable.

---

# Execution Flow

1. **Resolve Table**: Buscar `table_name` en `airtable_context.entities`.
2. **Resolve View**: Si `view_name` presente, buscar en `airtable_context.views`.
3. **Alias Handling**: Si el input es "mis tareas", buscar en `airtable_context.team` el mapeo correspondiente al usuario actual.
4. **Fuzzy Match**: Si no hay match exacto, buscar el más cercano por texto.
5. **Validation**: Verificar que el ID resuelto tenga el formato correcto (e.g. `tbl` para tablas).
6. **Emit**: Retornar `resolved_ids`.

---

# Constraints

Always:
- Priorizar match exacto.
- Si hay ambigüedad, listar opciones al orquestador.
- Resolver "mis tareas" usando el mapeo de `team` si está disponible.

---

# Output Contract

Artifact: resolved_ids

Schema:
    table_id: { type: string, pattern: "^tbl" }
    view_id: { type: string, pattern: "^viw", optional: true }
    base_id: { type: string, pattern: "^app" }
