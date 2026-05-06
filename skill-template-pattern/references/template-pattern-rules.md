# Template Pattern: Rules

> **Invariante** — Obligatorio en toda skill.

Toda skill **debe** terminar con una sección `## Rules` que liste reglas específicas de esa skill:

```markdown
## Rules

- EXECUTOR BOUNDARY: You are an EXECUTOR, not the orchestrator. Do the work yourself. Do NOT launch sub-agents, do NOT call `delegate` or `task`, and do NOT hand execution back unless you hit a real blocker.
- [Regla específica de la skill]
- [Otra regla específica]
- [Regla sobre límites de tamaño si aplica]
- [Regla sobre modo estricto si aplica]
- Return envelope per **Section D** from `[skill-common-file]`.
```

## Reglas sobre Rules

1. **Primera regla siempre es Executor Boundary**: toda skill ejecuta, no delega — esto es innegociable
2. **Siempre al final**: Es la última sección de la skill, antes del footer técnico
3. **Formato**: Lista bullet con `-`, no numerada
4. **Reglas de dominio**: Cada skill pone sus propias reglas (no repetir boilerplate del common)
5. **Size budget**: Skills que producen artifacts incluyen límite de palabras
6. **Return envelope**: Siempre termina con la referencia a Section D del common
7. **No repetir boilerplate**: No incluir reglas del common (como los 4 modos), solo las específicas de la skill
8. **Guard clauses, hard gates y STOP**: Usá los formatos canónicos (`Before...`, `GATE: Do NOT...`, `If... STOP and report`) para que la intención sea clara a primera vista

## Guard Clauses & Hard Gates

Tres patrones para declarar bloqueos y condiciones dentro de la lista de Rules:

| Patrón | Formato | Cuándo usarlo | Ejemplo |
|--------|---------|--------------|---------|
| **Guard Clause** | `Before [acción], check [condición]` | Pre-condición entre pasos | `Before writing any code, check for existing progress — if present, READ it first` |
| **Hard Gate** | `GATE: Do NOT proceed until [condición]` | Bloqueo absoluto entre pasos | `GATE: Do NOT proceed until the test is written AND confirmed RED` |
| **STOP / Escalate** | `If [condición bloqueante], STOP and report` | Abortar ejecución y devolver control | `If a task is blocked by something unexpected, STOP and report back` |

### Guard Clause

Pre-condición que debe verificarse entre pasos. Si no se cumple, la skill toma acción correctiva o aborta.

```
Ejemplos reales (sdd-apply):
  - Before starting work, check for existing apply-progress. If the orchestrator told you previous progress exists, you MUST read it.
  - NEVER implement tasks that weren't assigned to you
  - If the test runner fails for infrastructure reasons (not test failures), report as "Blocked" and continue to next task
```

### Hard Gate

Bloqueo absoluto. La skill NO puede continuar hasta que la condición se cumpla. Es más fuerte que una guard clause — no hay acción correctiva posible, solo esperar o abortar.

```
Ejemplos reales:
  - GATE: Do NOT proceed to GREEN until the test is written (RED phase complete)
  - GATE: Do NOT proceed until GREEN is confirmed by test execution
  - GATE: All spec scenarios for this task must have tests before REFACTOR
  - NEVER archive a change that has CRITICAL issues in its verification report
```

### STOP / Escalate

Abortar la ejecución completa de la skill y devolver el control al orchestrator con un reporte de bloqueo.

```
Ejemplos reales:
  - If a task is blocked by something unexpected, STOP and report back
  - If the merge would be destructive (removing large sections), WARN the orchestrator and ask for confirmation
  - If anything blocks execution (tests fail, design unclear, codebase too complex), STOP and explain — don't push through
```

## Ejemplos

```
code-review (con guard clauses):
  - EXECUTOR BOUNDARY: Do the work yourself. Do NOT launch sub-agents or delegate.
  - NEVER approve code you haven't read completely
  - ALWAYS cite specific lines in findings
  - If a finding is subjective, flag it as OPINION not FACT
  - **Size budget**: Review report MUST be under 600 words
  - Return envelope per Section D from `skills/_shared/code-review-common.md`.

sdd-propose (con guard clause + STOP):
  - EXECUTOR BOUNDARY: Do the work yourself. Do NOT launch sub-agents or delegate.
  - In `openspec` mode, ALWAYS create the `proposal.md` file
  - Before writing the proposal, check if the change directory already has one — READ it first and UPDATE it, don't overwrite
  - Keep the proposal CONCISE - it's a thinking tool, not a novel
  - Every proposal MUST have a rollback plan
  - **Size budget**: Proposal artifact MUST be under 450 words
  - Return envelope per Section D from `skills/_shared/sdd-phase-common.md`.

sdd-apply (con guard clause + STOP + hard gate):
  - EXECUTOR BOUNDARY: Do the work yourself. Do NOT launch sub-agents or delegate.
  - Before writing any code, check for existing apply-progress. If present, READ and include it in your artifact — overwriting loses prior batch work permanently.
  - ALWAYS read specs before implementing — specs are your acceptance criteria
  - ALWAYS follow the design decisions — don't freelance a different approach
  - GATE (Strict TDD only): Do NOT implement until a failing test exists
  - GATE (Strict TDD only): Do NOT proceed past GREEN until all tests pass
  - If a task is blocked by something unexpected, STOP and report back
  - NEVER implement tasks that weren't assigned to you
  - Return envelope per Section D from `skills/_shared/sdd-phase-common.md`.
```

## Anti-patrones

❌ Omitir Executor Boundary como primera regla — toda skill ejecuta, no delega
❌ Repetir reglas del Data Contract
❌ Omitir la referencia a Section D al final
❌ Usar formato narrativo en lugar de lista bullet
❌ Incluir reglas genéricas que ya están en el common
❌ Dejar la sección vacía o con solo "Follow the common protocol"
❌ Mezclar guard clauses, hard gates y STOP — cada uno tiene un formato y severidad distinta
❌ Usar "GATE" para condiciones que no son bloqueantes (usar guard clause en su lugar)
❌ Usar "STOP" para condiciones recuperables (STOP es abortar, no advertir)