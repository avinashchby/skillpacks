---
name: quality-test-generator
description: >
  Analyzes a source file and generates a complete test suite covering the happy
  path, edge cases, error cases, and all reachable branches. Triggered by phrases
  like "generate tests", "write tests for", "test this file", "add test coverage",
  "write unit tests", or "cover this with tests". Automatically detects the test
  framework already in use (Jest, Vitest, pytest, Go testing, Rust #[test], etc.),
  mirrors the project's existing naming and assertion conventions, and produces
  tests that are immediately runnable without modification. Each generated test
  includes a comment explaining what invariant it is verifying.
version: 1.0.0
---

# Quality Test Generator

## When to Use

Activate this skill when the user says any of the following:
- "generate tests"
- "write tests for"
- "test this file"
- "add test coverage"
- "write unit tests"
- "cover this with tests"
- "I need tests for this"
- "what should I test here"

Also activate when the user shares a function or module and asks "is this correct?" — generating tests is often the right way to verify correctness.

Do NOT activate for:
- Requests to run or debug existing tests ("why is this test failing").
- Requests for integration or end-to-end test strategies — those require broader context. Clarify scope first.

---

## Instructions

### Step 1 — Gather Context

1. Use `Read` to load the full source file the user wants tested.
2. Use `Glob` to find existing test files: patterns like `**/*.test.ts`, `**/*.spec.js`, `**/*_test.go`, `**/test_*.py`, `**/tests/**/*.rs`.
3. Read one or two existing test files to extract:
   - Test framework and version (Jest, Vitest, pytest, Go `testing`, Rust `#[test]`, etc.)
   - Assertion style (Jest `expect().toBe()`, pytest `assert x == y`, etc.)
   - Mocking library (jest.mock, unittest.mock, testify, mockall, etc.)
   - File naming convention (`*.test.ts` vs `*.spec.ts`, `test_foo.py` vs `foo_test.py`)
   - Any test helpers, fixtures, or factory functions already defined
   - Whether tests are co-located with source or in a separate `tests/` directory
4. If no existing tests exist, infer framework from `package.json`, `pyproject.toml`, `Cargo.toml`, or `go.mod`.

### Step 2 — Analyze the Source File

For every exported/public function, class method, or route handler, extract:

- **Signature**: parameter types, return type, throws/rejects declarations
- **Branches**: every `if`, `else`, ternary, `switch`, `try/catch`, early return
- **Side effects**: DB calls, HTTP calls, file I/O, event emissions, state mutations
- **Preconditions**: input validation, guard clauses, type checks
- **Postconditions**: what the function guarantees on success

Build a mental branch map. Every branch is a test candidate.

### Step 3 — Plan the Test Cases

For each function, plan tests in this order:

**Happy path (1–2 tests)**
- Valid, typical inputs that exercise the main success flow.
- One test per significant variation in the success path.

**Edge cases**
- Empty inputs: `""`, `[]`, `{}`, `0`, `null`, `undefined`
- Boundary values: off-by-one for numeric ranges, min/max lengths
- Type coercion traps (JS-specific): `"0"` vs `0`, `NaN`, `Infinity`
- Concurrent calls if the function has shared state

**Error cases**
- Each explicit `throw` or `reject` in the function
- Each validation failure path
- Each external dependency failure (mock the dependency to throw)
- Missing required fields

**Branch coverage**
- One test per branch not already covered above.
- Use a coverage comment `// covers: line N — else branch` to make intent clear.

### Step 4 — Handle Dependencies

For each external dependency (DB, HTTP, filesystem, time, randomness):

- **JavaScript/TypeScript**: Use `jest.mock()` / `vi.mock()` at module level. Use `jest.spyOn()` for method-level mocking. Reset mocks in `afterEach`.
- **Python**: Use `unittest.mock.patch` as a decorator or context manager. Use `pytest-mock`'s `mocker` fixture if already in use.
- **Go**: Define interfaces for dependencies; pass mock implementations. Use `testify/mock` if already in use.
- **Rust**: Use feature-flagged mock implementations or `mockall` if already in use.

Never make real network calls, DB calls, or filesystem writes in unit tests. If the code makes these calls without an injectable boundary, note that refactoring is needed and generate tests for the extractable pure logic.

### Step 5 — Write the Tests

Structure the test file as follows:

```
[imports — mirror existing test file import style]

[module-level mocks — before describe blocks]

describe('[ClassName or filename]', () => {
  describe('[functionName]', () => {
    // Setup shared across tests in this group
    beforeEach(() => { ... })
    afterEach(() => { ... })

    it('[does X when given Y]', () => {
      // Arrange
      // Act
      // Assert
    })
  })
})
```

Rules for test bodies:
- Follow Arrange / Act / Assert structure. One blank line between each section.
- One logical assertion per test. Multiple `expect` calls are fine if they all verify the same behavior.
- Test names must be full sentences describing behavior, not implementation: "returns empty array when input is empty" not "test empty".
- Never `console.log` inside tests.
- Never use `any` type assertions to paper over TypeScript errors in tests.

### Step 6 — Output

Produce the complete test file content. Then provide a brief summary table:

```
## Test Coverage Summary

| Function | Happy Path | Edge Cases | Error Cases | Branches Covered |
|----------|-----------|-----------|-------------|-----------------|
| foo()    | ✓         | 3         | 2           | 4/5             |
```

Note any branches that cannot be tested without refactoring, and explain what change would make them testable.

---

## Examples

### Example 1 — TypeScript utility function

**Source file: `src/utils/paginate.ts`**
```ts
export function paginate<T>(items: T[], page: number, pageSize: number): T[] {
  if (pageSize <= 0) throw new Error('pageSize must be positive');
  if (page < 1) throw new Error('page must be >= 1');
  const start = (page - 1) * pageSize;
  return items.slice(start, start + pageSize);
}
```

**Detected framework:** Jest + TypeScript

**Generated: `src/utils/paginate.test.ts`**
```ts
import { paginate } from './paginate';

describe('paginate', () => {
  const items = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

  describe('happy path', () => {
    it('returns the correct slice for page 1', () => {
      // Arrange / Act
      const result = paginate(items, 1, 3);

      // Assert
      expect(result).toEqual([1, 2, 3]);
    });

    it('returns the correct slice for a middle page', () => {
      const result = paginate(items, 2, 3);
      expect(result).toEqual([4, 5, 6]);
    });
  });

  describe('edge cases', () => {
    it('returns an empty array when page exceeds total pages', () => {
      // covers: slice returns [] when start >= items.length
      const result = paginate(items, 99, 3);
      expect(result).toEqual([]);
    });

    it('returns a partial page when items do not fill the last page', () => {
      const result = paginate(items, 4, 3);
      expect(result).toEqual([10]);
    });

    it('returns an empty array when source array is empty', () => {
      const result = paginate([], 1, 5);
      expect(result).toEqual([]);
    });
  });

  describe('error cases', () => {
    it('throws when pageSize is zero', () => {
      expect(() => paginate(items, 1, 0)).toThrow('pageSize must be positive');
    });

    it('throws when pageSize is negative', () => {
      expect(() => paginate(items, 1, -1)).toThrow('pageSize must be positive');
    });

    it('throws when page is less than 1', () => {
      expect(() => paginate(items, 0, 5)).toThrow('page must be >= 1');
    });
  });
});
```

### Example 2 — Python async function with external dependency

**Source file: `services/user_service.py`**
```python
async def get_user(user_id: int, db) -> dict:
    if user_id <= 0:
        raise ValueError("user_id must be positive")
    row = await db.fetchone("SELECT id, name FROM users WHERE id = ?", [user_id])
    if row is None:
        raise LookupError(f"User {user_id} not found")
    return {"id": row["id"], "name": row["name"]}
```

**Detected framework:** pytest + pytest-asyncio

**Generated: `tests/test_user_service.py`**
```python
import pytest
from unittest.mock import AsyncMock
from services.user_service import get_user


@pytest.fixture
def mock_db():
    return AsyncMock()


@pytest.mark.asyncio
async def test_get_user_returns_user_dict(mock_db):
    # Arrange
    mock_db.fetchone.return_value = {"id": 1, "name": "Alice"}

    # Act
    result = await get_user(1, mock_db)

    # Assert
    assert result == {"id": 1, "name": "Alice"}
    mock_db.fetchone.assert_awaited_once_with(
        "SELECT id, name FROM users WHERE id = ?", [1]
    )


@pytest.mark.asyncio
async def test_get_user_raises_lookup_error_when_not_found(mock_db):
    # Arrange
    mock_db.fetchone.return_value = None

    # Act / Assert
    with pytest.raises(LookupError, match="User 42 not found"):
        await get_user(42, mock_db)


@pytest.mark.parametrize("bad_id", [0, -1, -100])
@pytest.mark.asyncio
async def test_get_user_raises_value_error_for_non_positive_id(bad_id, mock_db):
    with pytest.raises(ValueError, match="user_id must be positive"):
        await get_user(bad_id, mock_db)

    # DB must not be called when validation fails
    mock_db.fetchone.assert_not_awaited()
```

---

## Anti-patterns

1. **Writing tests that only verify implementation, not behavior.** A test that asserts `spy.calledOnce` without asserting what the function returned tests nothing observable. Always assert the output or side effect, not just that internal methods were called.

2. **Generating tests that always pass regardless of the code.** For example: `expect(result).toBeDefined()`. This passes even if the function throws or returns `null`. Use specific equality assertions.

3. **Mocking the thing under test.** If you are testing `UserService`, do not mock `UserService`. Only mock its dependencies (DB, HTTP clients, etc.).

4. **Skipping the `afterEach` / teardown.** When using `jest.mock` or patching at module level, always restore mocks between tests. Leaked mock state causes order-dependent test failures that are extremely difficult to debug.

5. **Generating one giant test function that tests multiple behaviors.** Each test must test exactly one behavior. If a test function has more than one `// Act` section, split it. This ensures that when a test fails, the failure message is self-explanatory without reading the whole test body.
