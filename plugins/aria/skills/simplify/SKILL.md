---
name: simplify
description: Use when code needs cleanup before merging. Use when there's dead code, duplication, missing abstractions, or test gaps. Use after aria:exec to polish a feature branch. Use when the user says simplify, cleanup, refactor, or review code quality.
---

# Simplify — Post-Implementation Cleanup

Scan code for cleanup opportunities and test gaps, then generate an aria:exec-compatible plan to fix them.

**Announce at start:** "Using aria:simplify to scan for cleanup opportunities."

## Input Modes

- **No argument:** scope = all files changed on the current branch vs its base branch
- **With argument:** scope = free-text description interpreted via grep/glob (e.g., "the DiffRebuilder and its callers")

## The Process

### Step 1: Determine Scope

**No argument:**
```bash
# Detect base branch
git merge-base develop HEAD  # or main, master — check what exists
git diff <base>...HEAD --name-only
```

**With argument:**
- Interpret the free-text scope
- Use grep/glob to find relevant files
- Include files that call/are called by the target (one level of indirection)

Present scope summary:
```
## Scope

**Mode:** branch diff / targeted scope
**Files:** N files
[file list]

Confirm to begin scanning?
```

Wait for confirmation.

### Step 2: Pass 1 — Diff Scan

Read the diff (or files if targeted scope). Identify findings per category:

#### Categories

| Category | What to look for | Priority guidance |
|----------|-----------------|-------------------|
| **Dead code** | Unused methods, variables, usings, unreachable branches, orphaned imports | HIGH if public API, MEDIUM if private |
| **Duplication** | Similar code blocks (3+ lines repeated or near-identical logic) | HIGH if 3+ occurrences, MEDIUM if 2 |
| **Missing abstractions** | Repeated patterns that should be a helper, method, or shared component | MEDIUM — only if the pattern repeats 3+ times |
| **Naming** | Inconsistent or unclear names (abbreviations, misleading names, convention violations) | LOW unless actively misleading |
| **Complexity** | Over-engineering, unnecessary indirection, premature abstraction, deep nesting | HIGH if it obscures the intent, MEDIUM otherwise |
| **Leftovers** | TODO, FIXME, HACK comments, debug logging, commented-out code | MEDIUM for TODO/FIXME, LOW for minor debug traces |
| **Test gaps** | Public methods/classes modified in the diff that have no corresponding test | See below |

**Test gap detection:**
1. For each source file in the diff, identify public methods/classes
2. Search for corresponding test files (patterns: `*Tests.*`, `*Test.*`, `test_*`, `*_test.*`, `*_spec.*`)
3. If test file found: check which methods have test coverage (match by name patterns)
4. If no test file found: flag the entire file
5. Priority: **HIGH** if the method handles business logic, data mutation, or security; **MEDIUM** for helpers/utilities; **LOW** for trivial getters/DTOs/config

Record each finding with: file, line(s), category, description, priority (HIGH/MEDIUM/LOW).

### Step 3: Pass 2 — Targeted Expansion

For findings that need broader context:

- **Dead code:** grep for usages across the entire codebase (not just the diff). A method unused in the diff may be called elsewhere.
- **Duplication:** search for additional occurrences beyond the diff scope.
- **Test gaps:** check if tests exist in a different location or under a different naming convention.

Upgrade or downgrade findings based on expanded context:
- Method flagged as dead code but found to have callers elsewhere → **remove finding**
- Duplication found in 2 places in diff but also in 3 more outside → **upgrade to HIGH**
- Test exists but under unexpected name → **remove finding**

**Do NOT read every file in the codebase.** Only read files that pass 1 findings specifically demand.

### Step 4: Present Report

```
## Simplify Report — <branch or scope>

**Scope:** N files scanned
**Findings:** X total (H high, M medium, L low)

### Code Quality

| # | Category | File:Line | Description | Priority |
|---|----------|-----------|-------------|----------|
| 1 | Dead code | file.cs:45 | Method `Foo()` has no callers | HIGH |
| 2 | Duplication | a.cs:10, b.cs:30 | Similar 15-line block | MEDIUM |
| 3 | Complexity | c.cs:80 | 4 levels of nesting, extract method | MEDIUM |

### Test Gaps

| # | File | Method/Class | Reason | Priority |
|---|------|-------------|--------|----------|
| 4 | svc.cs | ProcessOrder() | No test found — handles data mutation | HIGH |
| 5 | helper.cs | FormatDate() | No test found — utility | MEDIUM |

**Remove items you don't want fixed, then confirm to generate the plan.**
```

Wait for the human to curate the list. They can:
- Remove items by number ("remove 3, 5")
- Change priority ("2 is actually LOW")
- Add items they spotted themselves
- Confirm to generate the plan

### Step 5: Generate Plan

Group related findings into tasks:
- All findings in the same file → one task (unless very different in nature)
- Test gaps → separate task(s) grouped by related functionality
- High priority tasks first
- Group by file to minimize context switching

Create the plan:
```
docs/plans/simplify-<branch>/
  plan.md
  task-01-<short-name>.md
  task-02-<short-name>.md
  ...
```

Each task file follows the standard aria:plan format:
- Objective (single sentence)
- Files to modify
- Steps (with verification)
- Commit message

**For test gap tasks:** the steps are write test → run (GREEN, since the code already works) → commit. No TDD red-green cycle here — the code exists, we're backfilling coverage.

### Step 6: Handoff

```
Plan saved to docs/plans/simplify-<branch>/

**Tasks:** N tasks
[Summary table]

Run aria:exec to begin cleanup.
```

## Key Rules

- **Read-only scan** — never modify code during steps 1-4. The scan phase is pure analysis.
- **Human curates** — always present findings and wait for approval before generating the plan.
- **No false positives over missed findings** — if unsure whether something is dead code or duplication, err on the side of including it with a note. The human will remove false positives.
- **Never create `.aria/`** — this skill has nothing to do with the knowledge system.
- If `.aria/project.md` exists, read it for project context (conventions, patterns to follow during the scan). **Never create it.**
- **If no findings:** "Code looks clean. No simplification plan needed."
- **Base branch detection:** try `develop` first, then `main`, then `master`. If none found, ask the user.

## Integration

- **aria:exec** — executes the generated plan
- **aria:resume** — resumes multi-session simplification
- **aria:review** — can be run after simplification for final check
- **aria:baseline** — simplify-generated plans modify existing code, so exec will trigger baseline automatically in MEDIUM mode
