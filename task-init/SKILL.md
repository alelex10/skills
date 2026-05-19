---
name: task-init
description: "Initialize work environment for a task: fetch task data from Airtable, create branch from main, open draft PR with assignee and in-progress label, then mark the Airtable record as in-progress. Trigger: when user runs /task-init, says 'arrancar tarea X', 'empezar tarea X', 'init task X', or asks to start a new task from Airtable."
metadata:
  type: orchestrator
  author: alelex10
  version: "0.1"
---

## Purpose

Orchestrator skill that bootstraps the work environment for a task:

1. Fetches task data from Airtable (by ID or interactive selection)
2. Creates a branch from `main` named after the task
3. Pushes an empty commit and opens a draft PR with title, body, assignee, and in-progress label
4. Updates the Airtable record status to in-progress

The skill replaces 6+ manual steps with a single command. It calls the `mcp__claude_ai_Airtable__*` tools directly for Airtable I/O — no script generation, no sub-agent delegation.

---

## Triggers

- `/task-init` (no args) → interactive mode, lists tasks filtered by `default_filter`
- `/task-init <id>` → direct mode, fetches by ID
- `/task-init --resume` → resume the last incomplete run from cache
- `/task-init --status` → show progress of the last run (or list in-progress Airtable tasks if no cache)
- `/task-init --all` → ignore `default_filter`, list all tasks
- `/task-init --mine` → list only tasks assigned to me
- Natural phrases in any language: "arrancar tarea 779", "empezar tarea X", "init task 779", "start task X"

---

## Tool Usage

| Action | How |
|--------|-----|
| Read config + template + `.last-task` (engram) | Inline `mem_get_observation` / `mem_search` |
| Fetch / list Airtable records | `mcp__claude_ai_Airtable__list_records_for_table` with structured filters from config (use `recordIds: [recXXX]` for single get) |
| Discover schema on first setup | `mcp__claude_ai_Airtable__list_tables_for_base` + `mcp__claude_ai_Airtable__get_table_schema` |
| PATCH Airtable status | `mcp__claude_ai_Airtable__update_records_for_table` with `records: [{id, fields}]` |
| `git status`, `git stash`, `git checkout`, `git commit --allow-empty`, `git push` | Inline Bash |
| `gh pr create --draft`, `gh pr edit`, `gh pr view`, `gh label list` | Inline Bash |
| Write to engram `last-task` after each step | Inline `mem_update` |

MCP responses always include `recordId` natively (in the `id` field of each record). No special handshake required.

---

## Setup Flow (first run only)

Triggered when no config exists at `.atl/task-init/config.yaml` (local backend) or no observation at `task-init/{project}/config` (engram backend).

```
1. Pick the Airtable base
   - mcp__claude_ai_Airtable__search_bases with the project name → suggest candidates
   - User picks one → capture base_id

2. Pick the table that holds the tasks
   - mcp__claude_ai_Airtable__list_tables_for_base
     (output can be large; if it exceeds the token limit, use jq on the saved file)
   - User picks one → capture table_id + table_name + every field's {id, name, type}

3. Discover singleSelect choice IDs for the Status field
   - mcp__claude_ai_Airtable__get_table_schema with [{tableId, fieldIds: [status_field_id]}]
   - Cache choices map: name → selXXX (e.g., "In progress" → seljNA...)

4. Ask which logical role each field plays (each question interactive — never auto-resolve)
   - id (autoNumber or unique number)
   - title (primary or chosen)
   - description (optional; can be the title or another field)
   - status (singleSelect)
   - assignee (multipleRecordLinks)
   - user's own recordId in the assignee table (to filter "my tasks")

5. Ask the remaining workflow fields:
   - **default_filter** — auto-compose from the answers above (Status in [todo, in_progress, en_revision] AND assignee hasAnyOf [user_record_id]); let the user edit if needed
   - **assignee (GitHub)** — suggest `gh api user --jq .login`; user confirms or overrides
   - **in-progress label** — list with `gh label list`, user picks
   - **base branch** — REQUIRED question, never silent. Detect candidates with `git branch -a` + `git symbolic-ref refs/remotes/origin/HEAD`. Suggest the repo's default first but ALWAYS confirm — some projects branch off `develop`, `release-*`, or a long-lived feature branch.

6. Ask: storage backend for task-init's own state (config + cache)? [local | engram]

7. Save pr.tmpl template to the chosen backend:
   - local → cp templates-default/pr.tmpl to .atl/task-init/templates/
   - engram → mem_save topic task-init/{project}/pr-template
   Tell the user the path/topic so they can edit before running.

8. Persist the config (mem_save topic task-init/{project}/config, or write .atl/task-init/config.yaml).
```

---

## Flow

