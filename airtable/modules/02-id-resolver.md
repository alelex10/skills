# Phase B — ID Resolution

## Purpose

Traducir nombres humanos de tablas, vistas y alias de personas a sus IDs técnicos de Airtable usando el contexto cargado en la fase anterior.

---

## Execution Procedure

1. **Resolve Table ID**: Buscar `table_name` en `airtable_context.entities`. Si no hay match exacto, intentar fuzzy matching por similitud de texto.
2. **Resolve View ID**: Si `view_name` está presente, buscarlo en `airtable_context.views`. Si no se proporciona, usar vista por defecto si existe.
3. **Resolve Person ID**: Si se proporciona un alias (ej: "mis tareas", "Alex"), buscar en `airtable_context.team` el mapeo correspondiente.
4. **Validate Format**: Verificar que cada ID resuelto tenga el prefijo correcto:
   - Tablas: `^tbl`
   - Vistas: `^viw`
   - Personas: `^rec`
   - Base: `^app`
5. **Handle Ambiguity**: Si hay múltiples matches, listar opciones al orquestador y fallar con `execution_status: failed`.
6. **Produce Artifact**: Generar el artefacto `resolved_ids` con todos los IDs resueltos.

---

## Constraints

Always:
- Priorizar match exacto sobre fuzzy matching.
- Si no se puede resolver un ID, reportar error específico.
- Validar el formato de cada ID antes de emitir el artefacto.

Never:
- Consultar Engram (usar solo el context provisto).
- Llamar a la API de Airtable para resolver IDs.

---

## Output Artifact

```yaml
resolved_ids:
  table_id: "tbl6eFGsNzS6PaB2m"
  view_id: "viws8uOr7j4zz3hZG"
  person_id: "recUCXUc1vwsrvth0"
  base_id: "app1dRELIjLmNCLLd"
  execution_status: complete | failed
```

---

## Failure Modes

- **Table not found**: `execution_status: failed`, error: "No se encontró tabla '{table_name}' en entities"
- **View not found**: `execution_status: failed`, error: "No se encontró vista '{view_name}' en views"
- **Person not found**: `execution_status: failed`, error: "No se encontró persona '{alias}' en team"
- **Invalid ID format**: `execution_status: failed`, error: "ID '{id}' no tiene formato válido"
