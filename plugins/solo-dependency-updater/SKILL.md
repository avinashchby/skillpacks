---
name: solo-dependency-updater
description: >
  Audits a project's dependencies for outdated packages, fetches changelogs or release
  notes for each outdated package, identifies breaking changes, and performs the update
  with explicit warnings before touching any major-version bumps. Triggers on phrases
  like "update deps", "outdated packages", "upgrade dependencies", "check for updates",
  "bump packages", or "are my dependencies up to date". Supports npm/yarn/pnpm (Node),
  pip/uv (Python), and Cargo (Rust). Never silently upgrades across major versions.
version: 1.0.0
---

# Solo Dependency Updater — Safe Dependency Upgrades

## When to Use

Activate this skill when the user says any of the following:

- "update deps" / "update dependencies"
- "outdated packages" / "check for outdated packages"
- "upgrade dependencies" / "upgrade packages"
- "bump packages" / "bump deps"
- "are my dependencies up to date"
- "check for updates"
- "update my lockfile"

Also activate when the user says "I need to patch a security vulnerability" or
"update X to the latest version" for a specific package.

Do NOT activate for `git` operations, deployment tasks, or Docker image updates
unless those are explicitly combined with a dependency update request.

---

## Instructions

### Step 1 — Detect the project type and package manager

Check for manifest files in order of precedence:

```bash
# Node.js
ls package.json yarn.lock pnpm-lock.yaml package-lock.json 2>/dev/null

# Python
ls pyproject.toml requirements.txt uv.lock Pipfile 2>/dev/null

# Rust
ls Cargo.toml Cargo.lock 2>/dev/null
```

Determine the active package manager:
- `yarn.lock` present → Yarn
- `pnpm-lock.yaml` present → pnpm
- `package-lock.json` present → npm
- `uv.lock` present → uv (preferred Python manager)
- `Pipfile` present → pipenv
- Only `pyproject.toml` / `requirements.txt` → pip
- `Cargo.toml` present → Cargo

If multiple ecosystems exist (e.g., a monorepo with Node + Python), handle each
separately and ask the user which to process first.

### Step 2 — Check for outdated packages

**Node (npm):**
```bash
npm outdated --json 2>/dev/null
```

**Node (yarn):**
```bash
yarn outdated --json 2>/dev/null
```

**Node (pnpm):**
```bash
pnpm outdated 2>/dev/null
```

**Python (uv):**
```bash
uv pip list --outdated 2>/dev/null
```

**Python (pip):**
```bash
pip list --outdated --format=json 2>/dev/null
```

**Rust (Cargo):**
```bash
cargo outdated 2>/dev/null
# If cargo-outdated is not installed:
# cargo install cargo-outdated
```

### Step 3 — Categorize updates by semver bump type

Parse the output and build a table:

| Package | Current | Latest | Bump | Type |
|---------|---------|--------|------|------|
| express | 4.18.0  | 4.21.1 | PATCH | direct |
| react   | 17.0.2  | 18.3.1 | MAJOR | direct |
| lodash  | 4.17.20 | 4.17.21 | PATCH | transitive |

Classify each package:
- **PATCH** (e.g., 4.18.0 → 4.18.3): Bug fixes, safe to update
- **MINOR** (e.g., 4.18.0 → 4.21.0): New features, backward-compatible, generally safe
- **MAJOR** (e.g., 4.18.0 → 5.0.0): May contain breaking changes — require explicit warning

### Step 4 — Fetch changelogs for MAJOR updates

For each package with a MAJOR version bump, attempt to retrieve the changelog:

**npm packages:**
```bash
npm view <package> repository.url
# Then fetch CHANGELOG.md from the GitHub repo between current and latest tag
```

Use the GitHub API when available:
```bash
# Get releases between current and latest version
gh api repos/<owner>/<repo>/releases --jq '[.[] | {tag: .tag_name, body: .body}]' 2>/dev/null | head -100
```

**PyPI packages:**
```bash
pip index versions <package> 2>/dev/null
# Fetch from PyPI JSON API
curl -s https://pypi.org/pypi/<package>/json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('info',{}).get('description','')[:2000])"
```

**Cargo crates:**
```bash
curl -s https://crates.io/api/v1/crates/<crate>/versions | python3 -c "import sys,json; d=json.load(sys.stdin); [print(v['num'], v.get('description','')) for v in d.get('versions',[])[:10]]"
```

If changelog retrieval fails, note it explicitly: "Could not fetch changelog for
`<package>`. Review manually at https://npmjs.com/package/<package> before updating."

### Step 5 — Identify breaking changes in changelogs

Scan the fetched changelog text for these signals:

- Lines containing: "BREAKING", "breaking change", "removed", "deprecated and removed",
  "no longer supported", "migration guide", "upgrade guide"
- Major API renames (e.g., "X has been renamed to Y")
- Removed CLI flags or configuration keys
- Changed default behavior

For each breaking change found, format it as a named warning block:

```
WARNING — react 17 → 18
  - Concurrent rendering is now the default. If you use ReactDOM.render() it
    must be replaced with ReactDOM.createRoot().
  - Removed: ReactDOM.hydrate() — use hydrateRoot() instead.
  - New JSX transform requires Babel ≥ 7.9 or TypeScript ≥ 4.1.
```

### Step 6 — Present update plan to user

