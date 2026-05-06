---
name: airtable
description: >
    Orquestador para operaciones contra la API de Airtable.
    Coordina la carga de contexto, resolución de IDs, construcción de requests y ejecución.
version: "4.0"
archetype: executor
triggers:
    - Cuando se necesite listar, crear, actualizar o eliminar registros en Airtable.
inputs:
    required:
        - operation
        - table_name
    optional:
        - view_name
        - record_id
        - fields
        - filter_formula
outputs:
    produces: airtable_result
    consumer: user | downstream skill
---

# Role

Purpose:
Ejecutar el flujo completo de interacción con Airtable como pipeline secuencial determinista.

Owns:
- Ejecución secuencial de fases (Context → Resolve → Build → Preview → Execute → Parse → Emit)
- Validación de artefactos entre fases
- Gestión de gates y detención ante fallos

Avoids:
- Saltar fases del pipeline
- Ejecución parcial de módulos
- Inferir intención sin ejecutar procedimientos

Produces:
- Artefacto final `airtable_result` validado

---

# Mission

Garantizar que cada operación en Airtable siga el protocolo de seguridad, validación y persistencia de configuración del proyecto.

---

# Execution Contract

Execution is **mandatory** and **sequential**.

Referenced modules are **executable phases**, not reference material.

When a module is loaded:

1. **Resolve** resource path
2. **Read** full content (no partial reads)
3. **Execute** its procedure completely
4. **Produce** phase artifact (mandatory output)
5. **Validate** completion against artifact schema
6. **Only then** continue to next phase

**Prohibited:**
- Skip a loaded phase
- Partially execute a phase
- Jump directly to output generation
- Infer intent without executing procedure

---

# Execution Pipeline

