---
name: investigate
description: "Use when codebase understanding is needed before implementation — traces execution paths, maps data flow, identifies edge cases, and produces a structured summary. Trigger: how does X work, understand this, where is Y, trace this flow, what calls what, investigate before implementing."
---

# investigate — Codebase Investigation

**Recommended Model:** sonnet

Structured codebase investigation that produces understanding, not code. Dispatches a general-purpose investigation agent to trace the system, then returns a focused summary.

**Use when:** scope is unfamiliar, spans multiple services or layers, or research is needed before writing a plan
**Output:** structured summary — not a fix, not a refactoring suggestion

---

## Step 1: Scope the Investigation

Before dispatching, confirm:
- **What to understand**: the feature, flow, or system area
- **Entry point**: where does it start? (HTTP route, queue event, cron job, CLI command, API endpoint)
- **Services/packages involved**: which directories or packages are in scope
- **Depth needed**: full trace (entry → DB) or focused (just one layer)?

If the question is ambiguous, ask before dispatching.

**If more than 2 independent areas are involved:** dispatch one investigation agent per area in parallel (using `superpowers:dispatching-parallel-agents`), then merge findings before presenting. Don't attempt a single-agent trace across all areas — it will be shallow.

## Step 2: Dispatch Investigation Agent

For a single area, dispatch one `general-purpose` agent:

```
Agent(
  description: "Investigate {TOPIC} in codebase",
  subagent_type: "general-purpose",
  prompt: "
Investigate how {TOPIC} works in this codebase. Produce structured findings — no code changes, no refactoring suggestions.

Entry point hint: {ENTRY_POINT_IF_KNOWN}
Directories to search: {DIRECTORIES}

## Investigation steps

1. Locate entry points — Grep/Glob for the feature across the listed directories
2. Consult project documentation if available:
   - Architecture docs, glossary, or README files that describe the system
   - OpenAPI/Swagger specs for API surface
   - ADRs (Architecture Decision Records) for why decisions were made
3. Trace execution path — handler → service → model → storage (following actual function calls, not guessing)
4. Read existing tests — understand expected behaviour and edge cases from test files
5. Check git history — run: git log --follow --oneline {key_file} to understand why decisions were made
6. If search returns no results for any layer: state 'Not found' — do not infer or guess the path based on what you'd expect to exist

Return structured findings:
- What it does (1 paragraph)
- Data flow: entry point file → intermediate layers (file names) → persistence layer
- Key files: path + one-line purpose each
- Edge cases and gotchas discovered from tests or code
- What to be careful of if changing this system
- Framework patterns that should be verified against official docs (list file:line and pattern name)
  "
)
```

**For multiple independent areas** — dispatch in parallel:

Use `superpowers:dispatching-parallel-agents` to dispatch one Agent per area simultaneously. Each agent gets its own scoped prompt. Collect all results before presenting findings.

## Step 3: Augment With Log Data (Optional)

If understanding runtime behaviour would add value — e.g. "what errors does this actually produce in production?":

Dispatch an additional agent alongside or after the code investigation, focused on log analysis:
- Application logs, error logs, or observability data for the feature area
- Time window: last 24 hours unless otherwise specified
- Focus: what errors appear, what entity IDs are affected, volume trend

Only worth doing when the feature's runtime behaviour is meaningfully different from what the code alone shows (e.g. async failures, race conditions, edge-case errors).

## Step 4: Present Findings

Format and return findings to the caller. Structure:

```markdown
## Investigation: {TOPIC}

**What it does**
{1 paragraph}

**Data flow**
{entry_file} → {service_file} → {model_file} → {storage_layer}

**Key files**
- `{path}` — {one-line purpose}
- `{path}` — {one-line purpose}

**Edge cases / gotchas**
- {gotcha 1}
- {gotcha 2}

**What to be careful of**
{paragraph}

**Patterns to verify against docs**
- `{file:line}` — uses `{framework_api}` pattern
```

When called by `write-plan` or `flow`: the "Key files" and "What to be careful of" sections inform `read-first` entries and risk notes in the plan. When called directly: output is a standalone summary for the user.

## Common Rationalizations

| Excuse | Counter |
|--------|---------|
| "I know roughly how this works — I'll skip investigation." | "Roughly" is the source of most scope violations. Investigation takes 5 minutes; a wrong plan takes hours to undo. |
| "The grep returned nothing — I'll infer the path." | No results means not found. State it. Guessing a path produces confident-sounding wrong plans. |
| "I'll read the files myself instead of dispatching an agent." | For unfamiliar multi-layer systems, agent dispatch produces broader, more parallel coverage. |

## Red Flags

- About to guess an execution path not found in grep/glob results
- About to present findings from only one layer (entry point but no service/model trace)
- About to suggest code changes — this skill produces understanding only

**All of these mean: stop, re-check the relevant step above, fix the gap before returning findings.**
