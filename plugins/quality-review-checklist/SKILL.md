---
name: quality-review-checklist
description: >
  Performs a systematic, multi-dimensional code review across security (OWASP Top 10),
  performance bottlenecks, accessibility (WCAG 2.1), and error handling patterns.
  Triggered by phrases like "review this code", "code review", "review checklist",
  "audit this file", "security review", or "quality audit". Produces a structured
  report with severity-rated findings, specific line references, and actionable
  remediation steps. Use this skill whenever a thorough, checklist-driven review
  is needed rather than a quick glance at logic.
version: 1.0.0
---

# Quality Review Checklist

## When to Use

Activate this skill when the user says any of the following:
- "review this code"
- "code review"
- "review checklist"
- "audit this file"
- "security review"
- "quality audit"
- "what's wrong with this code"
- "is this production-ready"

Also activate when the user pastes code and asks for general feedback without a specific question — a checklist review is the appropriate default response.

Do NOT activate for narrow questions like "why does this function return null" or "fix this bug" — those call for targeted debugging, not a full review.

---

## Instructions

### Step 1 — Read and Orient

1. Use the `Read` tool to load the full file. Never review a truncated version.
2. Identify: language, framework, file role (controller, model, utility, component, etc.).
3. Note the file's public surface area: exported functions, classes, HTTP endpoints, event handlers.
4. Skim for obvious structural issues before diving into categories.

### Step 2 — Security Review (OWASP Top 10)

Check each category and note findings with line numbers:

**A01 — Broken Access Control**
- Are authorization checks present before every privileged operation?
- Do object references (IDs in URLs/params) get validated against the authenticated user?
- Are directory traversal paths possible via user-supplied filenames?

**A02 — Cryptographic Failures**
- Is sensitive data (passwords, tokens, PII) ever logged or exposed in error messages?
- Are secrets hardcoded or loaded from environment correctly?
- Is TLS enforced for outbound requests? (`http://` vs `https://`)
- Are passwords hashed with bcrypt/argon2, not MD5/SHA1?

**A03 — Injection**
- SQL: are all queries parameterized? Flag any string concatenation into queries.
- Command injection: flag `exec`, `spawn`, `eval` with user-supplied input.
- Template injection: flag template engines rendering raw user input.
- Path injection: flag `path.join` or `fs` calls using unvalidated user input.

**A04 — Insecure Design**
- Are rate limits present on authentication or expensive endpoints?
- Is there any "security through obscurity" (hiding endpoints instead of authorizing them)?

**A05 — Security Misconfiguration**
- Are CORS origins set to `*` in production contexts?
- Are debug modes, stack traces, or verbose errors returned to clients?
- Are default credentials or example API keys present?

