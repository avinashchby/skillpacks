---
name: debugger
description: >
  Systematic debugging skill that enforces root-cause analysis before fixes.
  Use when: errors occur, tests fail, unexpected behavior happens, stack traces appear,
  "it's broken", "not working", "fix this bug", "debug this", "why is this crashing".
  NEVER jump to a fix. Always reproduce → analyze → hypothesize → verify → fix → harden.
version: 2.0.0
---

# Debugger — Systematic Root-Cause Debugging

## When to Use

Activate when the user reports an error, test failure, crash, unexpected behavior, or says anything like "fix", "debug", "broken", "not working", "failing", "error", "crash", "bug".

**Use ESPECIALLY when:**
- Under time pressure (emergencies make guessing tempting)
- "Just one quick fix" seems obvious
- You've already tried multiple fixes
- You don't fully understand the issue

## Protocol — STRICTLY follow these 6 phases IN ORDER

```
Phase 1: REPRODUCE → Phase 2: ANALYZE → Phase 3: HYPOTHESIZE → Phase 4: VERIFY → Phase 5: FIX → Phase 6: HARDEN
                                                       ↑                              |
                                                       └──── fix fails ───────────────┘
```

---

### Phase 1: REPRODUCE

Before anything else, confirm the problem exists and capture evidence.

1. Run the failing command/test/action exactly as described
2. Capture the COMPLETE error output — stack trace, error code, log messages
3. Note the exact file, line number, and function where it fails
4. If you cannot reproduce, ask the user for exact reproduction steps
5. STOP here if you cannot reproduce — do not guess at fixes

**For multi-component systems** (API → service → database, CI → build → deploy):
Add diagnostic instrumentation at each component boundary BEFORE diagnosing:
```
For EACH component boundary:
  - Log what data enters the component
  - Log what data exits the component
  - Verify environment/config propagation
  - Check state at each layer

Run once to gather evidence showing WHERE it breaks.
```
This reveals which layer fails (e.g., secrets → workflow OK, workflow → build BROKEN).

Output format:
```
REPRODUCTION:

Command: [what was run]
Error: [exact error message]
Location: [file:line]
Reproducible: yes/no
Components involved: [list if multi-component]
Failing layer: [which component boundary breaks, if applicable]
```

---

### Phase 2: ANALYZE

Find what works, compare it to what's broken. Do NOT skip this phase.

1. **Find working examples** — locate similar working code in the same codebase
2. **Compare broken vs working** — list every difference, however small
3. **Check recent changes** — `git log --oneline -10 -- <file>` and `git diff HEAD~5 -- <file>`
4. **Trace data flow backward** — from where the error appears, trace UP the call chain:
   - What code directly causes the error?
   - What called it with the bad value?
   - Keep tracing up until you find the source
   - The source is often 3-5 levels above where the error appears

Output format:
```
ANALYSIS:

Working example: [file:line — what works and is similar]
Key differences: [list each difference between working and broken]
Recent changes: [relevant commits that touched this area]
Data flow trace: [chain from error site back to origin of bad value]
```

---

### Phase 3: HYPOTHESIZE

List exactly 3 possible root causes, ranked by probability. For each:

1. State the hypothesis clearly in one sentence
2. Explain WHY you think this could be the cause (reference evidence from Phase 1 and 2)
3. Describe ONE specific check that would confirm or rule it out

Output format:
```
HYPOTHESES:

1. [Most likely] — ...
   Evidence: [what from Phase 1/2 supports this]
   Check: [specific action to verify]
2. [Second likely] — ...
   Evidence: [what from Phase 1/2 supports this]
   Check: [specific action to verify]
3. [Third likely] — ...
   Evidence: [what from Phase 1/2 supports this]
   Check: [specific action to verify]
```

---

### Phase 4: VERIFY

Test each hypothesis starting from most likely. For each:

1. Run the specific check you identified
2. Record the result: CONFIRMED, RULED OUT, or INCONCLUSIVE
3. If confirmed, move to Phase 5
4. If all ruled out, generate 3 new hypotheses based on what you learned
5. Never test more than 2 rounds of hypotheses (6 total) — if still stuck, escalate to user

Rules:
- Read the actual source code around the error location
- Check types, null checks, async/await patterns, import paths
- Check if the same pattern works elsewhere in the codebase
- DO NOT modify any code during this phase

---

### Phase 5: FIX

Only after root cause is CONFIRMED:

**Step 1: Create a failing test FIRST**
- Write the simplest possible test that reproduces the bug
- Run it — it MUST fail
- This test proves the bug exists and will prove the fix works
- If no test framework is available, a one-off script that exits non-zero is acceptable

**Step 2: Apply the minimal fix**
- State the confirmed root cause in one sentence
- Change the MINIMUM number of lines possible
- Never add try/catch to suppress an error — fix the cause
- Never change multiple things at once
- If you need to change more than 20 lines, pause and explain why

**Step 3: Verify the fix**
- Re-run the failing test from Step 1 — it MUST pass now
- Re-run the original failing command from Phase 1
- Run related tests to check for regressions

**Step 4: Escalation rule — 3 strikes**
- If the fix doesn't work, return to Phase 3 with new information
- Track how many fix attempts you've made
- **If 3+ fixes have failed: STOP.** This is no longer a bug — it's an architectural problem.
  - Each fix revealing new problems in different places = wrong architecture
  - Fixes requiring "massive refactoring" = wrong abstraction
  - Present the pattern of failures to the user and discuss fundamentals before attempting fix #4

---

### Phase 6: HARDEN

After the fix passes, prevent this class of bug from recurring.

