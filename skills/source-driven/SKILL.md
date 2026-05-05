---
name: source-driven
description: "Use when implementing any framework-specific or library-specific pattern — verify against official docs for the exact installed version before writing code. LLM training data goes stale; this skill enforces a \"check the source\" discipline across all projects. Trigger: implement framework pattern, use library API, \"how do I use X\", version-specific behaviour."
source: https://github.com/owenfranssen/ai-engineering-plugin
---

# source-driven — Source-Driven Development

**Core principle:** LLM training data is always stale. API signatures, configuration options, and behaviour change across versions. Before implementing any non-trivial framework or library pattern, check the official docs for the **exact installed version** — not your memory of the API.

**Use when:** implementing anything where the API may have changed between versions — routing, auth middleware, ORM queries, state management, build config, test runners, etc.

**Skip when:** pure business logic with no framework dependency — data transforms, pure functions, domain rules.

---

## Step 1: Detect Installed Version

Read the **actual installed version** from `node_modules` — not the semver range in `package.json`:

```bash
# Node/JS — read from installed package, not semver range
node -e "console.log(require('./node_modules/<package-name>/package.json').version)"

# Or from the relevant sub-package if monorepo
node -e "console.log(require('./<package-dir>/node_modules/<pkg>/package.json').version)"

# Python
pip show <package> | grep Version
# or: python -c "import <pkg>; print(<pkg>.__version__)"

# Ruby
bundle exec ruby -e "require '<gem>'; puts <Gem>::VERSION"
```

Note the exact resolved version (e.g. `21.3.3`, not `^21`).

> If the command fails (module not found), the package may not be installed. Run the project's install command first.

## Step 2: Fetch Official Docs

Use `WebFetch` to retrieve the relevant section from the official documentation. Always prefer:

1. **Official docs** for the specific version (include version in URL where supported, e.g. `?v=21.3.3`, `/docs/0.74/`)
2. **Official changelog** if the docs are version-agnostic
3. **MDN / web standards** for web APIs
4. **Source code / types** as a last resort

Never cite Stack Overflow, tutorials, blog posts, or AI summaries as primary sources.

**Finding the right URL:** If you don't know the docs URL, search `"<package-name> official documentation"` or check the `homepage` field in the package's `package.json`.

## Step 3: Surface Conflicts

Compare the documented pattern against existing code:

```bash
# Find existing usage of the API being implemented
Grep "<framework_api_or_method>" --output_mode=content -C 3
```

If existing code diverges from current docs:
- Flag the divergence explicitly — don't silently fix it
- Note it in your implementation summary
- Ask the user before changing existing usage patterns across the codebase

## Step 4: Implement

Follow the documented pattern for the installed version. If existing codebase code already uses the pattern and it matches the docs, follow the codebase convention for consistency.

If docs and existing code conflict, prefer docs and note the divergence.

## Step 5: Cite

For non-obvious framework patterns, add a source comment pointing to the exact version:

```typescript
// Pattern from Hapi lifecycle docs — https://hapi.dev/api/?v=21.3.3#-serverextevents
server.ext('onPreResponse', handler);
```

```python
# See SQLAlchemy 2.0 migration guide — https://docs.sqlalchemy.org/en/20/changelog/migration_20.html
session.execute(select(User).where(User.id == user_id))
```

Skip citation comments for obvious boilerplate any developer would recognise.

---

## Common Cases

| Ecosystem | How to detect version | Docs pattern |
|-----------|----------------------|--------------|
| npm/pnpm | `node -e "require('./node_modules/<pkg>/package.json').version"` | Check `homepage` in package.json or search `<pkg> docs` |
| Python pip | `pip show <pkg> \| grep Version` | https://pypi.org/project/<pkg>/ → project links |
| Ruby gems | `bundle exec ruby -e "require '<gem>'; puts <Gem>::VERSION"` | https://rubygems.org/gems/<gem> → documentation |
| Go modules | `go list -m <module>` | pkg.go.dev/<module> |

## Red Flags

If you find yourself doing any of these, stop and check the docs:

- Copying a pattern from memory without verifying the version
- Using a method name you recall from training ("I think it's `.findById()`")
- Assuming config option names without checking (`strict`, `legacy`, `experimental`)
- Guessing at callback signatures, hook names, or middleware ordering

**The 2-minute check pays for itself:** one wrong API call that passes type-checking but fails at runtime costs far more than reading the source.
