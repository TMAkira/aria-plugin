---
name: worktree
description: Use when starting feature work that needs isolation from the current workspace. Use before aria:exec to set up an isolated environment. Use when the user asks to work on a branch or feature in isolation.
---

# Worktree — Git Isolation

Create an isolated git worktree for feature work, keeping the main workspace clean.

**Announce at start:** "Using aria:worktree to set up an isolated workspace."

## Process

### Step 0: Pre-flight Check

Before creating a worktree:

```bash
# Am I already in a worktree?
git rev-parse --show-toplevel
git worktree list
```

If already in a worktree, ask the human: "You're already in a worktree at `<path>`. Create a nested one or use the existing one?"

### Step 1: Choose Location

Determine the project name dynamically and place the worktree as a sibling:

```bash
PROJECT_NAME=$(basename $(git rev-parse --show-toplevel))
# Worktree goes to ../<PROJECT_NAME>-wt-<feature-name>
```

### Step 2: Create Branch and Worktree

```bash
git worktree add ../${PROJECT_NAME}-wt-<feature-name> -b feature/<feature-name>
```

If the branch already exists:
```bash
# Branch exists — check if it has a worktree already
git worktree list | grep feature/<feature-name>
# If no worktree: reuse the branch
git worktree add ../${PROJECT_NAME}-wt-<feature-name> feature/<feature-name>
# If worktree exists: inform the user and ask what to do
```

### Step 3: Verify

```bash
git worktree list
cd ../${PROJECT_NAME}-wt-<feature-name>
git branch --show-current
```

### Step 4: Announce

> "Worktree created at `<path>` on branch `feature/<feature-name>`. Working from there now."

## Cleanup

When work is complete (after merge or abort):

```bash
# Return to the main worktree first
cd <main-worktree-path>

# Remove the worktree
git worktree remove ../${PROJECT_NAME}-wt-<feature-name>

# Delete the branch (only if merged — use -D for abandoned branches via aria:abort)
git branch -d feature/<feature-name>
```

## Integration

- **aria:exec** — should run inside a worktree for isolation
- **aria:abort** — cleans up the worktree (uses `git branch -D` for unmerged branches)
- **aria:design** — can create worktree early if exploration is needed