**A06 — Vulnerable Components**
- Flag any inline version pins for known-vulnerable libraries (note: full dependency audit is out of scope here, flag what's visible).

**A07 — Identification and Authentication Failures**
- Are session tokens sufficiently random and long (≥128 bits)?
- Is there brute-force protection on login flows?
- Are JWTs validated (signature, expiry, audience)?

**A08 — Software and Data Integrity Failures**
- Are deserialization operations performed on untrusted data?
- Is input validated before being passed to serializers/deserializers?

**A09 — Security Logging Failures**
- Are authentication events (success and failure) logged?
- Do logs include timestamps and user identifiers?

**A10 — SSRF**
- Are there any outbound HTTP requests using user-supplied URLs?
- Is there an allowlist of permitted outbound domains?

### Step 3 — Performance Review

**Database / I/O**
- Flag N+1 patterns: loops that contain DB queries or HTTP calls.
- Flag missing indexes implied by query patterns (e.g., `WHERE email =` with no index hint).
- Flag synchronous I/O in async contexts (blocking calls inside async functions).
- Flag missing pagination on list endpoints.

**Algorithmic Complexity**
- Flag nested loops over large collections: O(n²) or worse.
- Flag repeated array `.find()` / `.filter()` inside loops — suggest Map/Set lookups.
- Flag expensive operations (sort, regex) inside hot loops.

**Caching**
- Flag expensive computations repeated on every request that could be memoized.
- Flag missing cache headers on static or semi-static responses.

**Memory**
- Flag unbounded collection growth (pushing to arrays without eviction).
- Flag large object graphs held in module-level scope.
- Flag stream operations that buffer entire payloads into memory.

### Step 4 — Accessibility Review (for UI code only)

Skip this section for backend-only files. For frontend code:

**Semantic HTML**
- Are interactive elements using native `<button>` / `<a>` rather than `<div onClick>`?
- Is heading hierarchy logical (no skipped levels)?

**ARIA**
- Are `aria-label` or `aria-labelledby` present on icon-only buttons?
- Are dynamic regions (`aria-live`) used for status updates?
- Are custom widgets (modals, dropdowns) following ARIA Authoring Practices?

**Keyboard Navigation**
- Are all interactive elements reachable via Tab?
- Is focus managed correctly when modals open/close?
- Are custom `onKeyDown` handlers present alongside `onClick`?

**Color and Contrast**
- Flag hardcoded color values that may fail WCAG AA contrast (4.5:1 for text).
- Flag information conveyed by color alone (e.g., red = error, with no icon or text).

**Images and Media**
- Are all `<img>` elements present with non-empty `alt` (or `alt=""` for decorative)?
- Do videos have captions or transcripts?

### Step 5 — Error Handling Review

- Flag missing error handling on: async operations, I/O calls, external API calls, JSON parse operations.
- Flag bare `catch(e) {}` or `catch { return null }` that silently swallow errors.
- Flag error messages that leak stack traces, file paths, or DB schema details to end users.
- Flag missing distinction between operational errors (expected, recoverable) and programmer errors (unexpected, crash-worthy).
- Flag missing cleanup in error paths (open file handles, DB connections, locks).
- Flag retry logic absence on transient network/DB failures.
- Verify that error responses use consistent shapes (e.g., `{ error: { code, message } }`).

### Step 6 — Compose the Report

Structure output as follows:

```
## Code Review Report — [filename]

### Summary
[2-3 sentence overview of overall quality and most critical issues]

### Findings

#### CRITICAL
| # | Category | Line | Issue | Fix |
|---|----------|------|-------|-----|
...

#### HIGH
...

#### MEDIUM
...

#### LOW / INFORMATIONAL
...

### Positive Observations
[Note 2-3 things done well — reviewers should reinforce good patterns]

### Recommended Next Steps
[Ordered list of the top 3-5 actions to take]
```

**Severity definitions:**
- CRITICAL: Exploitable security vulnerability or data loss risk. Fix before merge.
- HIGH: Likely bug, significant security weakness, or severe performance issue.
- MEDIUM: Code smell with real-world impact, missing validation, unclear error handling.
- LOW: Style, naming, minor inefficiency, nice-to-have improvement.

---

## Examples

### Example 1 — SQL Injection + Missing Error Handling

**Input code (Node.js):**
```js
app.get('/user', async (req, res) => {
  const name = req.query.name;
  const result = await db.query(`SELECT * FROM users WHERE name = '${name}'`);
  return res.json(result.rows);
});
```

**Expected output:**
```
## Code Review Report — routes/user.js

### Summary
This endpoint contains a critical SQL injection vulnerability and no error
handling. An attacker can exfiltrate the entire database or drop tables.

### Findings

#### CRITICAL
| # | Category | Line | Issue | Fix |
|---|----------|------|-------|-----|
| 1 | A03 Injection | 3 | String interpolation into SQL query. User controls `name` directly. | Use parameterized query: `db.query('SELECT * FROM users WHERE name = $1', [name])` |

#### HIGH
| # | Category | Line | Issue | Fix |
|---|----------|------|-------|-----|
| 2 | Error Handling | 3–4 | No try/catch around async DB call. Unhandled rejection crashes the process. | Wrap in try/catch, return 500 on error without leaking the error message. |

#### MEDIUM
| # | Category | Line | Issue | Fix |
|---|----------|------|-------|-----|
| 3 | Performance | 3 | `SELECT *` retrieves all columns. Select only needed fields. | Replace `*` with explicit column list. |
| 4 | A01 Access Control | 1 | No authentication middleware on this route. Any caller can query users. | Add auth middleware before handler. |

### Positive Observations
- Async/await is used correctly over callbacks.

### Recommended Next Steps
1. Switch to parameterized queries immediately.
2. Add try/catch and a generic 500 response.
3. Add authentication middleware.
```

### Example 2 — Frontend Accessibility Issues

**Input code (React):**
```jsx
function IconButton({ onDelete }) {
  return <div onClick={onDelete} style={{ color: 'red' }}>🗑</div>;
}
```

**Expected output:**
```
#### HIGH
| # | Category | Line | Issue | Fix |
|---|----------|------|-------|-----|
| 1 | A11y — Semantics | 2 | `<div>` used as interactive control. Not keyboard reachable, not announced as button. | Replace with `<button onClick={onDelete}>`. |
| 2 | A11y — ARIA | 2 | Icon-only button has no accessible name. Screen readers announce nothing useful. | Add `aria-label="Delete item"` to the button. |
| 3 | A11y — Color | 2 | Hardcoded `color: red` — verify contrast ratio meets WCAG AA (4.5:1). | Use a design token; verify with a contrast checker. |
```

---

## Anti-patterns

1. **Reporting findings without line numbers.** Every finding must reference a specific line or line range. Vague findings like "there are SQL injection risks in this file" are not actionable.

2. **Skipping sections because the file "looks safe".** Always work through every category. Many vulnerabilities are non-obvious. Mark sections "No findings" explicitly rather than omitting them.

3. **Treating LOW findings as not worth reporting.** Low-severity findings belong in the report under their own section. Omitting them gives a false sense of completeness.

4. **Suggesting fixes that introduce new problems.** For example, recommending `try { ... } catch { return res.json({ error: err.stack }) }` to fix missing error handling — this leaks internals. Every suggested fix must itself be reviewed for correctness.

5. **Reviewing only the pasted snippet when the file has more context.** Always use the `Read` tool on the full file path. Partial context produces false negatives (missing cross-function vulnerabilities) and false positives (flagging code that is in fact protected upstream).
