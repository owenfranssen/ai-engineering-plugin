# AI Engineering Plugin by Owen Franssen

A collection of skills for AI-assisted engineering — built around **flow**, an orchestrator-plus-agents pattern for implementing features with parallel autonomous agents, zero trust in agent self-reporting, and per-task verification before merge.

Skills are plain markdown files. They're packaged as a [Claude Code](https://claude.ai/code) plugin for one-line install, but the files work with any AI tool that accepts system prompt or instruction files — Cursor rules, Copilot instructions, Gemini context, or plain context passed to any model.

→ [Quick start](#quick-start) · [Installation](#installation) · [Skills](#skills) · [flow in practice](#flow-in-practice)

---

## The core idea: flow

Most AI-assisted development uses a single agent that reasons through a task sequentially. Flow does something different: it breaks an implementation into discrete tasks, dispatches them to parallel agents, and independently verifies each result before squash-merging — regardless of what the agent claims it did.

```
spec / issue / description
         │
    write-plan          ← structures work into tasks with ACs and file scopes
         │
      flow              ← orchestrates agents, reviews output, squash-merges
    ┌────┴────┐
 agent 1   agent 2      ← parallel execution (non-overlapping files)
    └────┬────┘
   flow-review          ← reads the diff, not the agent's report — PASS / FAIL / FLAG
         │
        PR              ← raised only when all tasks pass independently
```

The three skills work together:

| Skill | Role |
|-------|------|
| `write-plan` | Turns a spec into a structured `plan.md` with tasks, ACs, file scopes, and verify commands |
| `flow` | Orchestrator — dispatches agents, calls `flow-review`, squash-merges verified work, raises PR |
| `flow-review` | Reads the diff against ACs — does not trust the agent's self-assessment |

### Why this architecture?

**Parallel execution.** Tasks with non-overlapping file scopes run simultaneously. A feature split into 4 independent tasks takes ~1/4 the wall-clock time of sequential execution.

**Scope containment.** Each agent gets an explicit `files-allowed` list. `flow-review` catches scope violations before they merge — agents can't contaminate sibling work.

**AC traceability.** Every task has behavioral acceptance criteria. `flow-review` checks the diff for evidence of each AC, not just the agent's self-reported summary.

**Decision surfacing.** Agents flag decisions (new deps, schema changes, security boundaries) rather than deciding silently. The orchestrator surfaces flags to the user before re-dispatching.

### Zero-trust model

Agents are optimistic narrators. Given the chance, they will:
- Report all ACs as passing when some have no implementation
- Drift outside their file scope without flagging it
- Make architectural decisions (new dep, schema change) without stopping

Flow is built on the assumption that you can't trust what an agent says it did — only what the diff shows.

`flow-review` operationalises this:
- **Reads the diff directly.** The agent's "ACs MET" section is advisory. `flow-review` cross-checks every AC against the actual code change.
- **Scope violations are mechanical FAIL.** A file outside `files-allowed` is a hard stop, not a judgment call. The rule exists precisely because agents exercise bad judgment about scope under pressure.
- **Verify runs independently.** Even if the agent ran the test suite and reported it passing, `flow-review` runs it again. A clean run by the agent is not evidence — a clean run after the fact is.
- **Decision flags are non-negotiable.** When an agent hits a flag trigger (new dep, schema change, security boundary), it stops and posts the question. The orchestrator never auto-resolves flags on the agent's behalf.

The result: you get parallel execution speed without trusting any single agent to stay in bounds.

---

## Skills

| Skill | Purpose |
|-------|---------|
| `flow` | Orchestrate multi-agent task flows with parallel execution and per-task review |
| `flow-review` | Verify agent output against ACs before squash-merge — reads diff, not agent report |
| `write-plan` | Turn a spec or GitHub issue into a structured plan.md for flow |
| `investigate` | Structured codebase investigation before implementation — traces execution paths, maps data flow |
| `code` | Project-agnostic code writer — TDD, quality gates, read-before-write |
| `context-engineering` | Structure and load project context for AI-assisted sessions |
| `documentation-and-adrs` | Write and maintain ADRs and technical documentation |
| `source-driven` | Not included — use [Addy Osmani's original](https://github.com/addyosmani/agent-skills/blob/main/skills/source-driven-development/SKILL.md) directly |

## Slash Commands

| Command | Purpose |
|---------|---------|
| `/checkpoint` | Save current task progress to `.project/current-task.md` |
| `/complete-task` | Archive a completed task |
| `/restore` | Resume session from `.project/` state files |
| `/source-driven` | Not included — see [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) |

---

## Quick start

### flow

```
/flow #123                  ← implement a GitHub issue
/flow path/to/spec.md       ← implement from a markdown spec
/flow "add CSV export"       ← implement from an inline description
/flow status                ← show in-progress flows
/flow done                  ← archive state and clean up
```

Flow writes plan state to `~/.flow/state/{id}/` — never committed to the repo.

### write-plan standalone

```
/write-plan #123            ← plan from a GitHub issue
/write-plan path/to/spec.md ← plan from a markdown file
/write-plan "description"   ← plan from inline description
```

Produces `~/.flow/state/{id}/plan.md` for review before dispatch.

---

## Installation

### Claude Code

One-line install at user scope — available across all your projects:

```bash
claude plugin add owenfranssen/ai-engineering-plugin --scope user
```

Or project-scoped (checked into the repo):

```bash
claude plugin add owenfranssen/ai-engineering-plugin
```

Skills are invoked with `/flow`, `/write-plan`, `/investigate`, etc.

### VS Code + GitHub Copilot

Copy the skill content you want into `.github/copilot-instructions.md` in your repo, or into VS Code's custom instructions (`Copilot` → `Configure` → `Custom Instructions`).

Each `SKILL.md` is self-contained — paste the content below the frontmatter directly:

```bash
# Example: add the flow skill to Copilot instructions
tail -n +6 skills/flow/SKILL.md >> .github/copilot-instructions.md
```

Copilot will follow the skill's steps when you describe the task in chat (`"implement issue #123 using flow"`).

### Cursor

Add skills as [Cursor rules](https://docs.cursor.com/context/rules) in `.cursor/rules/`:

```bash
# Copy a skill as a Cursor rule
cp skills/flow/SKILL.md .cursor/rules/flow.mdc
cp skills/write-plan/SKILL.md .cursor/rules/write-plan.mdc
```

Remove the YAML frontmatter block (`---` ... `---`) from each file — Cursor doesn't use it. Reference the rule in chat: `"use @flow to implement this"`.

### Gemini CLI

Add skills to your `GEMINI.md` file (project-level) or `~/.gemini/GEMINI.md` (user-level):

```bash
# Append a skill to project GEMINI.md
echo "" >> GEMINI.md
cat skills/flow/SKILL.md >> GEMINI.md
```

Or maintain them as separate files and reference them in `GEMINI.md`:

```markdown
# Skills
@skills/flow/SKILL.md
@skills/write-plan/SKILL.md
```

### Any other tool

Every skill is a plain markdown file. Paste the content (minus the YAML frontmatter) into whatever context window your tool uses — system prompt, instructions file, or inline context. The skills are written to be self-contained: no tool-specific syntax, no imports.

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

1. **Scope** — every file in the diff is in `files-allowed` (automatic FAIL otherwise)
2. **AC coverage** — diff contains positive implementation evidence for each AC
3. **Edge cases** — if the task named edge cases, the agent addressed them
4. **Decision flags** — any unresolved flags surface to the user before re-dispatch
5. **Verify output** — runs task verify commands independently

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

### `investigate`

Structured codebase investigation before writing a plan. Dispatches a `general-purpose` agent to trace the system — entry point to storage, through every layer — and returns structured findings: data flow, key files, edge cases, what to be careful of. For multi-service systems, dispatches one agent per service in parallel. Never produces code changes, only understanding.

### `code`

Project-agnostic code writer. Follows TDD (test first), reads existing code before writing, enforces no-duplication and type safety, runs quality gates on completion. Use for single-file changes where flow's orchestration overhead isn't warranted.

### `context-engineering`

Structures context files (CLAUDE.md, AGENTS.md) and memory for multi-session AI-assisted work. Use when agent output quality degrades — wrong patterns, hallucinated APIs, ignoring conventions. Establishes the conventions that `write-plan` captures as `project_context` and passes to every agent.

### `documentation-and-adrs`

Writes ADRs (Architecture Decision Records) when making architectural choices. Captures the decision, alternatives considered, and rationale — so future agents don't re-litigate settled decisions. Use when a design decision has come up twice or when you find unexplained code.

### `source-driven`

Not included in this repo. Use [Addy Osmani's source-driven-development](https://github.com/addyosmani/agent-skills/blob/main/skills/source-driven-development/SKILL.md) directly — it's the more complete version and worth reading.
