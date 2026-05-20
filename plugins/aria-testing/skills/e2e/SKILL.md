---
name: e2e
description: Plan and autonomously execute end-to-end (E2E) test plans for a feature branch using a browser-automation MCP (Playwright or equivalent). Use this skill whenever the user wants to E2E-test a feature branch, validate a feature works end-to-end before review/merge, generate a runnable E2E plan from a diff, autonomously drive a browser to verify changes with evidence (screenshots, console checks, optional DB integrity), or says "e2e", "e2e plan", "run e2e", "validate end to end before PR". Do NOT use for writing automated test code (unit / integration / E2E source files), running pre-existing automated test suites, or generating manual QA checklists (use the testplan skill for that).
---

# aria-testing:e2e — Plan & autonomous E2E execution

Generates an E2E test plan from the diff of the current branch, then executes it **rigorously** via a browser-automation MCP (Playwright or equivalent), capturing evidence (screenshots, console state, optional DB checks) for every item.

**Announce at start:** "Using aria-testing:e2e to {plan|run|report} the E2E plan."

**Usage:**
- `aria-testing:e2e plan` — generate/update the plan for the current branch (user must validate before run)
- `aria-testing:e2e run` — execute the plan, capture evidence, update statuses
- `aria-testing:e2e run T-03` — execute only item T-03 (debug / re-run after fix)
- `aria-testing:e2e report` — pass/fail/blocked summary with evidence links

---

## NON-NEGOTIABLE rules (`run` mode)

These rules are the reason this skill exists. If you bypass them, the skill loses its value.

1. **No item marked `[x]` without a verifiable evidence file.** After each browser screenshot, you MUST call `Read` on the file to confirm it exists and is non-empty. No verification = no pass.
2. **Missing data ≠ skip.** If a precondition is not satisfied, execute the `Seed` step (UI, API, or DB command per the plan). If no seed is provided or seed fails, mark the item `[!]` (blocked) and **continue the rest of the plan**. Never mark `[~]` (skipped) just because data is missing.
3. **Failure ≠ abort.** Red item → mark `[x]` FAIL (see status table), screenshot the error, note the cause (JS console, backend log, UI message), **continue**. The full plan must be executed end-to-end unless the user explicitly says otherwise.
4. **Invisible errors are failures.** On every page transition, inspect the browser MCP's console-messages tool — any red JS error flips the item to FAIL.
5. **No browser MCP available → STOP.** If no browser-automation tool is loaded, load it via `ToolSearch` first. If none exists in the environment, abort and tell the user.

### Status notation (ASCII, used in plan + report)

| Marker | Meaning | Where it appears |
|---|---|---|
| `[ ]` | pending | plan file |
| `[x]` | passed (with evidence) | plan file |
| `[F]` | failed (with evidence + notes) | plan file |
| `[!]` | blocked (seed impossible, dep missing) | plan file |
| `PASS` / `FAIL` / `BLOCKED` / `PENDING` | report table column | report output |

Do not use emoji or unicode marks (`✗`, `✅`, `⚠️`) anywhere — keep the plan grep-friendly across editors.

---

## Workflow

Follow these steps in order. Do NOT skip steps.

### Step 0: Load or set up config

Look for `.aria-testing/e2e.json` at the project root.

**If the file exists**, load it. Schema (all top-level keys optional except `app`):

```json
{
  "branches": { "main": "main" },
  "app": {
    "frontendUrl": "https://localhost:5173",
    "backendUrl": "https://localhost:5174",
    "portDiscovery": {
      "frontend": { "mode": "json", "file": "path/to/launchSettings.json", "jsonPath": "$.profiles.web.applicationUrl" },
      "backend":  { "mode": "command", "command": "node scripts/print-port.js api", "parse": "first-url" }
    },
    "healthCheck": { "frontend": "/", "backend": "/health" }
  },
  "auth": {
    "method": "none",
    "user": "qa@example.com",
    "password": "env:E2E_PASSWORD",
    "organizationPicker": "Acme QA",
    "loginUrl": null,
    "customStepsRef": null
  },
  "db": {
    "enabled": false,
    "command": "docker exec my_pg psql -U postgres -d mydb -c"
  },
  "analyze": {
    "fileGlobs": ["**/*.ts", "**/*.tsx", "**/*.js", "**/*.cs", "**/*.razor", "**/*.vue", "**/*.py"],
    "exclude": ["**/*.test.*", "**/*.spec.*", "**/node_modules/**", "**/dist/**", "**/migrations/**"]
  },
  "output": {
    "plansDir": "tests/e2e-plans",
    "evidenceDir": "tests/e2e-evidence"
  },
  "browserMcp": {
    "toolPrefix": "auto"
  }
}
```

