---
name: quality-doc-generator
description: >
  Generates complete API documentation (JSDoc, TSDoc, Python docstrings, Go doc
  comments, Rust doc comments) for all exported or public items in a file. Triggered
  by phrases like "document this", "add docs", "generate documentation", "add jsdoc",
  "write docstrings", "document the public API", or "add comments". Detects the
  language and existing documentation style automatically, follows the project's
  established conventions, and always includes @param/@returns/@throws/@example
  (or language-equivalent) tags. Produces documentation that is accurate to the
  actual implementation — it reads the function body before writing a single word.
version: 1.0.0
---

# Quality Doc Generator

## When to Use

Activate this skill when the user says any of the following:
- "document this"
- "add docs"
- "generate documentation"
- "add jsdoc"
- "write docstrings"
- "document the public API"
- "add comments to this"
- "this needs documentation"
- "annotate this file"

Also activate when the user asks "what does this function do?" and the function is non-trivial — generating documentation is often more useful than an inline explanation, because it improves the codebase.

Do NOT activate for:
- Inline explanatory comments within function bodies (that is a separate refactoring concern).
- README or module-level architectural documentation (that requires a different scope of context).
- Requests to explain code without modifying the file — answer directly instead.

---

## Instructions

### Step 1 — Read the Full File

1. Use `Read` to load the complete file. Never document a snippet; undocumented context in the same file affects what the docs should say.
2. Identify the language from the file extension and content.
3. Note the file's existing documentation style:
   - Are there any existing doc comments? If so, match their structure exactly.
   - What tags are already in use? (`@param`, `:param`, etc.)
   - Is `@example` used? Are examples inline or in a separate block?
   - Is there a `@since` or `@author` convention?
4. Check for a project-level documentation config: `jsdoc.config.js`, `typedoc.json`, `mkdocs.yml`, `rustdoc` flags in `Cargo.toml`. These constrain the tag set.

### Step 2 — Identify Items to Document

Collect every item that needs documentation:

**JavaScript / TypeScript**
- Exported `function`, `async function`, `const` (if it holds a function or class)
- Exported `class` and all its `public` and `protected` methods
- Exported `interface`, `type`, `enum`
- Module-level exported constants with non-obvious values

**Python**
- All `def` and `async def` at module level
- All class `def` methods with `public` naming (no leading underscore)
- All `class` definitions
- Module-level constants that are part of the public API (`__all__`)

**Go**
- All exported identifiers (start with uppercase): functions, types, methods, constants, variables
- The package itself if no package doc comment exists

**Rust**
- All `pub fn`, `pub struct`, `pub enum`, `pub trait`, `pub const`
- All `impl` block methods that are `pub`
- The crate root if no `//! ...` module doc exists

**Skip**: private/internal items, trivial getters/setters with self-evident names, `main()`, test functions.

### Step 3 — Read Each Item's Implementation

Before writing documentation for any function:
1. Read its full body, not just the signature.
2. Identify all parameters and their roles, including optional ones and defaults.
3. Identify the return value — what it represents, not just its type.
4. Identify all thrown errors or rejected promises — both explicit throws and propagated errors from dependencies.
5. Identify any important side effects (mutates input, writes to DB, emits events).
6. Identify preconditions the caller must satisfy.
7. Note any non-obvious behavior (what happens with `null` input, empty arrays, zero values).

### Step 4 — Write Documentation

#### JavaScript / TypeScript (JSDoc / TSDoc)

```ts
/**
 * [One-sentence summary. Start with a verb. No "This function".]
 *
 * [Optional: 1-3 sentence elaboration for non-trivial behavior,
 *  including important side effects or invariants.]
 *
 * @param paramName - [Description. For objects, describe shape if not typed.]
 * @param [optionalParam=defaultValue] - [Description including what default means.]
 * @returns [What the return value represents. Not just its type.]
 * @throws {ErrorType} [When this is thrown and why.]
 * @example
 * ```ts
 * const result = functionName(arg1, arg2);
 * // result: expected value
 * ```
 */
```

Rules:
- Summary line must be a complete sentence, imperative mood: "Paginates an array..." not "Paginate" or "This paginates".
- `@param` descriptions must state the role, not just restate the name.
- `@returns` must explain what the value means, not just say "the result".
- Include `@throws` for every distinct error condition.
- Include at least one `@example` for every public function. Two examples if behavior varies significantly with input.
- For TypeScript, omit type annotations in JSDoc if the TypeScript types are already expressive — no `@param {string} name`.
- Use `@deprecated` with a migration path if the item is deprecated.

#### Python (Google-style docstring, unless project uses NumPy or Sphinx)

```python
def function_name(param: type) -> return_type:
    """One-sentence summary. Start with a verb.

    Optional elaboration paragraph for non-trivial behavior.

    Args:
        param_name: Description. State the role, not just the type.
            For multi-line, indent continuation by 4 spaces.
        optional_param: Description. Include default behavior.

    Returns:
        Description of what the return value represents.

    Raises:
        ValueError: When and why this is raised.
        KeyError: When and why this is raised.

    Example:
        >>> result = function_name("input")
        >>> print(result)
        'expected output'
    """
```

If the project uses NumPy style (detected by existing `Parameters\n----------` patterns) or Sphinx style (detected by `:param:` tags), match that style instead.

#### Go

