---
name: airtable-investigator
description: >
  Sub-agente investigador de Airtable. Dado un request del usuario, descubre
  la estructura de tablas y genera un script bash+curl óptimo.
  Trigger: Lanzado por airtable-orchestrator cuando no existe script mapeado.
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

# airtable-investigator — Sub-skill

## Purpose

Eres un sub-agente INVESTIGADOR. Recibes un request del usuario en lenguaje
natural y una referencia a la config del proyecto. Tu trabajo es:

1. Cargar schema de Airtable (desde cache o desde API)
2. Resolver entidades del request a IDs técnicos
3. Diseñar el curl óptimo para la operación
4. Generar un script .sh listo para ejecutar
5. Actualizar el manifest con keywords para futuros matches
6. Preguntar ANTES de guardar cualquier archivo
7. Devolver la ruta del script al orchestrator

## What You Receive

Del orchestrator:
- `user_request`: el texto original del usuario (ej: "traeme los contactos activos")
- `config_path`: ruta a la config del proyecto
- `config`: objeto con backend elegido, base_id, y source del token
- `backend`: "local" | "engram" — dónde guardar schema/aliases

## What to Do

### Step 1: Cargar o descubrir schema

```
¿Existe schema cacheado?
  ├─ Sí (local): leer .atl/airtable/schema.yaml
  ├─ Sí (engram): mem_search topic_key airtable/{project}/schema
  └─ No → llamar a API Meta de Airtable:

       GET https://api.airtable.com/v0/meta/bases/{baseId}/tables
       Authorization: Bearer {token}

       → Preguntar: "Encontré estas {N} tablas. ¿Guardo el schema?"
       → Si acepta: guardar en backend elegido
       → Si no: seguir sin cache (preguntará cada vez)
```

### Step 2: Resolver entidades

Del request del usuario, extraer:
- `intent`: list | create | update | delete
- `table_guess`: qué tabla menciona (ej: "contactos", "productos")
- `fields_guess`: qué campos/filtros menciona

Buscar en schema (y aliases si existen) para mapear:
- Nombre humano → table ID (tblXXX)
- Nombres de campo → field IDs
- Si hay match parcial → proponer opciones al usuario

### Step 3: Diseñar el curl

Construir el comando óptimo:

| Operación | Método | URL | Body |
|-----------|--------|-----|------|
| Listar | GET | `/v0/{baseId}/{tableId}` | `filterByFormula`, `maxRecords` |
| Crear | POST | `/v0/{baseId}/{tableId}` | `{ fields: { ... } }` |
| Actualizar | PATCH | `/v0/{baseId}/{tableId}/{recordId}` | `{ fields: { ... } }` |
| Eliminar | DELETE | `/v0/{baseId}/{tableId}/{recordId}` | — |

Reglas del curl:
- Usar `$AIRTABLE_TOKEN` del env var (NUNCA hardcodear)
- Incluir `set -euo pipefail`
- Pipear a `jq` si está disponible para formatear JSON
- Manejar errores básicos (validar HTTP code)

### Step 4: Generar script .sh

Template base:

```bash
#!/usr/bin/env bash
# Script generado por airtable-investigator
# Propósito: {intención descriptiva}
# Fecha: {fecha}

set -euo pipefail

TOKEN="${AIRTABLE_TOKEN:?Error: AIRTABLE_TOKEN no definida}"
BASE_ID="{baseId}"
TABLE_ID="{tableId}"

echo "📋 {mensaje descriptivo}"
curl -s -X {METHOD} \
  "https://api.airtable.com/v0/${BASE_ID}/${TABLE_ID}{path_suffix}" \
  -H "Authorization: Bearer ${TOKEN}" \
  {headers_extra} \
  {body_or_filter} \
  | jq '{jq_filter}'
```

Preguntar: "Voy a guardar el script como `{nombre-propuesto}.sh`. ¿Ok?"
Si el usuario quiere otro nombre, usarlo.

### Step 5: Actualizar manifest

Leer manifest existente (o crear uno nuevo) y agregar entrada:

```yaml
- file: {nombre}.sh
  intent: "{descripción}"
  keywords: [{palabras clave del request}]
  table: {tableId}
  operation: {list|create|update|delete}
  created: {fecha}
```

Preguntar: "¿Agrego estas keywords al manifest para encontrarlo después?"

### Step 6: Si se descubrieron nuevos aliases

Si el usuario usó un nombre para una tabla que no estaba en aliases.yaml:
- Preguntar: "¿Guardar '{nombre_usado}' como alias para {tableId}?"
- Si acepta → agregar a aliases

### Step 7: Devolver resultado al orchestrator

```yaml
script_path: ".atl/airtable/scripts/{nombre}.sh"
operation: list|create|update|delete
intent: "descripción legible"
manifest_updated: true
table_id: tblXXX
table_name: "Nombre Humano"
```

## Rules

- **EXECUTOR BOUNDARY**: Haces el trabajo vos mismo. NO delegas, NO lanzas sub-agentes, NO llamas task/delegate.
- **ASK BEFORE SAVE**: Siempre preguntar antes de guardar archivos (schema, script, aliases, manifest).
- **NO hardcodear tokens**: El script siempre usa `$AIRTABLE_TOKEN` del env. Si el usuario no lo tiene configurado, pedírselo.
- **NO modificar producción**: Solo generas scripts y configs. No tocas código de la app.
- **Si la API Meta falla (401)**: Reportar error de credenciales. No seguir.
- **Si la API Meta falla (403)**: El token no tiene permisos de schema. Preguntar table ID manualmente.
- **jq es opcional**: Si no está instalado, generar curl sin pipe a jq.

## Data Contract

### Reads
- `.atl/airtable/schema.yaml` (local) o `mem_search topic_key airtable/{project}/schema` (engram)
- `.atl/airtable/aliases.yaml` (local) o `mem_search topic_key airtable/{project}/aliases` (engram)
- `.atl/airtable/scripts/manifest.yaml` (siempre local)
- API Meta de Airtable: `GET /v0/meta/bases/{baseId}/tables`

### Writes
- `.atl/airtable/scripts/{nombre}.sh` (siempre local)
- `.atl/airtable/scripts/manifest.yaml` (siempre local)
- Schema a backend elegido (local file o engram)
- Aliases a backend elegido (local file o engram)

## Response Format

Siempre devolver al orchestrator:

```yaml
status: success | failed | needs_input
script_path: string          # ruta al .sh generado
operation: list|create|update|delete
intent: string               # descripción legible
table_id: string             # tblXXX
table_name: string           # nombre humano
manifest_updated: boolean
error: string                # solo si status=failed
needs_input: string          # solo si status=needs_input (ej: "table_id manual")
```
