---
name: source-driven
description: Verify framework-specific implementation against official docs for the installed version before writing code. Use when implementing any framework, library, or runtime API where training data may be stale.
---

# source-driven — Source-Driven Development

**Use when:** implementing any framework, library, or runtime API where the correct pattern depends on the installed version
**Skip when:** version-agnostic logic — pure functions, data transforms, business rules with no framework dependency

---

## Step 1: Detect Version

Find the relevant `package.json` and read the installed version — not the semver range, the resolved version:

```bash
# From the directory containing the relevant package.json:
node -e "const p=require('./package.json'); console.log(p.dependencies?.['<package>'] || p.devDependencies?.['<package>'] || 'not found')"

# Or read the actual installed version from node_modules (more reliable):
node -e "console.log(require('./node_modules/<package>/package.json').version)"

# For non-JS runtimes, check the relevant lock/manifest:
# Python: cat pyproject.toml | grep '<package>' OR pip show <package>
# Ruby:   cat Gemfile.lock | grep '<package>'
# Rust:   cat Cargo.lock | grep 'name = "<package>"' -A1
# Go:     cat go.sum | grep '<package>'
```

Note the **exact resolved version** (e.g. `21.3.3`, not `^21`).

> If `node_modules` isn't present, install dependencies first (`pnpm install` / `npm install` / `yarn`) and retry.

## Step 2: Fetch Official Docs

Use `WebFetch` to retrieve the relevant section of the official docs for the **installed version**:

- Prefer versioned doc URLs (e.g. `https://hapi.dev/api/?v=21.3.3`) over unversioned ones
- Navigate directly to the API page or section covering the feature being implemented
- **Source hierarchy:** official docs → official changelog/migration guide → MDN/web standards
- Never cite Stack Overflow, tutorials, or AI summaries as primary sources

If no versioned URL exists, check the official changelog or migration guide for breaking changes between the version in your training data and the installed version.

## Step 3: Surface Conflicts

Compare the documented pattern against existing codebase usage before writing new code:

```bash
# Find existing usages of the API/method being implemented
# (adjust path to the relevant package directory)
grep -r "<api_method_or_pattern>" <package_dir>/src --include="*.ts" -n -C 3
```

If existing code diverges from current docs:
- Flag it explicitly — don't silently fix it
- Note the divergence in your implementation summary
- Ask the user whether to fix the existing usage or just follow docs for new code

## Step 4: Implement

Follow the documented pattern for the installed version. Where existing codebase code already uses the pattern **and it aligns with docs**, follow the codebase pattern for consistency.

## Step 5: Cite

For non-obvious framework patterns, add a source comment:

```typescript
// Pattern from Hapi lifecycle docs — https://hapi.dev/api/?v=21.3.3#-serverextevents
server.ext('onPreResponse', handler);
```

Do not add citation comments for obvious, boilerplate patterns that any developer would recognise.
