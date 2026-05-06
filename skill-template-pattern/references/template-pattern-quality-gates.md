# Template Pattern: Quality Gates & Severity Levels

> **Opcional** — Incluir cuando la skill evalúa, verifica o audita algo y necesita clasificar hallazgos por severidad.

Define cómo la skill clasifica findings y emite un veredicto final. Incluye severidad de issues, sistema de veredictos, y (opcionalmente) una matriz de compliance para skills de verificación.

---

## Template

```markdown
## Quality Gates

### Severity Levels

| Level | Icon | Meaning | Blocks? |
|-------|------|---------|---------|
| **CRITICAL** | 🔴 | Debe corregirse antes de continuar | Yes — blocks pipeline |
| **WARNING** | 🟡 | Debería corregirse, pero no bloquea | No |
| **SUGGESTION** | 🟢 | Mejora opcional, no requerida | No |

### Verdict

Un finding es clasificado con una severidad. El veredicto final de la skill se determina a partir del finding más alto:

| Verdict | Condición | Significado |
|---------|-----------|-------------|
| **PASS** | Cero findings o solo SUGGESTION | Todo bien, seguir adelante |
| **PASS WITH WARNINGS** | Al menos un WARNING, cero CRITICAL | Proceder con precaución |
| **FAIL** | Al menos un CRITICAL | Bloqueado — no continuar hasta resolver |

[IF VERIFICATION SKILL]
### Compliance Matrix (optional)

Para skills de verificación que validan contra specs:

| Status | Meaning | Condition |
|--------|---------|-----------|
| **COMPLIANT** | Implementado y probado | Test exists AND passed at runtime |
| **FAILING** | Implementado pero roto | Test exists BUT failed |
| **UNTESTED** | Sin evidencia de testing | No test found |
| **PARTIAL** | Cubierto parcialmente | Test exists, passes, but only covers part of the spec |

**Evidence rule**: Code existing in the codebase is NOT sufficient evidence of compliance — a passing test at runtime is required.
[ENDIF]
```

---

## Reglas

1. **Severity es binario en bloqueo**: Solo CRITICAL bloquea. WARNING y SUGGESTION nunca bloquean
2. **Veredicto es determinista**: Se deriva del finding más alto (FAIL > PASS WITH WARNINGS > PASS)
3. **Evidence rule**: Si la skill verifica contra specs, code existing no es evidencia — solo tests pasando en runtime
4. **Compliance matrix es opcional**: Solo para skills que validan implementación contra especificaciones
5. **Integración con Rules**: La sección Quality Gates define la taxonomía; las Rules la aplican con reglas como `NEVER archive a change that has CRITICAL issues`
6. **Integración con Response Format**: Cada finding en el Return Summary debe incluir el campo `severity` usando esta taxonomía

## Ejemplo real (sdd-verify)

```markdown
### Severity Levels

| Level | Meaning |
|-------|---------|
| **CRITICAL** | Must fix before archive. Missing tests for ADDED requirements, MODIFIED not implemented, REMOVED still present. |
| **WARNING** | Should fix but won't block. Missing edge cases, minor spec deviations, test quality concerns. |
| **SUGGESTION** | Nice to have. Refactoring opportunities, naming improvements, documentation gaps. |

### Verdict

| Verdict | Condition |
|---------|-----------|
| **PASS** | All ADDED/MODIFIED requirements are COMPLIANT, all REMOVED are gone |
| **PASS WITH WARNINGS** | All COMPLIANT or PARTIAL, with WARNING findings only |
| **FAIL** | Any CRITICAL finding (FAILING, UNTESTED, or REMOVED still present) |

### Compliance Status

| Status | Meaning |
|--------|---------|
| **COMPLIANT** | Test exists AND passed at runtime |
| **FAILING** | Test exists BUT failed |
| **UNTESTED** | No test found for this scenario |
| **PARTIAL** | Test exists and passes, but only covers part of the requirement |
```

## Ejemplo mínimo (code review, solo severidad)

```markdown
## Quality Gates

### Severity Levels

| Level | Meaning |
|-------|---------|
| **CRITICAL** | Security vulnerability, data loss risk, breaking change unannounced |
| **WARNING** | Performance regression, missing error handling, inconsistent pattern |
| **SUGGESTION** | Naming improvement, refactoring opportunity, comment clarification |

### Verdict

- **APPROVE**: No CRITICAL findings, zero or few WARNING
- **REQUEST CHANGES**: Any CRITICAL finding
```

## Anti-patrones

❌ Definir severidades sin criterio de bloqueo (¿CRITICAL bloquea o no?)
❌ Usar más de 3 niveles de severidad — se diluye
❌ Mezclar severidad con compliance (son conceptos distintos)
❌ No integrar con Rules (las reglas deben referenciar los niveles definidos en Quality Gates)
❌ No incluir el campo `severity` en el Response Format cuando la skill tiene Quality Gates