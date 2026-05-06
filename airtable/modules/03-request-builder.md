# Phase C — Request Building

## Purpose

Construir la estructura de la request (curl command) para la API de Airtable usando los IDs resueltos y las credenciales, priorizando fórmulas de filtrado sobre vistas para búsquedas dinámicas.

---

## Execution Procedure

1. **Endpoint Mapping**: Mapear `operation` al endpoint y método HTTP correspondiente:
   - `read` → GET `/v0/{base_id}/{table_id}`
   - `create` → POST `/v0/{base_id}/{table_id}`
   - `update` → PATCH `/v0/{base_id}/{table_id}/{record_id}`
   - `delete` → DELETE `/v0/{base_id}/{table_id}/{record_id}`
2. **URL Construction**: Construir la URL base con `resolved_ids.base_id` y `resolved_ids.table_id`.
3. **Query Params & Filtering Strategy**:
   - **Search by Field**: Si se busca un registro o estado específico, usar `filterByFormula` (ej: `filterByFormula=ID=753` o `filterByFormula=AND({Responsable}='Nombre', {Status}='In progress')`).
   - **Views**: Usar `?view={view_id}` solo para reportes pre-configurados. **ADVERTENCIA**: No confiar en vistas para búsquedas de trabajo activo si estas pueden tener filtros restrictivos.
   - **Active Status Filter (Default)**: Al buscar "tareas actuales" o trabajo activo, filtrar usando `airtable_context.configs.phase_c`:
   - **Active statuses**: Usar `active_statuses` (default: `Todo`, `In progress`, `en revision`)
   - **Exclusions**: Usar `exclude_patterns` (default: `deploy`)
   - **Fórmula**: `OR({Status}='Todo', {Status}='In progress', {Status}='en revision')`
   - **Deploy exclusion**: `NOT(SEARCH('deploy', LOWER({Tareas AMIA})))`
   - Si se combina con otros filtros, envolver en `AND()`
   - **NOTA**: Solo aplicar este filtro cuando el usuario pide "tareas actuales/activas/pendientes". Para búsquedas por ID específico o consultas generales, no aplicar.
4. **Sorting**: Si se necesita ordenar resultados, usar `sort[0][field]={field_name}&sort[0][direction]=desc`. **ADVERTENCIA**: El parámetro `sort[]` solo acepta campos de usuario definidos en la tabla (ej: `ID`, `Status`, `Fecha DONE`). Los campos del sistema como `createdTime` NO funcionan con `sort[]` y causan error `UNKNOWN_FIELD_NAME`. Para obtener registros ordenados por fecha de creación, usar la vista correspondiente (que ya tiene el orden configurado) o confiar en el orden por defecto de la API (que suele ser `createdTime` descendente).
5. **Body Build**: Si la operación es `create` o `update`, envolver `fields` en `{ "fields": { ... } }`.
6. **Headers**: Definir headers estándar:
   - `Authorization: Bearer $AIRTABLE_TOKEN`
   - `Content-Type: application/json`
7. **Curl Template**: Montar el comando curl usando la variable de entorno `$AIRTABLE_TOKEN` (no el valor plano).
8. **Add jq**: Incluir `| jq '.'` al final del comando para facilitar el parseo posterior.
9. **Determine Mutation**: Marcar `is_mutation: true` para create/update/delete, `false` para read.
10. **Produce Artifact**: Generar el artefacto `request_preview`.

---

## Constraints

Always:

- Usar `$AIRTABLE_TOKEN` en el comando curl generado (nunca el valor plano).
- **Priorizar `filterByFormula`** cuando la intención sea buscar trabajo activo o registros específicos por ID.
- Asegurar que las fórmulas estén correctamente URL-encoded.
- **No usar `sort[]` con campos del sistema** (`createdTime`, `lastModifiedTime`). Solo usar campos de usuario definidos en la tabla.
- Incluir `jq` en el comando para parseo posterior.
- **Active Status Filter**: Al buscar tareas actuales/activas, filtrar por estados `Todo`, `In progress`, `en revision` (inclusión). Excluir tareas de deploy por defecto. No aplicar este filtro para búsquedas por ID o consultas generales.

Never:

- Confiar ciegamente en una `view` para buscar tareas en progreso si no se conoce su configuración de filtros.
- Exponer el token en el preview.

---

## Output Artifact

```yaml
request_preview:
  method: "GET"
  url: "https://api.airtable.com/v0/app1dRELIjLmNCLLd/tbl6eFGsNzS6PaB2m?filterByFormula=ID=753"
  body: null
  curl_command: 'curl -X GET "https://api.airtable.com/v0/app1dRELIjLmNCLLd/tbl6eFGsNzS6PaB2m?filterByFormula=ID=753" -H "Authorization: Bearer $AIRTABLE_TOKEN" -H "Content-Type: application/json" | jq ''.'''
  is_mutation: false
  execution_status: complete | failed
```

---

## Failure Modes

- **Unknown field name (sort)**: `execution_status: failed`, error: "Campo '{field}' no existe en la tabla. Solo usar campos de usuario, no campos del sistema como createdTime"
- **Invalid formula syntax**: `execution_status: failed`, error: "Sintaxis de fórmula inválida"
- **Invalid operation**: `execution_status: failed`, error: "Operación '{operation}' no soportada"
- **Missing required IDs**: `execution_status: failed`, error: "Falta {table_id|base_id} para construir URL"
- **Invalid body format**: `execution_status: failed`, error: "Formato de body inválido para operación {operation}"
