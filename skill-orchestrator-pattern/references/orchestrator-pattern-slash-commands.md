# Orchestrator Pattern: Slash Command Routing

> **Invariante** — Obligatorio en todo orchestrator que expone comandos al usuario.

Los orchestrators exponen meta-comandos que mapean a fases del DAG. Sin slash commands, el usuario no tiene forma de iniciar, continuar o consultar el estado del workflow.

---

## Template

```markdown
## Commands

- `/[command-start] [params]`: Inicia el flujo completo. Si no se especifican params, lista opciones disponibles.
- `/[command-continue]`: Continúa con la siguiente fase pendiente. Usa state para determinar qué falta.
- `/[command-status]`: Muestra el estado actual del workflow activo.
- `/[command-ff]`: Ejecuta fases en secuencia desde el punto actual hasta completar (o hasta que una falle).
- `/[command-phase] [params]`: Ejecuta una fase específica directamente.

### Continue Logic

`/[command-continue]` determina la siguiente fase basándose en el DAG state:

1. Verificar artifacts existentes via backend (topic keys o filesystem)
2. Encontrar el primer nodo del DAG sin artifact completado
3. Verificar que sus dependencias requeridas estén satisfechas
4. Si dependencias faltan → informar al usuario qué artifacts necesita primero
5. Si dependencias OK → lanzar sub-agente para esa fase

### Fast-Forward Logic

`/[command-ff]` ejecuta múltiples fases en secuencia:

1. Verificar punto de partida (última fase completada)
2. Ejecutar cada fase secuencialmente según el DAG
3. Si una fase falla → STOP y reportar el error al usuario
4. Si todas pasan → DAG completado hasta la última fase
```

---

## Reglas

1. **Cada comando tiene un propósito único**: No mezclar start con continue. No mezclar status con status+continue.
2. **`/continue` SIEMPRE lee state**: No asume qué fase sigue — lo determina del state persistedo.
3. **`/ff` tiene fail-fast**: Si una fase falla, para inmediatamente y reporta. No continúa con la siguiente.
4. **Comandos sin params son interactivos**: Si el usuario no especifica qué PR/branch/change, el orchestrator lista opciones y pide al usuario que elija.
5. **El comando de status es de solo lectura**: No lanza sub-agentes, solo lee state y lo muestra.

## Ejemplo real (cr-orchestrator)

```markdown
## Commands

- `/cr-start [branch_o_pr]`: Inicia el flujo completo. Si no se especifica, lista las PRs disponibles del repositorio.
- `/cr-continue`: Continúa con la siguiente fase pendiente (Análisis o Publicación).
- `/cr-status`: Muestra el estado actual de la revisión activa.
```

## Ejemplo real (cy-orchestrator, con routing de usuario)

```markdown
## Commands

- `/cy-explore <component>` -> run `cy-explorer`
- `/cy-test <component>` -> run `cy-tester`
- `/cy-verify <component>` -> run `cy-verify`
- `/cy-test-all <components>` -> run `cy-explorer` -> `cy-tester` -> `cy-verify` for each component
```

## Ejemplo real (SDD DAG, completo con continue y ff)

```markdown
## Commands

| Command                  | Action                                                           |
| ------------------------ | ---------------------------------------------------------------- |
| `/sdd-init`              | Ejecutar `sdd-init`                                              |
| `/sdd-explore <tema>`    | Ejecutar `sdd-explore`                                           |
| `/sdd-new <cambio>`      | Ejecutar `sdd-explore` → `sdd-propose`                           |
| `/sdd-continue [cambio]` | Crear siguiente artifact faltante en la cadena de dependencias   |
| `/sdd-ff [cambio]`       | Ejecutar `sdd-propose` → `sdd-spec` → `sdd-design` → `sdd-tasks` |
| `/sdd-apply [cambio]`    | Ejecutar `sdd-apply` en batches                                  |
| `/sdd-verify [cambio]`   | Ejecutar `sdd-verify`                                            |
| `/sdd-archive [cambio]`  | Ejecutar `sdd-archive`                                           |

### Continue Logic

`/sdd-continue [cambio]`:

1. Verificar artifacts existentes via `mem_search` con topic keys
2. Encontrar el primer nodo del DAG sin artifact completado
3. Verificar que sus dependencias estén satisfechas
4. Si dependencias faltan → informar al usuario
5. Si dependencias OK → lanzar sub-agente para esa fase

Ejemplo:

- Si existe `proposal` pero no `spec` → lanzar `sdd-spec`
- Si existe `spec` y `design` pero no `tasks` → lanzar `sdd-tasks`

### Fast-Forward Logic

`/sdd-ff [cambio]`:

1. Verificar punto de partida (usualmente después de proposal)
2. Ejecutar cada fase secuencialmente
3. Si una fase falla → STOP y reportar
4. Si todas pasan → DAG completado hasta tasks
```

## Ejemplo real (judgment-day, sin commands de DAG)

````markdown
## Commands

```bash
# No CLI commands — this is a pure orchestration protocol.
# Execution happens via delegate() and delegation_read() tool calls.
# Trigger: User says "judgment day", "judgment-day", "doble review", etc.
```
````

Note: judgment-day no tiene slash commands porque se trigger por frase natural, no por comando. Su "routing" es un decision tree, no un DAG. Sigue el patrón de lanzar jueces → sintetizar → fix → re-juzgar, pero sin `/continue` ni `/ff`.

```

## Anti-patrones

❌ Comandos que hacen más de una cosa — `/start` que también hace continue, o `/status` que también modifica state
❌ `/continue` que asume la siguiente fase sin leer state — siempre verificar qué artifacts existen y qué dependencias están satisfechas
❌ `/ff` que continúa después de una fase fallida — fail-fast, no fail-continue
❌ No incluir `/status` — el usuario necesita consultar el estado del workflow sin ejecutar nada
❌ Comandos que reciben params pero no说明 qué hacer cuando faltan — siempre listar opciones disponibles si el usuario no especifica
❌ Hardcodear la secuencia en `/continue` — el orden viene del DAG, no del código
```