Resolved defaults when the field is missing:
- `branches.main`: auto-detect via `git symbolic-ref refs/remotes/origin/HEAD`, fall back to `"main"`.
- `app.frontendUrl` / `app.backendUrl`: REQUIRED if `portDiscovery` is not set — ask interactively. `backendUrl` may be left empty if the app has no separate backend.
- `app.portDiscovery.{frontend|backend}.mode`:
  - `"json"` — read `file`, parse JSON, evaluate `jsonPath` (e.g. `$.profiles.api.applicationUrl`). If the value contains multiple URLs separated by `;`, take the first `https://` (else first).
  - `"command"` — run `command` via shell and parse stdout. `parse: "first-url"` extracts the first URL; `parse: "trim"` uses raw stdout trimmed. Useful for `npm run print-port`, `dotenv-cli`, etc.
- `app.healthCheck.frontend`: `"/"`. `app.healthCheck.backend`: `"/"` (treat 2xx or 3xx as up).
- `auth.method`: `"none"`. Supported values:
  - `"none"` — app is open or already authenticated, skip login step. All other `auth.*` fields are ignored if set; do not warn.
  - `"basic-form"` — same-origin login form on the app itself. Navigate to `{frontend_url}{loginUrl or "/"}`. Fill the first text/email input with `user`, the first password input with `password`, then submit. Requires `user` + `password`. `organizationPicker` is ignored.
  - `"oidc-redirect"` — the app redirects to an external identity provider (Auth0, Okta, AWS Cognito, Azure AD, Keycloak, Google, etc.) for sign-in. Requires `user` + `password`. Steps: navigate to `{frontend_url}` → follow the redirect to the IdP page → fill identifier + password on the IdP form → submit. If `organizationPicker` is non-empty AND a picker/tenant-select page appears (common on Auth0 organizations and Okta-with-orgs setups), select the entry whose label equals `organizationPicker`. If `organizationPicker` is empty/null and a picker appears anyway, FAIL with "auth.organizationPicker is empty but the IdP displayed a picker".
  - `"custom"` — login steps are described in the plan's `Setup global > Login steps` section (mandatory for this method), executed by the agent. `auth.customStepsRef` may point to a file path holding the steps if they're long enough to keep out of the plan. Use this method for SAML, magic links, passkeys, or any flow not covered above.
- `auth.password` supports `"env:VAR_NAME"` to read from `$VAR_NAME` at runtime (never persist real secrets to the config).
- `auth.organizationPicker` is read **only** when `auth.method` is `"oidc-redirect"` AND the IdP exposes an organization/tenant picker. Ignored otherwise.
- `db.enabled`: `false`. When `true`, `db.command` is the prefix that accepts a SQL string as its next argument. When `false`, any `DB check` field encountered in a plan item is skipped at run time with a `[!]` blocked status and a "DB checks disabled in config" note (do not silently pass).
- `analyze.fileGlobs`: auto-detect based on what's present in the repo (e.g. `package.json` → JS/TS globs; `*.csproj` → C# globs; `pyproject.toml` → Python; `go.mod` → Go; `Cargo.toml` → Rust). In a polyglot monorepo, union the detected sets.
- `output.plansDir`: `"tests/e2e-plans"`. `output.evidenceDir`: `"tests/e2e-evidence"`.
- `browserMcp.toolPrefix`: `"auto"` — at run time, locate the loaded browser MCP (see Workflow `run` Step 1).

