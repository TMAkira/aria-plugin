# Aria — Claude Code Plugin

A human-in-the-loop development workflow for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Aria enforces a structured process: **design first, then plan, then execute stepwise with checkpoints**.

## Skills

### Core Workflow

| Skill | Purpose |
|-------|---------|
| `aria:design` | Functional & technical discussion before any implementation |
| `aria:plan` | Break validated designs into task files with TDD steps |
| `aria:exec` | Execute tasks one-by-one with human checkpoints between each |
| `aria:baseline` | Capture test state before modifying code for regression comparison |
| `aria:review` | Code review of changes (per-task or full) |
| `aria:worktree` | Git worktree isolation for feature work |
| `aria:resume` | Resume interrupted work from a previous session |
| `aria:abort` | Clean shutdown with options to keep, shelve, or discard work |

### Post-Implementation

| Skill | Purpose |
|-------|---------|
| `aria:simplify` | Scan for dead code, duplication, test gaps, and complexity — generate a cleanup plan |

### Project Knowledge (Experimental)

| Skill | Purpose |
|-------|---------|
| `aria:setup` | Initialize project knowledge in `.aria/` — interactive project scan |
| `aria:learn` | Capture session learnings to `.aria/learnings.md` |

## Workflows

### Feature Implementation
```
aria:design → aria:plan → aria:worktree → aria:exec → aria:review
                                              ↑            |
                                              └── aria:resume (if interrupted)
```

### Post-Implementation Cleanup
```
aria:simplify (scan + plan) → aria:exec → aria:review
```

### Project Knowledge (Experimental, Opt-in)
```
aria:setup (once per project) → work normally → aria:learn (after completing work)
```

Aria skills automatically load `.aria/` context when it exists. If you never run `aria:setup`, nothing changes — the knowledge system is fully opt-in.

The `.aria/` directory is meant to be committed to your repo so the whole team benefits from captured learnings.

## Installation

```bash
# 1. Add the marketplace
claude plugins marketplace add TMAkira/aria-plugin

# 2. Install the plugin
claude plugins install aria
```

## Update

```bash
claude plugins update aria
```

## Philosophy

- **No implementation without a validated design** — every change goes through design, even small ones
- **One task, one commit** — each task produces a commit with tests
- **Human checkpoints** — never proceed to the next task without approval
- **Tests are written WITH the task, never after** — TDD is enforced
- **Evidence before claims** — run tests, show output, then report

## License

MIT
