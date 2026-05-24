---
name: status-diario
description: "Genera `status.md` en la raíz del proyecto con el reporte diario de avance (tarea, PR, avance técnico, siguiente paso, bloqueos, ayuda). Auto-detecta contexto desde git + gh + engram, cura items para que el usuario elija y solo pregunta lo subjetivo. Trigger: cuando el usuario dice 'armá el status', 'status diario', 'status del día', 'daily status', /status-diario, o pide generar el reporte de avance del día."
metadata:
  type: orchestrator
  author: alelex10
  version: "0.5"
  delegation:
    subagent_type: general-purpose
    model: haiku
    pattern: "2-agent-split"
---

## Purpose

Generates a daily status report (`status.md`) at the root of the current project.

**Architecture (v0.4)**: 2-agent Haiku split.

```
Orchestrator (main thread)
  │
  ├── Spawns Agent #1 (Haiku) — "context-gatherer"
  │     • Pre-flight + context detection (git/gh/engram/WIP)
  │     • Task inference (returns flag if unknown)
  │     • Candidate curation (priority: engram > commits > WIP)
  │     • Siguiente paso inference
  │     • Returns STRUCTURED JSON contract (see below)
  │
  ├── Parses JSON → builds AskUserQuestion (chips + Bloqueos + Ayuda)
  │     • Reason this is in the orchestrator: Haiku does NOT reliably
  │       invoke AskUserQuestion as a tool call — it emits the
  │       questions as plain text. Observed twice in v0.3.
  │
  └── Spawns Agent #2 (Haiku) — "renderer"
        • Synthesize Avance from user-selected chips
        • Read template, substitute placeholders
        • Write {repo_root}/status.md
        • Return short confirmation
```

Output is Rioplatense Spanish (voseo). `status.md` always overwrites (file is ephemeral).

---

## Triggers

- `/status-diario`
- "armá el status" / "status diario" / "status del día" / "daily status"
- "generá el status de hoy"

---

## Orchestrator behavior (MANDATORY)

When this skill fires, the orchestrator MUST:

1. **Spawn Agent #1** with `subagent_type: "general-purpose"`, `model: "haiku"`, `description: "Status: gather context"`, and prompt from `## Agent #1 prompt` below (substitute `{{cwd}}`).
2. **Parse the JSON contract** returned by Agent #1. If JSON is malformed, fall back to inline execution.
3. **Show `AskUserQuestion`** to the user with:
   - Question A "¿Qué entra al Avance?" (multiSelect) — chips from `candidates[]` + `"Escribo yo el Avance"` (skip Q A entirely if `no_activity = true`)
   - Question B "¿Tenés bloqueos?" (default `"Ninguno por ahora"`)
   - Question C "¿Necesitás ayuda de alguien?" (default depends on `pr_state`)
   - Question D "¿Tarea?" only if `task_unknown = true`
4. **Spawn Agent #2** with `subagent_type: "general-purpose"`, `model: "haiku"`, `description: "Status: render"`, and prompt from `## Agent #2 prompt` below — passing context blob from Agent #1 + user answers.
5. **Relay** Agent #2's short confirmation (path + 1-line preview). Do NOT echo file content.

The orchestrator must NOT run git/gh/mem_* calls itself, except as a fallback if the Agent tool is unavailable.

---

## Agent #1 prompt (context-gatherer)

> Copy verbatim into the `Agent` tool's `prompt` for Agent #1. Substitute `{{cwd}}`.

