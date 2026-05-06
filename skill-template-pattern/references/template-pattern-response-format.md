# Template Pattern: Response Format

> **Opcional** — Incluir solo cuando el Return Summary tiene un formato complejo que merece su propio template. Si el Return Summary es simple (status + paths + siguiente paso + issues), queda claro en el What to Do.

## Contrato mínimo

Todo Return Summary **debe** incluir estos 4 elementos, sin importar el formato elegido:

| Elemento | Por qué | Ejemplo |
|----------|---------|---------|
| Estado | Confirmar qué se completó | "Proposal created", "Implementation done (3/5 tasks)" |
| Paths afectados | Referencia a archivos/locations | `sdd/{change}/proposal`, `src/auth/login.ts` |
| Siguiente paso | El orchestrator necesita encadenar | "Run sdd-spec next", "Await user decision" |
| Issues/bloqueos | Visibilidad de problemas | "2 specs unresolvable without clarification" |

## Reglas

1. **El creador define el formato**: Markdown, tablas, bullets — lo que mejor comunique el artifact
2. **Sin límite de palabras**: El summary puede ser más largo que el artifact persistido
3. **Nombre de sección**: Siempre "Return Summary" seguido de dos puntos
4. **Los 4 elementos del contrato son obligatorios**: No omitir ninguno

## Cómo crear un formato de response

Cuando crees una skill, definí el formato de response en un archivo dedicado dentro de la carpeta de formats de tu dominio:

```
skill-best-practices/
  [tu-dominio]/
    formats/
      [artifact-name]-response.md
```

Cada archivo define la estructura markdown exacta que la skill retorna. No hay un formato universal — cada dominio y artifact puede tener el suyo.

## Anti-patrones

❌ Copiar el formato de otro artifact sin adaptarlo — cada uno es distinct
❌ Usar JSON u otro formato no-markdown
❌ Poner el artifact completo en el response — usa referencias
❌ Omitir alguno de los 4 elementos del contrato mínimo