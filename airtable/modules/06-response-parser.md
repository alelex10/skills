# Phase F — Response Parsing

## Purpose

Procesar la respuesta cruda de la API de Airtable y convertirla en un artefacto estructurado `airtable_result`, normalizando errores y extrayendo datos relevantes.

---

## Execution Procedure

1. **Parse JSON**: Intentar convertir `raw_response.body` a objeto JSON. Si falla, generar error de parsing.
2. **Success Check**: Si `raw_response.http_code` está en rango 2xx:
   - Extraer datos relevantes según la operación original:
     - `read`: extraer `records`, `offset` si existe
     - `create/update/delete`: extraer el record creado/modificado
   - Normalizar formatos (fechas, IDs)
   - Generar objeto `success` con `data` y `meta`
3. **Error Mapping**: Si `raw_response.http_code` está en rango 4xx/5xx o JSON inválido:
   - Mapear código HTTP a código de dominio:
     - 400 → `INVALID_REQUEST`
     - 401 → `AUTH_ERROR`
     - 403 → `FORBIDDEN`
     - 404 → `NOT_FOUND`
     - 422 → `UNPROCESSABLE_ENTITY`
     - 429 → `RATE_LIMIT`
     - 5xx → `SERVER_ERROR`
   - Extraer `reason` del body de error si está disponible
   - Determinar `recoverable`: true para 429, 5xx; false para 4xx (excepto 429)
4. **Validation**: Asegurar que el resultado final siga el schema de `airtable_result`.
5. **Produce Artifact**: Generar el artefacto `airtable_result`.

---

## Constraints

Always:
- Devolver un objeto válido incluso en caso de error catastrófico.
- Incluir `recoverable: true/false` en los errores.
- Normalizar formatos de datos (fechas ISO, IDs con prefijos correctos).

Never:
- Reintentar requests (ya hecho por el executor).
- Devolver null o undefined en el artefacto final.

---

## Output Artifact

```yaml
airtable_result:
  success:
    data:
      records:
        - { id: "recXXX", fields: { ... } }
      offset: "recYYY"
    meta:
      http_code: 200
      count: 10
  error: null
  execution_status: complete | failed
```

O en caso de error:

```yaml
airtable_result:
  success: null
  error:
    code: "AUTH_ERROR"
    reason: "Invalid authentication token"
    recoverable: false
  execution_status: complete | failed
```

---

## Failure Modes

- **Invalid JSON**: `execution_status: failed`, error: "JSON inválido en respuesta"
- **Unexpected structure**: `execution_status: failed`, error: "Estructura de respuesta inesperada"
- **Missing required fields**: `execution_status: failed`, error: "Faltan campos requeridos en respuesta"
