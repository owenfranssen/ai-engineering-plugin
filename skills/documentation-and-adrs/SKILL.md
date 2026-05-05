---
name: documentation-and-adrs
description: "Use when making an architectural decision, choosing between competing approaches, or adding context that future-you or future agents will need to understand why the codebase is the way it is. Also use when a design decision has come up twice across sessions, or when you find commented code with no explanation of intent. Trigger: write an ADR, document this decision, add an ADR, explain why we chose, record this decision, document the architecture."
source: https://github.com/owenfranssen/ai-engineering-plugin
---

# Documentation and ADRs

## Purpose

Document decisions, not just code. The most valuable documentation captures the *why* — the context, constraints, and trade-offs that shaped a decision. Code shows *what* was built; documentation explains *why it was built this way*. This context is essential for future-you and future agents working in the project.

Agents in future sessions cannot see the reasoning that happened during implementation. Without ADRs, they will treat intentional decisions as problems to fix. A well-placed ADR prevents an agent from "improving" something that was deliberately designed that way.

**One concrete trigger:** if you find yourself explaining the same design decision twice across different sessions, write an ADR.

## When to Use

- Making an architectural decision that would be hard to reverse
- Choosing between competing libraries, databases, or approaches
- Designing a data model or schema
- Any decision whose "why" is not obvious from reading the code
- When you need to prevent future agents from undoing an intentional choice

**When NOT to use:** Don't document obvious code. Don't write ADRs for throwaway prototypes or decisions that can be freely revisited.

## Architecture Decision Records (ADRs)

ADRs capture the reasoning behind significant technical decisions. They are the highest-value documentation you can write in a personal project.

### When to Write an ADR

- Choosing a framework, library, or major dependency
- Designing a data model or database schema
- Selecting an authentication or authorization strategy
- Deciding on an API style (REST, tRPC, GraphQL)
- Choosing between hosting or infrastructure options
- Any decision that would be expensive to reverse
- Any decision whose rationale is not visible in the code

### ADR Location

Store ADRs under `docs/adrs/` with zero-padded four-digit numbering:

```
docs/adrs/0001-use-sqlite.md
docs/adrs/0002-session-auth-over-jwt.md
```

### ADR Template

```markdown
# ADR-0001: Use SQLite for local persistence

## Status
Accepted

## Context
This is a single-user CLI tool running on a local machine. There is no
concurrent write load and no requirement for a network-accessible database.
We need persistence for user preferences and command history.

## Decision
Use SQLite via better-sqlite3. No ORM — raw SQL queries kept in a single
queries.ts module. Schema migrations handled with a simple versioned integer
stored in user_version.

## Consequences
- Zero infrastructure: no server, no connection strings, no Docker required
- Suitable only for single-process, local use — not a web server backend
- If the project ever needs multi-user or remote access, this ADR should be
  superseded (not deleted) with the replacement approach documented
```

The three sections cover what any reader needs:
- **Context** — what situation prompted this decision
- **Decision** — what was chosen and why (include rejected paths only if the rejection is non-obvious)
- **Consequences** — trade-offs, limitations, and any follow-up work this creates

### ADR Lifecycle

```
PROPOSED → ACCEPTED → (SUPERSEDED or DEPRECATED)
```

Never delete old ADRs. When a decision changes, write a new ADR that references and supersedes the old one. Historical decisions explain why the codebase evolved the way it did — that context has value even after the decision is reversed.

Update the `Status` line of the old ADR to `Superseded by ADR-NNNN`.

## Inline Documentation

### Comment the WHY, not the WHAT

```typescript
// BAD: Restates the code
// Increment the retry count
retryCount += 1;

// GOOD: Explains non-obvious intent
// Cap retries at 3 before surfacing to the user — beyond this the failure
// is almost certainly a network or auth issue, not a transient blip
if (retryCount >= 3) {
  throw new PermanentError(lastError);
}
```

### When NOT to Comment

```typescript
// Don't comment self-explanatory code
function sum(a: number, b: number) {
  return a + b;
}

// Don't leave TODOs that should just be done now
// TODO: handle the error  ← Just handle it

// Don't leave commented-out code — git has history
// const oldApproach = () => { ... }
```

### Document Known Gotchas

Put warnings at the call site, not buried in a PR description:

```typescript
/**
 * IMPORTANT: Must be called after the config file is loaded.
 * If called before loadConfig(), it silently uses defaults and
 * the caller gets no error — this has caused subtle bugs before.
 *
 * See ADR-0003 for why initialization is deferred.
 */
export function initializeCache(config: Config): Cache {
  // ...
}
```

## README Structure

Every project needs a README that answers the four questions a future session will have:

```markdown
# Project Name

One-paragraph description of what this project does and for whom.

## Quick Start
1. Install dependencies: `npm install`
2. Set up environment: `cp .env.example .env`
3. Run: `npm run dev`

## Commands
| Command | Description |
|---------|-------------|
| `npm run dev` | Start development server |
| `npm test` | Run tests |
| `npm run build` | Production build |

## Architecture
Brief overview of structure and key design decisions.
See `docs/adrs/` for decision records.
```

Keep it to what a fresh session needs to get oriented. Link to ADRs rather than duplicating their content.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The code is self-documenting" | Code shows what. It does not show why, what alternatives were rejected, or what constraints were in play. |
| "Future-me will remember" | Future-you is effectively a different person. Future agents have no memory of this session at all. |
| "ADRs are enterprise overhead" | A 10-minute ADR prevents a future agent from spending an hour "fixing" something you intentionally designed that way. |
| "I'll document it once it stabilizes" | Decisions are easiest to document at the moment they are made. Retroactive ADRs miss the context of what was actually considered. |
| "Comments get outdated" | Comments on *why* are stable. Comments on *what* get outdated — that is why you only write the former. |
| "If I had to write an ADR, I'd just stop building" | You don't need an ADR for every choice. Only write one when the decision is hard to reverse or when the rationale isn't visible in the code. |

## Red Flags

- Architectural decision with no written rationale anywhere in the project
- An agent in a new session reverting or "cleaning up" an intentional design choice
- README that does not explain how to run the project
- Commented-out code with no explanation
- TODO comments older than the current branch
- Known gotchas for a function that only exist in a PR description or Slack thread
- Explaining the same past decision to an agent for the second time

## Verification

After documenting:

- [ ] ADR exists for every decision that was hard to reach or would be hard to reverse
- [ ] ADR template has all three sections: Context, Decision, Consequences
- [ ] Status line is correct (Proposed / Accepted / Superseded / Deprecated)
- [ ] README covers quick start, commands, and architecture overview
- [ ] Known gotchas documented inline at the relevant function or module
- [ ] No commented-out code remains
- [ ] No TODO comments for things that should just be done
