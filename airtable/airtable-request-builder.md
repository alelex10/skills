---
name: airtable-request-builder
description: Construye la estructura de la request (curl) para la API de Airtable.
version: "1.0"
archetype: transformer
triggers:
    - Cuando se tienen los IDs resueltos y se necesita preparar la ejecución.
inputs:
    required:
        - operation
        - resolved_ids
        - credentials
    optional:
        - fields
        - record_id
        - filter_formula
outputs:
    produces: request_preview
    consumer: airtable (orchestrator) | airtable-executor
---

# Role

Purpose:
Preparar la ejecución física abstrayendo la complejidad de la sintaxis de curl y la API de Airtable.

Owns:
- Construcción de URLs con query params.
- Formateo de bodies JSON para POST/PATCH.
- Inyección de headers (Authorization, Content-Type).

Avoids:
- Ejecutar la request.
- Manejar rate limits.

---

# Execution Flow

1. **Endpoint Map**: Mapear `operation` al endpoint correspondiente de Airtable.
2. **Query Params**: Si `view_id` o `filter_formula` están presentes, añadirlos a la URL (URL encoded).
3. **Body Build**: Si es create/update, envolver `fields` en `{ "fields": { ... } }`.
4. **Curl Template**: Montar el comando curl usando variables de entorno para el token.
5. **Preview Generation**: Crear una versión legible para el usuario sin secretos expuestos.
6. **Emit**: Retornar `request_preview`.

---

# Constraints

Always:
- Usar `$AIRTABLE_TOKEN` en el comando curl generado.
- Asegurar que las URLs estén correctamente encodeadas.
- Incluir `jq` en el comando para facilitar el parseo posterior.

---

# Output Contract

Artifact: request_preview

Schema:
    method: { type: string }
    url: { type: string }
    body: { type: object, optional: true }
    curl_command: { type: string }
    is_mutation: { type: boolean }
