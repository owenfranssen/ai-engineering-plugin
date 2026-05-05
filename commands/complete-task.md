---
name: complete-task
description: Move completed task to archive and clean up related context
---

# Complete Task

Mark a task as complete and archive it:

1. **Identify the task** - Ask user which task from `.project/tasks.md` to complete, or use context

2. **Update tasks.md** - Remove task from active list (or mark 🟢 DONE)

3. **Archive to completed.md** - Add entry:
   ```markdown
   ## [Today's date]

   - **Task:** [task description]
     - **Completed:** [ISO timestamp]
     - **Outcome:** [what was accomplished]
     - **Files Changed:** [key files modified]
     - **Related:** [PR/commit/ticket links]
   ```

4. **Clean up current-task.md** - If this was the current task:
   - Delete `.project/current-task.md`, OR
   - Ask user if there's a next task to checkpoint

5. **Update state.md** - Update "Next Steps" section

**Confirm:**
```
✅ Task completed and archived

Moved to: .project/completed.md
Updated: .project/tasks.md, .project/state.md
Cleared: .project/current-task.md

Next: [suggest next task from tasks.md or ask user]
```
