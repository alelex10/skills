# Template Pattern: Reference Appendix

> **Opcional** — Incluir cuando la skill necesita material de consulta rápida que no cabe en Rules, Steps ni Response Format.

Una Reference Appendix es contenido **suplementario de dominio** que el agente consulta durante la ejecución. Va después de Rules (la última sección de la skill). No es una regla, no es un paso, no es un formato de salida — es consulta rápida.

---

## Template

```markdown
## [Appendix Title]

[Breve descripción de qué es y cuándo consultarlo. Omitir si el título es auto-explicativo.]

[Tabla, catálogo de código, matriz o formato libre]
```

---

## Reglas

1. **Siempre después de Rules**: Es la última sección de la skill, después de Rules
2. **No es una Rule**: Las Rules no referencian el appendix — el appendix SUPLEMENTA las Rules. Ejemplo: "Always use RFC 2119 keywords" va en Rules, la tabla de keywords va en el appendix
3. **No es un Step**: El agente no "ejecuta" el appendix — lo consulta como referencia mientras ejecuta Steps o aplica Rules
4. **Formato libre**: Tabla, catálogo de código, matriz, lista de ejemplos — lo que mejor comunique el contenido
5. **No repetir**: Si el contenido ya existe en Rules, Response Format o Data Contract, no duplicarlo
6. **Contenido de dominio**: El appendix es específico de la skill, no del framework (no repetir boilerplate del common)

## Cuando incluir vs. no incluir

| Tiene sentido | No tiene sentido |
|--------------|-----------------|
| Catálogo de patrones prohibidos con ejemplos de código | Lista de reglas (eso va en Rules) |
| Tabla de consulta rápida de keywords/terminología | Definición de formato de salida (eso va en Response Format) |
| Matriz de decisiones con criterios | Instrucciones de ejecución (eso va en Steps) |
| Ejemplos de inputs/outputs correctos vs. incorrectos | Contenido que ya está en el common |
| Notas técnicas que la skill necesita pero no se repite todos los Steps | Referencias a archivos externos sin contenido propio |

## Ejemplos

### Tabla de consulta rápida (RFC 2119 — sdd-spec)

```markdown
## RFC 2119 Keywords Quick Reference

| Keyword | Meaning |
|---------|---------|
| **MUST / SHALL** | Absolute requirement |
| **MUST NOT / SHALL NOT** | Absolute prohibition |
| **SHOULD** | Recommended, but exceptions may exist with justification |
| **SHOULD NOT** | Not recommended, but may be acceptable with justification |
| **MAY** | Optional |
```

### Catálogo de patrones prohibidos con ejemplos (strict-tdd.md)

```markdown
## Banned Assertion Patterns (NEVER write these)

# ❌ Tautology — proves nothing
expect(true).toBe(true)
assert 1 == 1

# ❌ Ghost loop — NEVER executes if collection is empty
const items = screen.queryAllByTestId("item");  // returns []
for (const item of items) {
  expect(item).toHaveTextContent("value");       // dead code
}

# ❌ Smoke test — renders but asserts no behavior
render(<MyComponent data={mockData} />);
expect(screen.getByTestId("wrapper")).toBeInTheDocument();

# ❌ Type-only — proves existence, not behavior
expect(result).toBeDefined()         // alone is useless — WHAT is the value?
expect(result).not.toBeNull()        // same — assert the actual value
assert result is not None            // same
```

### Matriz de decisiones (ejemplo genérico)

```markdown
## Test Layer Selection Guide

| Scenario | Layer | Reason |
|----------|-------|--------|
| Pure data transformation | Unit | No dependencies, fast feedback |
| Component with API call | Integration | Tests wiring between layers |
| Full user journey | E2E | Tests real flow across boundaries |

**Rule**: Use the HIGHEST available layer that fits the task. If that layer is unavailable, degrade (E2E → Integration → Unit) rather than skip.
```

## Anti-patrones

❌ Poner contenido que debería ser una Rule — las Rules definen el comportamiento, el appendix solo lo documenta
❌ Repetir contenido del common — el appendix es específico de dominio
❌ Crear appendix para contenido que se referencia pero no se necesita consultar — si el agente solo "sabe" la info, no necesitás un appendix
❌ Poner el appendix antes de Rules — siempre va después
❌ Usar el appendix como "data dump" para no organizar mejor la skill — si el contenido cabe en otro lado, va en otro lado
