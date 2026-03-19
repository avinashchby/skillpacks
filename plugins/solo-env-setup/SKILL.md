---
name: solo-env-setup
description: >
  Detects the project type from manifest files, checks that all required tools and
  language runtime versions are installed and meet the project's stated requirements,
  reports missing or mismatched versions with specific install/fix commands, and
  optionally bootstraps the full development environment in one pass. Triggers on
  phrases like "setup env", "check setup", "bootstrap project", "dev environment",
  "is my environment ready", "check my tooling", "onboard to this project", or
  "what do I need to run this". Covers Node, Python, Rust, Go, Docker, and common
  CLI tools.
version: 1.0.0
---

# Solo Env Setup — Development Environment Checker & Bootstrapper

## When to Use

Activate this skill when the user says any of the following:

- "setup env" / "set up my environment"
- "check setup" / "check my setup"
- "bootstrap project" / "bootstrap my environment"
- "dev environment" / "development environment"
- "is my environment ready"
- "check my tooling"
- "onboard to this project"
- "what do I need to run this"
- "install dependencies"
- "first-time setup"

Also activate when the user clones a new repository and asks "how do I get this
running" or "what's the setup process".

Do NOT activate for deployment environment checks, production server audits, or
Docker container inspection unless the user explicitly asks about their local
development environment.

---

## Instructions

### Step 1 — Identify the project type(s)

Scan the repository root for manifest and config files:

```bash
ls -1 \
  package.json pyproject.toml requirements.txt Pipfile \
  Cargo.toml go.mod \
  Dockerfile docker-compose.yml docker-compose.yaml \
  .nvmrc .node-version .python-version .tool-versions \
  Makefile justfile \
  .env.example .env.template \
  2>/dev/null
```

Build a list of detected project types:

| File found             | Project type          |
|------------------------|-----------------------|
| `package.json`         | Node.js               |
| `pyproject.toml`       | Python                |
| `requirements.txt`     | Python                |
| `Cargo.toml`           | Rust                  |
| `go.mod`               | Go                    |
| `Dockerfile`           | Docker required       |
| `docker-compose.yml`   | Docker Compose required |

If no recognizable manifest is found, say: "No project manifest detected in the
current directory. Are you in the right project root?"

### Step 2 — Read version requirements from project files

**Node — read required version:**
```bash
# From .nvmrc
cat .nvmrc 2>/dev/null

# From .node-version
cat .node-version 2>/dev/null

# From package.json engines field
node -e "const p=require('./package.json'); console.log(p.engines?.node || 'not specified')" 2>/dev/null
```

**Python — read required version:**
```bash
# From .python-version
cat .python-version 2>/dev/null

# From pyproject.toml
grep -E 'python_requires|requires-python' pyproject.toml 2>/dev/null
```

**Rust — read required version:**
```bash
# From rust-toolchain.toml or rust-toolchain
cat rust-toolchain.toml 2>/dev/null || cat rust-toolchain 2>/dev/null
```

**Go — read required version:**
```bash
grep '^go ' go.mod 2>/dev/null
```

**asdf / mise — read all tool versions:**
```bash
cat .tool-versions 2>/dev/null
```

### Step 3 — Check installed tool versions

Run each check, capture both the version string and whether the requirement is met.

**Core runtime checks:**

```bash
# Node
node --version 2>/dev/null || echo "NOT INSTALLED"
npm --version 2>/dev/null || echo "NOT INSTALLED"

# Python
python3 --version 2>/dev/null || echo "NOT INSTALLED"
pip3 --version 2>/dev/null || echo "NOT INSTALLED"

# Rust
rustc --version 2>/dev/null || echo "NOT INSTALLED"
cargo --version 2>/dev/null || echo "NOT INSTALLED"

# Go
go version 2>/dev/null || echo "NOT INSTALLED"
```

**Package manager checks (Node):**
```bash
yarn --version 2>/dev/null || echo "NOT INSTALLED"
pnpm --version 2>/dev/null || echo "NOT INSTALLED"
```

**Python environment managers:**
```bash
uv --version 2>/dev/null || echo "NOT INSTALLED"
pyenv --version 2>/dev/null || echo "NOT INSTALLED"
```

