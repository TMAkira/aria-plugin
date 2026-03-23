---
name: resume
description: Use when resuming work on an existing plan from a previous session. Use when the user says to continue, resume, or pick up where we left off.
---

# Resume — Continue Interrupted Work

Resume execution of a plan that was started in a previous session.

**Announce at start:** "Using aria:resume to pick up where we left off."

## Process

### Step 1: Find the Plan

Look for active plans:

```bash
find docs/plans/ -name "plan.md" -type f 2>/dev/null
```

- If **no plans found:** inform the user — "No active plan found. Use aria:design to start a new one."
- If **multiple plans found:** list them and ask the user which one to resume.

### Step 2: Read State

Read the meta-plan (`plan.md`):
- Which tasks are completed?
- Which task was in progress when the session ended?
- Are there discoveries or warnings noted?
- Was the plan revised during execution?
- What is the exec mode (HARD/MEDIUM/EASY)?

### Step 3: Present Status

```
## Resuming: [Feature Name]

**Progress:** [N/Total] tasks completed
**Exec mode:** [HARD/MEDIUM/EASY] (change? just say so)
**Last completed:** Task N — [description]
**Next up:** Task N+1 — [description]
[If a task was in progress] **Interrupted task:** Task N — [was in progress, will restart from scratch]
**Discoveries from last session:** [any notes, or "None"]

Ready to continue?
```

The human can change the exec mode at this point (e.g., switch from HARD to MEDIUM if the risky part is done).

### Step 4: Handle In-Progress Tasks

If a task was marked "in progress" when the session ended:
- Check if it was committed (partially or fully)
- If **committed:** verify tests pass, mark as completed if good, re-do if not
- If **not committed:** restart the task from scratch — partial work without a commit is unreliable

### Step 5: Continue

- If there are discoveries or warnings, discuss them first
- Invoke aria:exec to continue from the next pending task, passing the exec mode from the meta-plan (or the human's override)

## Edge Cases

- **Plan was revised mid-session:** the task files reflect the latest version — follow them
- **Tasks were reordered:** compare the current meta-plan task order with completed tasks — flag any inconsistencies to the human before proceeding
- **Worktree still exists:** verify it's clean and on the right branch
- **Worktree was cleaned up:** re-create it with aria:worktree

## Integration

- **aria:exec** — resumed execution continues with the same mode and rules
- **aria:plan** — creates the meta-plan that this skill reads to resume
