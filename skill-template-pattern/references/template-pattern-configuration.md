# Configuration Section

Optional section #9. Include it when the skill needs **per-project setup state** to run (credentials, IDs, schemas, templates, user preferences). It is the formal contract for what was previously ad-hoc setup flows.

Configuration is different from Data Contract:

| Aspect | Data Contract | Configuration |
|--------|---------------|---------------|
| Lifecycle | Runtime read/write of artifacts every invocation | One-time setup, reused across invocations |
| Scope | Workflow artifacts (proposals, reports, progress) | Skill installation state (IDs, paths, credentials, prefs) |
| When written | Every run | Only on first run or when user reconfigures |

A skill can have BOTH (Configuration to know WHERE to read/write, Data Contract for the actual artifacts).

---

## Storage backends

Configuration is stored in exactly one of these backends, chosen by the user at setup:

| Backend | Where | When to use |
|---------|-------|-------------|
| `local` | `.atl/{skill-name}/config.yaml` (project-relative) | Team-shared config, committed to git, edited by hand |
| `engram` | Topic key `{skill-name}/{project}/config` | Solo work, no file footprint, survives across worktrees |
| `hybrid` | Both backends in sync | Team sharing + cross-session recovery |

The skill reads from whichever backend has the artifact. The user chooses at setup; the choice is persisted in the config itself (`storage.backend` field).

---

## File format: YAML

Always YAML. Reasons:
- Supports comments (humans edit configs)
- Same string serializes identically to file and to an engram observation
- Coherent with existing skills (`task-init`, `airtable`)

JSON only if the config is machine-generated and never edited by hand.

---

## Required top-level fields

Every config YAML MUST include:

```yaml
version: 1                       # schema version — bump on breaking changes
configured_at: 2026-05-24T...    # ISO 8601 timestamp of last setup
storage:
  backend: local | engram | hybrid
# ... skill-specific fields below
```

The skill compares `version` against the version it expects. Mismatch → prompt the user to re-run setup.

---

## Topic key convention (engram)

```
{skill-name}/{project}/config
```

Examples:
- `task-init/liga-ficba/config`
- `airtable/skills/config`
- `notebooklm/{project}/config`

Auxiliary topics use the same prefix:
- `{skill-name}/{project}/last-task` (cache)
- `{skill-name}/{project}/pr-template` (templates)

This keeps everything for a skill grouped under one searchable prefix.

---

## Local path convention

```
{project-root}/.atl/{skill-name}/config.yaml
{project-root}/.atl/{skill-name}/templates/...
{project-root}/.atl/{skill-name}/cache/...
```

The `.atl/` prefix is the Agent Teams Lite convention shared across all skills in this workspace.

---

## Setup detection (no booleans in SKILL.md)

The SKILL.md is static and versioned in git — never mutate it to track installation state. Instead, **the presence of the artifact IS the boolean**:

```
backend = local  → does .atl/{skill}/config.yaml exist?
backend = engram → mem_search topic_key {skill}/{project}/config → hit/miss
backend = hybrid → both must exist; if only one, repair from the other
```

If the artifact exists but `version` differs from what the skill expects → treat as "needs setup".

---

## Setup instructions: separate file (lazy-load)

The setup flow lives in `references/setup.md`, NOT in SKILL.md. Reasons:
- SKILL.md is loaded on every invocation — keep it lean
- Setup is rare; loading those instructions when not needed wastes context
- Editing setup steps shouldn't touch SKILL.md

The runtime skill only says:

> If config is missing or `version` is outdated, read `references/setup.md` and follow that flow.

The setup file contains:
1. Backend choice prompt
2. Field-by-field interactive questions
3. Defaults derived from environment (e.g., `gh api user --jq .login`)
4. Persistence step (write file OR `mem_save`)
5. Template seeding (copy `templates-default/*` to active location)

---

## Default templates

Ship blank/example templates in `templates-default/`:

```
{skill-name}/
  templates-default/
    config.yaml.tmpl       # rellenable, with comments explaining each field
    {other-templates}.tmpl
```

On setup, the skill renders these into the active backend so the user can edit them afterwards.

---

## Section structure (what to write in SKILL.md)

```markdown
## Configuration

**Backend**: `local` | `engram` | `hybrid` (chosen at setup)

**Locations**:
- local: `.atl/{skill-name}/config.yaml`
- engram: topic `{skill-name}/{project}/config`

**Detection**: artifact presence + `version` match.

**Setup**: if config is missing or version mismatches → read `references/setup.md` and run that flow.

**Schema** (skill-specific fields, see `templates-default/config.yaml.tmpl` for full shape):

```yaml
version: 1
configured_at: <ISO 8601>
storage:
  backend: local | engram | hybrid
{skill-specific-fields}:
  ...
```
```

Keep the runtime section short. The detailed schema lives in `templates-default/config.yaml.tmpl` (commented YAML). The setup walkthrough lives in `references/setup.md`.

---

## Rules to add when Configuration is present

In the `Rules` section, include:

- Before any work, check that config exists and `version` matches. If not, run setup before continuing.
- Never write config values inline in code or in the SKILL.md — they belong in the configured artifact.
- Never edit the user's config without confirmation (presenting a diff of what will change).
- When persisting config to engram, use `capture_prompt: false` — config saves are automated artifacts, not human prompts.

---

## When to include Configuration

| Skill needs... | Include Configuration? |
|----------------|------------------------|
| External credentials (API tokens, base IDs, project IDs) | Yes |
| Per-project field mappings or schemas (Airtable fields, label names) | Yes |
| User preferences that persist across invocations | Yes |
| Templates that the user edits before first run | Yes |
| Only runtime inputs from the orchestrator | No |
| Purely deterministic logic with no per-project state | No |