```
You are the context-gatherer for the `status-diario` skill. Do NOT ask the user anything. Do NOT call AskUserQuestion. Output ONLY a single JSON object in a fenced ```json``` block at the end of your response.

# Process

## 1. Pre-flight
- `git rev-parse --show-toplevel`. If fails, return JSON `{ "error": "not_a_git_repo" }` and stop.
- repo_root = returned path.

## 2. Detect context

Resolve activity windows (note: engram and git have DIFFERENT windows):

**Git window** (commits + WIP):
- `date +%u` (1=Mon ... 7=Sun)
- If today is Monday (`1`) → git_window = "4 days" (Fri+Sat+Sun+Mon). Use `git log --since='4 days ago'`.
- Otherwise → git_window = "today". Use `git log --since=midnight`.

**Engram window** (decisions/discoveries/patterns):
- ALWAYS last 7 days. Filter engram saves with timestamp >= today - 7 days.
- Rationale: design decisions and discoveries carry over multiple days before being committed. A "today only" filter on engram misses recent uncommitted design work.

Run in parallel:
- `git branch --show-current`
- `git log --since=<git_window> --pretty='%h %s' HEAD`
- `git log --oneline -15 HEAD` (fallback)
- `git status --short` + `git diff --stat`
- `gh pr view --json number,title,url,state,baseRefName,reviewDecision` (if fails, pr = null)
- `git config user.name`
- `mem_current_project`, then `mem_context`, then 1 targeted `mem_search` with keywords from branch + PR title + recent commit subjects. Filter results to engram_window (7 days).

Multiple PRs → most recent open. Branch is main/master → set `warning_main_branch = true`.

## 3. Task inference

Look in order:
1. PR title (regex `\(\(?(\d+)\)?\)`)
2. Branch name (regex `(\d+)`)
3. Top commit subject

If you can extract task_id with reasonable confidence → set `task_unknown = false`. Else → `task_unknown = true` and leave task_id/task_title null.

## 4. Curate candidate items

Build up to 3 `candidates[]`. Each item:
```
{
  "kind": "engram" | "commit" | "wip",
  "short_label": "≤ 60 chars, what shows on chip",
  "detail": "≤ 200 chars, what shows as chip description"
}
```

Priority:
1. Engram saves of type `bugfix`/`architecture`/`decision`/`pattern`/`discovery` from the **last 7 days**. Title → `short_label`; first sentence of `What:` → `detail`. When labeling, include the save date in `short_label` if it's older than today, e.g. `"Decisión cron diario (19/5)"` — helps the user spot stale items.
2. Commits with high-signal subjects (`feat`/`fix`/`refactor`) from the **git_window**. Skip merges and noise. `<sha> <subject>` → `short_label`; full subject → `detail`.
3. Working-tree: bundle ALL unstaged changes into ONE chip. `short_label: "WIP: <N> archivos modificados"`. `detail`: basenames + scope hint from diff (e.g. "fix placeholder en plan-subscription + logging webhook MP").

Engram-filtering nuances:
- Skip engram saves whose `topic_key` already appears in the current branch's commit history (already shipped).
- Prefer saves related to the inferred task (`task_id` in titles or content).
- Newest first when tied.

Rules:
- ≥ 3 strong engram saves → use 3 engram only.
- 2 engram + WIP changes → 2 engram + 1 WIP.
- No engram → commits first, WIP last.
- If no engram, no commits, no WIP in either window → `no_activity = true`, `candidates: []`.

## 5. Siguiente paso

| PR state | siguiente_paso |
|---|---|
| open, no review | `Esperar review del PR #{N}` |
| open, approved | `Mergear PR #{N} a \`{base}\`` |
| open, changes requested | `Incorporar feedback del review en PR #{N}` |
| no PR | `Abrir PR sobre \`{base_branch}\`` (default `main`) |
| merged | `Cerrar tarea en Airtable y arrancar la siguiente` |

## 6. Output

Return ONLY this JSON in a fenced ```json``` block (no other text after it):

```json
{
  "repo_root": "/abs/path",
  "branch": "feat/etapa-3",
  "name": "alelex10",
  "task_unknown": false,
  "task_id": 191,
  "task_title": "...",
  "pr": null | { "number": 1234, "url": "...", "state": "abierto|aprobado|con cambios solicitados|mergeado", "base": "main" },
  "no_activity": false,
  "warning_main_branch": false,
  "candidates": [
    { "kind": "engram", "short_label": "...", "detail": "..." },
    { "kind": "commit", "short_label": "...", "detail": "..." },
    { "kind": "wip", "short_label": "...", "detail": "..." }
  ],
  "siguiente_paso": "..."
}
```

DO NOT call AskUserQuestion. DO NOT write files. Return JSON only.
```

---

## Agent #2 prompt (renderer)

> Copy verbatim. Substitute `{{ctx}}` with the JSON blob from Agent #1 (compacted), and `{{user_answers}}` with the user's responses.

```
You are the renderer for the `status-diario` skill. Do NOT call AskUserQuestion. Do NOT gather more context. Use the inputs as authoritative.

