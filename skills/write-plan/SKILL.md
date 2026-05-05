---
name: write-plan
description: "Use when a GitHub issue, markdown spec, or inline description needs to become a structured plan.md before flow dispatches agents — or when authoring an implementation plan for a feature/script/skill build. Trigger: write plan, plan this, draft plan, plan from issue, plan implementation, write implementation plan."
---

# write-plan

**Recommended Model:** sonnet — planning involves judgment, codebase context, and clarification Q&A.

## Purpose

Turn a spec (GitHub issue, markdown file, or inline description) into a single `plan.md` artifact. Used two ways:

- **Runtime:** Consumed by `flow`'s orchestrator to dispatch agents and verify their work.
- **Authoring:** Used standalone to plan a feature or script before writing any code.

Both modes produce the same artifact. Plans are ephemeral — never committed.

## When NOT to use

- Trivial single-file change with one obvious outcome — just do it
- Pure research with no implementation outcome — just explore
- Re-planning an in-flight `{id}` without explicit user instruction — see Step 1

## Inputs

| Input kind | How to read it | `{id}` derivation |
|------------|---------------|-------------------|
| GitHub issue | `gh issue view {number} --json title,body,comments` | `issue-{number}` |
| Markdown file | Read the file directly | Filename slug |
| Inline spec | Use as-is; ask ≤2 clarifying questions for critical gaps | Generated slug from first 3-5 words |

## Output location

`~/.flow/state/{id}/plan.md` alongside `meta.json` and `progress.md`. Archived to `~/.flow/archive/{id}/` on `/flow done` or `/flow abort`. Never committed.

## Process

### Step 1 — Identify input + check for collisions

Derive `{id}` per the table above.

```bash
ls ~/.flow/state/{id}/ 2>/dev/null && echo "STATE EXISTS"
```

If state already exists, ask: **resume**, **archive existing + start fresh**, or **abort**. Never silently overwrite.

### Step 2 — Ingest spec

- **GitHub issue:** `gh issue view {number} --json title,body,comments` in the project directory. Capture title, description, and any acceptance criteria in comments.
- **Markdown file:** Read the file. Identify phase boundaries and AC sections.
- **Inline:** Parse the user's text. Identify outcomes and constraints.

### Step 3 — Clarify only critical gaps

Ask ONLY if one of these is missing:
- No clear behavioral outcome ("what should change?")
- No identifiable target (no file/component/system named or inferable)
- Conflicting constraints

Do NOT ask about file naming, test structure, internal data shapes, or code style — defer to CLAUDE.md or existing patterns.

Cap clarifications at 2 questions. If more are needed, the spec isn't ready — say so and stop.

### Step 4 — Capture project context

Read repo-root `CLAUDE.md` (and `AGENTS.md` or `.cursorrules` if present). Capture orientation that every agent prompt will receive:

- **Project name** (from CLAUDE.md or `package.json`)
- **Framework / runtime** (e.g. "Next.js 14 App Router, TypeScript strict")
- **Storage / tooling** (e.g. "Drizzle + Postgres", "Vite + Vitest")
- **Working directory** (absolute path)
- **Hard conventions** (3-5 bullets — things that cause review blockers if violated)

If CLAUDE.md is missing or thin, ask one targeted question: "What's the one-paragraph orientation a new contributor would need?" Store in `meta.json` as `project_context`.

### Step 5 — Decide phase structure

| Spec shape | Phases |
|-----------|--------|
| Single issue, single area, ≤5 ACs | 1 phase |
| Single issue, multi-area OR ≥5 ACs | Multi-phase by area |
| Markdown spec with explicit phases | Follow the spec's structure |

Do NOT introduce phases the spec doesn't motivate.

### Step 6 — Decompose each phase into tasks

A task is a unit of work that:
- Touches a small, identifiable set of files (typically 1-5)
- Has one clear behavioral outcome
- Can be completed by an autonomous agent in one session
- Is independently verifiable

**Heuristics:**
- If the task title contains "and", it's probably two tasks
- Tasks within a phase should be parallel-safe where possible (non-overlapping `files-allowed`)
- Mark dependencies explicitly so the orchestrator can sequence vs parallelise

### Step 7 — Fill the task schema

**Required fields:**

| Field | Purpose |
|-------|---------|
| `id` | `phase-N.task-M` (e.g. `1.2`) |
| `title` | Short imperative sentence |
| `outcome` | One sentence — what the agent delivers |
| `acceptance` | Bulleted ACs. Behavioral: `WHEN <precondition> THEN <observable outcome>`. Non-behavioral: plain bullets. |
| `files-allowed` | Explicit allow-list of files/dirs the agent may modify (scope guard) |
| `read-first` | Ordered list of files the agent MUST read first, each with a `— why` annotation. High-level context first, then specific files. |
| `dependencies` | Task IDs that must complete first |
| `verify` | Commands the orchestrator runs after each agent completes |

**Optional fields** (omit unless they apply):

