---
name: solo-pr-creator
description: >
  Creates a GitHub pull request with an AI-generated title, structured description,
  and reviewer suggestions derived from the repository's CODEOWNERS file. Triggers
  on phrases like "create pr", "open pull request", "submit pr", "make a PR",
  "open a PR", or "push and create PR". Reads the full commit log and diff between
  the current branch and its base to generate a meaningful description — never
  produces a generic placeholder PR body.
version: 1.0.0
---

# Solo PR Creator — AI-Generated Pull Requests

## When to Use

Activate this skill when the user says any of the following:

- "create pr" / "create a pr" / "create pull request"
- "open pull request" / "open a pr"
- "submit pr" / "submit pull request"
- "make a pr" / "make a pull request"
- "push and create pr"
- "pr this branch"

Also activate when the user says "I'm done with this feature" or "ready for review" if
they are on a non-default branch with unpushed commits.

Do NOT activate for draft PR requests, issue creation, or branch management alone
unless combined with a PR creation phrase.

---

## Instructions

Follow these steps in order. Do not run `gh` commands until you have confirmed the
repository state and the user's intent.

### Step 1 — Confirm git and gh CLI are available

```bash
git --version
gh --version
```

If `gh` is not installed, stop and tell the user: "The GitHub CLI (`gh`) is required.
Install it with `brew install gh` (macOS) or see https://cli.github.com. Then run
`gh auth login` to authenticate."

### Step 2 — Identify the current branch and base branch

```bash
# Current branch
git branch --show-current

# Check if the branch has a remote tracking branch
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null

# Identify the repo's default branch (usually main or master)
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'
```

If the current branch IS the default branch, stop and ask: "You are on the default
branch. PRs are typically opened from a feature branch. Did you mean to switch
branches first?"

### Step 3 — Push the branch if not already pushed

```bash
git status --short
```

If there are uncommitted changes, warn: "There are uncommitted changes in your working
tree. Should I commit them first (recommend running the solo-git-workflow skill), or
proceed with just the already-committed changes?"

```bash
# Push branch to remote, setting upstream if needed
git push -u origin $(git branch --show-current)
```

If push fails due to diverged history, report the error verbatim and ask the user to
resolve the conflict before proceeding.

### Step 4 — Collect the commit range between branch and base

```bash
BASE=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
BRANCH=$(git branch --show-current)

# All commits unique to this branch
git log origin/$BASE..$BRANCH --oneline

# Full diff stat
git diff origin/$BASE...$BRANCH --stat

# Full diff (used to write description)
git diff origin/$BASE...$BRANCH
```

Count the commits. If there is only 1 commit, derive the PR title directly from it.
If there are multiple commits, synthesize a higher-level title that describes the
overall change.

### Step 5 — Generate the PR title

Rules:
- 60–70 characters maximum
- Sentence case (first word capitalized, rest lowercase except proper nouns)
- Describes the purpose, not the implementation mechanism
- Do NOT copy the commit subject verbatim if it is too low-level

Bad:  `fix(auth): reject expired JWT tokens before hitting the database`
Good: `Enforce token expiry validation for all user roles`

### Step 6 — Generate the PR description

Structure the body using this exact template:

```markdown
## What changed

<1–3 bullet points summarizing the key changes. Be specific — name the files,
functions, or modules affected.>

## Why

<1–2 sentences explaining the motivation. Reference the issue or ticket if known.
If no issue exists, explain the business or technical reason.>

## How to test

<Numbered steps a reviewer can follow to verify the change works. Be concrete.
Include the exact command to run tests if applicable.>

## Notes for reviewer

<Optional. List non-obvious decisions, known limitations, follow-up work deferred,
or areas where you want specific feedback. Omit this section if there is nothing
special to flag.>
```

Derive all content from the actual diff and commit messages. Never use placeholder
text like "TODO" or "see diff".

### Step 7 — Read CODEOWNERS and suggest reviewers

```bash
# Check common CODEOWNERS locations
cat .github/CODEOWNERS 2>/dev/null || cat CODEOWNERS 2>/dev/null || cat docs/CODEOWNERS 2>/dev/null
```

Parse the CODEOWNERS file:
- Find entries whose path patterns overlap with files changed in the diff
- Extract GitHub usernames (prefixed with `@`)
- Deduplicate and exclude the PR author themselves

```bash
# Get the authenticated GitHub user (the PR author)
gh api user --jq '.login'
```

If CODEOWNERS does not exist, check `git log` for frequent contributors to the
changed files:

```bash
git log --follow --format='%ae' -- <changed_file> | sort | uniq -c | sort -rn | head -5
```