**If the file does NOT exist**, run interactive setup. Ask questions **one at a time**, then write the config and continue:

1. **Pre-fill** from sibling `.aria-testing/*.json` files (reuse `branches.main` if present).
2. **Questions** (skip a question if the value is already known from pre-fill):
   - "Main branch name?" — default: detected from git, else `main`.
   - "Frontend URL for local dev?" (must be reachable when you run `run`)
   - "Backend URL for local dev? (leave empty if the app has no separate backend or it does not need health-checking)"
   - "Auto-discover ports from a config file or command? (n / json-file / command)" —
     - `json-file`: ask for the file path + JSON path expression
     - `command`: ask for the shell command + parse mode (`first-url` / `trim`)
     - `n`: skip discovery, keep the static URLs above
   - "Auth method? (none / basic-form / oidc-redirect / custom)" — `oidc-redirect` covers any flow where the app redirects to an external identity provider (Auth0, Okta, Cognito, Azure AD, Keycloak, Google, etc.).
     - If `none`: no further question. (Other `auth.*` fields will be ignored even if set later.)
     - If `basic-form` or `oidc-redirect`: "Test user identifier?" then "Password — provide a literal string, or `env:VAR_NAME` to read from environment? (recommended: env)" then (basic-form only) "Login URL path? (default `/`)"
     - If `oidc-redirect`: also "Does your IdP show an organization/tenant picker after sign-in? If yes, the exact label to select; leave empty if no picker."
     - If `custom`: tell the user they will describe the login steps inside the generated plan's `Setup global > Login steps` section.
   - "Enable DB integrity checks? (y/n)" — if y, ask for the DB command prefix.
   - "File extensions to analyze in diffs?" — default: detected from repo (show the detected list and ask y/n to accept).
   - "Output directories?" — defaults: `tests/e2e-plans`, `tests/e2e-evidence`.
3. Write `.aria-testing/e2e.json`.
4. Continue with the rest of the workflow.

Store resolved values in variables for the rest of the run:
- Branches: `main_branch`.
- App: `frontend_url`, `backend_url`, `port_discovery` (object with sub-config or null), `health_frontend`, `health_backend`.
- Auth: `auth_method`, `auth_user`, `auth_password`, `auth_org_picker`, `auth_login_url`, `auth_custom_ref`.
- DB: `db_enabled`, `db_cmd`.
- Analyze: `file_globs`, `exclude_globs`.
- Output: `plans_dir`, `evidence_dir`.
- MCP: `mcp_prefix` (resolved in Workflow `run` Step 1; absent during `plan`).

---

### Step 1: Dispatch on subcommand

- `plan` → go to **Workflow `plan`**.
- `run` (optionally followed by `T-XX`) → go to **Workflow `run`**.
- `report` → go to **Workflow `report`**.
- Anything else → show usage and STOP.

---

## Workflow `plan`

### 1. Identify branch and scope

```bash
git rev-parse --abbrev-ref HEAD                              # current branch
git diff --name-status {main_branch}...HEAD                  # changed files
git log --oneline --no-merges {main_branch}..HEAD            # commits in diff
```

`branch_slug` = current branch name with `/` replaced by `-`. E.g. `feature/payments` → `feature-payments`.

If `{main_branch}..HEAD` is empty → **"No change vs {main_branch}, nothing to plan."** STOP.

### 2. Read the existing plan if it exists

If `{plans_dir}/{branch_slug}.md` already exists:
- Read it in full.
- Keep every existing item, its status, evidence, and notes.
- **Never overwrite existing statuses or notes.**
- When deriving new scenarios in step 3, **dedupe** against existing items: skip a candidate if an existing item has the same `URL` AND a fuzzy-matching title (ignore case + filler words). Existing IDs always win — never renumber.

### 3. Analyze the diff to derive scenarios

Read the changed files matching `file_globs` (minus `exclude_globs`). For each change, identify any of the following surfaces (only when applicable to this codebase — do not invent surfaces the project does not have):

