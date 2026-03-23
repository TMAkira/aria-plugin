---
name: testplan
description: Generate manual QA checklists from actual code changes (git diff). Use this skill whenever the user asks about manual testing, QA checklists, "what should I test", "what needs testing", test plans for a release, pre-release verification, or preparing for QA. Also use when the user says "testplan", "generate test plan", "what to test before deploying", "mark as tested", or wants a structured checklist of manual verification steps organized by domain/feature area. Do NOT use for running automated tests, writing unit/integration/E2E test code, or auditing test coverage.
---

# aria-ops:testplan — Generate manual QA checklist from code changes

Generate a specific, actionable manual test checklist based on **actual code changes** since the last tested release. Tests are grouped by page/domain, labeled by type, and optionally include DB integrity checks.

**Announce at start:** "Using aria-ops:testplan to generate a QA checklist."

**Usage:**
- `aria-ops:testplan` — checklist from last tested tag to main branch
- `aria-ops:testplan v1.13.0` — force a specific starting tag
- `aria-ops:testplan users` — filter tests to a specific domain
- `aria-ops:testplan mark` — mark the most recent tag as tested

## Test Type Labels

Use these labels to classify every test item:

| Label | Meaning |
|-------|--------|
| `[CRUD]` | Create / Read / Update / Delete operations |
| `[UI]` | Visual rendering, no crash, correct display |
| `[REALTIME]` | Real-time updates (WebSocket, SignalR, SSE, etc.) |
| `[AUTH]` | Permissions, 403 handling, role-based access |
| `[PERF]` | Performance-sensitive (grids 50+ items, batch ops) |
| `[DATA]` | Data integrity, no corruption, FK consistency |

## Workflow

Follow these steps in order. Do NOT skip steps.

### Step 0: Load or Setup config

Look for `.aria-ops/testplan.json` at the project root.

**If the file exists**, load it. The schema is:

```json
{
  "ado": {
    "project": "MyProject",
    "repoId": "guid-here",
    "org": "my-org"
  },
  "branches": {
    "main": "develop"
  },
  "test": {
    "command": "dotnet test --filter Category!=Manual"
  },
  "testplan": {
    "outputDir": "changelog/testing",
    "domains": [
      { "name": "Users", "patterns": ["**/User*", "**/Auth*", "**/Permission*"] },
      { "name": "Orders", "patterns": ["**/Order*", "**/Invoice*"] }
    ],
    "dbChecks": {
      "enabled": false,
      "command": "docker exec my_pg psql -U postgres -d mydb -c",
      "globalQueries": [
        { "query": "SELECT COUNT(*) FROM \"User\" WHERE \"Email\" IS NULL;", "expected": "0" }
      ]
    }
  }
}
```

All top-level keys are optional. Defaults:
- `ado`: `null` (ADO integration disabled)
- `branches.main`: auto-detect from `git symbolic-ref refs/remotes/origin/HEAD` or fall back to `"main"`
- `test.command`: auto-detect (see below)
- `testplan.outputDir`: `"changelog/testing"`
- `testplan.domains`: `null` (auto-detect by grouping changed files)
- `testplan.dbChecks.enabled`: `false`

**If the file does NOT exist**, run interactive setup. Ask questions **one at a time**:

1. **Pre-fill**: scan for other `.aria-ops/*.json` files. If any exist, extract `ado`, `branches`, and `test` values to use as defaults.

2. **Questions:**
   - "Azure DevOps project name? (leave empty to skip ADO integration)"
     - If non-empty: "Repository ID (GUID)?" then "Organization name?"
   - "Main branch name?" — default: detected from git, or `"develop"`
   - "Test command?" — default: auto-detect from project files:
     - `*.csproj` found → `"dotnet test"`
     - `package.json` found → `"npm test"`
     - `Makefile` found → `"make test"`
     - `Cargo.toml` found → `"cargo test"`
     - `pyproject.toml` / `setup.py` found → `"pytest"`
     - Otherwise → ask the user
   - "Output directory for test plans?" — default: `"changelog/testing"`
   - "Define domain mappings now? (y/n)"
     - If yes: interactive domain builder — ask for domain name + file patterns, repeat until done
     - If no: will auto-detect domains by grouping changed files by top-level directory/namespace
   - "Enable DB integrity checks? (y/n)"
     - If yes: "DB command prefix?" (e.g., `docker exec pg psql -U postgres -d mydb -c`)

