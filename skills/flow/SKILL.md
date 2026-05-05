---
name: flow
description: "Use when implementing a GitHub issue, markdown spec, or inline description through an orchestrator-plus-agents architecture — dispatches tasks in parallel, reviews each agent's output, squash-merges to the feature branch, and raises a PR when all tasks pass. Trigger: flow, run this in agents, dispatch agents, implement with agents, orchestrate this, flow issue 123."
---

# flow

**Recommended Model:** sonnet (orchestrator); agents inherit their own model context.

## Purpose

Implements a plan by dispatching tasks to autonomous subagents, reviewing each completion, squash-merging verified work to the feature branch, and raising a PR. Uses `write-plan` to produce the plan before dispatch.

**Use for:** multi-task features where parallel execution and AC coverage review add value.  
**Don't use for:** single-file changes with no ACs — just do them directly.

## Invocation

```
/flow #123
/flow path/to/spec.md
/flow "build a CSV export for the reports page"
/flow done          → archive state + clean up
/flow status        → show in-progress flow
/flow abort         → cancel and clean up
```

---

## Orchestrator Loop

### Step 0 — Autonomous check

```bash
echo "${ZARK_AUTONOMOUS:-0}"
```

- **Output `1`** → proceed.
- **Output `0`** → warn:

  > ⚠️  **flow works best in an autonomous session** — approval prompts will interrupt git operations and agent dispatch.
  >
  > Restart with: `claude-task autonomous`, then re-invoke.
  >
  > Reply **"continue"** to proceed in this session anyway.

  Wait for user reply before continuing.

### Step 1 — Plan

Invoke `write-plan` with the input. This produces:

```
~/.flow/state/{id}/
├── plan.md       — phases + tasks
├── meta.json     — id, branch, project_context
└── progress.md   — event log (orchestrator appends)
```

If a plan already exists for `{id}` (resuming), read it and skip to Step 2.

### Step 2 — Branch

```bash
git checkout -b {branch-name}
# branch name: flow/{id} unless user specifies one
```

Log to progress.md:
```
[{ISO timestamp}] [PHASE 1] Starting — N tasks
```

### Step 3 — Dispatch tasks

For each phase, identify tasks that can run in parallel (non-overlapping `files-allowed`, no unresolved dependencies).

**For each parallel group:**

Dispatch each task as a subagent using the Agent tool. Pass the agent prompt below. Collect all results before proceeding to review.

**Agent prompt template:**
```
You are implementing task {task.id} — "{task.title}" for project "{project_context.name}".

## Project context
- Stack: {project_context.framework}
- Working dir: {project_context.working_dir}
- Hard conventions:
{project_context.conventions as bullets}

## Your task
{task.outcome}

## Acceptance criteria
{task.acceptance as bullets}

## Read these files first (in order)
{task.read-first — each entry with its why annotation}

## Files you may modify
{task.files-allowed as bullets}

{if task.do-not-touch}
## DO NOT touch
{task.do-not-touch as bullets}
{endif}

{if task.expected-state-after}
## Expected state after this task
{task.expected-state-after — intentional broken state; do not "fix" it}
{endif}

{if task.pre-authorized-exceptions}
## Pre-authorized exceptions (not flag-worthy)
{task.pre-authorized-exceptions}
{endif}

{if task.requires-env}
## Required env vars
{task.requires-env}
{endif}

{if task.edge-cases}
## Edge cases that MUST be present in your output
{task.edge-cases}
{endif}

## Decision flags — MUST flag (stop + post decision) if:
- New dependency needed (package.json / pyproject.toml / etc.)
- Refactor required outside files-allowed
- Schema change not in this plan
- Auth or security boundary change
- Public API contract change
{if task.flag-decisions}{task.flag-decisions}{endif}

## Decision flags — MAY assume (no stop needed):
- File/function naming — follow CLAUDE.md or existing patterns
- Test structure — follow existing patterns
- Code style — defer to linter/formatter

## On completion, report:
1. SUMMARY: one paragraph of what you did
2. FILES CHANGED: list with brief note per file
3. ACs MET: each AC with pass/fail/partial + evidence
4. DECISION FLAGS: any stops you hit and how resolved
5. VERIFY RESULT: output of each verify command
{if task.output-fields}6. OUTPUT FIELDS: {task.output-fields}{endif}

Work on the current branch — do NOT push or create a PR.
```