```
1. Setup check
   - local: does .atl/task-init/config.yaml exist?
   - engram: mem_search topic_key task-init/{project}/config
   - If missing → run Setup Flow above

2. Load config and pr.tmpl
   - local: read files
   - engram: mem_get_observation for each topic

3. Acquire the task (inline MCP calls — no delegation)
   - If invoked with <id> (numeric task ID) → `list_records_for_table` with a filter
       `{ operator: "=", operands: [ config.airtable.fields.id, <id> ] }`
   - If no arg → `list_records_for_table` with `config.airtable.default_filter` (already structured)
     → show numbered list → ask user to pick → keep that record
   - Capture from the MCP response: `record.id` (this IS the recordId, recXXX), and from `cellValuesByFieldId`:
     `fields.id`, `fields.title`, `fields.description` (often empty), `fields.status`, `fields.assignee`.
   - The recordId is needed later for the PATCH; cache it in last-task immediately.

4. Resolve branch prefix (in this order)
   a. If config.branch.type_map[record.type] exists → use the mapping
   b. Else if record.type matches one of the 11 hardcoded prefixes
      (feat|fix|chore|docs|style|refactor|perf|test|build|ci|revert) → use it directly
   c. Else → ask user to pick from the 11 prefixes, with config.branch.fallback_suggestion highlighted

5. Render
   - slug = kebab-case(title), collapsed separators, truncated to 50 chars on word boundary
   - branch_name = render(config.branch.template, {type, id, slug})
   - Parse pr.tmpl: split YAML frontmatter (title) from markdown body
   - pr_title = render(frontmatter.title, placeholders)
   - pr_body  = render(body, placeholders)
   - Validate branch_name against ^(feat|fix|chore|docs|style|refactor|perf|test|build|ci|revert)\/[a-z0-9._-]+$

6. Working tree check
   - git status --porcelain
   - If dirty → ask: (a) git stash push -u -m "task-init: pre-{branch}" and continue
                    (b) abort
   - If user stashes → after success, tell them: "your changes are in `git stash list` — pop them with `git stash pop` when ready"

7. Existing branch check
   - git rev-parse --verify {branch}  (local)
   - git ls-remote --exit-code origin {branch}  (remote)
   - If exists → ask 3 options:
     (a) use existing → git checkout {branch}, then `gh pr list --head {branch} --state open --draft`
         - if draft PR found → show URL, skip to step 11
         - if no PR → continue at step 9 to create one
     (b) delete local + remote and recreate from main → git branch -D + git push origin --delete + continue
     (c) abort

8. Preview to the user
   - Branch name, base branch, PR title, PR body (snippet), assignee, labels
   - Ask: "continue?"

9. Execute the git/gh sequence (write .last-task.yaml after each successful step)
   git fetch origin {base}
   git checkout -b {branch} origin/{base}
     → last-task.branch.created_local = true
   git commit --allow-empty -m "{empty_commit_message}"
   git push -u origin {branch}
     → last-task.branch.pushed_remote = true
   gh pr create --draft --title "{pr_title}" --body "{pr_body}" --base {base}
     → capture PR number + URL
     → last-task.pr.{created, number, url}
   gh pr edit <num> --add-assignee {config.pr.assignee}
     → last-task.pr.assignee_added = true
   gh pr edit <num> --add-label {label1} --add-label {label2} ...
     → If a label does not exist → gh label create {label} or warn, then retry
     → last-task.pr.labels_added = true

10. Update Airtable (only if config.airtable_update.enabled)
    - Preview to the user: `Status: '{current_status_name}' → '{in_progress_choice_name}'`
    - Ask: "confirm?"
    - On yes → call `mcp__claude_ai_Airtable__update_records_for_table` inline:
      ```
      baseId  = config.airtable.base_id
      tableId = config.airtable.table_id
      records = [{ id: <recordId>, fields: { <status_field_id>: <in_progress_choice_name> } }]
      ```
      (use the plain string name `"In progress"`, NOT the `{id, name}` object)
    - → last-task.airtable_update.applied = true

11. Mark last-task.completed_at = now ISO 8601

12. Report to the user
    - PR URL
    - Branch created/used
    - Airtable status updated (or skipped)
    - Next steps: "implement your changes; when ready run /branch-pr to mark the PR ready for review"
```

---

## Config

Loaded from `.atl/task-init/config.yaml` (local backend) or engram topic `task-init/{project}/config`.

