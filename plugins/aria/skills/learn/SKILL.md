---
name: learn
description: "(Experimental) Use when the user wants to capture learnings from a work session. Use after completing a plan or when something surprising happened. Use when the user says learn, capture, or remember for the project."
---

# Learn — Capture Session Learnings (Experimental)

Extract non-obvious insights from a work session and append them to `.aria/learnings.md`.

**Announce at start:** "Using aria:learn to capture learnings from this session. (This feature is experimental.)"

## Pre-flight

```bash
ls .aria/learnings.md 2>/dev/null
```

- **If not found:** "No `.aria/` found. Run `aria:setup` first to initialize project knowledge." — STOP.
- **Never create `.aria/`** — only aria:setup does that.

## Process

### Step 1: Gather Context

Silently collect what happened this session:

```bash
# Recent commits (since last learning entry date, or last 20)
git log --oneline -20

# Current plan if one exists
ls docs/plans/*/plan.md 2>/dev/null
```

- Read the plan's Discoveries section if it exists
- Note which tasks were completed, any plan revisions

### Step 2: Propose Learnings

Based on what happened, propose candidate learnings **one at a time**:

- "I noticed [X pattern was used successfully] — worth capturing?"
- "The plan had to be revised because [Y] — should we remember this?"
- "We discovered [Z] wasn't documented — add it?"
- "[Technical decision] was made during implementation — record the rationale?"

**What makes a good learning:**
- Non-obvious — can't be derived from reading the code or CLAUDE.md
- Reusable — would help a future session avoid a mistake or repeat a success
- Concise — expressible in 2-3 sentences

**What is NOT a learning:**
- Code patterns visible in the code itself
- Things already in CLAUDE.md
- Ephemeral state (current branch, in-progress work)
- Implementation details of what was just built

After proposing, ask: "Anything else surprising or non-obvious from this session?"

### Step 3: Format Entries

For each accepted learning:

```markdown
### <date> — <short title>

**Context:** [what was being done — one sentence]
**Learning:** [the insight — what to know next time]
**Tags:** [comma-separated: architecture, pattern, pitfall, convention, performance, tooling, testing]
```

### Step 4: Preview and Confirm

Show all entries that will be appended:

```
## Learnings to capture

[formatted entries]

Append these to .aria/learnings.md?
```

Wait for approval. The user can edit, remove, or add entries.

### Step 5: Append and Commit

Append to `.aria/learnings.md` — **never overwrite** existing content.

```bash
git add .aria/learnings.md
git commit -m "docs: capture learnings from <brief context>"
```

## Key Rules

- **Never create `.aria/`** — if it doesn't exist, point to aria:setup and stop
- **Append only** — never overwrite, edit, or delete existing entries
- **Concise entries** — 2-4 lines per learning, not paragraphs
- **Non-obvious only** — if it's in the code or CLAUDE.md, it's not a learning
- **Human curates** — propose, don't impose. The user decides what's worth keeping.
- **No duplicates** — read existing learnings.md before proposing, skip topics already captured

## Integration

- **aria:setup** — creates `.aria/learnings.md` that this skill appends to
- **aria:exec** — proposes learn at plan completion (when `.aria/` exists)
