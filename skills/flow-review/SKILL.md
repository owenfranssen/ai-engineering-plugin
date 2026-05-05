---
name: flow-review
description: Use when flow's orchestrator needs to verify an agent's task completion before squash-merging — checks AC coverage against the agent's diff and report. Called by flow; not typically invoked directly. Called by flow; not typically invoked directly.
---

# flow-review

**Recommended Model:** sonnet — AC coverage requires reading code and evaluating evidence.

## Purpose

Verify an agent's task completion before the orchestrator squash-merges. Returns a structured verdict: PASS, FAIL, or FLAG.

This is an internal skill. The `flow` orchestrator calls it after each agent returns. It is not a general-purpose code reviewer — scope is limited to the specific task's ACs and files-allowed.

## Input

Called with three pieces of context:

1. **Task spec** — the full task block from `plan.md` (ACs, files-allowed, verify commands, edge-cases)
2. **Agent report** — the agent's SUMMARY, FILES CHANGED, ACs MET, DECISION FLAGS, VERIFY RESULT
3. **Diff** — `git diff {base}...HEAD -- {files-allowed}` for the agent's work

## Review process

### Check 1 — Scope guard

Verify that every file in the diff is in `files-allowed` (or a subdirectory of an allowed dir).

Any file outside `files-allowed` is a **scope violation** → immediate FAIL.

Exception: test files co-located with allowed source files are permitted without explicit listing.

### Check 2 — AC coverage

For each acceptance criterion in the task:

- Read the diff for implementation evidence
- Read the agent's "ACs MET" section for their self-assessment
- Classify:
  - **pass** — diff contains clear implementation + agent confirms
  - **fail** — no implementation evidence in diff, or agent marks fail
  - **partial** — some evidence but AC cannot be fully verified from diff alone

If any AC is **fail**: verdict is FAIL.  
If any AC is **partial**: run the task's `verify` commands (if not already run by agent) to resolve.

### Check 3 — Edge cases (if present)

If the task has `edge-cases`, verify that the agent's report explicitly addresses each numbered case. Any unaddressed edge case → FAIL.

### Check 4 — Decision flags

If the agent's report lists unresolved DECISION FLAGS: verdict is FLAG (surface to orchestrator for user resolution).

### Check 5 — Verify output

If the agent reported verify results, check for failures. A failing verify → FAIL.

If the agent did not run verify, run the task's verify commands now:

```bash
# Per the verify field entries in the task spec
# test:<pattern>  → npm test -- --testPathPattern=<pattern>
# lint            → npm run lint
# typecheck       → npm run typecheck
# cmd:<command>   → run verbatim
```

## Output

Return structured JSON:

```json
{
  "verdict": "PASS | FAIL | FLAG",
  "scope_violation": false,
  "acs": [
    { "ac": "WHEN user submits form THEN...", "result": "pass", "evidence": "reservations-service.ts:42" },
    { "ac": "Error is logged", "result": "fail", "evidence": null }
  ],
  "edge_cases": [
    { "case": "Orphaned ticket", "addressed": true }
  ],
  "flags": [],
  "verify_passed": true,
  "reason": "One AC has no implementation evidence in the diff."
}
```

**Verdict rules:**
- `PASS` — all ACs pass or partial-resolved-by-verify, no scope violation, no unresolved flags, verify passing
- `FAIL` — any AC fail, scope violation, or verify failure
- `FLAG` — unresolved decision flags that need user input (fix ACs and scope first; flags only if those pass)

## Common Rationalizations

Excuses to resist when reviewing — each paired with the factual counter.

| Excuse | Counter |
|--------|---------|
| "The AC seems implied by the surrounding code — I'll mark it pass." | Pass requires positive evidence in the diff. "Implied" is partial. Run verify or mark partial. |
| "The file is just outside files-allowed by one directory level — I'll let it through." | Scope violations are automatic FAIL regardless of severity. The orchestrator needs this to be mechanical. |
| "The agent said all ACs pass in their report — I'll trust it." | The agent's self-assessment is advisory. Cross-check against the diff. Agents are optimistic. |

## Red Flags

- About to return PASS when any AC has no diff evidence (mark partial, run verify)
- About to ignore a file outside files-allowed (scope violation = FAIL, always)
- About to return PASS without checking the verify output

**All of these mean: stop, re-check the relevant step, fix before returning verdict.**
