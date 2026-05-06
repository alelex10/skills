# Phase D — Preview & Confirm

## Purpose

Presentar un resumen legible de la operación al usuario sin exponer secretos. Si es una mutación, exigir confirmación explícita antes de continuar.

---

## Execution Procedure

1. **Build Summary**: Crear un resumen legible de la operación usando `request_preview`:
   - Operación (read/create/update/delete)
   - Tabla/vista objetivo
   - Campos a modificar (si aplica)
   - Filtros aplicados (si aplica)
2. **Sanitize Secrets**: Asegurar que el token no aparezca en el resumen (usar `$AIRTABLE_TOKEN` o `[REDACTED]`).
3. **Determine Confirmation Need**: Si `request_preview.is_mutation = true`, marcar que se requiere confirmación.
4. **Present Preview**: Mostrar el resumen al usuario en formato claro.
5. **Request Confirmation**: Si es mutación, preguntar explícitamente: "¿Confirmar esta operación? (sí/no)".
6. **Capture Response**: Esperar respuesta del usuario. Si es mutación y el usuario responde negativamente, fallar con `execution_status: failed`.
7. **Produce Artifact**: Generar el artefacto `preview_result` con el estado de confirmación.

---

## Constraints

Always:
- Nunca exponer el token en el preview.
- Para mutaciones, obtener confirmación explícita antes de continuar.
- Si el usuario rechaza, fallar con `execution_status: failed` y detener el pipeline.

Never:
- Continuar con una mutación sin confirmación.
- Asumir confirmación implícita.

---

## Output Artifact

```yaml
preview_result:
  operation_summary: "Leer registros de Tareas AMIA usando Vista Alex"
  is_mutation: false
  user_confirmed: true
  execution_status: complete | failed
```

---

## Failure Modes

- **User rejected mutation**: `execution_status: failed`, error: "Usuario rechazó la operación"
- **Invalid confirmation response**: `execution_status: failed`, error: "Respuesta de confirmación inválida"