**Version manager checks:**
```bash
nvm --version 2>/dev/null || echo "NOT INSTALLED"
asdf --version 2>/dev/null || echo "NOT INSTALLED"
mise --version 2>/dev/null || echo "NOT INSTALLED"
```

**Docker:**
```bash
docker --version 2>/dev/null || echo "NOT INSTALLED"
docker compose version 2>/dev/null || echo "NOT INSTALLED"
# Verify Docker daemon is running
docker info > /dev/null 2>&1 && echo "daemon: running" || echo "daemon: NOT RUNNING"
```

**Git:**
```bash
git --version 2>/dev/null || echo "NOT INSTALLED"
```

**GitHub CLI (check if repo uses gh):**
```bash
gh --version 2>/dev/null || echo "NOT INSTALLED"
```

**Optional — check for project-specific tools defined in Makefile or justfile:**
```bash
# Extract tool names from Makefile commands
grep -E '^[a-z]' Makefile 2>/dev/null | head -20

# Check for common tools referenced in scripts
grep -rE 'command -v|which|type ' scripts/ Makefile justfile 2>/dev/null | head -20
```

### Step 4 — Compare installed versions against requirements

For each required tool, compare the installed version against the required version:

1. Parse the required version string (e.g., `>=18.0.0`, `~20`, `3.11`)
2. Parse the installed version string
3. Apply semver comparison rules:
   - `>=X.Y.Z` — installed must be X.Y.Z or higher
   - `~X.Y` — installed must match major.minor exactly, any patch is fine
   - `^X` — installed major must match
   - Exact pin (`X.Y.Z`) — installed must match exactly

Flag the result as one of:
- **OK** — installed version meets requirement
- **OUTDATED** — installed but below the required version
- **TOO NEW** — installed major version exceeds pinned requirement (rare but warn)
- **MISSING** — not installed at all

### Step 5 — Check for .env setup

```bash
# Look for example env file
ls .env.example .env.template .env.sample 2>/dev/null

# Check if .env exists
ls .env 2>/dev/null
```

If `.env.example` (or equivalent) exists but `.env` does not:
- List the required variables from `.env.example`
- Flag any variables without default values as "REQUIRED — must be filled in"
- Suggest: `cp .env.example .env` and note which variables need real values

Never read or print the contents of `.env` — it may contain secrets.

### Step 6 — Generate a status report

Format the report as a clear table:

```
Environment Check — my-project
══════════════════════════════════════════════════════════
Tool            Required    Installed   Status
──────────────────────────────────────────────────────────
node            >=20.0.0    v22.3.0     ✓ OK
npm             >=10.0.0    10.7.0      ✓ OK
python3         >=3.11      3.10.12     ✗ OUTDATED
uv              any         0.1.40      ✓ OK
docker          any         26.1.4      ✓ OK
docker daemon   running     running     ✓ OK
git             any         2.43.0      ✓ OK
gh              any         NOT FOUND   ⚠ OPTIONAL — needed for PR workflows

.env            —           MISSING     ✗ Copy .env.example → .env and fill in:
                                          DATABASE_URL (required)
                                          SECRET_KEY   (required)
                                          PORT         (default: 8080)
══════════════════════════════════════════════════════════
```

### Step 7 — Provide specific fix commands for every failing check

For each MISSING or OUTDATED item, provide the exact install or upgrade command for
the user's OS (detect with `uname -s`):

**Node MISSING (macOS):**
```bash
# With nvm (recommended):
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install 20
nvm use 20

# Or with Homebrew:
brew install node@20
```

**Python OUTDATED (macOS):**
```bash
# With pyenv:
pyenv install 3.11.9
pyenv local 3.11.9

# Or with Homebrew:
brew install python@3.11
```

