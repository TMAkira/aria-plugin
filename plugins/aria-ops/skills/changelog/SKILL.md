---
name: changelog
description: Generate structured changelogs (technical + user-facing) from git history and optionally Azure DevOps PRs/work items. Use this skill whenever the user mentions changelog, release notes, "what changed", "what's new", documenting a version, writing up changes for a release, or backfilling historical changelogs. Also triggers on "generate changelog", "write release notes", "document the changes since last version", or any request to produce a summary of changes between versions or tags. Supports --backfill for retroactive generation of all missing versions. Do NOT use for git log browsing, README updates, or blog posts about features.
---

# aria-ops:changelog — Generate changelogs from git history

Generate structured changelogs combining git history and (optionally) Azure DevOps PR/work item data. Produces both a technical changelog (for developers) and a client-facing changelog (for end users).

**Announce at start:** "Using aria-ops:changelog to generate changelog."

**Usage:**
- `aria-ops:changelog` — generate for the latest undocumented version
- `aria-ops:changelog v1.9.5` — generate for a specific version
- `aria-ops:changelog --backfill` — generate for ALL missing versions
- `aria-ops:changelog --backfill v1.5.0..v1.8.0` — generate for a specific range

## Workflow

Follow these steps in order. Do NOT skip steps.

---

### Step 0: Load or Setup Configuration

1. Determine project root: `git rev-parse --show-toplevel`
2. Check if `.aria-ops/changelog.json` exists at the project root

**If found:** load it, proceed to Step 1.

**If NOT found:** run interactive setup.

**Pre-fill:** scan `.aria-ops/*.json` for existing configs. If any contain `ado.project`, `ado.repoId`, `ado.org`, or `branches.main`, reuse those values as defaults.

Ask the user these questions **one at a time**, showing the default in parentheses:

1. "Azure DevOps project name? (leave empty if not using ADO)" — default: pre-fill from other configs or empty
2. **If ADO project was provided:**
   - "ADO Repository ID (GUID)?" — default: pre-fill or empty
   - "ADO Organization name?" — default: pre-fill or empty
3. "Main branch name?" — default: auto-detect via `git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null` or `develop`
4. "Technical changelog directory?" — default: `changelog/technical`
5. "Client changelog directory?" — default: `changelog/client`
6. "Target persona for client changelog? (describes the typical end user)" — default: `a non-technical user of the application`
7. "Technical changelog language?" — default: `en`
8. "Client changelog language?" — default: `fr`

Write `.aria-ops/changelog.json`:
```json
{
  "ado": {
    "project": "<answer or null>",
    "repoId": "<answer or null>",
    "org": "<answer or null>"
  },
  "branches": {
    "main": "<answer>"
  },
  "changelog": {
    "technicalDir": "<answer>",
    "clientDir": "<answer>",
    "clientPersona": "<answer>",
    "technicalLanguage": "<answer>",
    "clientLanguage": "<answer>"
  },
  "backfill": {
    "outputMode": null
  }
}
```

Create the `.aria-ops/` directory if it doesn't exist.
Tell the user: **"Config saved in `.aria-ops/changelog.json`. Continuing with changelog generation."**

---

### Step 1: Parse arguments and determine mode

Parse `$ARGUMENTS`:

**Backfill mode** (`--backfill` present):
- If a range is provided (e.g., `v1.5.0..v1.8.0`), extract start and end versions
- Jump to **Step 1B: Backfill Setup**

**Normal mode** (no `--backfill`):
- If a version is provided (e.g., `v1.9.5`): normalize (strip leading `v` if present), find the tag `git tag -l "v<version>"` — abort if not found
- If no version provided: auto-detect (see Step 1A)

#### Step 1A: Auto-detect target version

1. List all tags: `git tag --sort=-v:refname`
2. Scan `<config.changelog.technicalDir>/` for existing version files (`vX.Y.Z.md`) AND check `<config.changelog.technicalDir>/CHANGELOG.md` for documented versions
3. Identify versions with a git tag but NO changelog file and NO entry in the cumulative CHANGELOG.md
4. If all versions are documented: **"All versions are documented. Use `aria-ops:changelog vX.Y.Z` to regenerate a specific version."** — stop here
5. Otherwise, take the **most recent** undocumented version as target

#### Step 1B: Backfill Setup

