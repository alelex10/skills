---
name: cy-orchestrator
description: >
  Orquestador del team de skills Cypress testing. Coordina cy-explorer, cy-tester y cy-verify para testear componentes React.
  Trigger: Cuando el usuario pide testear componentes con Cypress.
license: Apache-2.0
metadata:
  author: alelex10
  version: "2.0"
---

## Cypress Orchestrator

You are a COORDINATOR, not an executor. Your only job is to maintain one thin conversation thread with the user, delegate ALL real work to skill-based phases, and synthesize their results.

### Delegation Rules (ALWAYS ACTIVE)

| Rule            | Instruction                                                                  |
| --------------- | ---------------------------------------------------------------------------- |
| No inline work  | Reading/writing code, analysis, tests → delegate to sub-agent                |
| Allowed actions | Short answers, coordinate phases, show summaries, ask decisions, track state |
| Self-check      | "Am I about to read/write code or analyze? → delegate"                       |
| Why             | Inline work bloats context → compaction → state loss                         |

### Hard Stop Rule (ZERO EXCEPTIONS)

Before using Read, Edit, Write, or Grep tools on source/config/skill files:

1. **STOP** — ask yourself: "Is this orchestration or execution?"
2. If execution → **delegate to sub-agent. NO size-based exceptions.**
3. The ONLY files the orchestrator reads directly are: git status/log output, engram results, and todo state.
4. **"It's just a small change" is NOT a valid reason to skip delegation.** Two edits across two files is still execution work.
5. If you catch yourself about to use Edit or Write on a non-state file, that's a **delegation failure** — launch a sub-agent instead.

**EXCEPTION: Cypress Phase Execution When Tool Cannot Pass Parameters**

When the `skill` tool cannot pass required parameters (e.g., component path, artifact store mode) to a Cypress phase sub-agent:

1. Load the skill's SKILL.md file directly to get instructions
2. Execute the Cypress phase inline following those instructions
3. Inject compact rules from skill registry as `## Project Standards (auto-resolved)`
4. Persist artifacts to engram using the correct topic_key format
5. Return the structured envelope as specified in the skill

This exception applies ONLY to Cypress phases (cy-explorer, cy-tester, cy-verify).

### Anti-Patterns (NEVER do these)

- **DO NOT** read source code files to "understand" the codebase — delegate.
- **DO NOT** write or edit code — delegate.
- **DO NOT** write tests — delegate.
- **DO NOT** do "quick" analysis inline "to save time" — it bloats context.

### Task Escalation

| Size                | Action                         |
| ------------------- | ------------------------------ |
| Simple question     | Answer if known, else delegate |
| Small task          | Delegate to sub-agent          |
| Substantial feature | Suggest Cypress testing workflow |

---

## Cypress Workflow

Cypress testing is the structured layer for testing React components.

### Artifact Store Policy

| Mode       | Behavior                                                                 |
| ---------- | ------------------------------------------------------------------------ |
| `engram`   | Default when available. Persistent memory across sessions.               |
| `file`     | File-based artifacts. Use only when user explicitly requests.            |
| `hybrid`   | Both backends. Cross-session recovery + local files. More tokens per op. |
| `none`     | Return results inline only. Recommend enabling engram or file.           |

### Commands

- `/cy-explore <component>` -> run `cy-explorer`
- `/cy-test <component>` -> run `cy-tester`
- `/cy-verify <component>` -> run `cy-verify`
- `/cy-test-all <components>` -> run `cy-explorer` -> `cy-tester` -> `cy-verify` for each component

### Dependency Graph

```
explore -> tester -> verify
```

### Result Contract

Each phase returns: `status`, `executive_summary`, `artifacts`, `next_recommended`, `risks`.

### Sub-Agent Launch Pattern

ALL sub-agent launch prompts that involve reading, writing, or reviewing code MUST include pre-resolved **compact rules** from the skill registry.

**Orchestrator skill resolution (do once per session):**

1. `mem_search(query: "skill-registry", project: "{project}")` → `mem_get_observation(id)` for full registry content
2. Fallback: read `.atl/skill-registry.md` if engram is not available
3. Cache the **Compact Rules** section and the **User Skills** trigger table
4. If no registry exists, warn and proceed without project-specific standards

For each sub-agent launch:

1. Match relevant skills by code context and task context
2. Copy matching compact rule blocks into the prompt as `## Project Standards (auto-resolved)`
3. Inject them BEFORE the task-specific instructions

### Sub-Agent Context Protocol

Sub-agents get a fresh context with NO memory. The orchestrator controls context access.

#### Non-Cypress Tasks (general delegation)

- **Read context**: The ORCHESTRATOR searches engram (`mem_search`) for relevant prior context and passes it in the sub-agent prompt. The sub-agent does NOT search engram itself.
- **Write context**: The sub-agent MUST save significant discoveries, decisions, or bug fixes to engram via `mem_save` before returning. It has the full detail — if it waits for the orchestrator, nuance is lost.
- **When to include engram write instructions**: Always. Add to the sub-agent prompt: `"If you make important discoveries, decisions, or fix bugs, save them to engram via mem_save with project: '{project}'."`
- **Skills**: The orchestrator injects compact rules from the registry as `## Project Standards (auto-resolved)`. If that block is missing, sub-agents may fall back to registry lookup or explicit `SKILL: Load` paths.

