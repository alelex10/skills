# Orchestrator Patterns

Patrones específicos del rol de **orchestrator** — el agente que coordina, NO ejecuta. Son el espejo de los executor patterns pero con lógica inversa.

Todo orchestrator sigue **4 invariantes** y **1 opcional**. Los invariantes son obligatorios. El opcional se incluye cuando el orchestrator lanza sub-agentes que retornan resultados estructurados.

## Invariantes

| # | Patrón | Qué define | Archivo |
|---|--------|-----------|---------|
| 1 | Delegation Boundary | Regla "NEVER execute", tabla de responsabilidades, skill resolution + feedback loop | `orchestrator-pattern-delegation-boundary.md` |
| 2 | DAG / Dependency Graph | Orden de ejecución, dependencias required/optional, reference-passing contract | `orchestrator-pattern-dag.md` |
| 3 | State Persistence for Recovery | Persistir estado del workflow para sobrevivir compaction | `orchestrator-pattern-state-persistence.md` |
| 4 | Slash Command Routing | Meta-comandos que mapean a fases, con `/continue` y `/ff` | `orchestrator-pattern-slash-commands.md` |

## Opcional

| # | Patrón | Incluir cuando... | Archivo |
|---|--------|------------------|---------|
| 5 | Result Contract | El orchestrator lanza sub-agentes que retornan resultados estructurados. | `orchestrator-pattern-result-contract.md` |

## Relación con Executor Patterns

| Dimensión | Executor | Orchestrator |
|-----------|----------|--------------|
| **Boundary** | "Do the work yourself, don't delegate" | "NEVER execute, ONLY coordinate" |
| **Flow** | Pasos secuenciales (What to Do) | DAG con dependencias |
| **State** | Stateless por invocación | Stateful — persiste para `/continue` |
| **Commands** | Invocado por orchestrator | Expone `/commands` al usuario |
| **Sub-agentes** | Prohibido lanzarlos | Es su trabajo coordinarlos |
| **Artifacts** | Lee/escribe los suyos | Pasa referencias, no contenido |
| **Result format** | Response Format (para sí mismo) | Result Contract (exige a sub-agentes) |

## Referencia cruzada

Los executor patterns están en el directorio padre: `../template-pattern-*.md`. Los orchestrator patterns complementan pero no reemplazan — un orchestrator skill completa usa los 4 patrones de este directorio MÁS los patrones de executor que aplican (Quality Gates, Response Format, Reference Appendix según necesidad).