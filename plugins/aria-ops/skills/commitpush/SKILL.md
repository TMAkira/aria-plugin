---
name: commitpush
description: Commit all changes and push to remote with a structured report. Use this skill whenever the user wants to commit, push, save their work to the remote, or says things like "commitpush", "commit and push", "push my changes", "save and push", "ship it", "send my changes", or any variation of committing + pushing code. Also use when the user asks to stage everything and push, or wants a commit message auto-generated from their diff. Do NOT use for amending commits, rebasing, squashing, reverting, or branch creation.
---

# aria-ops:commitpush — Commit all changes and push

Commit and push all pending changes on the current branch.

**Announce at start:** "Using aria-ops:commitpush to commit and push changes."

**Usage:**
- `aria-ops:commitpush` — auto-generate commit message from changes
- `aria-ops:commitpush fix: resolve null reference in UserService` — use provided message

## Workflow

Follow these steps in order. Do NOT skip steps.

### Step 0: Load Configuration

1. Check if `.aria-ops/commitpush.json` exists at the project root (use `git rev-parse --show-toplevel` to find root)
2. **If found:** load it, proceed to Step 1
3. **If NOT found:** run interactive setup:

   **Pre-fill:** scan `.aria-ops/*.json` for existing configs to reuse common fields (e.g., co-author name)

   Ask the user these questions one at a time:
   - "Co-author for commits? (name + email)" — default: `Aria <noreply@anthropic.com>`
   - "Use conventional commits? (feat:/fix:/refactor:/chore:/etc.)" — default: yes
   - "Path to architecture docs to auto-update on structural commits? (leave empty to skip)" — default: empty

   Write `.aria-ops/commitpush.json`:
   ```json
   {
     "commit": {
       "coAuthor": "<answer>",
       "conventionalCommits": true|false
     },
     "architectureDocs": {
       "path": "<answer or null>"
     }
   }
   ```

   Create the `.aria-ops/` directory if it doesn't exist.
   Tell the user: **"Config saved in `.aria-ops/commitpush.json`. Continuing with commit."**
   Proceed to Step 1.

### Step 1: Check preconditions

Run in parallel:
- `git status` — verify there are changes to commit
- `git branch --show-current` — get current branch name
- `git log --oneline -5` — get recent commit style

If there are **no changes** (working tree clean, nothing staged), abort: **"Nothing to commit."**

**Main branch protection:** Detect the main branch from the project's CLAUDE.md, git config, or common conventions (`develop`, `main`, `master`). If the current branch is the main branch, warn the user: **"You are on `<main-branch>`. Do you really want to commit directly here, or should we create a branch first?"** Do NOT proceed until the user explicitly confirms.

### Step 2: Analyze all changes

Run in parallel:
- `git diff --stat` — summary of unstaged changes
- `git diff --cached --stat` — summary of staged changes
- `git status --short` — list all files with their status (A/M/D/?)

**Verify completeness:**
- Check that ALL modified/new files are accounted for
- If there are untracked files that look related to the current work, warn the user before continuing

### Step 3: Stage all relevant files

Stage all changes:
```bash
git add -A
```

Then run `git diff --cached --stat` to confirm what will be committed.

### Step 4: Update architecture docs (if configured and structural changes detected)

**Skip entirely if `config.architectureDocs.path` is null or empty.**

If configured, analyze the staged files for structural changes:
- New/deleted source files in controller, service, or entity directories
- New database migrations that add tables or columns
- Changes to dependency injection registration, middleware, or external service configuration
- New project files or project reference changes

If structural changes are detected:
1. Read the architecture docs index at `<config.architectureDocs.path>/INDEX.md` (or similar)
2. Read the relevant domain file(s)
3. Apply targeted updates (add/remove/rename entries)
4. Stage the updated architecture files

**Rules:**
- Only update what changed — never rewrite an entire file
- If unsure whether a change is structural, skip (false negatives > false positives)

### Step 5: Generate commit message

**If $ARGUMENTS is provided** (not empty):
- Use `$ARGUMENTS` as the commit message verbatim

**If $ARGUMENTS is empty:**
- Analyze the staged diff to determine the commit type
- If `config.commit.conventionalCommits` is true: use conventional commit format (`feat:`, `fix:`, `refactor:`, `chore:`, `test:`, `docs:`, `perf:`)
- Write a concise commit message in English, imperative mood, under 72 characters
- If changes span multiple concerns, use the dominant type and summarize

### Step 6: Commit

Create the commit using a HEREDOC for proper formatting:

```bash
git commit -m "$(cat <<'EOF'
<commit message>

Co-Authored-By: <config.commit.coAuthor>
EOF
)"
```

If the commit fails (e.g., pre-commit hook), fix the issue and retry with a NEW commit (never `--amend`).

### Step 7: Push

Push to the remote with upstream tracking:

```bash
git push -u origin <current-branch>
```

### Step 8: Report

After a successful push, produce a **structured report**:

#### 8.1 — Files report

Count files by status from the commit diff (`git diff --name-status HEAD~1`):

| Status | Count | Files |
|--------|-------|-------|
| Added (A) | N | list |
| Modified (M) | N | list |
| Deleted (D) | N | list |
| **Total** | **N** | |

#### 8.2 — Tests report

Scan the committed diff for test changes. Detect test methods by common decorators across frameworks:
- C#: `[TestMethod]`, `[Fact]`, `[Theory]`, `[Test]`
- JS/TS: `it(`, `test(`, `describe(`
- Python: `def test_`, `@pytest.mark`
- Java: `@Test`, `@ParameterizedTest`

For each test file in the commit, count added/modified/deleted tests.

Classify tests by folder/namespace conventions:
- **Unit**: default
- **Integration**: folder/namespace contains "Integration" or "integration"
- **E2E**: folder/namespace contains "E2E", "e2e", or "end-to-end"

| Type | Added | Modified | Deleted | Total |
|------|-------|----------|---------|-------|
| Unit | N | N | N | N |
| Integration | N | N | N | N |
| E2E | N | N | N | N |
| **Total** | **N** | **N** | **N** | **N** |

If no tests were changed: **"No tests modified in this commit."**

#### 8.3 — Summary line

```
Commit <short-hash> pushed to origin/<branch> — N file(s), N test(s) added
```