3. Write the config to `.aria-ops/testplan.json`.
4. Continue with the rest of the workflow.

Store resolved config values in variables for use throughout:
- `main_branch` = `config.branches.main` or detected default
- `output_dir` = `config.testplan.outputDir` or `"changelog/testing"`
- `domain_mappings` = `config.testplan.domains` or `null` (auto-detect)
- `ado_enabled` = `config.ado` is non-null
- `db_checks_enabled` = `config.testplan.dbChecks.enabled` is `true`
- `test_command` = `config.test.command` or detected default

### Step 1: Handle `testplan mark`

**If $ARGUMENTS is `mark`:**

1. Find the most recent tag: `git tag --sort=-v:refname | head -1`
2. Read `{output_dir}/state.json` (create if missing)
3. Update `lastTestedTag` to the most recent tag
4. Write `{output_dir}/state.json`
5. Tell the user: **"Tag {tag} marked as tested."**
6. Ask if they want to commit. **STOP here — do not continue.**

### Step 2: Determine the range

**If $ARGUMENTS contains a version** (e.g. `v1.13.0`):
- Normalize: add `v` prefix if missing
- Verify tag exists: `git tag -l "<tag>"` — abort if not found
- `from_tag` = that tag

**If $ARGUMENTS is empty or a domain filter:**
- Read `{output_dir}/state.json`
  - If exists: `from_tag` = `lastTestedTag`
  - If missing: find the second most recent tag and use it as `from_tag`
- `to_ref` = `{main_branch}`

If $ARGUMENTS is non-empty and NOT a version and NOT `mark`, treat it as `domain_filter` (used in step 5 to filter output).

Show: **"Test plan: changes from {from_tag} to {main_branch}"**

Count commits: `git log --oneline --no-merges <from_tag>..{main_branch} | wc -l`
If zero: **"No changes since {from_tag}. Nothing to test."** — STOP.

### Step 3: Analyze actual code changes

This is the critical step. Run in parallel:

**Agent 1 — File-level diff analysis:**

Run `git diff --name-status <from_tag>..{main_branch}` and `git diff --stat <from_tag>..{main_branch}`.

Classify every changed file into **page/domain groups**:

- **If `domain_mappings` is configured:** use the patterns from config to match files to domains. Files matching no pattern go to **Cross-cutting**.
- **If `domain_mappings` is null (auto-detect):** group changed files by their top-level source directory or namespace. Infer domain names from directory names (e.g., `src/Orders/` → "Orders", `components/auth/` → "Auth"). Files in shared/common/utility directories go to **Cross-cutting**.

**Agent 2 — Detailed diff content for test generation:**

Run `git diff <from_tag>..{main_branch}` on source files (exclude binaries, lock files, generated code).

For each changed file, identify:
- **New/modified endpoints** (controller methods, route attributes, API handlers)
- **Changed service methods** (business logic)
- **Modified UI components** (new forms, grid changes, page modifications)
- **Real-time changes** (WebSocket, SignalR, SSE handlers)
- **Permission/auth changes** (authorization attributes, guards, middleware)
- **Entity/model changes** (new properties, FK changes, schema migrations)

**Agent 3 — Git history for context:**

Run `git log --oneline --no-merges <from_tag>..{main_branch}` and `git log --oneline --merges <from_tag>..{main_branch}`.

**If `ado_enabled`:** extract PR numbers from merge commits. For each PR, use `mcp__ado__repo_get_pull_request_by_id` (Project: `config.ado.project`, repositoryId: `config.ado.repoId`) to get title and linked work items.

**If ADO is not configured:** use commit messages and merge commit titles for context. Skip PR/work item enrichment.

### Step 4: Filter noise

**Remove from analysis:**
- Merge commits, version bumps, changelog commits
- Test files (they ARE tests, not things to test)
- CI/CD config, dependency bumps with no API change
- AI assistant command/skill files (`.claude/`, `.cursor/`, etc.)
- Lock files (`package-lock.json`, `yarn.lock`, `Cargo.lock`, etc.)
- Migration files (test via the entity/service that uses them, not directly)
- Generated code files (source generators, API clients, etc.)

