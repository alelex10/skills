---
name: ligaficba-api-test
description: >
  Pruebas de peticiones HTTP contra la API local de liga-ficba.
  Trigger: Cuando necesites probar un endpoint, hacer un curl, verificar una respuesta, o testear la API.
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

## When to Use

- Probar un endpoint nuevo después de implementarlo
- Verificar que un caso de uso responde correctamente
- Debuggear errores de API
- Testear permisos y autenticación
- Validar respuestas y errores

## Critical Patterns

### Autenticación: NO usar "Bearer "

**Esta es la diferencia crítica con APIs estándar.** El backend toma el token directamente del header `authorization` sin procesar (línea 63 de `create-http-endpoints.ts`):

```typescript
const token = req.headers["authorization"]; // toma el raw value, sin "Bearer "
```

Por lo tanto:

```bash
# ❌ INCORRECTO - va a fallar con AccessTokenInvalidError
curl -H "Authorization: Bearer <token>"

# ✅ CORRECTO - token sin prefijo
curl -H "Authorization: <token>"
```

### Endpoints

- Todos los endpoints son **POST**
- Content-Type por defecto: `application/json`
- Los endpoints que aceptan archivos usan `multipart/form-data` (definido en `acceptsFiles` del caso de uso)
- URL base: `http://localhost:3000`

## Code Examples

### Probar un endpoint simple (sin payload)

```bash
curl -s -X POST http://localhost:3000/getSubscribers \
  -H "Content-Type: application/json" \
  -H "Authorization: <token>" \
  -d '{}'
```

### Probar con payload

```bash
curl -s -X POST http://localhost:3000/getAdminPaymentsSummary \
  -H "Content-Type: application/json" \
  -H "Authorization: <token>" \
  -d '{"page": 1, "pageSize": 10}'
```

### Obtener un token (login)

```bash
curl -s -X POST http://localhost:3000/signIn \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@email.com","password":"password123"}'
```

O desde el frontend: DevTools → Application → Local Storage → copiar el token.

### Verificar el contenido del token JWT

```bash
# Decodificar el payload (segunda parte del JWT)
echo "<payload-base64>" | base64 -d
```

### Probar un error esperado

```bash
# Sin token - debería dar error de autenticación
curl -s -X POST http://localhost:3000/getSubscribers \
  -H "Content-Type: application/json" \
  -d '{}'
```

## Commands

```bash
# Probar endpoint sin payload
curl -s -X POST http://localhost:3000/<endpoint> \
  -H "Content-Type: application/json" \
  -H "Authorization: <token>" \
  -d '{}'

# Probar endpoint con payload
curl -s -X POST http://localhost:3000/<endpoint> \
  -H "Content-Type: application/json" \
  -H "Authorization: <token>" \
  -d '{"key": "value"}'
```

## Resources

- **Source**: `/packages/api/src/app.ts` — registro de dependencias
- **Use-cases**: `domain/src/use-cases/use-cases.ts` — lista de `httpUseCases`
- **Endpoint handler**: `packages/libraries/api-lib/src/http/create-http-endpoints.ts`
