---
name: checkpoint
description: Save current task progress to .project/current-task.md for context persistence
---

# Checkpoint Current Progress

Create or update `.project/current-task.md` with:

1. **Current task description** - what you're working on right now
2. **Progress summary** - what's been done
3. **Files modified** - list of changed files with brief notes
4. **Next steps** - specific actions to resume from
5. **Open questions** - anything unresolved
6. **Timestamp** - ISO 8601 format

**Format:**

```markdown
# Current Task Checkpoint

**Saved:** [ISO timestamp]

## Task

[Brief description of current work]

## Progress

- [x] Completed item
- [ ] In progress item

## Files Modified

- `path/to/file.ts` - [what changed]

## Next Steps

1. [Specific next action]

## Open Questions

- [Any blockers or decisions needed]
```

After writing the checkpoint:
- Confirm file saved
- Show timestamp
- Remind user to use `/restore` in next session