1. List all tags sorted by version: `git tag --sort=v:refname`
2. Scan `<config.changelog.technicalDir>/` for existing version files AND `CHANGELOG.md` for documented versions
3. If a range was specified, filter tags to only those within the range (inclusive)
4. Remove already-documented versions from the list
5. Build consecutive tag pairs: `[(null, v1.0.0), (v1.0.0, v1.1.0), ...]` where `null` means "beginning of history"
6. **Check `config.backfill.outputMode`:**
   - If `null` or missing: ask the user:
     - "Backfill output mode?"
     - `individual` — one file per version (like normal changelog)
     - `cumulative` — single CHANGELOG.md with all versions
     - `both` — both formats
   - Save the answer back to `.aria-ops/changelog.json` under `backfill.outputMode`
7. Show the user:
   ```
   Versions to document: v1.0.0, v1.1.0, v1.4.0, ... (N versions)
   Already documented: (list or "none")
   Output mode: <outputMode>
   ```
8. Ask: **"Proceed with generation for these N versions?"**
9. **Do NOT proceed without explicit confirmation.**

After confirmation, for each version pair, execute Steps 2–7 sequentially with progress updates after each version:
```
[done] v1.0.0 — 5 entries (2 feat, 2 fix, 1 chore)
[done] v1.1.0 — 12 entries (8 feat, 3 fix, 1 refactor)
...
```

After all versions are processed, jump to **Step 7B: Backfill Write** for final output, then to **Step 8B: Backfill Summary**.

---

### Step 2: Determine tag range

For the target version:
- `from_tag` = the tag immediately before the target version (by version sort)
- `to_tag` = the target version tag
- Special case: for the very first tag, use `git log <first_tag> --oneline` (all history up to that tag)

Show: **"Generating changelog for version {version} ({from_tag}..{to_tag})"**

Get the tag date:
```bash
git log -1 --format='%as' <to_tag>
```

---

### Step 3: COLLECT — Gather data (parallel)

Launch **two parallel agents** using the Task tool:

**Agent 1 — Git history:**

```bash
# For first version (no from_tag):
git log --oneline --no-merges <to_tag>

# For all other versions:
git log --oneline --no-merges <from_tag>..<to_tag>
```

For each commit, extract:
- `hash` (short)
- `message`
- `prefix` (feat/fix/refactor/chore/perf/docs/test — if conventional commit format)

Also extract merge commits to find PR numbers:
```bash
git log --oneline --merges <from_tag>..<to_tag>
```
Extract PR numbers from messages like "Merge pull request NNN" or "Merged PR NNN".

Return the structured list of commits and PR numbers.

**Agent 2 — Azure DevOps PRs & work items:**

**Skip entirely if `config.ado.project` is null or empty.** Return empty results — the pipeline continues in git-only mode.

If ADO is configured, for each PR number found in merge commits:
1. Use `mcp__ado__repo_get_pull_request_by_id` with:
   - **Project**: `config.ado.project`
   - **repositoryId**: `config.ado.repoId`
   - **pullRequestId**: the PR number (as a NUMBER, not string)
2. Extract: PR title, description
3. For each PR, check linked work items via the PR's work item refs
4. For each linked work item, use `mcp__ado__wit_get_work_item` to get:
   - Title
   - Type (Bug, User Story, Task)
   - Description (first 200 chars for context)

Return the structured list of PRs with their work items.

**If no PR numbers are found** (older versions without PRs), Agent 2 returns empty — that's fine.

---

### Step 4: MERGE — Combine sources

Merge the results from both agents into unified records:

```
{
  hash, message, category,
  pr_number?, pr_title?,
  work_item_id?, work_item_title?, work_item_type?
}
```

**Deduplication rules:**
- If a commit message matches a PR title (or is clearly the same change), keep the PR as primary source — PR titles are usually better written
- Commits that are part of a merged PR (identified via merge commit) get linked to that PR
- Orphan commits (no PR) stay as standalone entries

---

### Step 5: CLASSIFY — Categorize entries

Assign each entry to a category:

| Category | Key | Icon | Criteria |
|----------|-----|------|----------|
| Features | `feat` | ✨ | `feat:` prefix, or work item type = User Story |
| Fixes | `fix` | 🐛 | `fix:` prefix, or work item type = Bug |
| Performance | `perf` | ⚡ | `perf:` prefix |
| Refactoring | `refactor` | ♻️ | `refactor:` prefix |
| Infrastructure | `infra` | 🔧 | `chore:`, `docs:`, `test:`, CI/CD-related |