**uv MISSING (all platforms):**
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Rust MISSING:**
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
```

**Docker daemon not running (macOS):**
```bash
open -a Docker
# Wait for Docker to start, then re-run the check
```

**gh MISSING (macOS):**
```bash
brew install gh
gh auth login
```

Detect the OS:
```bash
uname -s    # Darwin = macOS, Linux = Linux
uname -m    # x86_64 or arm64
```

Adjust install commands accordingly (e.g., `apt-get` for Debian/Ubuntu Linux).

### Step 8 — Bootstrap the project (optional)

After all tools are confirmed present, ask: "Environment looks good. Should I run the
project bootstrap now? (install dependencies, run migrations, etc.)"

If yes, detect and run the standard bootstrap sequence:

**Node:**
```bash
npm install        # or yarn install / pnpm install based on lockfile
```

**Python (uv):**
```bash
uv sync
```

**Python (pip):**
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

**Rust:**
```bash
cargo build
```

**If a Makefile or justfile has a `setup`, `bootstrap`, or `install` target:**
```bash
make setup 2>/dev/null || just setup 2>/dev/null
```

**If docker-compose.yml exists and the user wants services running:**
```bash
docker compose up -d
```

Report the result of each step clearly. If any step fails, print the error output and
stop — do not proceed to the next step on failure.

### Step 9 — Final summary

Print a one-line summary of the environment state:

```
All required tools are installed and up to date.
Bootstrap complete — run `npm run dev` to start the development server.
```

Or, if issues remain:
```
3 issues found. Fix the items marked ✗ above, then re-run `check setup`.
```

---

## Examples

### Example 1 — Node project, Python too old, .env missing

**User says:** "check setup"

**Project files found:** `package.json`, `.nvmrc` (requires `20`), `.env.example`

**Tool check output:**
```
node     required: 20   installed: v18.19.0   ✗ OUTDATED
npm      required: any  installed: 10.2.3     ✓ OK
.env     —              MISSING               ✗ Required
```

**Claude provides:**
```
Issues found:

1. Node is outdated (v18 installed, v20 required):
   nvm install 20 && nvm use 20

2. .env file is missing. Copy the template and fill in required values:
   cp .env.example .env
   # Then set: DATABASE_URL, SECRET_KEY (no defaults provided)
```

---

### Example 2 — Rust project, fully clean environment

**User says:** "bootstrap project"

**Project files found:** `Cargo.toml`, `rust-toolchain.toml` (stable 1.77)

**Tool check output:**
```
rustc    required: 1.77   installed: 1.78.0   ✓ OK (minor patch ahead — fine)
cargo    required: any    installed: 1.78.0   ✓ OK
```

**Claude runs:**
```bash
cargo build
```

**Output:**
```
All checks passed. Running `cargo build`...
Compiling my_project v0.1.0
Finished dev [unoptimized + debuginfo] target(s) in 12.4s

Bootstrap complete. Run `cargo run` to start the application.
```

---

### Example 3 — No manifest found, wrong directory

**User says:** "setup env"

**No manifest files found.**

**Claude responds:**
> No project manifest found in `/Users/avinash/Projects`. This does not appear to be
> a project root. Common causes:
>
> - You may be in the wrong directory. Try `ls` to check what's here.
> - The project may use a non-standard layout. Let me know the project type and I
>   can check for specific files.

---

## Anti-patterns

1. **Printing raw version strings without comparing to requirements.** Telling the
   user "node v18.0.0 is installed" with no pass/fail verdict is useless. Always
   compare against the project's stated requirement and render a clear OK / FAIL.

2. **Recommending global package installs over version managers.** Prefer `nvm`,
   `pyenv`, `rustup`, or `asdf` over direct global installs. Version managers let
   the user maintain multiple project environments without conflicts.

3. **Running the bootstrap (npm install, cargo build, etc.) before fixing version
   mismatches.** An install run on the wrong runtime version can generate a corrupt
   lockfile or build artifacts. Always fix version issues first, then bootstrap.

4. **Reading or printing .env file contents.** The `.env` file may contain secrets,
   API keys, and credentials. Only read `.env.example` or `.env.template` to list
   required variable names. Never cat or display the actual `.env`.

5. **Assuming the OS without checking.** Install commands differ between macOS, Linux
   (Debian vs. Alpine vs. Arch), and Windows. Always run `uname -s` and `uname -m`
   before generating install instructions, and tailor the output to the detected
   platform.