Display a structured plan before touching any files:

```
Dependency Update Plan
══════════════════════

SAFE (PATCH/MINOR) — will update automatically:
  ✓ express         4.18.0 → 4.21.1
  ✓ axios           1.3.2  → 1.7.4
  ✓ typescript      5.3.2  → 5.4.5

MAJOR — requires your approval (breaking changes detected):
  ⚠ react           17.0.2 → 18.3.1  [breaking changes — see above]
  ⚠ webpack         4.46.1 → 5.94.0  [breaking changes — see above]

  Skipping majors? [yes / no / update specific package]
```

Wait for confirmation before proceeding.

### Step 7 — Perform the updates

**Safe (PATCH + MINOR) updates:**

Node (npm):
```bash
npx npm-check-updates -u --target minor
npm install
```

Node (yarn):
```bash
yarn upgrade --pattern "*" --latest 2>/dev/null || yarn upgrade
```

Node (pnpm):
```bash
pnpm update
```

Python (uv):
```bash
uv pip install --upgrade <package1> <package2> ...
```

Python (pip):
```bash
pip install --upgrade <package1> <package2> ...
```

Rust:
```bash
cargo update
```

**Major updates (only if user approved each one):**

Node:
```bash
npx npm-check-updates -u --filter "<package>"
npm install
```

Python (uv):
```bash
uv pip install "<package>==<new_version>"
```

Rust:
```bash
# Edit Cargo.toml manually, then:
cargo update -p <crate>
```

### Step 8 — Run the test suite after updating

```bash
# Node
npm test 2>/dev/null || yarn test 2>/dev/null || pnpm test 2>/dev/null

# Python
pytest 2>/dev/null || python -m pytest 2>/dev/null

# Rust
cargo test 2>/dev/null
```

If tests fail:
1. Report which tests failed and which package update is most likely responsible
2. Suggest reverting that specific package: `npm install <package>@<old_version>`
3. Do NOT automatically revert — ask the user to decide

### Step 9 — Summarize what changed

Print a concise summary:

```
Update complete.

Updated 8 packages (6 patch/minor, 2 major with your approval).
Lockfile updated: package-lock.json

Major updates applied:
  react 17.0.2 → 18.3.1   ← review ReactDOM.createRoot() migration
  webpack 4.46.1 → 5.94.0 ← review webpack.config.js for removed options

All tests passed. ✓

Next steps:
  - Review the breaking change migrations listed above
  - Run `git diff package-lock.json` to verify lockfile changes
  - Commit: git commit -m "chore(deps): upgrade to react 18 and webpack 5"
```

---

## Examples

### Example 1 — Node project, mix of safe and major updates

**User says:** "update deps"

**Detected:** `package.json` + `package-lock.json` → npm

**Outdated packages found:**
- `axios` 1.3.2 → 1.7.4 (MINOR)
- `jest` 28.1.0 → 29.7.0 (MAJOR)
- `dotenv` 16.0.3 → 16.4.5 (PATCH)

**Breaking changes found for jest 29:**
> `jest-environment-jsdom` is no longer bundled. Install it separately.
> `--experimental-vm-modules` flag renamed behavior.

**Claude presents:**
```
SAFE — updating automatically:
  axios   1.3.2  → 1.7.4
  dotenv  16.0.3 → 16.4.5

MAJOR — approval required:
  jest    28.1.0 → 29.7.0
  WARNING: jest-environment-jsdom must be installed separately after upgrade.
  Update require? [yes/no]
```

---

### Example 2 — Python project with uv, no major bumps

**User says:** "outdated packages"

**Detected:** `pyproject.toml` + `uv.lock` → uv

**Output:**
```
All outdated packages are PATCH or MINOR — safe to update.

  httpx        0.25.1 → 0.27.0
  pydantic     2.4.2  → 2.8.0
  pytest       7.4.0  → 8.3.2

Running: uv pip install --upgrade httpx pydantic pytest
Tests: pytest ... 47 passed in 3.21s
```

---

### Example 3 — User requests a specific package update

**User says:** "update lodash to the latest version"

**Claude runs:**
```bash
npm view lodash version          # latest: 4.17.21
npm view lodash@4.17.21 repository.url
```

Fetches changelog, finds no breaking changes (same major version).

```bash
npm install lodash@4.17.21
```

Reports: "Updated lodash 4.17.20 → 4.17.21 (PATCH). No breaking changes. Tests passed."

---

## Anti-patterns

1. **Silently upgrading across major versions.** Always surface breaking changes from
   the changelog and require explicit user approval for each major bump. A surprise
   breaking change in production is worse than staying on an old version.

2. **Updating everything at once without categorization.** Mixing patch bumps with
   major bumps in a single `npm install --latest` makes it impossible to bisect a
   test failure. Update patch/minor first, then handle majors one at a time.

3. **Skipping the test run.** Running the test suite after updating is mandatory. If
   you skip it, a broken dependency may go undetected until deployment.

4. **Updating transitive (indirect) dependencies manually.** Only directly manage
   packages listed in the project's own manifest. Let the package manager resolve
   transitive dependencies through the lockfile.

5. **Not committing after a successful update.** After a clean update + passing tests,
   remind the user to commit the manifest and lockfile changes together. A lockfile
   committed without the manifest (or vice versa) creates inconsistent state.
