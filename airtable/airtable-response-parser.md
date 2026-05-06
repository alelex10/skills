---
name: airtable-response-parser
description: Procesa la respuesta cruda de la API de Airtable y la convierte en un artefacto estructurado.
version: "1.0"
archetype: transformer
triggers:
  - Cuando se recibe una respuesta de airtable-executor.
inputs:
  required:
    - raw_response
outputs:
  produces: airtable_result
  consumer: airtable (orchestrator)
---

# Role

Purpose:
Transformar la respuesta técnica de la API en un formato útil para el usuario o para otras skills, normalizando errores.

Owns:

- Parsing de JSON.
- Mapeo de errores HTTP a códigos de dominio.
- Extracción de metadata (records, offsets).

Avoids:

- Reintentar requests (ya hecho por el executor).

---

# Execution Flow

1. **Parse JSON**: Intentar convertir `raw_response.body` a objeto.
2. **Success Check**: Si HTTP 2xx, extraer datos relevantes según la operación original.
3. **Error Map**: Si HTTP 4xx/5xx o JSON inválido, generar objeto de error siguiendo el schema.
4. **Normalization**: Asegurar que los campos sigan el formato esperado (e.g. fechas, IDs).
5. **Emit**: Retornar `airtable_result`.

---

# Constraints

Always:

- Devolver un objeto válido incluso en caso de error catastrófico.
- Incluir `recoverable: true/false` en los errores.

Never:

- Guardar airtable_result o datos de registros operacionales en Engram. Los datos de Airtable cambian constantemente y se vuelven stale.
- Persistir estados de tareas o campos de negocio en la memoria del proyecto.

---

# Output Contract

Artifact: airtable_result

Schema:
success: { type: object, optional: true }
error: { type: object, optional: true }
meta: { type: object }
