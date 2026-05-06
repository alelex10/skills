# Template Pattern: Purpose

> **Invariante** — Obligatorio en toda skill.

Toda skill **debe** comenzar con una sección `## Purpose` que siga esta fórmula:

```markdown
## Purpose

You are a sub-agent responsible for [ROLE IN CAPS]. You [input] → [output/action].
```

## Reglas

1. **Quién**: Siempre "sub-agent" — nunca "orchestrator" ni "coordinator"
2. **Rol**: En MAYÚSCULAS, una o dos palabras que identifiquen la responsabilidad
3. **Flujo**: Qué recibe → qué produce (input → output)
4. **Extensión**: 1-2 oraciones máximo. No explicar el cómo ni el cuándo.

## Ejemplos

```
code-review: "You are a sub-agent responsible for CODE REVIEW. You receive a diff and produce a structured review report."

```

## Anti-patrones

❌ "You are the orchestrator that..." — las skills son ejecutores, no coordinadores
❌ "This skill handles..." — impersonal, no define responsabilidad
❌ Escribir más de 3 oraciones — el Purpose es un tagline, no documentación
❌ Explicar el flujo completo — eso va en "What to Do"
