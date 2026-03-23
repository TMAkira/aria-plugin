---
name: merge
description: Merge a pull request on Azure DevOps and clean up the local environment (worktree, branch deletion, fetch). Use this skill whenever the user wants to merge a PR, complete a PR, finish a PR, or says things like "merge PR 123", "the PR is approved, merge it", "complete the pull request", "merge and cleanup", "merge and delete the branch". Handles autocomplete, squash/merge/rebase strategy, worktree removal, and local branch cleanup. Do NOT use for local git merges (no PR involved), PR creation, conflict resolution, cherry-picks, or rebasing.
---

# aria-ops:merge — Merge a PR and clean up

Merge an Azure DevOps pull request and clean up the local environment (worktree or branch).

**Announce at start:** "Using aria-ops:merge to merge PR #<id> and clean up."

**Usage:**
- `aria-ops:merge 348` — merge PR #348 and clean up

## Workflow

Follow these steps in order. Do NOT skip steps.

### Step 0: Load Configuration

1. Find the project root: `git rev-parse --show-toplevel`
2. Check if `.aria-ops/merge.json` exists at the project root
3. **If found:** load it, proceed to Step 1
4. **If NOT found:** run interactive setup:

   **Pre-fill:** scan `.aria-ops/*.json` for existing configs to reuse common ADO fields (`ado.project`, `ado.org`, `ado.repoId`).

   Ask the user these questions **one at a time**:
   - "Azure DevOps organization name?" — pre-fill from other configs if available
   - "Azure DevOps project name?" — pre-fill from other configs if available
   - "Repository ID (GUID)? (run `az repos show --repository <name> --query id` to find it)" — pre-fill from other configs if available
   - "Merge strategy? (Squash / Merge / Rebase)" — default: `Squash`
   - "Worktree path convention? (relative to repo root, e.g. `.worktrees/`)" — default: auto-detect from `git worktree list`, fallback to `.worktrees/`

   Write `.aria-ops/merge.json`:
   ```json
   {
     "ado": {
       "org": "<answer>",
       "project": "<answer>",
       "repoId": "<answer>"
     },
     "merge": {
       "strategy": "Squash"
     },
     "worktree": {
       "pathConvention": ".worktrees/"
     }
   }
   ```

   Create the `.aria-ops/` directory if it doesn't exist.
   Tell the user: **"Config saved in `.aria-ops/merge.json`. Continuing with merge."**
   Proceed to Step 1.

### Step 1: Validate the PR

Use `mcp__ado__repo_get_pull_request_by_id` with:
- **repositoryId**: `config.ado.repoId`
- **pullRequestId**: the PR number from `$ARGUMENTS` — **MUST be a number, not a string** (ADO MCP rejects string values with `Invalid arguments: Expected number, received string`)

Verify:
- The PR exists
- Note the source branch name (`sourceRefName`) and target branch (`targetRefName`)
- Extract the short branch name (strip `refs/heads/` prefix)

**Status handling:**
- Status 1 (Active) — proceed to Step 2 (merge) then Step 3+
- Status 3 (Completed) — skip Step 2, go directly to Step 3 (cleanup only)
- Status 2 (Abandoned) — warn the user, ask if they still want to clean up locally. Stop if they decline.

### Step 2: Set autocomplete and merge

Use `mcp__ado__repo_update_pull_request` with:
- **repositoryId**: `config.ado.repoId`
- **pullRequestId**: the PR number (as a number)
- **autoComplete**: `true`
- **deleteSourceBranch**: `true`
- **mergeStrategy**: `config.merge.strategy`

Then poll the PR status using `mcp__ado__repo_get_pull_request_by_id` every few seconds (up to 30s):
- Status 3 = Completed (merged) — continue
- Status 1 = Still active (policies pending) — inform the user that autocomplete is set but policies may be blocking. Continue to cleanup anyway.

### Step 3: Release all locks before cleanup

Before attempting any worktree/directory removal:

1. **Stop any background servers** running in the worktree (dotnet, node, npm, etc.):

   **On Windows:**
   ```bash
   tasklist /FI "IMAGENAME eq dotnet.exe" /V 2>/dev/null
   taskkill //F //IM dotnet.exe 2>/dev/null || true
   ```

   **On Unix/macOS:**
   ```bash
   pkill -f dotnet 2>/dev/null || true
   ```

   Detect the platform from `$OSTYPE` or `uname` and use the appropriate command.

2. **Change directory** to the main repo root (from `git rev-parse --show-toplevel`) to ensure no shell is inside the worktree.

3. **Verify no locks** — quick check that the worktree directory is accessible:
   ```bash
   ls <worktree-path>/ > /dev/null 2>&1
   ```

### Step 4: Clean up local environment

**Detect workspace type:**

Run `git worktree list` from the main repo root. Check if there is a worktree for the source branch.

**If a worktree exists for the source branch:**

Extract the worktree directory path from `git worktree list` output.

```bash
# Attempt clean removal
git worktree remove <worktree-path>
```

If `git worktree remove` fails (Permission denied, device busy, etc.), use the fallback:
```bash
rm -rf <worktree-path>
git worktree prune
```

Then delete the local branch:
```bash
git branch -D <source-branch>
```

**If no worktree (simple branch):**

Determine the target branch name (strip `refs/heads/` from `targetRefName`).

```bash
git checkout <target-branch>
git branch -D <source-branch>
```

### Step 5: Update main branch

Determine the target branch from the PR's `targetRefName`.

```bash
git fetch --prune
git pull origin <target-branch>
```

Verify we are on the target branch (or switch to it if cleanup left us elsewhere) and that it is up to date.

### Step 6: Verify architecture docs (optional)

**Skip this step entirely if no `architecture/` directory (or similar docs directory) exists at the project root.**

If an architecture docs directory is detected:

1. Look at the files changed by the merged PR (from the most recent commit on the target branch):
   ```bash
   git diff --name-only HEAD~1
   ```
2. If the changes include structural modifications (new/deleted controllers, services, entities, migrations, DI registration changes) **and** no architecture doc files were part of the commit:
   - Warn the user: "Architecture docs were not updated for this structural change. Consider updating them."
3. If architecture files were already included, or no structural changes detected — skip silently.

### Step 7: Report

Produce a structured summary:

```
PR #<id> — <merge status> (<strategy>) into <target-branch>
Branch <source-branch> deleted (local + remote)
Worktree cleaned up: <worktree-path> (if applicable)
<target-branch> is up to date at <short-hash>
```

Where `<merge status>` is one of:
- "merged" — PR was completed during this run
- "autocomplete set" — PR still active, autocomplete enabled
- "already merged" — PR was already completed before this run