- **HTTP endpoints / API routes** added or modified (controllers, route handlers, OpenAPI handlers).
- **UI components / pages / views** modified (forms, grids, dialogs, route components).
- **Business logic** changed (services, use-cases, domain methods, validators).
- **Schema / migrations** (new tables, columns, constraints, indices).
- **Real-time mechanisms** (WebSocket, SSE, push channels, polling jobs).
- **Permission / authorization** changes (guards, role checks, middleware, policies).

For each change, derive 1+ E2E scenarios **from the user's point of view** (not the developer's). One verifiable action per scenario.

### 4. Plan format (parseable, strict schema)

```markdown
# E2E Plan — {branch}

**Branch:** {branch}
**Base:** {main_branch}
**Generated:** {YYYY-MM-DD}
**Frontend:** {frontend_url}
**Backend:** {backend_url or "n/a"}

## Setup global

- **Test user:** {auth_user or "n/a (auth: none)"}
- **Organization picker:** {auth_org_picker or "n/a"}
- **Login method:** {auth_method}
- **Login steps:** _(REQUIRED if auth_method = "custom" — describe the exact steps a human would take, one per line)_
- **Pre-flight:** frontend reachable at `{frontend_url}{health_frontend}`, backend reachable at `{backend_url}{health_backend}` _(only if backend_url is set)_, DB available _(only if db_enabled)_.

## Scenarios

### T-01 — {short user-oriented title}

- **Type:** [CRUD] | [UI] | [REALTIME] | [AUTH] | [PERF] | [DATA]
- **Origin:** {triggering file or commit, e.g. `src/pages/Cart.tsx` (commit abc1234)}
- **URL:** {path/of/the/page}
- **Preconditions:** {precise list — e.g. "1 active user, ≥1 product in catalog, empty cart"}
- **Seed:** {how to create the precondition if missing. UI steps OR API call OR SQL via the configured db command. EXPLICIT — never "create manually"}
- **Steps:**
  1. {user action}
  2. {user action}
- **Expected:** {observable result — UI message, displayed value, redirect, etc.}
- **DB check:** _(only generate this field if `db_enabled` is true — otherwise omit it entirely)_ `{db_cmd} "SELECT ..."` → {expected result}
- **Evidence:** _(filled at run time)_
- **Status:** `[ ]`
- **Notes:** _(filled at run time on failure/blocked)_

### T-02 — ...
```

### 5. Write the plan and STOP

1. Create `{plans_dir}/` if needed.
2. Write the plan to `{plans_dir}/{branch_slug}.md`.
3. Show a summary: number of items per `Type`, files analyzed, count of items deduped vs existing.
4. **STOP** — tell the user: *"Plan saved to `{plans_dir}/{branch_slug}.md`. Review it, edit/add/remove scenarios, then run `aria-testing:e2e run` when ready."*

**Never auto-run `run` after `plan`.** The human must validate.

---

## Workflow `run`

### 1. Load the browser MCP

Check whether a browser MCP is already loaded. The signature: any tool whose name contains a navigate verb (`*navigate*`, `*navigate_page*`, `*browser_navigate*`).

If not loaded, search via `ToolSearch`:

```
ToolSearch(query="browser navigate playwright chrome-devtools")
```

From the first match whose name matches `*navigate*`:
- **Derive `mcp_prefix`** as everything before the navigate verb (e.g. `mcp__plugin_playwright_playwright__browser_navigate` → prefix `mcp__plugin_playwright_playwright__browser`; `mcp__chrome-devtools__navigate_page` → prefix `mcp__chrome-devtools`).
- **Identify the verb mapping** for this MCP (different MCPs use different suffixes — `_navigate` vs `_navigate_page`, `_take_screenshot` vs `_screenshot`, `_console_messages` vs `_list_console_messages`). Record the actual tool names you'll call rather than blindly appending suffixes.

Load the rest of the family in one `ToolSearch` call referencing the actual matched names.

If no browser MCP exists in the environment → **"No browser-automation MCP available. Install Playwright MCP (or equivalent — chrome-devtools, puppeteer-mcp, etc.) before running."** STOP.

### 2. Pre-flight

