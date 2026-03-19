---
name: solo-git-workflow
description: >
  Analyzes staged git changes and writes a well-formed conventional commit message,
  optionally auto-tagging a semver version based on commit type. Triggers on phrases
  like "commit", "smart commit", "conventional commit", "auto commit", "commit my
  changes", or "write a commit message for this". Reads the actual diff to infer
  scope, type, and breaking changes rather than relying on vague user descriptions.
version: 1.0.0
---

# Solo Git Workflow — Smart Commits

## When to Use

Activate this skill when the user says any of the following:

- "commit" / "commit this" / "commit my changes"
- "smart commit"
- "conventional commit"
- "auto commit"
- "write a commit message"
- "stage and commit"
- "commit and tag"

Also activate when the user asks you to finalize work on a feature, fix, or refactor
and there are staged or unstaged changes present in the repository.

Do NOT activate for bare `git status` queries, branch management, or push operations
unless the user explicitly combines them with a commit request.

---

## Instructions

Follow these steps in order. Do not skip steps. Do not proceed to a later step if an
earlier step surfaces a blocking problem.

### Step 1 — Verify you are inside a git repository

```bash
git rev-parse --show-toplevel
```

If this fails, stop and tell the user: "This directory is not inside a git repository.
Run `git init` first or navigate to the correct project root."

### Step 2 — Inspect staged and unstaged changes

```bash
# Show what is already staged
git diff --cached --stat

# Show the full staged diff (used to write the message)
git diff --cached

# Show unstaged changes so the user knows what is NOT being committed
git diff --stat
```

If `git diff --cached` is empty (nothing staged), check whether the user intended to
stage everything first:

- If unstaged changes exist, ask: "Nothing is staged. Should I stage all changes with
  `git add -A` before committing, or do you want to stage specific files?"
- If both are empty, tell the user there is nothing to commit.

### Step 3 — Determine the conventional commit type

Read the staged diff carefully and pick **one** primary type:

| Type       | When to use                                                        |
|------------|--------------------------------------------------------------------|
| `feat`     | A new capability visible to end users or API consumers             |
| `fix`      | Corrects a defect or unintended behavior                           |
| `refactor` | Code restructuring with no behavior change                         |
| `perf`     | A change that measurably improves performance                      |
| `test`     | Adding or correcting tests only                                    |
| `docs`     | Documentation, comments, README only                               |
| `chore`    | Build scripts, CI config, dependency bumps, tooling config         |
| `style`    | Formatting, whitespace, lint-only changes (no logic)               |
| `ci`       | Changes to CI/CD pipeline definitions                              |
| `revert`   | Reverting a previous commit                                        |

Rules for type selection:
- If the diff touches both source and tests, the primary type is determined by the
  source change (`feat`, `fix`, etc.), not `test`.
- If the diff is purely lock-file or `package.json` version bumps, use `chore`.
- Never combine two types in one commit message. If the changes span multiple types,
  warn the user: "These changes span multiple concerns (feat + refactor). Consider
  splitting into separate commits for a cleaner history."

### Step 4 — Determine the scope

Scope is the module, package, or area of the codebase most affected. Derive it from:

1. The directory name closest to the changed files (e.g., `auth`, `api`, `ui`)
2. The package name in `package.json`, `Cargo.toml`, or `pyproject.toml` if changes
   are confined to one package
3. Omit scope entirely if changes are genuinely cross-cutting

Format: `type(scope): description` — scope is lowercase, no spaces.

### Step 5 — Detect breaking changes

Scan the diff for:

- Removed or renamed public functions, methods, routes, or CLI flags
- Changed function signatures (parameter type, order, or count)
- Removed fields from serialized structs / API response shapes
- Database schema column drops or renames

If any breaking change is found:
- Append `!` after the type/scope: `feat(api)!: ...`
- Add a `BREAKING CHANGE:` footer explaining exactly what broke and how to migrate

### Step 6 — Write the commit subject line

Rules:
- Imperative mood: "add", "fix", "remove" — NOT "added", "fixes", "removes"
- 72 characters maximum (including type and scope prefix)
- No period at the end
- Specific: describe WHAT changed, not that you "updated" or "modified" something

Bad:  `fix: update user service`
Good: `fix(auth): reject expired JWT tokens before hitting the database`

### Step 7 — Write the commit body (when needed)