Log to progress.md:
```
[{timestamp}] [DISPATCH] Task {id}: "{title}" — agent started
```

### Step 4 — Review each completed task

After each agent returns, invoke `flow-review` passing:
- The task spec (from plan.md)
- The agent's report
- The diff: `git diff {base-branch}...HEAD -- {files-allowed}`

If `flow-review` returns **PASS**:
```bash
# Squash-merge the agent's work to the feature branch
git add {files-allowed}
git commit -m "{task.id}: {task.title}"
```
Log: `[{timestamp}] [MERGE] Task {id}: verified + merged`

If `flow-review` returns **FAIL** (scope violation or critical AC miss):
Log: `[{timestamp}] [FAIL] Task {id}: {reason}`
Re-dispatch the task with the failure reason appended to the agent prompt. Max 2 retries per task; escalate to user on third failure.

If `flow-review` returns **FLAG** (decision needed):
Log: `[{timestamp}] [FLAG] Task {id}: {decision question}`
Surface to user, get answer, update plan, re-dispatch.

### Step 5 — Run verify after each merged task

Run the task's `verify` commands:

```bash
# Each verify entry in the task is one of:
# test:<pattern>   → npm test -- --testPathPattern=<pattern>  (or project-specific)
# lint             → npm run lint (or project-specific)
# typecheck        → npm run typecheck (or project-specific)  
# cmd:<command>    → run the command verbatim
```

If verify fails: log `[VERIFY-FAIL]` with last 20 lines of output, re-dispatch a corrective task scoped to the failure.

### Step 6 — After all tasks complete

Run the full test suite once:

```bash
# Auto-detect from CLAUDE.md or package.json scripts
npm test   # or equivalent
```

If tests pass: proceed to PR.  
If tests fail: surface to user with output; do not create PR until passing.

### Step 7 — Raise PR

Before creating the PR, assess size. If the diff exceeds **20 non-test application files** or **800 net line additions**, pause and present the user with Continue / Split / Review options. Do not auto-proceed.

```bash
# Count changed app files (exclude tests, migrations, planning)
git diff --name-only main...HEAD \
  | grep -vE '(migrations/|__tests__|\.test\.|\.spec\.|snapshots/|\.flow/)' \
  | wc -l

# Net line changes
git diff --stat main...HEAD | tail -1
```

**PR body template.** Every feature PR follows this structure. Omit sections with no content rather than leaving them empty or writing "N/A".

