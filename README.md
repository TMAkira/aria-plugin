# Aria Marketplace — Claude Code Plugins

Two plugins for [Claude Code](https://docs.anthropic.com/en/docs/claude-code): a structured dev workflow and a release operations toolkit.

```
plugins/
  aria/          # Human-in-the-loop dev workflow
  aria-ops/      # Release operations toolkit
```

---

## aria — Development Workflow

A human-in-the-loop development workflow. Aria enforces a structured process: **design first, then plan, then execute stepwise with checkpoints**.

### Skills

#### Core Workflow

| Skill | Purpose |
|-------|---------|
| `aria:design` | Functional & technical discussion before any implementation |
| `aria:plan` | Break validated designs into task files with TDD steps |
| `aria:exec` | Execute tasks one-by-one with human checkpoints between each |
| `aria:baseline` | Capture test state before modifying code for regression comparison |
| `aria:review` | Code review of changes (per-task or full) |
| `aria:worktree` | Git worktree isolation for feature work |
| `aria:resume-plan` | Resume interrupted work from a previous session |
| `aria:abort` | Clean shutdown with options to keep, shelve, or discard work |

#### Post-Implementation

| Skill | Purpose |
|-------|---------|
| `aria:simplify` | Scan for dead code, duplication, test gaps, and complexity — generate a cleanup plan |

#### Project Knowledge (Experimental)

| Skill | Purpose |
|-------|---------|
| `aria:setup` | Initialize project knowledge in `.aria/` — interactive project scan |
| `aria:learn` | Capture session learnings to `.aria/learnings.md` |

### Workflows

**Feature Implementation:**
```
aria:design -> aria:plan -> aria:worktree -> aria:exec -> aria:review
                                               ^            |
                                               +-- aria:resume-plan (if interrupted)
```

**Post-Implementation Cleanup:**
```
aria:simplify (scan + plan) -> aria:exec -> aria:review
```

**Project Knowledge (Opt-in):**
```
aria:setup (once per project) -> work normally -> aria:learn (after completing work)
```

Aria skills automatically load `.aria/` context when it exists. If you never run `aria:setup`, nothing changes — the knowledge system is fully opt-in. The `.aria/` directory is meant to be committed to your repo so the whole team benefits.

### Philosophy

- **No implementation without a validated design** — every change goes through design
- **One task, one commit** — each task produces a commit with tests
- **Human checkpoints** — never proceed to the next task without approval
- **Tests are written WITH the task, never after** — TDD is enforced
- **Evidence before claims** — run tests, show output, then report

### Installation

Enable `aria@aria-marketplace` in your Claude Code settings.

---

## aria-ops — Release Operations Toolkit

A collection of release and operations skills. Each skill is self-contained with its own config file at `.aria-ops/<skill>.json` in your project root. Works with any tech stack.

### Skills

| Skill | Purpose | Config |
|-------|---------|--------|
| `aria-ops:commitpush` | Commit all changes and push with a structured report | `.aria-ops/commitpush.json` |
| `aria-ops:changelog` | Generate technical + user-facing changelogs from git history and optional ADO PRs. Supports `--backfill` for retroactive generation | `.aria-ops/changelog.json` |
| `aria-ops:testplan` | Generate manual QA checklists from code diff. Auto-detects domains or uses configured mappings | `.aria-ops/testplan.json` |
| `aria-ops:merge` | Merge ADO pull request + cleanup worktree/branch | `.aria-ops/merge.json` |
| `aria-ops:release` | Full release workflow: version bump + changelog + merge to production + tag. Supports PR-based or direct push | `.aria-ops/release.json` |

### Key Features

- **Config per skill** — each skill stores its config in `.aria-ops/<skill>.json` at project root
- **Interactive setup** — first run prompts for configuration if the config file is missing
- **Cross-config prefill** — common fields (ADO settings, branches) are pre-filled from other existing configs
- **ADO optional** — most skills work in git-only mode; Azure DevOps integration is opt-in
- **Stack agnostic** — not tied to any language or framework

### Configuration

| Skill | ADO Required | Key Settings |
|-------|-------------|--------------|
| `commitpush` | No | Co-author, conventional commits, architecture docs path |
| `changelog` | No | Output dirs, languages, client persona |
| `testplan` | No | Test command, domains, DB checks |
| `merge` | **Yes** | Merge strategy, worktree convention |
| `release` | No | Branches, version file, release method |

### Installation

Enable `aria-ops@aria-marketplace` in your Claude Code settings.

---

## Marketplace Setup

```bash
# Add the marketplace
claude plugins marketplace add TMAkira/aria-plugin

# Update plugins
claude plugins update aria
claude plugins update aria-ops
```

## License

MIT