Include a body when:
- The subject line cannot capture the full reasoning
- A non-obvious decision was made
- There is migration guidance for a breaking change

Body format:
- Blank line between subject and body
- Wrap at 72 characters
- Explain WHY, not just what (the diff already shows what)
- Reference issue numbers: `Closes #42`, `Fixes #17`

### Step 8 — Compose and show the full message to the user

Print the full commit message in a code block for the user to review before committing.

### Step 9 — Commit (with user confirmation)

```bash
git commit -m "$(cat <<'EOF'
<subject line>

<body if present>

<footers if present>
EOF
)"
```

Never use `--no-verify` unless the user explicitly asks to skip hooks.

### Step 10 — Semver auto-tag (optional, only if requested)

If the user asked to also tag the release:

```bash
# Get the latest existing semver tag
git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0"
```

Increment the version according to these rules:
- `BREAKING CHANGE` footer or `!` suffix → bump **MAJOR** (e.g., v1.2.3 → v2.0.0)
- `feat` → bump **MINOR** (e.g., v1.2.3 → v1.3.0)
- `fix`, `perf`, `refactor`, `chore` → bump **PATCH** (e.g., v1.2.3 → v1.2.4)
- `docs`, `style`, `test`, `ci` → no tag unless user insists

```bash
git tag -a v<new_version> -m "<subject line>"
```

Inform the user: "Tagged v<new_version>. Run `git push origin v<new_version>` to push
the tag to the remote."

---

## Examples

### Example 1 — Simple bug fix

**User says:** "smart commit"

**Staged diff summary:**
```
src/auth/jwt.rs  — removed a condition that skipped expiry validation for admin tokens
```

**Claude produces:**

```
fix(auth): enforce token expiry check for admin roles

Admin tokens bypassed the expiry validation introduced in v1.4.0 because
the role check short-circuited before reaching the expiry guard. All
tokens now go through the same validation path regardless of role.

Closes #88
```

**Then runs:**
```bash
git commit -m "$(cat <<'EOF'
fix(auth): enforce token expiry check for admin roles

Admin tokens bypassed the expiry validation introduced in v1.4.0 because
the role check short-circuited before reaching the expiry guard. All
tokens now go through the same validation path regardless of role.

Closes #88
EOF
)"
```

---

### Example 2 — New feature with breaking API change

**User says:** "conventional commit and tag this"

**Staged diff summary:**
```
src/api/users.rs  — renamed `get_user_by_id(id: i32)` to `find_user(id: Uuid)`
                    changed return type from `User` to `Result<User, NotFound>`
```

**Claude produces:**

```
feat(api)!: migrate user lookup to UUID-based identity

BREAKING CHANGE: `get_user_by_id(id: i32)` has been removed. Replace all
call sites with `find_user(id: Uuid)` which also returns a Result instead
of panicking on missing records. Integer user IDs are no longer accepted
by the public API.
```

**Auto-tag calculation:**
- Previous tag: `v1.3.1`
- Breaking change → bump MAJOR → `v2.0.0`

```bash
git tag -a v2.0.0 -m "feat(api)!: migrate user lookup to UUID-based identity"
```

---

### Example 3 — Nothing staged, user asks to commit everything

**User says:** "commit my changes"

**`git diff --cached` output:** (empty)
**`git diff --stat` output:** 3 files changed

**Claude responds:**
> Nothing is staged. I can see 3 modified files. Should I run `git add -A`
> to stage all of them, or would you like to specify which files to stage?

**User says:** "yes, stage all"

```bash
git add -A
```

Then continues from Step 3 with the newly staged diff.

---

## Anti-patterns

1. **Writing vague subject lines.** "fix: bug fix" or "chore: update stuff" conveys
   nothing. Always derive a specific description from the actual diff content.

2. **Staging unrelated changes together.** If the diff mixes a `feat` and a `fix` in
   unrelated modules, warn the user rather than silently combining them into one commit.
   One commit should represent one logical change.

3. **Using `--no-verify` by default.** Pre-commit hooks exist for a reason (linting,
   tests, secret scanning). Never bypass them unless the user explicitly says to skip
   hooks and understands the risk.

4. **Committing without showing the message first.** Always display the composed message
   in a code block and give the user a chance to edit it before running `git commit`.

5. **Tagging every commit.** Tags should mark release points, not every individual
   commit. Only create a tag when the user explicitly requests it or when the workflow
   is part of a release process.