**For commits without conventional commit prefix** (common in older versions):
- Analyze the message content to infer the category
- Messages in any language: look for keywords indicating add/new/create = `feat`, fix/correct/repair = `fix`, improve/optimize/speed = `perf`, refactor/restructure/rewrite = `refactor`
- When ambiguous, prefer `feat` over `refactor`

---

### Step 6: PRUNE — Filter noise

**Remove entirely** (never appears in any changelog):
- Merge commits (`Merge pull request...`, `Merged PR...`, `merge settings`)
- Version bumps (`chore: bump version to...`)
- Agent/tool/skill setup commits (`chore: add /xxx command`, `chore: add xxx skill`, `chore: initialize ... knowledge`)
- Pure CI/CD config changes (only `.yml`, `Dockerfile`, pipeline files changed)
- Duplicate entries (same change appearing as both commit and PR — keep PR version)
- **Explicitly excluded entries**: if a PR description or commit message contains `DO NOT SHOW IN CHANGELOG`, `NO CHANGELOG`, `skip changelog`, or similar exclusion markers — remove from ALL changelogs

**Technical only** (appears in technical changelog but NOT in client changelog):
- `refactor:` entries
- `test:` entries
- `chore:` entries that are substantive (not trivial config)
- `docs:` entries
- Infrastructure/DevOps changes (review apps, pipelines, agent pools)
- Dependency upgrades that don't affect users (package updates, SDK bumps)
- **All backend-only performance optimizations** (auth, SQL queries, caching, telemetry) — users don't see these directly
- **CSS/UI micro-fixes** that users wouldn't have reported as bugs
- **Internal data sync fixes** (background sync, data consistency between internal models)
- **Logging/monitoring changes** (log levels, telemetry config)

**Client changelog inclusion test — apply this for EVERY candidate entry:**

> "Could {config.changelog.clientPersona} describe this change in their own words?"

- If **YES** — include. The user can see it, use it, or was affected by it.
- If **NO** — technical only. It requires technical context to understand.

**What DOES make it into the client changelog:**
- New screens, buttons, dialogs, or workflows the user can see and use
- Fixes for bugs the user would have encountered and could describe
- Noticeably faster pages (only if the user would have complained about slowness before)

**What does NOT make it into the client changelog:**
- Backend performance (auth, SQL, caching, websockets) — invisible to users
- CSS tweaks users never noticed (z-index, hover states, overflow)
- Data sync/consistency fixes between internal models
- Anything where you'd need to explain the technical context for the user to understand

---

### Step 7: GENERATE & WRITE — Produce changelogs (parallel)

Launch **two parallel agents** using the Task tool:

**Agent 1 — Technical changelog:**

Generate markdown in `config.changelog.technicalLanguage`:

```markdown
## [X.Y.Z] - YYYY-MM-DD

### Features
- Description of feature (#PR if available)

### Fixes
- Description of fix (#PR if available)

### Performance
- Description of perf improvement (#PR if available)

### Refactoring
- Description of refactoring

### Infrastructure
- Description of infra change
```