```yaml
version: 2
provider: mcp_airtable

storage:
  backend: engram                       # local | engram (for task-init's own state)

airtable:
  base_id: appXXXXXXXXXXXXXX
  table_id: tblXXXXXXXXXXXXXX
  table_name: "Tareas AMIA"

  # Field IDs (cached from list_tables_for_base)
  fields:
    id: fldXXXXXXXXXXXXXX                # autoNumber, role: id
    title: fldXXXXXXXXXXXXXX              # primary or chosen title field
    description: fldXXXXXXXXXXXXXX        # optional; same as title is OK
    status: fldXXXXXXXXXXXXXX             # singleSelect
    assignee: fldXXXXXXXXXXXXXX           # multipleRecordLinks

  # Choice IDs for the Status field (from get_table_schema)
  status_choices:
    todo:        selXXXXXXXXXXXXXX
    in_progress: selXXXXXXXXXXXXXX
    en_revision: selXXXXXXXXXXXXXX
    done:        selXXXXXXXXXXXXXX

  # Structured filter for /task-init interactive mode (MCP filter shape)
  default_filter:
    operator: and
    operands:
      - operator: isAnyOf
        operands:
          - <status_field_id>
          - [<todo_choice>, <in_progress_choice>, <en_revision_choice>]
      - operator: hasAnyOf
        operands:
          - <assignee_field_id>
          - [<user_record_id>]

  user:
    airtable_record_id: recXXXXXXXXXXXXXX
    name: "Your Name"

branch:
  template: "{type}/{id}-{slug}"
  type_map: {}                           # empty if there is no Type field
  fallback_suggestion: chore
  base: main                              # always asked at setup, never assumed

pr:
  draft: true
  empty_commit_message: "chore: init task ({id})"
  template_topic: task-init/{project}/pr-template   # for engram backend
  # or  template: templates/pr.tmpl                  # for local backend
  assignee: alelex10
  labels:
    - "status:in-progress"
  auto_push: true

airtable_update:
  enabled: true
  status_field_id: fldXXXXXXXXXXXXXX
  in_progress_choice_name: "In progress"  # plain string; MCP writes use the name, not the {id, name}
```

---

## Template format (`pr.tmpl`)

Single file. YAML frontmatter contains the `title`. Body below the closing `---` is the PR body. Placeholders use `{name}` syntax.

Supported placeholders:

| Placeholder | Source |
|-------------|--------|
| `{id}` | record.id |
| `{title}` | record.title |
| `{description}` | record.description (empty string if missing) |
| `{type}` | resolved branch prefix |
| `{slug}` | derived from title |
| `{branch}` | full branch name |
| `{assignee}` | config.pr.assignee |
| `{airtable_url}` | https://airtable.com/{base_id}/{table_id}/{recordId} |
| `{date}` | current date ISO 8601 |

---

## Cache `.last-task.yaml` (local) or topic `task-init/{project}/last-task` (engram)

Updated after every successful step in the flow. Used by `--resume` and `--status`.

```yaml
task:
  id: 779
  title: "Mercadopago method service coverage"
  type: test
  airtable_record_id: recABC123
branch:
  name: test/779-mercadopago-method-service-coverage
  created_local: true
  pushed_remote: true
pr:
  created: true
  number: 1234
  url: https://github.com/org/repo/pull/1234
  assignee_added: true
  labels_added: true
airtable_update:
  applied: true
started_at: 2026-05-19T14:32:00Z
completed_at: 2026-05-19T14:33:12Z
```

### `--resume`

1. Load cache. If `completed_at` is set → nothing to resume, report.
2. Otherwise find the first incomplete step and continue from there without re-fetching Airtable.

### `--status`

1. If cache exists → print a summary of the cached run.
2. If not → call `list_records_for_table` inline with a filter
   `{ operator: "=", operands: [config.airtable.fields.status, config.airtable.status_choices.in_progress] }`
   and show the result.

---

## Rules

- Never push without explicit confirmation
- Never create a PR without showing the preview first
- If the branch already exists → ask the user, never decide silently
- If the working tree is dirty → offer `git stash`, never abort silently
- Use conventional commits for the empty init commit
- Write `.last-task.yaml` (or update the engram topic) after each successful step — this is what makes the run resumable
- The PR body must be in English (project convention); only persona reply text uses Spanish

---

## Error handling

| Error | Action |
|-------|--------|
| MCP returns 0 records | Abort with a clear message ("no task matches the filter / ID") |
| MCP returns a permission error | The user may have interface-only access; suggest `list_records_for_page` instead, or fix base permissions |
| Working tree dirty | Ask: stash + continue, or abort |
| Branch exists local or remote | Ask: use existing, delete + recreate, or abort |
| Task has empty `description` | Render `{description}` as empty string and continue |
| Task `type` not in map and not in 11 valid prefixes | Ask the user to pick a prefix |
| `gh` not authenticated | Tell the user to run `gh auth login` |
| Label does not exist in the repo | Try `gh label create {label}`; if that fails, warn and continue without that label |
| `git push` rejected | Show stderr to the user and stop — do not retry |
