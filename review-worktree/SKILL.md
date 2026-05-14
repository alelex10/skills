---
name: review-worktree
description: Crea un git worktree en una carpeta separada para revisar otra rama sin salir de la rama actual. Pregunta interactivamente la rama a revisar y el path destino, ejecuta `git worktree add`, y al final ofrece eliminarlo con `git worktree remove`. Usar cuando el usuario quiera hacer code review de una rama (local o remota) o de un PR sin perder su working directory actual.
---

# review-worktree

Esta skill prepara un git worktree aislado para revisar otra rama mientras el usuario sigue trabajando en la suya.

## Flujo

1. **Detectar contexto**:
   - Correr `git rev-parse --show-toplevel` para obtener la raíz del repo.
   - Correr `git branch --show-current` para saber la rama actual del usuario (no la toques).
   - Correr `git fetch origin` para tener refs remotas actualizadas (avisar al usuario que estás fetcheando).

2. **Preguntar al usuario** (con AskUserQuestion, una sola tanda):
   - **Rama a revisar**: nombre de la rama (local o `origin/xxx`) o número de PR de GitHub.
   - **Path del worktree**: por defecto sugerí `../<nombre-repo>-review-<slug-rama>` (slug = rama sin `/`, truncada a 30 chars). Permitir override.

3. **Validaciones antes de crear**:
   - Si el path destino ya existe → avisar y preguntar si sobrescribir (requiere `git worktree remove --force` previo) o elegir otro path.
   - Si la rama no existe localmente pero sí en `origin/`, usar `git worktree add <path> -b <rama-local> origin/<rama>` para trackear la remota.
   - Si es un número de PR (input puramente numérico o con `#`), usar `gh pr checkout <n> --detach` dentro del worktree, o equivalente: `gh pr view <n> --json headRefName,headRepository` para resolver la rama y branch correspondiente.

4. **Crear el worktree**:
   ```bash
   git worktree add <path> <rama>
   ```
   Mostrar al usuario el path absoluto resultante y un resumen del diff vs main:
   ```bash
   git -C <path> log --oneline main..HEAD
   git -C <path> diff --stat main...HEAD
   ```

5. **Indicar próximos pasos** al usuario:
   - Cómo navegar al worktree (`cd <path>`).
   - Que puede correr lint/tests/builds ahí sin afectar su rama actual.
   - Que cuando termine, puede pedirte `/review-worktree cleanup` o correr `git worktree remove <path>` manualmente.

## Cleanup

Si el usuario invoca la skill con argumento `cleanup` o pide eliminar el worktree:

1. Listar worktrees existentes: `git worktree list`.
2. Si hay más de uno además del principal, preguntar cuál eliminar.
3. Verificar que no haya cambios sin commitear en ese worktree (`git -C <path> status --porcelain`). Si los hay, avisar y pedir confirmación explícita antes de forzar.
4. Ejecutar `git worktree remove <path>` (o `--force` solo si el usuario lo confirmó).

## Reglas

- **Nunca** cambiar la rama del working directory principal del usuario.
- **Nunca** correr `git worktree remove --force` sin confirmación si hay cambios sin commitear.
- Si el repo no es un repo git, abortar con un mensaje claro.
- Si `gh` no está instalado y el usuario pasó un PR, sugerir instalarlo o pasar el nombre de rama directo.
- Reportar el path absoluto del worktree creado para que el usuario lo pueda copiar.