```markdown
## Summary

{1–3 sentences of intent.}

{If the PR introduces no behaviour change — refactors, scaffolding, tooling: say "Zero app behaviour change." explicitly.}

See the plan: ~/.flow/state/{id}/plan.md — this PR delivers the "{scope}" section.

## Commits

{git log --oneline main..HEAD — short SHA + subject per line}

## Key decisions worth a second look

{One subsection per non-obvious decision. Rules: name the alternative, explain why this was the minimal correct fix, use ⚠️ on decisions the reviewer should scrutinise (dep additions, shared-config changes, diverging from plan). Do not list mechanical choices — file naming, test layout, style — those belong in commit messages.

If the PR has no non-obvious decisions, omit this entire section. A missing section is better than a "nothing here" section.}

### 1. {Decision title} ⚠️
{What was chosen, what the alternative was, and why this is the minimal correct fix. 2–4 sentences.}

### 2. {Decision title}
{Same structure.}

## {Baselines / Metrics — optional}

{Include only if this PR captures a baseline for later enforcement (coverage, bundle size, perf). Use tables for anything numeric. Omit otherwise.}

## Acceptance criteria (verbatim from plan)

{Copy AC bullets from plan.md verbatim — do not paraphrase. Use `- [x]` for ACs verified locally with a specific command; `- [ ]` for ACs that still need preview URL, CI, or reviewer spot-check.}

- [x] {AC verified locally — mention how}
- [ ] {AC needing preview/CI verification}

## Test plan

{Checklist of verifications the reviewer runs — things CI doesn't cover. 3–6 items typical.}

- [ ] Preview build succeeds
- [ ] CI green on all lanes
- [ ] Fresh-clone install works on reviewer's machine

## Next in the queue — optional

{Pointer to follow-up work so this PR isn't read in isolation. One sentence. Omit if there's no follow-up.}

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

**Writing discipline for Key decisions:** flag with ⚠️ when the decision adds a dep, touches shared config, diverges from the plan, or has a cheaper alternative the reviewer might wonder about. Always name the alternative and say why it was rejected. If scope extended beyond the task's files-allowed, say so.

**Writing discipline for Acceptance criteria:** copy verbatim. Paraphrasing breaks reviewer trace back to the source.

**Writing discipline for Test plan:** describe what the *reviewer* does, not what the author already verified. Preview URLs, fresh-clone installs, spot checks — things CI can't confirm alone.

**Create the PR:**

```bash
PR_BODY=$(cat <<'EOF'
{body content following template above}
EOF
)

gh pr create \
  --base main \
  --title "{plan title}" \
  --body "$PR_BODY"
```

Log: `[{timestamp}] [DONE] PR raised: {url}`

### Step 8 — `/flow done`

Archive state and clean up:

```bash
mkdir -p ~/.flow/archive
mv ~/.flow/state/{id} ~/.flow/archive/{id}
```

Print: "Flow `{id}` archived. Branch `{branch}` ready for review."

---

## `/flow status`

Read `~/.flow/state/*/progress.md` for all active flows. Print a summary table:

```
ID           Branch              Tasks    Last event
issue-42     flow/issue-42       3/5      [MERGE] Task 2.1: verified + merged
```

---

## `/flow abort`

Ask for confirmation, then:

```bash
mv ~/.flow/state/{id} ~/.flow/archive/{id}-aborted-{timestamp}
git checkout main
```

---

## Common Rationalizations

Excuses to resist when orchestrating — each paired with the factual counter.

| Excuse | Counter |
|--------|---------|
| "The agent's output looks reasonable — I'll skip flow-review." | Flow-review catches scope violations and AC misses that look fine in prose. Always invoke it. |
| "This task failed verify but the failure looks minor — I'll merge anyway." | A failing verify is a broken tree. Fix it before merging; the next task builds on a broken base if you don't. |
| "Verify passed — I'll run it again to be sure." | A clean pass is authoritative. Only re-run if code changed since that run. Re-running after a pass wastes time and adds noise. |
| "I'll dispatch all tasks at once, including dependent ones." | Dependencies exist because task B reads files task A writes. Dispatch only tasks with satisfied dependencies. |
| "The PR is ready — I'll skip the full test suite." | Per-task verify is narrow by design. The full suite catches cross-task regressions that narrow verify misses. |
| "The agent hit a decision flag but it seems obvious — I'll resolve it myself." | Decision flags surface to the user. You may suggest an answer but the user confirms. Flags exist because agents get scope wrong under pressure. |

## Red Flags

Signals that mean STOP and re-check this skill:

- About to merge a task without running flow-review
- About to create a PR with a failing test suite
- About to dispatch a task whose dependency hasn't merged yet
- Resolving a decision flag without surfacing it to the user
- More than 2 retries on the same task without asking the user

**All of these mean: stop, re-read the relevant step above, fix the gap before continuing.**
