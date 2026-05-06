# Phase E — Execution

## Purpose

Ejecutar físicamente la request contra la API de Airtable, gestionando rate limits y reintentos básicos.

---

## Execution Procedure

1. **Token Pre-check**: Verificar si `AIRTABLE_TOKEN` está configurado como variable de entorno (`echo $AIRTABLE_TOKEN`). Si está vacío, exportarlo usando el valor de `airtable_context.credentials.token`: `export AIRTABLE_TOKEN="{token_value}"`. Esto evita errores 401 por token faltante.
2. **Rate Limit Check**: Verificar si se ha excedido el límite de 5 requests por segundo. Si es así, esperar hasta que pase el límite.
3. **Execute Command**: Ejecutar el `curl_command` del artefacto `request_preview`.
4. **Capture Output**: Capturar stdout, stderr y exit code del comando.
5. **Inspect HTTP Code**: Extraer el código de respuesta HTTP del output (si está disponible).
6. **Handle Errors**:
   - Si exit code ≠ 0: marcar error y continuar a parsing.
   - Si HTTP 429 (Rate Limit): esperar 30 segundos y reintentar una vez.
   - Si HTTP 401 (Auth Error): no reintentar, fallar con error de credenciales.
   - Si HTTP 5xx (Server Error): reintentar una vez.
7. **Limit Retries**: No reintentar más de una vez por request.
8. **Produce Artifact**: Generar el artefacto `raw_response` con el body, http_code y error si lo hay.

---

## Constraints

Always:

- **Exportar `AIRTABLE_TOKEN` antes de ejecutar curl** si la variable de entorno no está configurada. Usar el valor de `airtable_context.credentials.token`.
- Respetar el límite de 5 requests por segundo de Airtable.
- Capturar tanto el body como el código HTTP.
- Reintentar con backoff solo para 429 y 5xx (máximo 1 reintento).

Never:

- Reintentar indefinidamente.
- Silenciar errores de red.
- Reintentar errores de autenticación (401).

---

## Output Artifact

```yaml
raw_response:
  body: '{"records": [...], "offset": "..."}'
  http_code: 200
  error: null
  execution_status: complete | failed
```

---

## Failure Modes

- **Rate limit exceeded**: Esperar 30s, reintentar una vez. Si falla nuevamente, `execution_status: failed`, error: "Rate limit persistente después de reintento"
- **Auth error (401)**: `execution_status: failed`, error: "Error de autenticación: token inválido o expirado"
- **Server error (5xx)**: Reintentar una vez. Si falla, `execution_status: failed`, error: "Error del servidor persistente"
- **Network error**: `execution_status: failed`, error: "Error de red: {stderr}"
