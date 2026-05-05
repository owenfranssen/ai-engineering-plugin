---
name: restore
description: Restore project context from .project/ state files
---

# Resume Project Context

Read and summarize the following files to restore session context:

1. **`.project/state.md`** - Overall project status
2. **`.project/current-task.md`** (if exists) - Last checkpoint
3. **`.project/tasks.md`** - Active task list
4. **`.project/decisions.md`** - Recent decisions (last 3-5 entries)

**Output Format:**

```
📋 Project Context Restored

Current Focus: [from state.md]
Active Branch: [branch name]
Last Checkpoint: [timestamp from current-task.md]

Active Tasks:
- 🟡 [in progress task]
- 🔵 [todo task]

Recent Decisions:
- [decision title] (date)

Ready to continue: [next step from current-task.md or state.md]
```

If `.project/current-task.md` exists, ask: "Continue from last checkpoint, or start something new?"

If files are missing or empty, note that and offer to initialize project state.