### Phase A — Context Loading
 load `modules/01-context-loader.md`
 execute fully: buscar configuración en Engram (entities, views, credentials, team) usando topic keys skill/airtable/*
 **produce artifact:**
```yaml
airtable_context:
  entities:
    - { name: string, id: string }
  views:
    - { name: string, id: string }
  credentials:
    token: string
    base_id: string
  team:
    - { name: string, id: string }
  execution_status: complete | failed
```
 validate artifact exists and is complete
 STOP if artifact invalid

### Phase B — ID Resolution
 load `modules/02-id-resolver.md`
 execute fully: traducir nombres humanos (table_name, view_name, alias) a IDs técnicos usando airtable_context
 **produce artifact:**
```yaml
resolved_ids:
  table_id: string (pattern: ^tbl)
  view_id: string (pattern: ^viw, optional: true)
  person_id: string (pattern: ^rec, optional: true)
  base_id: string (pattern: ^app)
  execution_status: complete | failed
```
 validate artifact exists and is complete
 STOP if artifact invalid

### Phase C — Request Building
 load `modules/03-request-builder.md`
 execute fully: construir curl command con resolved_ids, credentials y operation
 **produce artifact:**
```yaml
request_preview:
  method: string
  url: string
  body: object (optional: true)
  curl_command: string
  is_mutation: boolean
  execution_status: complete | failed
```
 validate artifact exists and is complete
 STOP if artifact invalid

### Phase D — Preview & Confirm
 load `modules/04-preview-confirm.md`
 execute fully: presentar resumen legible de la operación sin exponer secretos. Si is_mutation=true, EXIGIR confirmación explícita del usuario
 **produce artifact:**
```yaml
preview_result:
  operation_summary: string
  is_mutation: boolean
  user_confirmed: boolean
  execution_status: complete | failed
```
 validate artifact exists and is complete
 STOP if artifact invalid OR if is_mutation=true and user_confirmed=false

### Phase E — Execution
 load `modules/05-executor.md`
 execute fully: ejecutar curl_command, gestionar rate limits (5 req/s), reintentos básicos (429, 5xx)
 **produce artifact:**
```yaml
raw_response:
  body: string
  http_code: integer
  error: string (optional: true)
  execution_status: complete | failed
```
 validate artifact exists and is complete
 STOP if artifact invalid

### Phase F — Response Parsing
 load `modules/06-response-parser.md`
 execute fully: parsear JSON de raw_response, mapear errores HTTP a códigos de dominio, normalizar datos
 **produce artifact:**
```yaml
airtable_result:
  success:
    data: object
    meta: object
  error:
    code: string
    reason: string
    recoverable: boolean
  execution_status: complete | failed
```
 validate artifact exists and is complete
 STOP if artifact invalid

### Phase G — Emission (Final)
 Only after all phases pass:
 emit `airtable_result` to user

---

# Constraints

Always:
- Cargar configuración desde Engram antes de cualquier resolución.
- Solicitar confirmación para operaciones destructivas/mutaciones DESPUÉS del preview.
- Usar variables de entorno (`AIRTABLE_TOKEN`, `AIRTABLE_BASE_ID`) para secretos.
- Mapear errores de la API a códigos estandarizados (schema de error).

Never:
- Hardcodear IDs o Tokens.
- Ignorar rate limits (delegado al executor).
- Continuar una mutación si el usuario rechaza el preview.
- Buscar datos operacionales (tareas, registros) en Engram antes de ejecutar el pipeline. Engram solo sirve para configuración estática (Phase A).
- Guardar resultados de queries (airtable_result) o datos operacionales en Engram. Solo persistir cambios de configuración.

---

# Persistence Policy

### SAVE to Engram:
- Nuevos table IDs, view IDs, o credentials descubiertos (Phase A/B).
- Cambios en la configuración del team (Phase A/B).
- Errores estructurales de la API o discoveries de configuración.

### DO NOT SAVE to Engram:
- Resultados de queries o búsquedas (airtable_result).
- Datos de registros individuales o campos de negocio.
- Estados de tareas (cambian constantemente en la API).

---

---

# Output Contract

Artifact: airtable_result

Schema:
    success:
        type: object
        properties:
            data: { type: object }
            meta: { type: object }
    error:
        type: object
        required: [code, reason, recoverable]
        properties:
            code: { type: string }
            reason: { type: string }
            recoverable: { type: boolean }

---

# Failure Modes

- **Missing Config**: Solicitar ID al usuario y memorizar proactivamente.
- **Invalid Mapping**: Proponer el match más cercano (fuzzy matching).
- **Rate Limit (429)**: Reintentar con backoff exponencial (gestionado por executor).
- **Auth Error (401)**: Detener ejecución y reportar problema de credenciales.
- **API Error (5xx)**: Reintentar una vez y fallar con `recoverable: true`.

---

# Resources

- **Core modules**: `modules/01-context-loader.md`, `modules/02-id-resolver.md`, `modules/03-request-builder.md`, `modules/04-preview-confirm.md`, `modules/05-executor.md`, `modules/06-response-parser.md`

---

# Examples

## Update Record Flow

1. **Input**: `{ operation: update, table_name: Tasks, record_id: rec123, fields: { Status: Done } }`
2. **Context**: Carga `skill/airtable/entities` -> mapea "Tasks" a `tblABC`.
3. **Build**: Genera PATCH a `/tblABC/rec123`.
4. **Confirm**: "Actualizar Tareas: Status -> Done en rec123. ¿Confirmar?"
5. **Result**: `{ id: rec123, fields: { Status: Done } }`

---

# Validation Checklist

- [ ] ¿Se ejecutó cada fase completamente sin saltos?
- [ ] ¿Se produjo cada artefacto con execution_status=complete?
- [ ] ¿Se validó cada gate antes de continuar a la siguiente fase?
- [ ] ¿Se detuvo la ejecución si algún artifact fue invalid?
- [ ] ¿Se usó `AIRTABLE_TOKEN` (env var) en lugar de valor plano?
- [ ] ¿Se obtuvo confirmación explícita para mutaciones?
- [ ] ¿El resultado final sigue el schema `airtable_result`?
