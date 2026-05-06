# Template Pattern: Skill Structure

Toda skill sigue **4 invariantes** y **4 opcionales**. Los invariantes son obligatorios. Los opcionales se incluyen según necesidad.

## Invariantes

| # | Sección | Qué define | Archivo |
|---|---------|-----------|---------|
| 1 | Purpose | Identidad y flujo de la skill | `template-pattern-purpose.md` |
| 2 | What You Receive | Inputs declarados del orchestrator | `template-pattern-what-you-receive.md` |
| 3 | What to Do | Pasos de ejecución | `template-pattern-what-to-do.md` |
| 4 | Rules | Reglas de dominio + executor boundary | `template-pattern-rules.md` |

## Opcionales

| # | Sección | Incluir cuando... | Archivo |
|---|---------|------------------|---------|
| 5 | Data Contract | La skill lee/escribe contexto externo. Siempre se incluyen lectura y escritura juntas. | `template-pattern-data-contract.md` |
| 6 | Response Format | El Return Summary tiene un formato complejo que merece su propia definición. | `template-pattern-response-format.md` |
| 7 | Quality Gates | La skill evalúa, verifica o audita y necesita clasificar findings por severidad. | `template-pattern-quality-gates.md` |
| 8 | Reference Appendix | Material de consulta rápida (tablas, catálogos de patrones prohibidos, matrices de decisión) que no cabe en Rules. | `template-pattern-reference-appendix.md` |

## Acoplamiento

- **Data Contract incluye lectura y escritura juntas** — no existe una skill que lee de storage pero no escribe, ni viceversa
- Si incluís Data Contract, `Artifact Store Mode` es obligatorio en What You Receive
- Si NO incluís Data Contract, What You Receive no necesita Artifact Store Mode
- Si la skill emite findings clasificados (CRITICAL/WARNING/SUGGESTION), Quality Gates y Response Format van juntos — el Response Format debe incluir el campo `severity`

## Ensamblaje

Ver `skill-skeleton.md` para el template rellenable que ensambla todas las secciones.