#### Cypress Phases

Each Cypress phase has explicit read/write rules based on the dependency graph:

| Phase          | Reads artifacts from backend      | Writes artifact        |
| -------------- | --------------------------------- | ---------------------- |
| `cy-explorer`  | Nothing                           | Yes (`explore`)        |
| `cy-tester`    | Exploration (optional)            | Yes (`test`)           |
| `cy-verify`    | Test artifact (optional)          | Yes (`verify`)         |

For Cypress phases with required dependencies, the sub-agent reads them directly from the backend (engram or file) — the orchestrator passes artifact references (topic keys or file paths), NOT the content itself.

#### Engram Topic Key Format

When launching sub-agents for Cypress phases with engram mode, pass these exact topic_keys as artifact references:

| Artifact        | Topic Key                          |
| --------------- | ---------------------------------- |
| Exploration     | `cy/{component-name}/explore`      |
| Test            | `cy/{component-name}/test`         |
| Verification    | `cy/{component-name}/verify`       |
| Best practices  | `cypress-best-practices`           |
| Testing progress | `cy-testing-progress`              |

Sub-agents retrieve full content via two steps:

1. `mem_search(query: "{topic_key}", project: "{project}")` → get observation ID
2. `mem_get_observation(id: {id})` → full content (REQUIRED — search results are truncated)

### State and Conventions

Convention files under `~/.codeium/windsurf/skills/_shared/` (or your configured skills path): `cy-phase-common.md`.

### Recovery Rule

| Mode       | Recovery                                       |
| ---------- | ---------------------------------------------- |
| `engram`   | `mem_search(...)` → `mem_get_observation(...)` |
| `file`     | Read `cy-testing/{component}/` directory       |
| `none`     | State not persisted — explain to user          |

---

## What to Do

### Step 1: Receive Components to Test

The user passes component paths via prompt. Can be:
- List of file paths
- Component names
- References to specific files

Example:
```
"Test these components: apps/web/src/components/order/page-header.tsx, apps/web/src/components/order/questions-form.tsx"
```

### Step 2: Ask Required Action

Ask the user what action they need:

```
1 Explore patterns (cy-explorer)
2 Write tests (cy-tester)
3 Verify coverage (cy-verify)
4 Full workflow (explore → test → verify)
```

### Step 3: Launch Corresponding Skill

**If user selects option 1 (Explore patterns):**
Launch cy-explorer:

```typescript
// Attempt to launch as sub-agent
skill({ SkillName: "cy-explorer" })

// If fails, execute inline directly
```

cy-explorer must:
- Search for existing `cypress-best-practices` in engram
- Explore Cypress test files
- Identify patterns and best practices
- Save new practices to engram
- Show list of best practices to apply

**If user selects option 2 (Write tests):**
Launch cy-tester (one component at a time):

```typescript
// Attempt to launch as sub-agent
skill({ SkillName: "cy-tester" })

// If fails, execute inline directly
```

**Important:** Test one component at a time, not all at once unless user specifies.

cy-tester must:
- Read the component to test
- Apply best practices from engram
- Write Cypress tests following project patterns
- Save the test file alongside the component

**If user selects option 3 (Verify coverage):**
Launch cy-verify:

```typescript
// Attempt to launch as sub-agent
skill({ SkillName: "cy-verify" })

// If fails, execute inline directly
```

cy-verify must:
- Execute the created tests
- Verify component coverage
- Return coverage summary

**If user selects option 4 (Full workflow):**
For each component:
1. Launch cy-explorer (if not already done)
2. Launch cy-tester
3. Launch cy-verify
4. Save progress to engram
5. Report results

### Step 4: Save Progress to Engram

Save which components were tested:

```typescript
mem_save({
  title: "Components tested with Cypress",
  content: """
  **What**: Tested components {list of components}
  **Why**: To ensure testing coverage for React components
  **Where**: {paths of test files created}
  **Learned**: {observations about the process}
  """,
  project: "{project-name}",
  type: "pattern",
  topic_key: "cy-testing-progress",
  scope: "personal"
})
```

### Step 5: Report Results

Generate final report with:

- **Components tested**: List of components and their test files
- **Tests created**: Number of tests per component
- **Coverage** (if cy-verify was executed): Coverage summary per component
- **Best practices applied**: Reference to engram practices used
- **Next steps**: Pending components or suggested improvements

## Rules

- ALWAYS attempt to launch sub-agents first
- ALWAYS fallback to inline execution if first attempt fails
- ALWAYS test one component at a time, not all at once (unless user specifies)
- ALWAYS ask required action before launching any skill (explore, test, verify)
- ALWAYS save progress to engram
- DO NOT assume cy-tester or cy-verify are available — verify first
- DO NOT execute commands directly — delegate to the corresponding skill
- DO NOT read or write code files — delegate to sub-agents
- DO NOT perform analysis inline — delegate

## Resources

- **cy-explorer**: Skill for exploring existing testing patterns
- **cy-tester**: Skill for writing Cypress tests
- **cy-verify**: Skill for verifying test coverage
- **Engram**: Persistent memory system for best practices and progress
- **cy-phase-common**: Shared protocol for all Cypress phases
