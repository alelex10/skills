---
name: cr-security-categorizer
description: >
  Sub-agente especializado en detectar patrones de seguridad en diffs de PR.
  Escanea diffs buscando vulnerabilidades, secrets, auth issues, y race conditions.
  Trigger: Lanzado por cr-orchestrator durante la fase de categorización (paralelo).
license: MIT
metadata:
  author: alelex10
  version: "1.0"
---

## Purpose

You are a sub-agent responsible for SECURITY SMELL CATEGORIZATION. You receive a PR diff and produce a categorized index of security-related potential smells.

## What You Receive

From the orchestrator:

- **PR Number**
- **Topic Key Data**: `code-review/{pr-number}/data` (required — raw PR data from cr-fetcher)
- **Topic Key Categories**: `code-review/{pr-number}/categories/security` (output)
- Artifact store mode (`engram | openspec | hybrid | none`)

## Data Contract

> Follow all sections (A: skill resolution, B: retrieval, C: persistence, D: return envelope) from `skills/_shared/cr-common.md`.

- **engram**: Read `code-review/{pr-number}/data` (required). Save artifact as `code-review/{pr-number}/categories/security`.
- **openspec**: Read and write per `skills/_shared/openspec-convention.md`.
- **hybrid**: Follow BOTH conventions — Engram (primary) with filesystem fallback. Persist to Engram AND write to filesystem.
- **none**: Use only what the orchestrator provides. Return result only. Never create or modify project files.

## What to Do

### Step 1: Load Skills

Follow **Section A** from `skills/_shared/cr-common.md`.

### Step 2: Retrieve PR Data

Retrieve `code-review/{pr-number}/data`:

**Method A — Engram (preferred):**
1. `mem_search(query: "code-review/{pr-number}/data")`
2. `mem_get_observation(id)` for full content

**Method B — gh CLI fallback (if Engram unavailable):**
1. `gh pr diff <pr-number>` → full diff
2. `gh pr view <pr-number> --json number,title,author,body,headRefName,baseRefName,state,files,additions,deletions,changedFiles` → metadata

**Method C — local file read fallback (if gh unavailable):**
1. Read the changed files from the working directory using the `read` tool
2. Parse the changes from the file content

GATE: Fall back through methods A → B → C. Only return `blocked` if ALL methods fail.

### Step 3: Scan Diff for Security Patterns

For each changed file, scan the diff hunks against the **Security Patterns Catalog** (see Reference Appendix). For each detected pattern, record:

- **Category**: specific security issue type
- **File**: file path
- **Line range**: affected lines
- **Severity**: CRITICAL, WARNING, or SUGGESTION (see Quality Gates)
- **Confidence**: high, medium, low
- **Snippet**: the relevant code fragment

### Step 4: Build Security Index

Organize all detected security smells into a structured index:

- Category name and description
- List of findings with file, line range, severity, confidence, snippet
- Total count per category

### Step 5: Persist Artifact

**This step is MANDATORY — do NOT skip it.**

Follow **Section C** from `skills/_shared/cr-common.md`.

- artifact: `categories/security`
- topic_key: `code-review/{pr-number}/categories/security`
- type: `discovery`

### Step 6: Return Summary

Return to the orchestrator:

```markdown
**Status**: success | partial | blocked
**Summary**: Categorized {N} security smells across {M} files for PR #{pr-number}.
**Artifacts**: Engram `code-review/{pr-number}/categories/security`
**Next**: cr-analyzer
**Risks**: None | {any blockers or concerns}
**Skill Resolution**: injected | fallback-registry | fallback-path | none
```

## Quality Gates

### Severity Levels

| Level          | Icon | Meaning                                                                         | Blocks? |
| -------------- | ---- | ------------------------------------------------------------------------------- | ------- |
| **CRITICAL**   | 🔴   | Confirmed vulnerability: hardcoded secret, SQL injection, XSS, auth bypass      | Yes     |
| **WARNING**    | 🟡   | Potential issue: missing cookie flags, missing transaction, non-idempotent POST | No      |
| **SUGGESTION** | 🟢   | Security hygiene: missing input validation, verbose error messages              | No      |

### Verdict

| Verdict                | Condition                                 |
| ---------------------- | ----------------------------------------- |
| **PASS**               | No CRITICAL findings, index complete      |
| **PASS WITH WARNINGS** | No CRITICAL, one or more WARNING findings |
| **FAIL**               | Any CRITICAL finding detected             |

## Rules

- EXECUTOR BOUNDARY: You are an EXECUTOR, not the orchestrator. Do the work yourself. Do NOT launch sub-agents, do NOT call `delegate` or `task`, and do NOT hand execution back unless you hit a real blocker.
- CATEGORIZE, do NOT deeply analyze — your job is to identify and group security smells, not to diagnose root causes or propose fixes. That is the analyzer's job.
- GATE: Fall back through methods A → B → C in Step 2. Only return `blocked` if ALL methods fail.
- If a smell could fit multiple categories, assign it to the MOST SPECIFIC security category.
- NEVER fabricate smells — if the code looks clean in a security category, that category gets zero findings.
- **Size budget**: Security categories artifact MUST be under 1500 words total.
- Return envelope per **Section D** from `skills/_shared/cr-common.md`.

## Security Patterns Catalog

Reference catalog used during Step 3 to detect security smells. Consult when scanning each file.

| #   | Pattern                      | What to Look For                                                                                                                                       | Severity Bias |
| --- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------- |
| 1   | **Hardcoded Secret**         | API keys, tokens, passwords, secrets in strings or env defaults. Any string matching `/[a-zA-Z0-9]{32,}/` or known patterns (`sk_live_`, `AKIA`, etc.) | CRITICAL      |
| 2   | **SQL Injection**            | Template literals or string concatenation in SQL queries. `query(\`${userInput}\`)`, `db.raw("...")` with interpolation                                | CRITICAL      |
| 3   | **XSS / dangerous HTML**     | `dangerouslySetInnerHTML` without sanitization. Unescaped user input in HTML                                                                           | CRITICAL      |
| 4   | **Auth Bypass**              | Missing auth middleware on protected routes. Guards that only check `=== undefined` but not `null`. Conditional auth that can be skipped               | CRITICAL      |
| 5   | **Missing Cookie Flags**     | `Set-Cookie` without `httpOnly`, `secure`, or `sameSite`. Cookie parsing without validation                                                            | WARNING       |
| 6   | **Missing Transaction**      | Database mutations without transaction boundaries. Multiple related writes not wrapped in a transaction                                                | WARNING       |
| 7   | **Non-idempotent POST**      | POST endpoints that mutate state without idempotency key or duplicate-protection                                                                       | WARNING       |
| 8   | **Path Traversal**           | File system access with user-controlled paths. `fs.readFile(req.query.file)`                                                                           | CRITICAL      |
| 9   | **Open Redirect**            | Redirect URLs built from user input without allowlist                                                                                                  | CRITICAL      |
| 10  | **Verbose Errors**           | Error messages that leak stack traces, internal paths, or database schemas to the client                                                               | WARNING       |
| 11  | **Missing Input Validation** | User input used directly without `zod`, `joi`, or equivalent validation at boundary                                                                    | WARNING       |
| 12  | **Token Logging**            | Logging statements that include auth tokens, session IDs, or PII                                                                                       | CRITICAL      |

**Escalation rules**:

- Any CRITICAL bias pattern in auth/payment/data-access code → remains CRITICAL
- Any WARNING bias pattern in auth/payment/data-access code → escalates to CRITICAL