# Inputs

CONTEXT (from agent #1):
{{ctx}}

USER_ANSWERS:
{{user_answers}}

`user_answers` shape:
- `avance_selection`: array of selected chip `short_label` values, OR `"Escribo yo el Avance"`, plus optional `other` free-text addendum.
- `bloqueos`: string (default `"Ninguno por ahora"`).
- `ayuda`: string (default depends on PR).

# Process

## 1. Synthesize `avance`

- If user_answers.avance_selection == "Escribo yo el Avance" → `avance = user_answers.other` (verbatim).
- Else → write **2-4 sentences** in Rioplatense Spanish (voseo), first person ("Extraje", "Apliqué", "Sumé"), paraphrasing the `detail` of each selected chip (lookup chips in `ctx.candidates[]` by `short_label`). Do NOT pad with un-selected items.
- If `user_answers.other` has text AND chips were selected → append as final sentence(s).
- If `ctx.no_activity == true` → `avance = "Recién arrancando, todavía sin actividad registrada hoy."`

## 2. Render template

Read `/home/alelex10/.claude/skills/status-diario/resources/status-template.md`. Substitute:

| Placeholder | Value |
|---|---|
| `{{name}}` | `ctx.name` (fallback: first token capitalized) |
| `{{task_id}}` | `ctx.task_id` |
| `{{task_title}}` | `ctx.task_title` |
| `{{pr_number}}` | `ctx.pr.number` (if pr is null, see below) |
| `{{pr_url}}` | `ctx.pr.url` |
| `{{pr_state}}` | `ctx.pr.state` |
| `{{pr_base}}` | `ctx.pr.base` |
| `{{avance}}` | step 1 |
| `{{siguiente_paso}}` | `ctx.siguiente_paso` |
| `{{bloqueos}}` | `user_answers.bloqueos` |
| `{{ayuda}}` | `user_answers.ayuda` |

If `ctx.pr == null` → replace the entire "Estoy trabajando en" line with:
`Tarea #{ctx.task_id} — {ctx.task_title}. Sin PR abierto todavía (rama \`{ctx.branch}\`).`

## 3. Write

Write the rendered content to `{ctx.repo_root}/status.md` (overwrite silently).

## 4. Return

SHORT message (do NOT echo file content):
- `Status escrito en {ctx.repo_root}/status.md`
- 1-line preview of the first content line (`Tarea #{N} — ...`)

# Conventions
- File body in Rioplatense Spanish (voseo). Identifiers/commands in English.
- PR links: `[#{N}]({url})`.
- No emojis.
```

---

## Resources

- `resources/status-template.md` — markdown template with `{{placeholders}}` (loaded by Agent #2).

---

## Failure modes

| Scenario | Behavior |
|---|---|
| Not a git repo | Agent #1 returns `{ "error": "not_a_git_repo" }` → orchestrator aborts with friendly message |
| `gh` not authenticated | `pr = null`, continue |
| No engram available | `candidates[]` built from git only |
| No activity in window | `no_activity = true`, orchestrator skips Q A, avance = boilerplate |
| Branch is `main` / `master` | `warning_main_branch = true`, orchestrator warns but continues |
| Agent tool unavailable | Orchestrator falls back to inline (run both prompts itself) |
| Agent #1 returns invalid JSON | Orchestrator falls back to inline |

---

## Notes for maintainers

- **Why 2 agents instead of 1**: Haiku does NOT reliably invoke `AskUserQuestion` — observed twice in v0.3 that it emits the questions as plain text instead of calling the tool. Splitting keeps the orchestrator responsible for `AskUserQuestion` (which it can do reliably) while still offloading expensive context-gathering and template-rendering work to Haiku.
- **Cost**: 2 Haiku calls + 1 orchestrator-resident `AskUserQuestion` ≈ still way cheaper than Sonnet/Opus and keeps the main thread lean.
- **Alternative if Haiku output quality drops**: bump `metadata.delegation.model` and both Agent calls to `sonnet`. If Sonnet is used, `AskUserQuestion` could go back into a single sub-agent (collapse to v0.2 pattern), but the 2-agent split is robust to model swaps.
- Keep each sub-agent prompt self-contained — sub-agents receive fresh contexts.
