# owen-plugin

A Claude Code plugin for AI-assisted engineering — built around **flow**, an orchestrator-plus-agents pattern for implementing features with parallel autonomous agents.

## The core idea: flow

Most AI-assisted development uses a single agent that reasons through a task sequentially. Flow does something different: it breaks an implementation into discrete tasks, dispatches them to parallel agents, reviews each result before merging, and raises a PR when all tasks pass verification.

```
spec / issue / description
         │
    write-plan          ← structures work into tasks with ACs and file scopes
         │
      flow              ← orchestrates agents, reviews output, squash-merges
    ┌────┴────┐
 agent 1   agent 2      ← parallel execution (non-overlapping files)
    └────┬────┘
   flow-review          ← AC coverage check + scope guard before each merge
         │
        PR              ← raised when all tasks pass
```

The three skills work together:

| Skill | Role |
|-------|------|
| `write-plan` | Turns a spec into a structured `plan.md` with tasks, ACs, file scopes, and verify commands |
| `flow` | Orchestrator — dispatches agents, calls `flow-review`, squash-merges verified work, raises PR |
| `flow-review` | Reviews each agent's diff against ACs before merge — returns PASS / FAIL / FLAG |

### Why this architecture?

**Parallel execution.** Tasks with non-overlapping file scopes run simultaneously. A feature split into 4 independent tasks takes ~1/4 the wall-clock time of sequential execution.

**Scope containment.** Each agent gets an explicit `files-allowed` list. `flow-review` catches scope violations before they merge — agents can't contaminate sibling work.

**AC traceability.** Every task has behavioral acceptance criteria. `flow-review` checks the diff for evidence of each AC, not just the agent's self-reported summary.

**Decision surfacing.** Agents flag decisions (new deps, schema changes, security boundaries) rather than deciding silently. The orchestrator surfaces flags to the user before re-dispatching.

---

## Skills

| Skill | Purpose |
|-------|---------|
| `flow` | Orchestrate multi-agent task flows with parallel execution and per-task review |
| `flow-review` | Review agent output against ACs before squash-merge |
| `write-plan` | Turn a spec or GitHub issue into a structured plan.md for flow |
| `code` | Project-agnostic code writer — TDD, quality gates, read-before-write |
| `context-engineering` | Structure and load project context for AI-assisted sessions |
| `documentation-and-adrs` | Write and maintain ADRs and technical documentation |
| `source-driven` | Verify framework/library patterns against official docs before writing code |

## Slash Commands

| Command | Purpose |
|---------|---------|
| `/checkpoint` | Save current task progress to `.project/current-task.md` |
| `/complete-task` | Archive a completed task |
| `/restore` | Resume session from `.project/` state files |
| `/source-driven` | Check framework/library docs before implementing a pattern |

---

## Quick start

### Using flow

```
/flow #123                  ← implement a GitHub issue
/flow path/to/spec.md       ← implement from a markdown spec
/flow "add CSV export"       ← implement from an inline description
/flow status                ← show in-progress flows
/flow done                  ← archive state and clean up
```

Flow writes plan state to `~/.flow/state/{id}/` — never committed to the repo.

### Using write-plan standalone

```
/write-plan #123            ← plan from a GitHub issue
/write-plan path/to/spec.md ← plan from a markdown file
/write-plan "description"   ← plan from inline description
```

Produces `~/.flow/state/{id}/plan.md` for review before dispatch.

---

## Installation

```bash
claude plugin add owenfranssen/owen-plugin
```

Or to install at user scope (survives across all repos):

```bash
claude plugin add owenfranssen/owen-plugin --scope user
```

---

## flow in practice

### The plan format

`write-plan` produces a structured `plan.md` where each task specifies:

- **outcome** — one sentence describing the behavioral change
- **acceptance** — `WHEN <precondition> THEN <observable outcome>` format
- **files-allowed** — explicit list of files the agent may touch (scope guard)
- **read-first** — ordered files the agent must read before writing, each with a `— why` annotation
- **verify** — commands the orchestrator runs after each merge

### The review loop

For every completed task, `flow-review` checks:

1. **Scope** — every file in the diff is in `files-allowed`
2. **AC coverage** — diff contains implementation evidence for each AC
3. **Edge cases** — if the task named edge cases, the agent addressed them
4. **Decision flags** — any unresolved flags surface to the user
5. **Verify output** — runs task verify commands if the agent didn't

Returns `{ "verdict": "PASS | FAIL | FLAG", "acs": [...], "reason": "..." }`.

### Decision flags

Agents stop and post a decision question when they hit:
- A new dependency (package.json / pyproject.toml)
- A refactor outside `files-allowed`
- A schema change not in the plan
- An auth or security boundary change
- A public API contract change

The orchestrator surfaces these to the user before re-dispatching. Agents never decide these silently.

### PR body

When all tasks pass, flow raises a PR with:
- Summary of intent
- Commit list (`git log --oneline`)
- Key decisions worth a second look (with alternatives named)
- ACs copied verbatim from the plan
- Test plan for the reviewer

---

## Other skills

### `code`

Project-agnostic code writer. Follows TDD (test first), reads existing code before writing, enforces no-duplication and type safety, runs quality gates on completion. Use for single-file changes where flow's orchestration overhead isn't warranted.

### `context-engineering`

Structures context files (CLAUDE.md, AGENTS.md) and memory for multi-session AI-assisted work. Use when agent output quality degrades — wrong patterns, hallucinated APIs, ignoring conventions. Establishes the conventions that `write-plan` captures as `project_context` and passes to every agent.

### `documentation-and-adrs`

Writes ADRs (Architecture Decision Records) when making architectural choices. Captures the decision, alternatives considered, and rationale — so future agents don't re-litigate settled decisions. Use when a design decision has come up twice or when you find unexplained code.

### `source-driven`

Verifies framework and library usage against official docs for the installed version before writing code. LLM training data goes stale; this skill enforces a "check the source" discipline. Use when implementing any version-specific API.