```go
// FunctionName does X to Y and returns Z.
//
// Elaboration for non-trivial behavior. Go doc comments use plain prose,
// not tags. Parameter roles are described inline in the prose.
//
// Returns an error if [condition]. The caller is responsible for [cleanup].
//
// Example:
//
//	result, err := FunctionName(ctx, input)
//	if err != nil { ... }
func FunctionName(...) ...
```

Rules:
- First line must be `// FunctionName` (starts with the symbol name).
- No `@` tags. All documentation is prose.
- Code examples use the `//\t` (tab-indented) convention for `go doc` rendering.
- Document error return values explicitly in prose.

#### Rust

```rust
/// One-sentence summary. Start with a verb.
///
/// Optional elaboration. Use markdown — backticks for code, `**bold**` sparingly.
///
/// # Arguments
///
/// * `param_name` - Description of role and valid values.
///
/// # Returns
///
/// Description of the return value meaning.
///
/// # Errors
///
/// Returns [`ErrorType`] when [condition].
///
/// # Panics
///
/// Panics if [condition] (document only if the function can panic).
///
/// # Examples
///
/// ```
/// let result = function_name(arg);
/// assert_eq!(result, expected);
/// ```
pub fn function_name(...) -> ...
```

### Step 5 — Output the Edited File

Use the `Edit` tool to add documentation in-place. Do not rewrite the entire file. Make one `Edit` call per item, or batch non-overlapping edits.

After all edits are applied, provide a summary:

```
## Documentation Added

| Item | Type | Tags Added |
|------|------|-----------|
| functionName | function | @param ×2, @returns, @throws ×1, @example |
| ClassName | class | summary |
| ClassName.methodName | method | @param ×1, @returns, @example |

[N] items documented. [M] items skipped (private/trivial).
```

---

## Examples

### Example 1 — TypeScript function, no existing docs

**Before:**
```ts
export async function fetchUser(id: string, options?: { timeout?: number }): Promise<User | null> {
  if (!id) throw new TypeError('id is required');
  const res = await httpClient.get(`/users/${id}`, { timeout: options?.timeout ?? 5000 });
  if (res.status === 404) return null;
  return res.data as User;
}
```

**After:**
```ts
/**
 * Fetches a user record by ID from the remote API.
 *
 * Returns `null` if no user exists with the given ID (HTTP 404).
 * All other non-2xx responses are thrown as `HttpError`.
 *
 * @param id - The unique user identifier. Must be a non-empty string.
 * @param options - Optional request configuration.
 * @param [options.timeout=5000] - Request timeout in milliseconds.
 * @returns The matching `User` object, or `null` if not found.
 * @throws {TypeError} If `id` is empty or falsy.
 * @throws {HttpError} If the API returns a non-404 error status.
 * @example
 * ```ts
 * const user = await fetchUser('usr_123');
 * if (user === null) {
 *   console.log('User not found');
 * }
 * ```
 */
export async function fetchUser(id: string, options?: { timeout?: number }): Promise<User | null> {
```

### Example 2 — Python class with methods

**Before:**
```python
class TokenBucket:
    def __init__(self, capacity: int, refill_rate: float):
        self._tokens = capacity
        self._capacity = capacity
        self._refill_rate = refill_rate
        self._last_refill = time.monotonic()

    def consume(self, tokens: int = 1) -> bool:
        self._refill()
        if self._tokens >= tokens:
            self._tokens -= tokens
            return True
        return False
```

**After:**
```python
class TokenBucket:
    """Rate limiter using the token bucket algorithm.

    Tokens are added at `refill_rate` per second up to `capacity`.
    Callers consume tokens to represent units of work; if insufficient
    tokens are available, the request is rejected without blocking.
    """

    def __init__(self, capacity: int, refill_rate: float):
        """Initialize a new token bucket.

        Args:
            capacity: Maximum number of tokens the bucket can hold.
                Also the initial token count.
            refill_rate: Number of tokens added per second.

        Raises:
            ValueError: If capacity <= 0 or refill_rate <= 0.
        """

    def consume(self, tokens: int = 1) -> bool:
        """Attempt to consume tokens from the bucket.

        Triggers a refill based on elapsed time before checking
        availability. Does not block if tokens are unavailable.

        Args:
            tokens: Number of tokens to consume. Defaults to 1.

        Returns:
            True if the tokens were consumed successfully.
            False if the bucket had insufficient tokens.

        Example:
            >>> bucket = TokenBucket(capacity=10, refill_rate=1.0)
            >>> bucket.consume(5)
            True
            >>> bucket.consume(10)
            False
        """
```

---

## Anti-patterns

1. **Restating the type information already present in the signature.** In TypeScript, `@param {string} name - The name string` is redundant. The doc should explain the role: `@param name - The display name shown in the user profile header.`

2. **Writing summary lines that begin with "This function".** Correct: "Validates an email address against RFC 5322." Wrong: "This function is used to validate an email address."

3. **Documenting only the happy path.** If a function can return `null`, throw, or produce different output shapes based on input, every variant must be documented. Omitting error conditions from docs is the most common source of misuse bugs.

4. **Copying the function name into the summary verbatim.** `getUserById` → "Gets user by ID" is noise. The doc should explain what "getting" means here: does it hit the DB? Does it cache? What happens if the user doesn't exist?

5. **Writing examples that don't actually demonstrate the non-obvious behavior.** If a function's interesting behavior is what it returns for missing input, the example must show that case. Generic happy-path-only examples waste the `@example` slot.