### Step 5: Generate structured test checklist

For each domain group that has changes, generate a section. If `domain_filter` is set, only output matching sections.

**Each domain section MUST follow this structure:**

```markdown
## {Domain Name} (`/{route-path}`)

### [AUTH] {Test group title}
- [ ] Access without permission -> 403 displayed properly
- [ ] Access with permission -> page loads normally

### [CRUD] {Test group title}
- **Pre-requisites:** {data needed}
- [ ] Create a {entity} -> {expected behavior}
- [ ] Update {field} -> {expected behavior}
- [ ] Delete -> {expected behavior, including cascade effects}

### [REALTIME] {Test group title}
- **Pre-requisites:** two tabs open on the same page
- [ ] {Action in tab 1} -> {expected update in tab 2}

### [UI] {Test group title}
- **Pre-requisites:** {data needed}
- **URL:** `https://localhost:{port}/{path}`
- [ ] {Step 1} -> {expected result}
- [ ] {Step 2} -> {expected result}

### [PERF] {Test group title}
- **Pre-requisites:** {large dataset}
- [ ] Load grid with 100+ items -> {response time < Xs, no freeze}

### [DATA] Post-test integrity — {Domain}
- [ ] `{db_command} "{query}"` -> {expected count/result}
```

**Only include subsections relevant to the changes.** If no real-time changes in a domain, omit `[REALTIME]`. If no permission changes, omit `[AUTH]`.

Only include `[DATA]` subsections if `db_checks_enabled` is true AND the domain has entity changes.

**Rules for generating test items:**

1. **Derive tests from actual diff** — read the changed methods/components and generate tests that exercise those specific changes. Do NOT generate generic CRUD checklists for unchanged code.

2. **One test item = one verifiable action.** Each checkbox must have a clear action and expected result separated by `->`.

3. **Include page URLs** — derive from route attributes in controllers and UI page/route directives.

4. **Include pre-conditions** — what data must exist to test meaningfully.

5. **Generate DB integrity queries** (only if `db_checks_enabled`) based on changed entities:
   - New FK: verify no orphans with a LEFT JOIN ... WHERE ... IS NULL query
   - Modified entity: verify no nulls in required columns
   - Deleted/archived feature: verify no stale data
   - Include any `globalQueries` from config in the global integrity section

6. **Group related commits** — if 3 commits all touch the same page, generate one test group (not three).

7. **Priority order within each domain:** `[AUTH]` -> `[DATA]` -> `[CRUD]` -> `[REALTIME]` -> `[UI]` -> `[PERF]`

8. **Tester perspective** (not developer). Write test steps as actions a QA person would perform.

9. **Reference PR/work item numbers** inline when available: `(#PR123)` or `(ADO #456)`.

### Step 6: Assemble final output

Structure the final markdown as:

```markdown
# Test Plan — {main_branch} since {from_tag}

Generated on {YYYY-MM-DD} | Commits: {N} | PRs: {N}
Impacted domains: {list of domain names}

---

{Domain sections ordered by number of changes, most changed first}

---

## [DATA] Global integrity (post-session)

Run these checks after all tests:

- [ ] {global DB queries from config, if db_checks_enabled}
- [ ] Check backend logs: no unhandled exceptions during the test session
```

If `db_checks_enabled` is false, omit the global DB integrity section entirely. Always keep the "check backend logs" item.

### Step 7: Save and display

1. Create directory if needed: `{output_dir}/`
2. Write checklist to `{output_dir}/checklist-{from_tag}-to-{main_branch}.md`
3. Display the full checklist in the terminal
4. Summary:
   - **"Checklist saved to `{output_dir}/checklist-{from_tag}-to-{main_branch}.md`"**
   - Item count per domain and per type label (e.g., "Users: 3 [CRUD], 2 [UI], 1 [DATA]")
   - **"Ready to commit? After testing, use `testplan mark` to mark the tag as tested."**

**Do NOT commit automatically.** Wait for the user.

## Integration

- `testplan` is standalone — not part of a release command
- Typical flow: `testplan` -> test manually -> `testplan mark` -> release
- Suggest `testplan` when the user mentions testing, QA, or preparing a release
