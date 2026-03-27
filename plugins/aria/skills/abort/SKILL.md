---
name: abort
description: Use when the user wants to stop current work cleanly. Use when implementation should be abandoned. Use when a plan is no longer viable and work needs to be discarded or shelved.
---

# Abort — Clean Shutdown

Stop current work, clean up resources, and leave the repo in a good state.

**Announce at start:** "Using aria:abort to cleanly stop this work."

## Process

### Step 1: Assess Current State

```bash
git status
git worktree list
git log --oneline -10
```

Determine:
- Are there uncommitted changes?
- Are there completed tasks with commits?
- Is there a worktree to clean up? (might not be — could be working directly on a branch)

### Step 2: Ask the User

Present only the relevant options based on the current state:

```
## Abort Options

**Completed tasks:** [N tasks with commits — or "None"]
**Uncommitted changes:** [yes — description / no]
**Worktree:** [path / not using a worktree]
```

**If completed tasks exist AND other work remains:**
```
A) **Keep completed work** — merge completed tasks, discard uncommitted changes
B) **Shelve everything** — save WIP, keep the branch (can resume later)
C) **Discard everything** — delete branch and all changes
```

**If no completed tasks:**
```
B) **Shelve everything** — save WIP, keep the branch (can resume later)
C) **Discard everything** — delete branch and all changes
```

Wait for the user's choice. NEVER discard work without explicit confirmation.

### Step 3: Execute Choice

**A) Keep completed work:**
```bash
# Discard uncommitted changes
git checkout -- .
git clean -fd

# Return to main worktree
cd <main-worktree-path>

# Merge completed work
git merge feature/<name>

# Clean up worktree (if exists)
git worktree remove <path>
git branch -d feature/<name>
```

**B) Shelve:**
```bash
# Commit any WIP
git add -A && git commit -m "wip: shelved — [reason]"

# Clean up worktree if exists, but keep the branch
cd <main-worktree-path>
git worktree remove <path>
# Branch stays — can resume later with aria:resume-plan
```

**C) Discard:**
```bash
# Return to main worktree
cd <main-worktree-path>

# Remove worktree if exists
git worktree remove --force <path>

# Delete branch
git branch -D feature/<name>
```

**If not using a worktree:** skip worktree removal steps, just handle the branch.

### Step 4: Update Meta-Plan

If keeping or shelving and a meta-plan exists, update it:
```markdown
## Status
- **Started:** [date]
- **Aborted:** [date]
- **Reason:** [why]
- **Completed tasks:** [list]
- **State:** shelved / partially merged
```

If discarding or no meta-plan exists, skip this step.

### Step 5: Confirm

> "Work [kept/shelved/discarded]. Repository is clean. [Branch still exists at feature/xxx / Branch deleted]."

## Integration

- **aria:exec** — can invoke abort at any checkpoint
- **aria:worktree** — abort cleans up the worktree
