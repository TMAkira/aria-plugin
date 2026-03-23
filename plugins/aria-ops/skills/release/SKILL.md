---
name: release
description: Full release workflow — bump version, generate changelog, merge to production branch, create git tag. Use this skill whenever the user wants to release, cut a release, ship to production, bump the version, create a release tag, or says things like "release", "release minor", "release 2.0.0", "ship it to prod", "tag a new version", "deploy a new version", "time to release". Supports both PR-based (locked production branch) and direct push methods. Do NOT use for just generating a changelog (use changelog skill), just merging a PR (use merge skill), just editing a version file, or deploying to staging.
---

# aria-ops:release — Release workflow

Release the current main branch to production by bumping the version, generating a changelog, merging to the production branch, and creating a git tag.

**Announce at start:** "Using aria-ops:release to cut a new release."

**Usage:**
- `aria-ops:release` — bump patch (1.9.3 -> 1.9.4)
- `aria-ops:release minor` — bump minor (1.9.3 -> 1.10.0)
- `aria-ops:release major` — bump major (1.9.3 -> 2.0.0)
- `aria-ops:release 2.0.0` — set explicit version

## Workflow

Follow these steps in order. Do NOT skip steps.

### Step 0: Load Configuration

1. Check if `.aria-ops/release.json` exists at the project root (`git rev-parse --show-toplevel`)
2. **If found:** load it, proceed to Step 1
3. **If NOT found:** run interactive setup:

   **Pre-fill:** scan `.aria-ops/*.json` for existing configs to reuse common fields (ADO, branches)

   Ask the user these questions one at a time:

   - "Azure DevOps project name? (leave empty for direct push releases)" — default: empty
   - If ADO provided:
     - "Repository ID (GUID)?" — no default, required
     - "Organization name?" — pre-fill from other configs if available
   - "Main development branch?" — default: detect from git (`develop`, `main`, `master`) or pre-fill from other configs
   - "Production branch?" — default: detect (`master`, `main`, `production`). Must differ from main branch.
   - "Version file path?" — default: auto-detect by scanning for:
     - `VERSION` file at root
     - `version` field in `package.json`
     - `<Version>` tag in `*.csproj`
     - If none found, ask the user
   - "Release method? (pr/push)" — default: `pr` if ADO configured, `push` otherwise
   - "Generate changelog on release? (y/n)" — default: yes if `.aria-ops/changelog.json` exists, no otherwise
     - If yes and no `changelog.json`: inform the user that `aria-ops:changelog` setup will run on first release

   Write `.aria-ops/release.json`:
   ```json
   {
     "ado": {
       "project": "<answer or null>",
       "repoId": "<answer or null>",
       "org": "<answer or null>"
     },
     "branches": {
       "main": "<answer>",
       "production": "<answer>"
     },
     "versionFile": "<answer>",
     "release": {
       "method": "pr|push",
       "generateChangelog": true|false
     }
   }
   ```

   Create the `.aria-ops/` directory if it doesn't exist.
   Tell the user: **"Config saved in `.aria-ops/release.json`. Continuing with release."**

### Step 1: Read the current version

Read the version from `config.versionFile`:

- **`VERSION` file:** read content, trim whitespace
- **`package.json`:** parse JSON, read `.version`
- **`*.csproj`:** parse XML, read `<Version>` or `<VersionPrefix>`

Parse as `MAJOR.MINOR.PATCH`. If the format is unexpected, show the raw value and ask the user.

### Step 2: Compute the next version

Based on `$ARGUMENTS`:

| Argument | Action | Example |
|----------|--------|---------|
| *(empty)* | Bump patch | 1.9.3 -> 1.9.4 |
| `patch` | Bump patch | 1.9.3 -> 1.9.4 |
| `minor` | Bump minor, reset patch | 1.9.3 -> 1.10.0 |
| `major` | Bump major, reset minor+patch | 1.9.3 -> 2.0.0 |
| `X.Y.Z` (explicit) | Use as-is | 1.9.3 -> 2.0.0 |

### Step 3: Check preconditions

Before proceeding, verify ALL of these:

1. **Current branch is `config.branches.main`** — if not, abort with error
2. **Working tree is clean** — if uncommitted changes exist, ask the user if they want to commit first (suggest `aria-ops:commitpush`)
3. **Main branch is up to date with origin** — run `git fetch origin` and compare `HEAD` with `origin/<main>`

