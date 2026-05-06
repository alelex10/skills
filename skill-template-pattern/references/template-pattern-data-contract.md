# Template Pattern: Data Contract

> **Opcional** — Incluir cuando la skill lee/escribe contexto externo (Engram, filesystem, etc.). Si la skill solo procesa lo que el orchestrator le pasa y retorna inline, no necesita este contrato.
>
> **Acoplamiento**: Lectura y escritura siempre van juntas. No existe una skill que lee de storage pero no escribe, ni viceversa. Si incluís Data Contract, incluís ambos.

Toda skill con almacenamiento externo **debe** incluir una sección `## Data Contract` que declare cómo lee y escribe:

```markdown
## Data Contract

> Follow **Section B** (retrieval) and **Section C** (persistence) from `[skill-common-file]`.

- **engram**: Read `[topic_keys de dependencias]` (required/optional). Save artifact as `[topic_key de salida]`.
- **openspec**: Read and follow `[openspec-convention-file]`. Write per `[openspec-convention-file]`.
- **hybrid**: Follow BOTH conventions — Engram (primary) with filesystem fallback. Persist to Engram AND write to filesystem.
- **none**: Use only what the orchestrator provides. Return result only. Never create or modify project files.
```

## Reglas

1. **Referencia inicial**: Siempre comienza citando el archivo de protocolo compartido (Secciones B y C)
2. **Cuatro modos**: Siempre en el mismo orden: engram, openspec, hybrid, none
3. **Engram**: Especifica qué topic_keys lee (required/optional) y a cuál guarda
4. **Openspec**: Siempre referencia el archivo de convención correspondiente
5. **Hybrid**: Siempre dice "Follow BOTH conventions" y especifica primario/fallback
6. **None**: Retorna inline, sin leer ni escribir nada externo

## Ejemplos

```
code-review:
  - **engram**: Read `code-review/{project}/config` (optional). Save artifact as `code-review/{project}/review-report`.
  - **openspec**: Read and write per `skills/_shared/openspec-convention.md`.
  - **hybrid**: Follow BOTH conventions — Engram (primary) with filesystem fallback. Persist to Engram AND write to filesystem.
  - **none**: Use only what the orchestrator provides. Return result only. Never create or modify project files.

sdd-propose:
  - **engram**: Read `sdd/{change-name}/explore` (optional). Save artifact as `sdd/{change-name}/proposal`.
  - **openspec**: Read and write per `skills/_shared/openspec-convention.md`.
  - **hybrid**: Follow BOTH conventions — Engram (primary) with filesystem fallback. Persist to Engram AND write to filesystem.
  - **none**: Use only what the orchestrator provides. Return result only. Never create or modify project files.

sdd-apply:
  - **engram**: Read `sdd/{change-name}/proposal`, `sdd/{change-name}/spec`, `sdd/{change-name}/design`, `sdd/{change-name}/tasks` (all required). Save progress as `sdd/{change-name}/apply-progress`. Mark tasks complete via `mem_update`.
  - **openspec**: Read and write per `skills/_shared/openspec-convention.md`. Update `tasks.md` with `[x]` marks.
  - **hybrid**: Follow BOTH conventions — Engram (primary) with filesystem fallback. Persist to Engram AND update `tasks.md` with `[x]` marks.
  - **none**: Use only what the orchestrator provides. Return progress only. Do not update project artifacts.
```

## Anti-patrones

❌ Omitir la referencia al protocolo compartido
❌ Inventar comportamientos para los modos que no están en la convención compartida
❌ Cambiar el orden de los 4 modos
❌ Omitir el modo `none`
❌ Declarar lectura sin escritura o escritura sin lectura — siempre van juntas
❌ Leer dependencies que no se necesitan para la ejecución