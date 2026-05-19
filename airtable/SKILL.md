---
name: airtable
description: "Thin wrapper for ad-hoc Airtable queries via the claude.ai Airtable MCP. Lists records, fetches by recordId, updates, creates, and deletes. Trigger: when the user asks anything about Airtable bases, tables, or records in natural language (lista tareas, listar contactos, traeme el registro X, actualizá el status, etc.). For programmatic access from other skills, prefer calling the MCP tools directly."
metadata:
  type: orchestrator
  author: alelex10
  version: "2.0"
---

## Purpose

Translate natural-language Airtable requests into `mcp__claude_ai_Airtable__*` tool calls. This is a thin skill — it does NOT generate shell scripts, NOT spawn sub-agents, NOT maintain a manifest. The MCP server handles auth, the API, and rate limits.

For repetitive workflows (like `/task-init`), the calling skill should hold its own schema cache and call MCP tools inline. Use this skill only for one-off questions by a human user.

---

## Available MCP tools

| Tool | When to use |
|------|-------------|
| `search_bases` / `list_bases` | Find a base by name. Setup only. |
| `list_tables_for_base` | Inventory of tables and field IDs. Can be large — write the result to a file if it triggers the token limit. |
| `get_table_schema` | Get the choice IDs for singleSelect / multipleSelects. Required before any filter that involves a select field. |
| `list_records_for_table` | Filtered read. Supports structured `filters` (NOT `filterByFormula`), `recordIds`, `sort`, `pageSize`, `cursor`. |
| `update_records_for_table` | PATCH up to 50 records. For singleSelect, write the plain string name, not the object. |
| `create_records_for_table` | Insert up to 50 records. |
| `delete_records_for_table` | Delete up to 50 records by recordId. |

---

## Triggers

- "lista tareas / contactos / registros …"
- "traeme el registro recXXX / por ID X"
- "actualizá el status de …"
- "creá un nuevo …"
- "eliminá el registro …"
- Any natural-language reference to a base, table, or record

---

## State

The skill caches per-project context in engram (topic `airtable/{project}/config`) so repeat invocations don't re-discover the base. Cached fields:

```yaml
base_id: appXXX
default_table:
  id: tblXXX
  name: "..."
fields_cache:                 # populated on demand
  Status: { id: fldXXX, type: singleSelect, choices: {...} }
  ...
```

Cache is filled lazily — the first time a filter on `Status` is needed, the skill calls `get_table_schema` and stores the choice IDs.

---

## Flow

```
1. Load config from engram (topic airtable/{project}/config)
   - If missing → setup:
     a. Call search_bases with the project name to suggest candidates
     b. User picks the base
     c. Call list_tables_for_base; user picks a default table (optional)
     d. mem_save the config

2. Parse the user's request → identify operation (read / write / search) and target

3. Resolve field IDs if needed:
   - If cache has them → use
   - Otherwise call list_tables_for_base or get_table_schema and update cache

4. Execute the right MCP tool:
   - read  → list_records_for_table (with structured filter)
   - write → update_records_for_table / create_records_for_table / delete_records_for_table

5. Return the parsed result to the user (or to the caller, if invoked from another skill)
```

---

## Filter syntax (structured, NOT formula)

The MCP does not accept `filterByFormula`. All filters use the operator + operands shape:

```json
{
  "operator": "and",
  "operands": [
    { "operator": "isAnyOf",  "operands": ["fldStatusXXX", ["selTodoXXX", "selInProgressXXX"]] },
    { "operator": "hasAnyOf", "operands": ["fldAssigneeXXX", ["recUserXXX"]] }
  ]
}
```

Common operators: `=`, `!=`, `<`, `>`, `<=`, `>=`, `contains`, `doesNotContain`, `isEmpty`, `isNotEmpty`, `isAnyOf`, `isNoneOf`, `hasAnyOf`, `hasAllOf`, `isWithin`.

Rules:
- Operand 1 of a leaf is ALWAYS a field ID (`fldXXX`), never a name.
- For singleSelect / multipleSelects, operand 2 is a **choice ID** (`selXXX`), obtained from `get_table_schema`.
- For multipleRecordLinks (assignee, etc.), use `hasAnyOf` / `hasAllOf` with recordIds.

---

## Confirmation policy

When invoked from a human user: confirm before writes (PATCH / POST / DELETE) with a preview ("Status: Todo → In progress on record recXXX").

When invoked from another skill (callable context): **trust the caller** — do not re-ask. The caller is responsible for any confirmation. This avoids double-prompts in compound flows like `/task-init`.

---

## What this skill no longer does

(All previously handled by the deprecated `airtable-investigator` + `airtable-executor` sub-skills and `.atl/airtable/scripts/`. None of those exist anymore.)

- Generate shell scripts. The MCP replaces them.
- Maintain a `manifest.yaml`. There are no scripts to map.
- Spawn sub-agents to discover schema. `list_tables_for_base` + `get_table_schema` do it inline.
- Manage `AIRTABLE_TOKEN`. The MCP handles auth.

---

## Output format

Return a structured response to the caller:

```yaml
operation: list | get | create | update | delete
status: ok | error
records:                      # for read operations
  - id: recXXX
    fields: { ... }
record_ids: [recXXX, ...]      # for create/update/delete
error: "..."                   # only when status=error
```

If the caller is the user (human), also print a human-readable summary above the structured block.

---

## Rules

- NEVER substitute user-facing names for IDs in MCP calls. Always operate on `appXXX` / `tblXXX` / `fldXXX` / `recXXX` / `selXXX`.
- For singleSelect writes, use the plain string name (`"In progress"`), not the `{id, name}` object returned by reads.
- Update cache lazily — discovery is cheap, but redundant calls waste tokens.
- If `list_tables_for_base` exceeds the inline token limit, the MCP will save the JSON to a file; access it with `jq` on that file rather than re-calling.