Suggest up to 3 reviewers. Do not add reviewers automatically — present suggestions
and ask the user to confirm.

### Step 8 — Confirm all details with the user

Print a preview:

```
PR Title:    <title>
Base branch: <base>
Reviewers:   @alice, @bob (from CODEOWNERS — confirm to add)

Description preview:
─────────────────────
<full markdown body>
─────────────────────

Proceed? [yes / edit title / edit description / skip reviewers]
```

Wait for user confirmation before running `gh pr create`.

### Step 9 — Create the PR

```bash
gh pr create \
  --title "<title>" \
  --body "$(cat <<'EOF'
<description>
EOF
)" \
  --reviewer "<reviewer1>,<reviewer2>"
```

If the user skipped reviewers, omit the `--reviewer` flag entirely.

If `--draft` was requested by the user, add `--draft` to the command.

### Step 10 — Report the result

After successful creation, print:
- The PR URL returned by `gh pr create`
- A reminder: "Run `gh pr view --web` to open it in your browser."

---

## Examples

### Example 1 — Single-commit feature branch

**User says:** "create pr"

**Branch:** `feat/jwt-expiry-fix`
**Base:** `main`
**Commits:** 1 commit — `fix(auth): enforce token expiry check for admin roles`
**Changed files:** `src/auth/jwt.rs`, `tests/auth/jwt_test.rs`
**CODEOWNERS match:** `@security-team` owns `/src/auth/**`

**Claude generates:**

```
PR Title:    Enforce token expiry validation for all user roles
Base branch: main
Reviewers:   @security-team (from CODEOWNERS)

## What changed

- `src/auth/jwt.rs`: removed the early-return that allowed admin tokens to bypass
  the expiry guard in `validate_token()`
- `tests/auth/jwt_test.rs`: added regression test for expired admin tokens

## Why

Admin tokens were silently accepted after expiry because the role check ran before
the expiry check. This closes a privilege escalation window reported in issue #88.

## How to test

1. Run `cargo test auth` — all tests should pass
2. Manually issue an expired admin token and confirm the API returns 401

## Notes for reviewer

The fix is a single-line reorder. The test added covers the exact scenario from
the bug report.
```

---

### Example 2 — Multi-commit branch, no CODEOWNERS

**User says:** "submit pr"

**Branch:** `feat/user-uuid-migration`
**Commits:** 4 commits touching `src/api/`, `src/db/`, `migrations/`

**Claude synthesizes:**

```
PR Title:    Migrate user identity from integer IDs to UUIDs
Base branch: main
Reviewers:   (no CODEOWNERS found — suggesting @db-contributor based on git log)

## What changed

- `src/api/users.rs`: replaced `get_user_by_id(i32)` with `find_user(Uuid)`
- `src/db/schema.rs`: updated `users` table schema; `id` column is now UUID
- `migrations/004_uuid_users.sql`: adds migration script with backfill logic

## Why

Integer IDs are enumerable and leak user count. UUID-based identity is required
for the upcoming multi-tenant rollout (see RFC-12).

## How to test

1. Run `cargo test` — full test suite
2. Apply migration: `diesel migration run`
3. Verify existing users retain data: `SELECT count(*) FROM users` should be unchanged

## Notes for reviewer

The migration includes a backfill from the old integer sequence to UUID v4. There is
a brief window during migration where the old endpoint returns 404 — noted in the
migration runbook.
```

---

### Example 3 — User is on default branch

**User says:** "open a pr"

**Current branch:** `main`

**Claude responds:**
> You are currently on `main`, which is the default branch. Pull requests should be
> opened from a feature or fix branch. Would you like me to create a new branch from
> your current changes first?

---

## Anti-patterns

1. **Creating the PR without user confirmation.** Always show the full title and
   description preview and wait for explicit confirmation before running `gh pr create`.
   The user may want to adjust the title or remove a reviewer.

2. **Generic, placeholder descriptions.** Never write "See diff for details" or
   "Various bug fixes". Every section of the PR body must be derived from the actual
   diff and commit history.

3. **Adding reviewers automatically without checking CODEOWNERS.** Only suggest
   reviewers from CODEOWNERS or git history. Never guess reviewer usernames.
   Always let the user confirm before the `--reviewer` flag is included.

4. **Ignoring uncommitted changes.** If the working tree is dirty, flag it explicitly.
   Silently creating a PR that is missing changes leads to incomplete reviews.

5. **Pushing to the remote without warning.** Inform the user before running
   `git push`. Some users have branch protection rules or want to rebase before
   pushing.