If any check fails, inform the user and stop.

### Step 4: Confirm with the user

Show the user:
- Current version: `X.Y.Z`
- Next version: `X.Y.Z`
- Release method: `pr` or `push`
- Commits to be released: `git log origin/<production>..<main> --oneline`

Ask: **"Release v{next_version} with these commits?"**

**Do NOT proceed without explicit confirmation.**

### Step 5: Generate changelog (if configured)

**Skip if `config.release.generateChangelog` is false.**

Generate the changelog for the new version **before** committing, using `origin/<production>..<main>` as the commit range.

**If `.aria-ops/changelog.json` exists:**
Run the aria-ops:changelog pipeline (steps 1-7) with:
- `from_tag` = latest tag on production (`git describe --tags --abbrev=0 origin/<production>`)
- `to_ref` = HEAD (main branch, not yet tagged)
- Version label = `{next_version}`
- Tag date = today's date

This produces files in the configured changelog directories.

**If `.aria-ops/changelog.json` does NOT exist:**
Inform the user: "No changelog config found. Run `aria-ops:changelog` first to set it up, or set `generateChangelog: false` in release.json to skip."
Ask if they want to continue without changelog or set up changelog config now.

### Step 6: Commit version + changelog

Write the new version to `config.versionFile`:

- **`VERSION` file:** overwrite with `{next_version}\n`
- **`package.json`:** update `.version` field
- **`*.csproj`:** update `<Version>` tag

Stage and commit:
```bash
git add <versionFile> <changelog files if generated>
git commit -m "chore: bump version to {next_version}"
git push origin <main>
```

### Step 7: Merge to production

**Method depends on `config.release.method`:**

#### Method: `push` (direct merge)

```bash
git checkout <production>
git pull origin <production>
git merge <main> --no-edit
git push origin <production>
```

Then proceed to Step 8.

#### Method: `pr` (pull request)

**Requires ADO config.** If `config.ado.project` is null, abort: "PR method requires ADO configuration. Either configure ADO in release.json or switch to push method."

Create a pull request using `mcp__ado__repo_create_pull_request`:
- **repositoryId:** `config.ado.repoId`
- **sourceRefName:** `refs/heads/<main>`
- **targetRefName:** `refs/heads/<production>`
- **title:** `Release v{next_version}`
- **description:** (use real newlines, not literal \n)
  ```
  ## Release v{next_version}

  Automated release from <main> to <production>.

  ### Changes
  <commit summary from Step 4>
  ```

Set autocomplete using `mcp__ado__repo_update_pull_request`:
- **autoComplete:** true
- **deleteSourceBranch:** false (main branch must NOT be deleted)
- **mergeStrategy:** Squash or as configured

Poll PR status (up to 60s):
- Status 3 (Completed) -> proceed to Step 8
- Status 1 (Active, policies pending) -> inform user: "PR created but policies are pending. Tag will be created after PR completes. Monitor at: `https://dev.azure.com/<org>/<project>/_git/<repo>/pullrequest/<id>`"
  - Ask: "Wait for PR completion, or create tag now on the main branch?"
  - If wait: poll every 10s up to 5 minutes
  - If tag now: proceed to Step 8 (tag on main instead of production)

### Step 8: Create and push the tag

```bash
git tag v{next_version}
git push origin v{next_version}
```

**If method was `push`:** tag is on the merge commit on production.
**If method was `pr` and PR completed:** checkout production, pull, then tag.
**If method was `pr` and PR still pending:** tag on main branch HEAD (the version commit).

### Step 9: Return to main branch

```bash
git checkout <main>
git pull origin <main>
```

### Step 10: Report

Present a structured summary:

```
## Release v{next_version} Complete

- Version: {previous} -> {next_version}
- Tag: v{next_version} created and pushed
- Method: {pr|push}
- Changelog: {generated / skipped}
```

**If ADO configured:**
```
- Pipeline: https://dev.azure.com/<org>/<project>/_build
```

**If method was `pr`:**
```
- PR: #<id> — {status}
```

**Suggest next steps:**
- If changelog was generated: "Review the changelog files if needed."
- If method was `pr` and pending: "Monitor the PR for completion."
- "CD pipelines should trigger automatically from the tag."
