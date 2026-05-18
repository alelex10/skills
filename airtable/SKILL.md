---
name: airtable
description: >
  Orquestrador de operaciones Airtable. Busca scripts existentes en el manifest,
  o delega a airtable-investigator para investigar y generar el script óptimo.
  Trigger: cuando el usuario pide listar, crear, actualizar o eliminar registros
  en Airtable, o menciona tablas/conceptos de Airtable.
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

# airtable — Orchestrator

## Role

Eres un COORDINADOR, no un ejecutor. Tu trabajo es:
1. Recibir el request del usuario sobre Airtable
2. Buscar si ya existe un script mapeado para ese request
3. Si existe → mostrarlo y preguntar si ejecutar (delegando a airtable-executor)
4. Si NO existe → lanzar airtable-investigator para que investigue y genere el script
5. Nunca ejecutar curl, nunca leer el contenido de scripts, nunca escribir archivos de código

## Delegation Boundary

| Acción | ¿Quién? |
|--------|---------|
| Leer config / manifest (YAML chico) | Orchestrator (inline) |
| Leer schema de Airtable | Delegado a investigator |
| Generar script .sh | Delegado a investigator |
| Ejecutar script | Delegado a executor |
| Escribir archivos de código/config | Delegado a investigator |
| Setup inicial (preguntar backend) | Orchestrator (inline) |
| Mostrar resultados al usuario | Orchestrator (inline) |

## Flow

### Fase 0: Setup (solo primera vez)

```
¿Existe config?
  ├─ No → preguntar:
  │       - ¿Backend? (local | engram)
  │       - ¿Token? (env var | guardar)
  │       - Guardar decisión
  └─ Sí → cargar config
```

### Fase 1: Matching

```
Request del usuario → extraer entidades e intención
  ├─ Leer manifest (.atl/airtable/scripts/manifest.yaml
  │   o mem_search si backend=engram)
  ├─ Buscar keywords que matcheen
  │
  ├─ Match encontrado:
  │   ├─ Mostrar: "Encontré el script X para {intención}"
  │   ├─ ¿Ejecutar?
  │   │   ├─ Sí + lectura  → task(airtable-executor, ruta, modo=read)
  │   │   ├─ Sí + escritura → task(airtable-executor, ruta, modo=write)
  │   │   └─ No → mostrar script y terminar
  │   └─ FIN
  │
  └─ Sin match:
      ├─ task(airtable-investigator, request + referencia a config)
      ├─ Recibir: { script_path, manifest_updates }
      ├─ Mostrar script al usuario
      ├─ ¿Ejecutar? (misma lógica que arriba)
      └─ FIN
```

### Reglas de ejecución

| Operación | ¿Preguntar antes de ejecutar? |
|-----------|-------------------------------|
| `list` / `get` (lectura) | ❌ No — ejecutar directo |
| `create` / `update` / `delete` (escritura) | ✅ Sí — preview + confirmación |

El manifest ya tiene el campo `operation`. El orchestrator lo lee del manifest match para decidir.

## State Persistence

### Config de proyecto

Guarda la decisión de backend y token apenas se configura:

- **Local** (`config.yaml` en `.atl/airtable/`):
  ```yaml
  backend: local
  token_source: env_var  # o "local_file"
  base_id: appXXXXXXXXXX
  ```
- **Engram** (topic key `airtable/{project}/config`):
  `mem_save` con `capture_prompt: false`, `type: config`

### Scripts SIEMPRE en disco

Sin importar el backend, los scripts .sh y el manifest viven en `.atl/airtable/scripts/`.
Esto permite ejecución directa con `bash archivo.sh` sin que la IA lea contenido.

## Input esperado

Lenguaje natural del usuario describiendo qué quiere hacer:

- "Traeme los contactos activos"
- "Creá un producto nuevo llamado X con precio Y"
- "Actualizá el estado del contacto recXXX a Inactivo"
- "Eliminá el registro recXYZ"
- "Listá los últimos 10 pedidos"

## Output

- Script encontrado o generado (se muestra al usuario)
- Resultado de ejecución (si aplica)
- Siempre preguntar antes de mutaciones

## Commands

No requiere slash command específico. La skill se auto-activa por contexto
cuando el usuario menciona Airtable o pide operaciones sobre tablas/registros.

## Resources

- **Investigator**: `~/.config/opencode/skills/airtable-investigator/SKILL.md`
- **Executor**: `~/.config/opencode/skills/airtable-executor/SKILL.md`
- **Documentación de diseño**: `skills-doc/airtable-skill-design.md` en Obsidian Vault
