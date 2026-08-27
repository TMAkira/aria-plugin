# Knowledge Location

Where aria's project knowledge lives. Every skill that reads or writes `project.md` or
`learnings.md` resolves the location the same way.

## Two modes

| Mode | Location | For |
|---|---|---|
| **repo** | `<repo>/.aria/` | Knowledge the team should share. Committed with the code. |
| **personal** | `~/.claude/aria/projects/<slug>/` | Knowledge that must not land in someone else's repository. Never committed. |

The mode is chosen once, by `aria:setup`. Nothing else creates either directory.

**Why personal mode exists.** `.aria/` is committed by design, which makes it unusable on a
repository you do not own the content policy of — a client's, an employer's, a shared
open-source one. Without a personal mode the only options were to push private working notes
into someone else's history, or to never run `aria:setup` at all. The second is what people
actually did, and it silently disabled `aria:learn` for those projects: it stopped on a
missing `.aria/` and captured nothing, for as long as the project lasted.

## Resolving the location

Run this before reading or writing knowledge. It is the same three steps everywhere.

```bash
# 1. Repo mode wins when the directory is there.
ls -d .aria 2>/dev/null

# 2. Otherwise look for personal knowledge for this repository.
#    --git-common-dir, not --show-toplevel: inside a linked worktree the toplevel is the
#    worktree, which would give every worktree of one repository a different slug.
slug=$(basename "$(dirname "$(git rev-parse --path-format=absolute --git-common-dir 2>/dev/null)")")
ls -d "$HOME/.claude/aria/projects/$slug" 2>/dev/null

# 3. Neither: this project has no aria knowledge.
```

- **Reading skills** (`design`, `plan`, `exec`, `simplify`) use whichever they find, and skip
  silently when there is none. They never create either directory.
- **`aria:learn`** appends to whichever it finds. When there is none it points at
  `aria:setup` and stops — including the personal option, so the answer to a repository you
  cannot commit to is not "give up".
- **`aria:setup`** creates one or the other, after asking.

When both exist, repo mode wins: it is the deliberate, shared choice.

## Slug

The directory name of the repository — `basename` of the parent of the common git dir. For
`~/dev/acme-api` and every worktree of it, the slug is `acme-api`. Outside a git repository,
fall back to the current directory's name.

Two unrelated repositories with the same directory name would collide. That is a real but
narrow case; `aria:setup` shows the resolved path before writing, so it is visible before it
matters.

## Committing

- **repo mode** — `git add .aria/…` and commit, as before.
- **personal mode** — never run git. The files are outside the repository; a `git add` would
  either fail or, worse, succeed against the wrong repository. Say where the file was written
  instead.
