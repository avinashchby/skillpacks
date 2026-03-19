---
name: solo-release-notes
description: >
  Generates a structured release notes document by reading the git log between two
  semver tags (or between a tag and HEAD), grouping commits by conventional commit
  type, summarizing breaking changes prominently, and outputting markdown suitable
  for GitHub Releases, a CHANGELOG.md, or a blog post. Triggers on phrases like
  "release notes", "changelog", "what changed since", "generate release notes",
  "write changelog", "diff since last release", or "what's new in v<X>". Requires
  conventional commit history to produce accurate output; falls back gracefully when
  commits are non-conventional.
version: 1.0.0
---

# Solo Release Notes — Changelog Generator

## When to Use

Activate this skill when the user says any of the following:

- "release notes" / "generate release notes" / "write release notes"
- "changelog" / "write changelog" / "update changelog"
- "what changed since <tag>" / "what changed since the last release"
- "what's new in v<version>"
- "diff since last release"
- "summarize this release"

Also activate when the user is preparing a GitHub Release and asks for a description,
or when the user says "tag and release" in combination with a request for notes.

Do NOT activate for PR descriptions, commit messages, or per-file diff summaries
unless they are explicitly part of a release documentation workflow.

---

## Instructions

### Step 1 — Identify the tag range

**If the user specified two tags (e.g., "release notes from v1.2.0 to v1.3.0"):**
```bash
FROM_TAG="v1.2.0"
TO_TAG="v1.3.0"
```

**If the user specified one tag (e.g., "what changed since v1.2.0"):**
```bash
FROM_TAG="v1.2.0"
TO_TAG="HEAD"
```

**If the user specified no tags (e.g., "release notes" or "changelog"):**
```bash
# List the two most recent semver tags
git tag --sort=-version:refname | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+' | head -2
```

Use the most recent tag as `TO_TAG` and the second most recent as `FROM_TAG`.
If there is only one tag, use the initial commit as `FROM_TAG`:
```bash
git rev-list --max-parents=0 HEAD
```

If there are no tags at all, use the full git history from the initial commit to HEAD
and note in the output: "No tags found — showing full project history."

### Step 2 — Validate the tags exist

```bash
git rev-parse --verify "$FROM_TAG" 2>/dev/null
git rev-parse --verify "$TO_TAG" 2>/dev/null
```

If a tag does not exist, list available tags and ask the user to correct the input:
```bash
git tag --sort=-version:refname | head -20
```

### Step 3 — Collect the commit log

```bash
# Oneline log for the range (used to count and categorize)
git log "${FROM_TAG}..${TO_TAG}" --oneline --no-merges

# Full log with author, date, body (used for BREAKING CHANGE footers)
git log "${FROM_TAG}..${TO_TAG}" \
  --no-merges \
  --format="%H%n%s%n%b%n---COMMIT_END---"
```

Count the commits. If there are 0 commits, tell the user: "No commits found between
`$FROM_TAG` and `$TO_TAG`. These tags may point to the same commit."

### Step 4 — Detect the new version number (if TO_TAG is HEAD)

If `TO_TAG` is `HEAD` (i.e., unreleased changes), infer the next version by scanning
the commit types:

```bash
# Check for BREAKING CHANGE footers or ! suffix
git log "${FROM_TAG}..HEAD" --format="%s %b" | grep -iE "BREAKING CHANGE|^[a-z]+(\(.+\))?!:"
```

- Any breaking change → bump MAJOR
- Any `feat:` commit (no breaking) → bump MINOR
- Only `fix:`, `perf:`, `chore:`, `docs:` → bump PATCH

Display the inferred next version to the user: "Based on commit types, the next
version should be `v<X.Y.Z>`."

### Step 5 — Parse and group commits by type

Read every commit subject line. Classify each using the conventional commit prefix:

| Group heading (in notes)     | Commit prefixes matched                  |
|------------------------------|------------------------------------------|
| Breaking Changes             | Any commit with `!` or `BREAKING CHANGE` footer |
| New Features                 | `feat:`                                  |
| Bug Fixes                    | `fix:`                                   |
| Performance Improvements     | `perf:`                                  |
| Refactoring                  | `refactor:`                              |
| Documentation                | `docs:`                                  |
| Tests                        | `test:`                                  |
| Build & CI                   | `chore:`, `ci:`, `build:`                |
| Reverts                      | `revert:`                                |
| Other                        | Any commit that does not match a prefix  |

For **Breaking Changes**, also extract the `BREAKING CHANGE:` footer text from the
full commit body — this becomes the migration note.

For **Other** (non-conventional commits): group them at the bottom under "Other
Changes" and add a note: "(These commits do not follow conventional commit format.)"

### Step 6 — Format each entry

For each commit, format the entry as:

```
- <description> ([<short-hash>](commit-url))
```

- Strip the type prefix from the description: `feat(auth): add OAuth2 login` →
  `Add OAuth2 login`
- Capitalize the first word
- Include the scope in parentheses if meaningful: `Add OAuth2 login (auth)`
- Include the short commit hash as a link when the remote URL is known:

```bash
git remote get-url origin
# Convert git@github.com:owner/repo.git → https://github.com/owner/repo/commit/<hash>
```

### Step 7 — Compose the release notes document

Use this exact structure:

```markdown
# Release v<version> — <date>

> <One-sentence summary of the release theme, e.g., "Focuses on authentication
> improvements and performance fixes in the query layer.">

## Breaking Changes

> Upgrade guide: [link to migration doc if applicable]

- **Remove `get_user_by_id(i32)`** — replaced by `find_user(Uuid)`. Update all call
  sites. See [migration notes](#) for details. ([a1b2c3d](url))

## New Features

- Add OAuth2 login via GitHub and Google (auth) ([d4e5f6a](url))
- Support dark mode preference from system settings (ui) ([b7c8d9e](url))

## Bug Fixes

- Reject expired JWT tokens before hitting the database (auth) ([f0a1b2c](url))

## Performance Improvements

- Cache user lookups with 60-second TTL (db) ([c3d4e5f](url))

## Other Changes

- Update README with deployment instructions ([a6b7c8d](url))

---

**Full changelog:** https://github.com/owner/repo/compare/v1.2.0...v1.3.0
```

Date format: `YYYY-MM-DD` using today's date for unreleased, or the tag date for
tagged releases:
```bash
git log -1 --format=%ai "$TO_TAG" 2>/dev/null | cut -d' ' -f1
```

Omit any section that has zero entries. Do not include empty headings.

### Step 8 — Handle non-conventional commit history

If more than 50% of commits lack a conventional prefix, warn:
> "Most commits in this range do not follow conventional commit format. The grouping
> below is based on best-effort keyword matching. Consider adopting conventional
> commits for more accurate future changelogs."

Fall back to keyword-based grouping:
- Subject contains "fix", "bug", "patch", "correct" → Bug Fixes
- Subject contains "add", "new", "implement", "introduce" → New Features
- Subject contains "refactor", "clean", "restructure" → Refactoring
- Everything else → Other Changes

### Step 9 — Write to CHANGELOG.md (optional)

Ask the user: "Should I prepend this to `CHANGELOG.md`? (yes / no / show only)"

If yes:
```bash
# Prepend new release section above existing content
# Keep existing content intact below the new section
```

Read the existing `CHANGELOG.md` (or create it if absent), insert the new section
at the top (after any title line), and write the file back.

### Step 10 — Create GitHub Release (optional)

Ask: "Should I create a GitHub Release for `v<version>` using these notes? (yes / no)"

If yes:
```bash
gh release create v<version> \
  --title "v<version>" \
  --notes "$(cat <<'EOF'
<release notes content>
EOF
)"
```

---

## Examples

### Example 1 — Generate notes between two tags

**User says:** "what changed since v1.2.0"

**Git log (v1.2.0..HEAD):**
```
feat(auth): add OAuth2 login via GitHub
fix(auth): reject expired JWT tokens for admin roles
perf(db): cache user lookups with 60s TTL
chore(deps): upgrade axios to 1.7.4
docs: update README deployment section
```

**Claude generates:**

```markdown
# Release v1.3.0 — 2026-03-19

> Introduces OAuth2 social login, hardens token validation, and improves
> database query performance.

## New Features

- Add OAuth2 login via GitHub (auth) ([a1b2c3d](https://github.com/owner/repo/commit/a1b2c3d))

## Bug Fixes

- Reject expired JWT tokens for admin roles (auth) ([d4e5f6a](https://github.com/owner/repo/commit/d4e5f6a))

## Performance Improvements

- Cache user lookups with 60-second TTL (db) ([b7c8d9e](https://github.com/owner/repo/commit/b7c8d9e))

## Build & CI

- Upgrade axios to 1.7.4 (deps) ([c3d4e5f](https://github.com/owner/repo/commit/c3d4e5f))

---

**Full changelog:** https://github.com/owner/repo/compare/v1.2.0...v1.3.0
```

---

### Example 2 — Breaking change release

**User says:** "release notes from v1.3.0 to v2.0.0"

**Git log includes:**
```
feat(api)!: migrate user lookup to UUID-based identity

BREAKING CHANGE: get_user_by_id(i32) removed. Use find_user(Uuid) instead.
```

**Claude generates:**

```markdown
# Release v2.0.0 — 2026-03-19

> Major release migrating user identity to UUID, removing deprecated integer ID APIs.

## Breaking Changes

- **Remove `get_user_by_id(i32)`** — replaced by `find_user(Uuid)`. All integer
  user ID references must be updated. See the migration guide for backfill steps.
  ([f1e2d3c](url))

## New Features

- Support UUID-based user identity across all API endpoints (api) ([f1e2d3c](url))

---

**Full changelog:** https://github.com/owner/repo/compare/v1.3.0...v2.0.0
```

---

### Example 3 — No tags found, full project history

**User says:** "generate a changelog"

**No tags exist in the repository.**

**Claude responds:**
> No semver tags found. I'll generate notes covering the full commit history from
> the initial commit to HEAD. Consider tagging your first release once you review
> these notes.

Then produces a full changelog from `$(git rev-list --max-parents=0 HEAD)` to `HEAD`.

---

## Anti-patterns

1. **Including merge commits.** Always use `--no-merges` when fetching the git log.
   Merge commit messages ("Merge branch 'feat/x' into main") add noise with no value.

2. **Listing every chore/docs commit prominently.** Keep `Build & CI` and
   `Documentation` sections at the bottom. Users care most about features, fixes,
   and breaking changes. Burying a breaking change below 20 chore entries is a
   documentation failure.

3. **Fabricating commit details.** Never paraphrase or embellish beyond what the
   commit message says. If the commit message is vague ("fix stuff"), reproduce it
   faithfully in the "Other Changes" section rather than guessing what was fixed.

4. **Overwriting CHANGELOG.md without reading it first.** Always read the existing
   file before writing. Prepend the new section; never discard historical entries.

5. **Skipping the version inference step for unreleased changes.** If `TO_TAG` is
   HEAD, always infer and display the recommended next version based on commit types
   before generating notes. The version is part of the release notes header.
