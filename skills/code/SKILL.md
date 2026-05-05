---
name: code
description: "Use for any code writing, editing, fixing, or refactoring task — follows TDD, reads before writing, enforces no-duplication, security, and type safety, and runs quality gates. Project-agnostic. Trigger: write code, implement, add feature, fix bug, refactor, create component, update service, add endpoint, change field, edit this file, add validation, create a hook, write a migration, update the model, update the schema, fix the test, add this logic, write the service, apply this change, change this function, debug this, make it work."
---

# /code — General Purpose Code Writer

**Recommended Model:** sonnet

Standards-enforcing code writer for ad-hoc tasks. No planning workflow, no ticket dependency.

**Use this for:** one-shot tasks — adding a field, creating a component, fixing a bug, writing a service, refactoring a function.

---

## Guardrails

**Before starting, check:**

- **Multi-surface task?** If this touches more than one major surface (e.g. API + frontend, new DB schema + new service layer), stop and confirm scope with the user before proceeding.
- **Architectural decision?** New database tables, new services, new API contracts, or changes to existing API contracts require human sign-off before implementation. Ask, don't assume.
- **Never commit or push** without explicit user instruction. Complete the task, summarise what changed, then wait.

---

## Step 1: Discover Project Context

If this is a new project or the language/tooling isn't known, establish:

```bash
ls package.json Cargo.toml pyproject.toml go.mod 2>/dev/null | head -3
```

- **Language / runtime** (Node, Python, Go, Rust, …)
- **Package manager** (npm/pnpm/yarn, pip/uv, cargo, …)
- **Test runner** (Jest, Vitest, pytest, go test, cargo test, …)
- **Lint / typecheck command** (eslint, ruff, tsc, clippy, …)

Then read the relevant files before writing anything.

---

## Step 2: Read Before Writing

```
Glob / Read / Grep: {locate the files the task affects}
```

Establish:
- What already exists — don't reinvent or duplicate
- Naming, structure, and patterns used in adjacent code
- Which packages/modules are affected

**If touching a shared module:** identify all consumers before changing anything.
**If touching an API route:** locate handler, service, model, and any API spec together.

**Unfamiliar area?** If this area of the codebase is unfamiliar or the task touches multiple service layers:
→ Investigate first. Glob + Grep the relevant area. Proceed to implementation only after establishing clear context.

---

## Step 3: Implement

Write code following these standards:

**Framework-specific pattern?** If implementing a routing, ORM, state management, auth middleware, or build config pattern:
→ Invoke `source-driven` to verify the pattern against official docs for the installed version before writing code.

### Structure
- Functions: `verbNoun` (`buildUserResponse`, `validatePayload`)
- No deep nesting (>3 levels) — extract named helpers
- One export style per file (default OR named, not both — where convention allows)
- No `console.log` in production — only `console.warn`, `console.error`, `console.info`

### No Duplication
- Never copy-paste existing logic — import or extract a shared utility
- If the same block appears in 2+ places: extract it first
- Check shared/utility modules before writing new utilities

### Types & Validation
- Validate at system boundaries (route handlers, queue consumers, CLI entry points) — not deep in services
- Prefer explicit guards over non-null assertions
- Narrow union types early

### Error Handling
- Throw typed/custom errors where actionable
- Include context (IDs, counts) — never secrets or PII
- Log once per failure path

### Security
- No hardcoded secrets, tokens, or passwords
- Parameterize all SQL via the ORM/query builder — never string interpolation
- Sanitize any HTML rendered from user input
- Validate user input at API boundaries

### Comments
- No comments that restate what code does — rewrite unclear code instead
- Only comment for non-obvious external constraints — explain *why*, never *what*

---

## Step 4: Test (TDD)

1. **Red** — write the failing test first
2. **Green** — minimum code to pass
3. **Refactor** — clean up without breaking

Minimum per change: happy path + one error/edge case.

**Required edge cases — always include:**
- **NULL / missing record:** what happens when a key foreign key or config value is absent?
- **Zero / boundary values:** `0`, `1`, and the configured maximum
- **No associated record:** the parent exists but has no related entry
- **Shared-state timing:** if background processes or async responses can change state mid-flow, test stale/mid-update state

