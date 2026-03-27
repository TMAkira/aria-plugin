---
name: plan
description: Use when a design is validated and implementation needs to be planned. Use when the user asks to plan, break down, or organize implementation work. Use after aria:design completes.
---

# Plan — Implementation Planning

Create a detailed implementation plan structured as a meta-plan + individual task files. Each task has a single testable objective and produces working code.

**Announce at start:** "Using aria:plan to create the implementation plan."

## Output Structure

```
docs/plans/<feature-name>/
  plan.md                    # Meta-plan: goal, architecture, task order, status, dependencies
  task-01-<short-name>.md    # Detailed task 1
  task-02-<short-name>.md    # Detailed task 2
  task-03-<short-name>.md    # Detailed task 3
  ...
```

User preferences for plan location override this default.

## The Process

### Step 1: Understand the Scope

Read the design document (from aria:design). Extract:
- The impact level (Refacto complet / Refacto / Ajout / Suppression)
- The key components to build or modify
- The dependencies between components
- The verification criteria
- If `.aria/project.md` exists, read it for project structure and conventions. If `.aria/learnings.md` exists, check for learnings relevant to the planned work. **Never create `.aria/`** — skip silently if absent.

### Step 2: Identify the Vertical Slice

Ask: **"What is the minimum work needed to get a first working result?"**

Order tasks so that:
1. The first task produces something testable and visible
2. Each subsequent task enriches something that already works
3. Interfaces and contracts emerge naturally in the tasks that need them — no artificial "scaffold" task

### Step 3: Decompose into Tasks

Each task must have:
- **A single testable objective** — not "build feature X" but "implement function Y that does Z"
- **Clear inputs and outputs** — what exists before, what exists after
- **Verification criteria** — how to prove this task is done
- **Estimated scope** — which files, roughly how much code

**Task granularity check:** Can you describe what this task achieves in one sentence without using "and"? If not, split it.

### Step 3.1: Present Task Outline for Approval

Before writing any detailed files, present the task list to the human:

```
## Proposed Tasks

| # | Task | Objective | Files touched | Dependencies |
|---|------|-----------|---------------|--------------|
| 1 | [Name] | [One sentence] | [key files] | — |
| 2 | [Name] | [One sentence] | [key files] | Task 1 |
| 3 | [Name] | [One sentence] | [key files] | Task 1 |

Does this breakdown look right? Any task too big, missing, or in the wrong order?
```

Wait for the human's feedback. Adjust the decomposition before writing detailed files. This is cheap to change now, expensive to change later.

### Step 4: Write the Meta-Plan

```markdown
# [Feature Name] Implementation Plan

**Goal:** [One sentence]
**Impact level:** [Refacto complet / Refacto / Ajout / Suppression]
**Design doc:** [path to design doc]
**Exec mode:** [HARD / MEDIUM / EASY]

## Architecture

[2-3 sentences about the technical approach]

## Task Order

| # | Task | Status | Dependencies | File |
|---|------|--------|--------------|------|
| 1 | [Short description] | pending | — | task-01-xxx.md |
| 2 | [Short description] | pending | Task 1 | task-02-xxx.md |
| 3 | [Short description] | pending | Task 1 | task-03-xxx.md |
| 4 | [Short description] | pending | Tasks 2, 3 | task-04-xxx.md |

## Discoveries

[Updated during execution — notes from aria:exec about things that affect future tasks]

## Status

- **Started:** [date]
- **Current task:** —
- **Completed:** 0/N
- **Last updated:** [date]
```

### Step 5: Write Individual Task Files

````markdown
# Task N: [Task Name]

**Objective:** [Single sentence — what this achieves]
**Depends on:** [Task numbers, or "None"]
**Impact on future tasks:** [What later tasks assume about this one]

## Files

- **Create:** `exact/path/to/new-file.ext`
- **Modify:** `exact/path/to/existing-file.ext` (functions: X, Y)
- **Test:** `exact/path/to/test-file.ext`

## Steps

### 1. Write the failing test

```
// Test following the project's test conventions and naming patterns
// Check docs/patterns/ and existing tests for the expected style
// Complete test code — not pseudocode
```

### 2. Run test — expect RED

```bash
<project test command> --filter "<test name>"
```

Expected: FAIL

### 3. Implement

```
// Complete implementation code — not pseudocode, not "add validation here"
// Follow patterns from docs/patterns/ and existing code conventions
// No magic strings, no hardcoded values — check how similar code does it
```

### 4. Run test — expect GREEN

```bash
<project test command> --filter "<test class>"
```

Expected: ALL PASS

### 5. Commit

```bash
git add <specific files>
git commit -m "feat: [what this task achieves]"
```

## Verification

- [ ] Test passes
- [ ] No regressions (run broader test suite)
- [ ] [Any task-specific verification]
````

## Task Ordering Principles

1. **Vertical slice first** — get something working end-to-end before widening
2. **Dependencies flow downward** — Task N never depends on Task N+M
3. **Test infrastructure early** — if you need test fixtures or helpers, that's Task 1
4. **Risky tasks early** — the thing most likely to force a plan revision should come first, so you discover problems early
5. **Each task is independently valuable** — if you stopped after Task N, you'd still have working software (maybe incomplete, but not broken)

## Scope Check

If the plan exceeds 10 tasks, consider:
- Is this actually multiple features that should be separate plans?
- Can some tasks be combined without losing testability?
- Is the granularity too fine? (Remember: single testable objective, not single function)

If the plan has fewer than 3 tasks, consider:
- Is each task too large? Can you describe it without "and"?
- Are there verification steps missing?

## Handoff

After saving the plan, present it to the human:

```
Plan saved to docs/plans/<feature-name>/

**Tasks:** N tasks, estimated exec mode: [HARD/MEDIUM/EASY]

[Summary table from meta-plan]

Review the plan. When ready, run aria:exec to begin implementation.
```

Wait for the human to review and approve before invoking aria:exec.

## Integration

- **aria:design** — produces the design this skill consumes
- **aria:exec** — executes the plan this skill produces
- **aria:resume-plan** — reads the meta-plan to resume interrupted work