1. **Validate at the source** — add input validation where the bad data originates, not where it crashes
2. **Consider adjacent layers** — if data passes through 3 layers before crashing, add validation at each layer:
   - Entry point: reject obviously invalid input
   - Business logic: ensure data makes sense for this operation
   - Environment guard: prevent dangerous operations in specific contexts (e.g., refuse destructive ops outside tmp dirs in tests)
3. **Only add hardening that is proportional** — a typo fix doesn't need 4 layers of validation. A data-flow bug that crossed 3 boundaries does.

Skip this phase if: the bug was a simple typo, wrong import path, or config value. Not everything needs hardening.

---

## Red Flags — STOP and Return to Phase 1

If you catch yourself thinking any of these, you are rationalizing. STOP.

| Thought | Reality |
|---------|---------|
| "Quick fix for now, investigate later" | Investigation IS the fix. Skipping it guarantees rework. |
| "Just try changing X and see if it works" | That's shotgun debugging. Form a hypothesis first. |
| "It's probably X, let me fix that" | "Probably" means you haven't verified. Go to Phase 4. |
| "I don't fully understand but this might work" | Partial understanding = partial fix = new bugs. |
| "Add multiple changes, run tests" | Can't isolate what worked. One change at a time. |
| "One more fix attempt" (after 2+ failures) | 3 strikes = architectural problem. Escalate. |
| "The issue seems simple, skip the process" | Simple bugs have root causes too. Process is fast for simple bugs. |
| "Let me add error handling around this" | Error handling hides bugs. Fix the cause. |

**User signals you're doing it wrong:**
- "Stop guessing" — you're proposing fixes without understanding
- "Is that not happening?" — you assumed without verifying
- "We're stuck?" (frustrated) — your approach isn't working
- Any of these → STOP. Return to Phase 1.

---

## Anti-Patterns — NEVER do these

- Jumping straight to a fix without reproducing first
- Wrapping errors in try/catch instead of fixing the root cause
- Changing multiple files simultaneously hoping something works
- "Shotgun debugging" — random changes without a hypothesis
- Saying "I think the issue might be..." without testing the hypothesis
- Adding console.log everywhere instead of reading the code carefully
- Reverting to a working state without understanding what broke
- Assuming the user's description is the root cause without verifying
- Fixing at the symptom point instead of tracing back to the source
- Raising max_iterations / max_retries to "fix" a loop (just makes failure more expensive)
- Deleting node_modules / clearing caches as a first resort (find the actual cause)

---

## Examples

### Example 1: Test Failure (simple — single file)

User: "my login test is failing"

RIGHT approach:
→ **Phase 1 (Reproduce):** Run the test, capture: `TypeError: Cannot read property 'email' of undefined at auth.test.ts:23`
→ **Phase 2 (Analyze):** Find working signup test with mock user `{ name: "test", email: "t@t.com" }`. Login test mock is `{ name: "test" }` — missing email field. `git log` shows mock was copy-pasted from a different test 3 days ago.
→ **Phase 3 (Hypothesize):** 1) Mock user missing email field — supported by diff with working test. Check: read mock object.
→ **Phase 4 (Verify):** Mock user is `{ name: "test" }` with no email. CONFIRMED.
→ **Phase 5 (Fix):** Write failing test asserting login returns user with email. Add `email: "test@test.com"` to mock. Test passes.
→ **Phase 6 (Harden):** Skip — simple data omission, no hardening needed.

### Example 2: Runtime Crash (multi-component)

User: "app crashes when I click submit"

RIGHT approach:
→ **Phase 1 (Reproduce):** Click submit, capture: `Unhandled promise rejection: Network error`. Add network logging — request goes to `/api/users`, server has no such route.
→ **Phase 2 (Analyze):** Working "get profile" feature calls `/api/user` (singular). Submit calls `/api/users` (plural). `git log` shows API route was renamed from plural to singular 2 weeks ago, but the submit form wasn't updated.
→ **Phase 3 (Hypothesize):** 1) Frontend URL is stale after API rename — supported by git history and working vs broken comparison.
→ **Phase 4 (Verify):** API route file defines `/api/user`. Frontend submit calls `/api/users`. CONFIRMED.
→ **Phase 5 (Fix):** Fix frontend URL from `/api/users` to `/api/user`. Submit works.
→ **Phase 6 (Harden):** Add a shared route constants file imported by both API and frontend so URLs can't drift independently.

### Example 3: Multi-Layer Failure (tracing required)

User: "CI build passes but the deployed app shows a blank page"

RIGHT approach:
→ **Phase 1 (Reproduce):** Deploy locally with production build. Blank page. Browser console: `Uncaught TypeError: window.CONFIG is undefined at main.js:1`. Add instrumentation at each layer: build output → deploy script → HTML template → runtime.
→ **Phase 2 (Analyze):** Local dev works because Vite injects `window.CONFIG` via plugin. Production build skips this. Trace: `index.html` has `<script>window.CONFIG = %CONFIG%</script>` — placeholder never replaced. Deploy script should replace it. `git log` shows deploy script was refactored last week.
→ **Phase 3 (Hypothesize):** 1) Deploy script no longer runs config injection — refactor broke the `sed` replacement. 2) Build output changed the placeholder format. 3) Env vars missing in CI.
→ **Phase 4 (Verify):** Read deploy script — it now uses `envsubst` which expects `$CONFIG` not `%CONFIG%`. CONFIRMED — format mismatch after refactor.
→ **Phase 5 (Fix):** Change HTML placeholder from `%CONFIG%` to `${CONFIG}` to match `envsubst` syntax. Deploy. Page loads.
→ **Phase 6 (Harden):** Add a post-deploy smoke test that checks `window.CONFIG` is defined. Add validation in deploy script that fails if any `%...%` placeholders remain in output.