| Field | When to include |
|-------|----------------|
| `do-not-touch` | When sibling tasks own files this agent might drift into |
| `expected-state-after` | When this task intentionally leaves something broken that a later task fixes |
| `pre-authorized-exceptions` | Pre-resolved decisions that would otherwise need a flag stop |
| `output-fields` | Specific facts to extract from the agent's report |
| `requires-env` | Env vars the task needs + fallback if missing |
| `edge-cases` | Specific scenarios that MUST be present in the output |
| `size` | `XS` / `S` / `M` / `L` — never `XL` (split instead) |
| `coordination-note` | One-liner about sibling/successor tasks |

### Step 8 — Decision-flag defaults (all tasks inherit)

**MUST flag** (agent stops and posts a decision question):
- New dependency (any new entry in `package.json`, `pyproject.toml`, etc.)
- Any refactor outside `files-allowed`
- Schema change not specified in the plan
- Auth or security boundary change
- Public API contract change

**MAY assume** (agent decides without flagging):
- File and function naming — follow CLAUDE.md or existing patterns
- Test names and structure — follow existing patterns
- Code style — defer to linter/formatter
- Internal variable names and data structures

### Step 9 — Write plan.md, meta.json, progress.md

Write all three to `~/.flow/state/{id}/`. `progress.md` starts empty; the orchestrator appends to it.

`meta.json`:
```json
{
  "id": "{id}",
  "kind": "issue|markdown|inline",
  "source": "#{issue-number} | path/to/spec.md | inline",
  "branch": "<feature-branch>",
  "machine": "<hostname>",
  "created": "<ISO timestamp>",
  "plan_version": 1,
  "project_context": {
    "name": "<project name>",
    "framework": "<one-line framework/runtime/storage>",
    "working_dir": "<abs path>",
    "conventions": ["<bullet>", "<bullet>"]
  }
}
```

### Step 10 — Confirm with user

Show the plan inline. Ask:

> **"Plan ready. Ship it / refine / cancel?"**

This is the one blocking gate. If user says "refine", edit in place and re-confirm (max 3 iterations — after that, ask if the spec itself needs work).

## plan.md template

````markdown
# Plan: {title}

**ID:** {id}
**Kind:** {issue|markdown|inline}
**Source:** #{issue-number} | path/to/spec.md | "inline"
**Branch:** {feature-branch}
**Created:** {ISO timestamp}

## Project Context

- **Project:** {name}
- **Stack:** {framework / runtime / storage in one line}
- **Working dir:** {absolute path}
- **Hard conventions:**
  - {bullet from CLAUDE.md}
  - {bullet}

## Goal

{1-3 sentence statement of what done looks like}

## Phases

1. {Phase 1 name}

## Decision-Flag Defaults

- MUST flag: new dependencies, out-of-scope refactor, unspecified schema change, security boundary change, public API contract change
- MAY assume: naming, test structure, code style, internal data shapes

---

## Phase 1 — {name}

**Outcome:** {what this phase delivers}

### Task 1.1 — {title}

**Outcome:** {one-sentence behavior change}

**Acceptance:**
- WHEN {precondition} THEN {observable outcome}
- {non-behavioral bullet}

**Read first (in order):**
- `path/to/spec.md §N` — explains the goal
- `path/to/schema.ts` — current data shape
- `path/to/related-lib.ts` — pattern to follow

**Files allowed:**
- `path/to/file-a.ts`
- `path/to/dir/**`

**Dependencies:** none

**Size:** S

**Verify:**
- `npm test -- --testPathPattern=path/to/foo.test.ts`
- `npm run typecheck`

---

## Progress

(maintained by flow at runtime; do not edit by hand)

| Task | Status | Started | Completed |
|------|--------|---------|-----------|
| 1.1 | pending | — | — |
````

## Common Rationalizations

Excuses to resist when writing a plan — each paired with the factual counter.

| Excuse | Counter |
|--------|---------|
| "The spec has gaps but I can fill them with reasonable defaults." | Gaps become scope violations during execution. Cap clarifications at 2; if more are needed, the spec needs work — say so. |
| "This is a big task but splitting it would be overhead." | Any XL task MUST be split. If the title contains "and", it's two tasks. |
| "The agent will figure out what to read on its own." | Without ordered `read-first`, agents miss context and produce off-pattern code. Mandatory. |
| "Files-allowed is enough — I don't need do-not-touch." | Sibling-owned files need explicit naming with reason. Implicit forbidden ≠ explicit forbidden. |
| "The edge cases are obvious from the ACs." | Edge cases are deliberate scenarios that must be present. Calling them out prevents skipping as "polish". |

## Red Flags

Signals that mean STOP and re-check this skill:

- Writing plan tasks with titles containing "and" (split them)
- Asking more than 2 clarifying questions (spec needs work, not more Q&A)
- Including empty optional fields (pad — omit fields that don't apply)
- Plan state already exists for `{id}` and you didn't ask before overwriting
- `verify` field is empty for any task

**All of these mean: stop, re-read the relevant step above, fix the gap before continuing.**