- Read `{plans_dir}/{branch_slug}.md`. If missing → **"No plan. Run `aria-testing:e2e plan` first."** STOP.
- Resolve actual ports if `port_discovery` is configured:
  - `mode: "json"` — Read the configured file with `Read`, parse JSON, evaluate `jsonPath`. If the file doesn't exist → STOP with the missing path.
  - `mode: "command"` — Run the configured command via `Bash`. Apply `parse` mode. If the command fails → STOP with stderr.
  - Update `frontend_url` / `backend_url` in memory and persist the new URLs in the plan header (`Frontend:` / `Backend:` lines).
- Health-check the frontend at `{frontend_url}{health_frontend}` → expect 2xx/3xx. Use whatever HTTP probe is available in the environment: `curl -k -s -o NUL -w "%{http_code}" {url}` on Bash, `Invoke-WebRequest -SkipCertificateCheck -Uri {url}` on PowerShell, or `WebFetch` if you have it. If KO → **"Frontend not reachable at {frontend_url}. Start it and retry."** STOP.
- Health-check the backend at `{backend_url}{health_backend}` if `backend_url` is set. STOP on failure.
- If `db_enabled`, run a trivial query (`{db_cmd} "SELECT 1"`) to confirm the DB command works. STOP on failure.
- Create `{evidence_dir}/{branch_slug}/` if needed.
- Initialize a log file: `{evidence_dir}/{branch_slug}/run-{timestamp}.log`.
- **Track whether you started the app yourself** during pre-flight (e.g. via `Bash run_in_background` because the health check failed and the user told you to start it). If yes, remember to stop it during cleanup.

### 3. Login (once per session)

Execute the login flow based on `auth_method`:

- **`none`** — skip. Other `auth.*` fields are ignored silently if set.
- **`basic-form`** — navigate to `{frontend_url}{auth_login_url or "/"}`. Resolve `env:` in `auth_password` if needed. Fill the first visible text/email input with `auth_user` and the first visible password input with `auth_password`, then submit. `auth_org_picker` is ignored.
- **`oidc-redirect`** — navigate to `{frontend_url}`. Expect a redirect to an external IdP login page. Fill identifier + password on the IdP form and submit. If `auth_org_picker` is non-empty, on the subsequent picker/tenant-select page click the entry whose visible label equals `auth_org_picker`. If `auth_org_picker` is empty/null and a picker appears anyway, FAIL with "auth.organizationPicker is empty but the IdP displayed a picker — set it in .aria-testing/e2e.json".
- **`custom`** — execute the steps described in the plan's `Setup global > Login steps`. If those steps are missing → STOP with "auth.method=custom requires Setup global > Login steps in the plan".

After login, capture `{evidence_dir}/{branch_slug}/00-login.png` and verify the file exists via `Read`.

### 4. Execution loop

**For each item in plan order (or only the one provided as argument if `run T-XX`):**

If status is already `[x]`, `[F]`, or `[!]` (and no forced re-run) → skip.

Otherwise:

1. Mark in-progress internally (not in the file).
2. **Preconditions:** verify they hold. If not, execute `Seed`. If seed fails → mark `[!]`, log reason, **move to next item**.
3. **Navigate** to the item's URL.
4. **Execute steps** one by one via the browser MCP.
5. **Verify Expected** by matching DOM content via snapshot or text assertion.
6. **DB check** if present in the plan: if `db_enabled` is true, run it via `db_cmd` and compare. If `db_enabled` is false but a `DB check` is present, mark the item `[!]` with note "DB check skipped: db.enabled is false in .aria-testing/e2e.json" and continue (do NOT mark as passed).
7. **Console:** call the MCP's console-messages tool, filter for `level: error`. Any error → item becomes `[F]`.
8. **Final screenshot:** `{evidence_dir}/{branch_slug}/T-{id}.png`. Verify with `Read`.
9. **Update the plan**: replace `Status: [ ]` with the final status, fill `Evidence` with the path, fill `Notes` on failure/blocked. Write the file fully on each update (partial state must survive a crash).

**Special case REALTIME (multi-tab):**

Some browser MCP contexts **do not share cookies between tabs**: opening a new tab can re-trigger login. For REALTIME items:

1. Open tab 2 at the relevant URL via the MCP's tabs-new tool.
2. If the new tab lands on the login flow, replay the login (same `auth_method` path) on tab 2.
3. Snapshot the "before" state on tab 2, then switch back to tab 0 via the MCP's tabs-select tool.
4. Trigger the action on tab 0.
5. Switch to tab 1, wait 3–5 seconds, snapshot again.
6. Compare DOM before/after — prefer the MCP's `evaluate`-style tool to read exact values rather than relying on the screenshot alone.

### 5. Cleanup

- Close the browser via the MCP's close tool.
- **Stop any servers you started yourself** during pre-flight. If the user had the app running before the session, leave it running.
- Append a summary line to the run log.
- Display the aggregated result (see `report`).

### 6. Post-run integrity (optional, only when `db_enabled`)

If the config has any `db.postRunQueries` (an array of `{ "query": "...", "expected": "..." }` — optional schema extension), run them after the loop. Add a `## Post-run integrity` section at the bottom of the plan with one `[x]` / `[F]` line per query and its result. If the array is empty or absent, skip this section entirely.

---

## Workflow `report`

Read `{plans_dir}/{branch_slug}.md` and produce a table:

```
| ID    | Type     | Title                          | Status  | Evidence                                  |
|-------|----------|--------------------------------|---------|-------------------------------------------|
| T-01  | [CRUD]   | Create item                    | PASS    | tests/e2e-evidence/{slug}/T-01.png        |
| T-02  | [UI]     | Sidebar collapse               | FAIL    | tests/e2e-evidence/{slug}/T-02.png — console JS error |
| T-03  | [DATA]   | No orphans                     | BLOCKED | seed SQL refused (missing FK)             |
```

Status mapping: `[x]` → `PASS`, `[F]` → `FAIL`, `[!]` → `BLOCKED`, `[ ]` → `PENDING`.

Followed by:
- Counters: `X PASS, Y FAIL, Z BLOCKED, W PENDING`.
- List of FAIL items with notes (bug summary).
- List of BLOCKED items (seed to fix or human input required).
- Suggestion: *"Fix the failures? Use the project's `fix` flow, or proceed to review."*

---

## Conventions

- **Stable IDs `T-XX`**: never renumber on re-plan. Add new IDs (`T-08`, `T-09`...) without reordering.
- **HTTPS-friendly**: if URLs are `https://localhost`, ignore self-signed cert warnings during health checks (use `-k` on curl, `-SkipCertificateCheck` on PowerShell).
- **Allowed roots**: screenshot paths MUST be **relative to the repo root** (e.g. `tests/e2e-evidence/{slug}/T-XX.png`). Most browser MCPs reject absolute paths or `..` traversal. If you see a "file access denied: outside allowed roots" error, drop any leading `C:\` or absolute prefix.
- **No hot reload assumption**: if the source code changes mid-run, ask the user to rebuild/restart before continuing.
- **Output language**: identifiers (`T-XX`, filenames, status markers, type labels) stay ASCII. For free-text fields (titles, notes, messages to the user), match the language of the existing plan if one exists; otherwise default to English. Do not auto-detect from repo docs — keep it predictable.
- **No secrets in config**: passwords go through `env:VAR_NAME`, never as plain strings.
- **Token economy**: do not load browser MCP tools in `plan` mode.
- **No emoji in plan/report**: keep ASCII markers (`[x]`, `[F]`, `[!]`, `[ ]`, `PASS`, `FAIL`, `BLOCKED`, `PENDING`) for grep-friendliness across editors.

## Integration

- `aria-testing:e2e` is standalone. Typical flow: implement a feature → `aria-testing:e2e plan` → review the plan → `aria-testing:e2e run` → `aria-testing:e2e report` → fix or ship.
- Suggest `aria-testing:e2e` after a non-trivial feature implementation, before requesting code review, or when the user wants E2E validation distinct from manual QA.
- Sister skill `aria-testing:testplan` covers **manual** QA checklists for releases — use that instead when the human will do the testing.
