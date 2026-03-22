---
name: debugger
description: >
  Systematic debugging skill that enforces root-cause analysis before fixes.
  Use when: errors occur, tests fail, unexpected behavior happens, stack traces appear,
  "it's broken", "not working", "fix this bug", "debug this", "why is this crashing".
  NEVER jump to a fix. Always reproduce → hypothesize → verify → fix.
version: 1.0.0
---

# Debugger — Systematic Root-Cause Debugging

## When to Use
Activate when the user reports an error, test failure, crash, unexpected behavior, or says anything like "fix", "debug", "broken", "not working", "failing", "error", "crash", "bug".

## Protocol — STRICTLY follow these 4 phases IN ORDER

### Phase 1: REPRODUCE
Before anything else, confirm the problem exists and capture evidence.

1. Run the failing command/test/action exactly as described
2. Capture the COMPLETE error output — stack trace, error code, log messages
3. Note the exact file, line number, and function where it fails
4. If you cannot reproduce, ask the user for exact reproduction steps
5. STOP here if you cannot reproduce — do not guess at fixes

Output format:
```
REPRODUCTION:

Command: [what was run]
Error: [exact error message]
Location: [file:line]
Reproducible: yes/no
```

### Phase 2: HYPOTHESIZE
List exactly 3 possible root causes, ranked by probability. For each:

1. State the hypothesis clearly in one sentence
2. Explain WHY you think this could be the cause
3. Describe ONE specific check that would confirm or rule it out

Output format:
```
HYPOTHESES:

1. [Most likely] — ...
   Check: [specific action to verify]
2. [Second likely] — ...
   Check: [specific action to verify]
3. [Third likely] — ...
   Check: [specific action to verify]
```

### Phase 3: VERIFY
Test each hypothesis starting from most likely. For each:

1. Run the specific check you identified
2. Record the result: CONFIRMED, RULED OUT, or INCONCLUSIVE
3. If confirmed, move to Phase 4
4. If all ruled out, generate 3 new hypotheses based on what you learned
5. Never test more than 2 rounds of hypotheses (6 total) — if still stuck, escalate to user

Rules:
- Read the actual source code around the error location
- Check types, null checks, async/await patterns, import paths
- Look at recent git changes: `git log --oneline -10 -- <file>`
- Check if the same pattern works elsewhere in the codebase
- DO NOT modify any code during this phase

### Phase 4: FIX
Only after root cause is CONFIRMED:

1. State the confirmed root cause in one sentence
2. Describe the minimal fix (fewest lines changed)
3. Apply the fix
4. Re-run the original failing command to verify it passes
5. Run related tests to check for regressions
6. If fix doesn't work, return to Phase 2 with new information

Rules:
- Change the MINIMUM number of lines possible
- Never add try/catch to suppress an error — fix the cause
- Never change multiple things at once
- If you need to change more than 20 lines, pause and explain why
- Always verify the fix by running the exact reproduction from Phase 1

## Anti-Patterns — NEVER do these

- Jumping straight to a fix without reproducing first
- Wrapping errors in try/catch instead of fixing the root cause
- Changing multiple files simultaneously hoping something works
- "Shotgun debugging" — random changes without a hypothesis
- Saying "I think the issue might be..." without testing the hypothesis
- Adding console.log everywhere instead of reading the code carefully
- Reverting to a working state without understanding what broke
- Assuming the user's description is the root cause without verifying

## Examples

### Example 1: Test Failure
User: "my login test is failing"

WRONG approach:
→ Immediately rewrite the login test

RIGHT approach:
→ Phase 1: Run the test, capture error: "TypeError: Cannot read property 'email' of undefined at auth.test.ts:23"
→ Phase 2: Hypotheses: 1) Mock user object missing email field, 2) Auth service returning undefined, 3) Test setup not running
→ Phase 3: Check hypothesis 1 — read test file, find mock user is { name: "test" } with no email. CONFIRMED.
→ Phase 4: Add email to mock: { name: "test", email: "test@test.com" }. Re-run. Passes.

### Example 2: Runtime Crash
User: "app crashes when I click submit"

WRONG approach:
→ Add error boundaries everywhere

RIGHT approach:
→ Phase 1: Reproduce by clicking submit, capture: "Unhandled promise rejection: Network error"
→ Phase 2: 1) API endpoint is wrong/down, 2) CORS blocking the request, 3) Request body is malformed
→ Phase 3: Check network tab — API returns 404. Check API route file — endpoint is /api/user but frontend calls /api/users. CONFIRMED.
→ Phase 4: Fix frontend URL from /api/users to /api/user. Test. Works.

### Example 3: Build Error
User: "npm run build fails"

WRONG approach:
→ Delete node_modules and reinstall

RIGHT approach:
→ Phase 1: Run build, capture: "TS2307: Cannot find module './utils/helper'"
→ Phase 2: 1) File was renamed/moved, 2) Import path is wrong, 3) tsconfig paths misconfigured
→ Phase 3: Check filesystem — file exists at ./utils/helpers.ts (plural). CONFIRMED — typo in import.
→ Phase 4: Fix import from './utils/helper' to './utils/helpers'. Build passes.
