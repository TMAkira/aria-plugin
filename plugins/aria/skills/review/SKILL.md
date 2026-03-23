---
name: review
description: Use when a task or set of tasks is complete and needs code review. Use when the user asks for a review of changes. Use at the end of aria:exec or per-task in HARD mode.
---

# Review — Code Review

Review code changes for correctness, quality, and adherence to the design.

**Announce at start:** "Using aria:review to review the changes."

## Scope

Determine what to review:
- **Per-task review** (during HARD exec): review only the changes from the last task
- **Full review** (end of exec): review all changes since the worktree was created

## Checklist

For each changed file:

1. **Correctness** — Does this do what the design says it should?
2. **Tests** — Are the tests meaningful? Do they test behavior, not implementation?
3. **Regressions** — Could this break something not covered by tests?
4. **Patterns** — Does this follow `docs/patterns/` and existing codebase conventions? No magic strings, no hardcoded values?
5. **Simplicity** — Is this the simplest solution? Could anything be removed?
6. **Security** — Any injection, auth bypass, data leak risks? Check project-specific security concerns in CLAUDE.md or `docs/patterns/`
7. **Performance** — Any obvious N+1 queries, unnecessary allocations, blocking calls?
8. **New patterns** — Did this task introduce a new pattern worth documenting in `docs/patterns/`? If yes, flag it as a SUGGESTION.

## Output

```
## Code Review

**Scope:** [Task N only / All N tasks]
**Files reviewed:** [count]

**Issues:**
- [BLOCKER] [file:line] — [description]
- [WARNING] [file:line] — [description]
- [SUGGESTION] [file:line] — [description]

**Strengths:**
- [What's done well]

**Verdict:** [APPROVED / NEEDS CHANGES]
```

**BLOCKER** = must fix before proceeding
**WARNING** = should fix, but not blocking
**SUGGESTION** = optional improvement

## Integration

- **aria:exec** — calls review after each task (HARD) or at completion
- Can be invoked standalone: `/review` to review current uncommitted changes