```bash
# Run tests — adapt to project:
{test command}       # e.g. pnpm test, pytest, cargo test, go test ./...
```

Only mock external boundaries (HTTP clients, queues, third-party SDKs) — not modules you control.

---

## Step 5: Quality Gates

Run in order. Fix before proceeding.

```bash
{lint/typecheck command}    # e.g. npm run lint, ruff check ., cargo clippy
{test command}              # all pass, ≥80% coverage on new code
```

---

## Step 5b: Stuck Logic

If a quality gate is failing after 3 attempts without clear progress, stop and escalate:

**TypeScript / type errors after 3 attempts:**
1. Report the exact error(s) — file, line, message
2. If it involves an unfamiliar external package API: `WebSearch("site:{official-docs-domain} {error pattern}")` — official docs only
3. If still stuck: ask the user. Include the error and what you tried.

**Test failures after 3 attempts:**
1. Confirm the test is asserting the right thing — it may be testing the wrong assumption
2. If framework-specific (routing, ORM, async behaviour): invoke `source-driven` to verify the pattern
3. If still stuck: ask the user — include the failing test, actual output, and what you tried

**Lint failures after 3 attempts:**
1. Run the lint command and read the first errors carefully
2. If a rule fires on code you can't change: disable it inline with a comment explaining why
3. If still stuck: ask the user

**General rule:** Never spin on the same failing command more than 3 times. Report what's failing and ask.

---

## Step 5c: Self-Review — Logic, Exceptions, Security

Before claiming completion, re-read the implementation with deliberate attention to failure modes. This is not a re-read for style — read for *ways it can break*.

**Logic bugs:**
- Read each conditional: is the condition correct? `&&` vs `||`? Inverted check?
- Off-by-one: loops, array slices, pagination offsets, `<` vs `<=`
- Null-unsafe access: any path where `.property` is accessed on a value that could be `undefined` or `null`?
- Ordering: if two operations must happen in sequence, is that enforced?
- Return values: does every branch return or throw? Any implicit `undefined` return?

**Unhandled exceptions:**
- What can throw in this code? (DB errors, network, parse errors, missing env vars)
- Is there a catch at the right level? Or does an error propagate or vanish silently?
- If catching: logging? Re-throwing? Returning a typed error result?

**Silent failures:**
- Any `try { ... } catch (e) {}` with an empty catch body?
- Any async function where the caller ignores a rejected promise?
- Any failure path that returns `undefined` or `null` without signal to the caller?

**Security:**
- Raw SQL strings with string interpolation instead of parameterization?
- Secrets, tokens, or PII appearing in log statements?
- User input reaching a dangerous operation (SQL, shell, HTML render) without validation?

**Quality:**
- Any function >50 lines? Note for follow-up or extract now.
- Nesting >3 levels? Add early return or extract a helper.
- Same logic block appearing in 2+ places? Extract.

If anything is found: fix it, then re-run quality gates before proceeding.

---

## Step 6: Simplify

Run if more than 3 application files changed, or if you introduced new abstractions:

```
/simplify
```

Re-run quality gates if fixes are applied. Skip for single-file, migration-only, or test-only changes.

---

## Step 7: PR Size Check (before raising a PR)

If the user is about to create a PR, check whether the diff is still human-reviewable:

```bash
git diff --name-only HEAD origin/main \
  | grep -vE '(__tests__|\.test\.|\.spec\.|snapshots/)' \
  | wc -l

git diff --stat HEAD origin/main | tail -1
```

Alert (advisory only — do not block) if:
- **>20** non-test files changed
- **>600** net line additions (excluding tests)

---

## Step 8: Summary

```
✅ {task description}

Changed: {file} — {what}, {file} — {what}
Tests: {N} new, all passing, {X}% coverage
Gates: ✅ lint  ✅ tests  {✅ /simplify | —}
{⚠️  Manual: {follow-up if needed}}
```