**Rules:**
- Omit empty sections (if no perf changes, don't include the Performance header)
- Each entry is one line, starting with `- `
- Include PR number as `(#NNN)` suffix when available
- Include work item ID as `(ADO #NNN)` suffix when available and no PR
- Write in `config.changelog.technicalLanguage`
- Entries sorted by relevance within each section (most impactful first)

**Write** to `<config.changelog.technicalDir>/vX.Y.Z.md`:
- The file contains ONLY this version's content (no `# Changelog` header — just the `## [X.Y.Z]` section)
- Overwrite if file already exists

**Agent 2 — Client changelog:**

Generate markdown in `config.changelog.clientLanguage`:

For `clientLanguage = "fr"`:
```markdown
## Version X.Y.Z — DD mois YYYY

### Nouvelles fonctionnalites
- **Short title** — What the user can do now that they couldn't before.

### Corrections
- **Short title** — What visible problem the user won't encounter anymore.
```

For `clientLanguage = "en"`:
```markdown
## Version X.Y.Z — Month DD, YYYY

### New Features
- **Short title** — What the user can do now that they couldn't before.

### Fixes
- **Short title** — What visible problem the user won't encounter anymore.
```

**Rules:**
- Write in `config.changelog.clientLanguage`
- **Strictly non-technical** — the audience is `config.changelog.clientPersona`
- Date format: localized (e.g., `18 fevrier 2026` for fr, `February 18, 2026` for en)
- Each entry has a **bold short title** (functional area or action) followed by ` — ` and a description
- **Description = user's perspective.** Describe what they see, not what was changed in code
- Group related commits into ONE entry per user-visible outcome
- Omit empty sections
- **No "Performance improvements" section** unless the user would have noticed slowness
- **Self-test before including any entry:** "Could {config.changelog.clientPersona} describe this problem or feature?" If not — omit.
- If a version has NO user-facing changes, write a single line: "Internal improvements and technical fixes." (localized)
- Prefer fewer, meaningful entries over a long list. **3 clear entries > 10 obscure ones.**

**Client changelog rewriting rules:**
- Describe the **user's experience**, not the technical change
- Write from the user's perspective: what can they DO now, or what PROBLEM is gone
- If you can't write a description without technical jargon, the entry probably doesn't belong in the client changelog
- **Good:** "Document export — You can now export documents directly from the detail page without navigating away."
- **Bad:** "Improved synchronization of reference data tables."

**Write** to `<config.changelog.clientDir>/vX.Y.Z.md`:
- Same rules as technical: one file per version, just the version section
- Overwrite if file already exists

---

### Step 7B: Backfill Write (backfill mode only)

After all versions have been processed through Steps 2-7, write cumulative files based on `config.backfill.outputMode`:

**If outputMode is `individual` or `both`:**
- Individual files were already written in Step 7 for each version. Nothing extra needed.

**If outputMode is `cumulative` or `both`:**

**`<config.changelog.technicalDir>/CHANGELOG.md`:**
- If file doesn't exist: create with `# Changelog\n\n`
- Insert all generated sections in reverse chronological order (newest first)
- If some versions already exist, preserve them and only add missing ones

**`<config.changelog.clientDir>/CHANGELOG.md`:**
- Same structure
- Header localized based on `config.changelog.clientLanguage` (e.g., `# Changelog` for en, `# Nouveautes` for fr)

---

### Step 8: Present & Confirm (normal mode)

Show the user a summary:
- Number of entries per category
- Any entries that were ambiguous to classify (ask if reclassification needed)
- Preview of the generated content (first 30 lines of each file)

Ask: **"Changelog for {version} has been generated. Want me to show the full content, modify something, or is it good?"** (in the user's interaction language)

If the user wants changes, edit the files accordingly. Otherwise, done.

**Do NOT commit automatically.** Suggest committing when the user is ready.

---

### Step 8B: Backfill Summary (backfill mode only)

Show a consolidated summary table:

```
====================================================
    CHANGELOG BACKFILL — N versions
====================================================

| Version | Date | Feat | Fix | Perf | Refactor | Infra | Total |
|---------|------|------|-----|------|----------|-------|-------|
| v1.9.5  | 2026-02-18 | 2 | 3 | 0 | 0 | 1 | 6 |
| v1.9.4  | ... | ... | ... | ... | ... | ... | ... |

Total: N technical entries, M client entries
Output mode: <outputMode>
Files: <list of written files>
```

Ask: **"Changelogs have been generated. Want to commit and push?"** (in the user's interaction language)

**Do NOT commit automatically.**

---

## Test Detection (multi-framework)

When analyzing commits for test-related entries, detect test methods by common decorators across frameworks:

| Framework | Markers |
|-----------|---------|
| C# (MSTest) | `[TestMethod]`, `[DataTestMethod]` |
| C# (xUnit) | `[Fact]`, `[Theory]` |
| C# (NUnit) | `[Test]`, `[TestCase]` |
| JS/TS (Jest/Vitest) | `it(`, `test(`, `describe(` |
| Python (pytest) | `def test_`, `@pytest.mark` |
| Python (unittest) | `class Test`, `def test_` |
| Java (JUnit) | `@Test`, `@ParameterizedTest` |
| Go | `func Test` |
| Rust | `#[test]`, `#[cfg(test)]` |

Use these markers to identify test-related commits for classification as `infra` category.

---

## Notes

- **ADO is optional.** If `config.ado.project` is null/empty, the entire pipeline works in git-only mode. PR enrichment and work item data are simply absent — classification relies solely on commit messages.
- **Old versions are messy.** Older versions may have no conventional commit prefixes, messages in various languages, no PRs (direct pushes), and few/no linked work items. The pipeline handles this gracefully — classification falls back to message content analysis.
- **Cumulative CHANGELOG.md files** (from backfill) are NOT updated by normal single-version runs. Normal runs only produce individual version files.
- **Language flexibility.** Technical and client changelogs can be in different languages. Section headers, date formats, and prose adapt to the configured language.
