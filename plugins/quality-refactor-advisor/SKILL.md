---
name: quality-refactor-advisor
description: >
  Identifies code smells and structural problems in a file, including long functions,
  deep nesting, god objects, duplicate code, primitive obsession, feature envy, and
  shotgun surgery candidates. Triggered by phrases like "refactor", "code smells",
  "improve this code", "clean up", "this is messy", "how would you restructure this",
  or "what's wrong with the design". Produces a severity-rated list of findings, each
  with a concrete, named refactoring technique and a before/after snippet demonstrating
  the transformation. Does not silently rewrite files — it advises and waits for
  confirmation before making changes.
version: 1.0.0
---

# Quality Refactor Advisor

## When to Use

Activate this skill when the user says any of the following:
- "refactor"
- "code smells"
- "improve this code"
- "clean up"
- "this is messy"
- "how would you restructure this"
- "what's wrong with the design"
- "technical debt"
- "this is getting hard to maintain"

Also activate when the user pastes code that visually appears complex (deeply indented, very long, many responsibilities) and asks a vague question like "thoughts on this?".

Do NOT activate for:
- Bug fix requests — address the bug directly.
- Performance-specific requests — use the `quality-perf-profiler` skill instead.
- Security-specific requests — use the `quality-review-checklist` skill instead.

---

## Instructions

### Step 1 — Read the Full File

1. Use `Read` to load the complete file. Smells like duplicate code and feature envy only become visible across the full file.
2. Note the language, framework, and apparent purpose (utility module, domain model, controller, etc.).
3. Read any related files the user mentions or that are imported, if context is needed to identify feature envy or inappropriate intimacy.

### Step 2 — Measure Structural Metrics

Compute or estimate the following for each function/method:

| Metric | Threshold for Flag |
|--------|--------------------|
| Function length (lines) | > 30 lines |
| Nesting depth | > 3 levels |
| Parameter count | > 4 parameters |
| Return points | > 3 `return` statements |
| Cyclomatic complexity (branch count) | > 10 |
| Class method count | > 15 methods |
| Class field count | > 10 fields |
| File length | > 400 lines |

These are not hard rules — use judgment. A 35-line function that is a simple switch with well-named cases may be fine. Flag the metric and explain why it is or isn't a problem.

### Step 3 — Identify Code Smells by Category

Work through each category. Note every finding with: smell name, location (line range), severity, and the specific symptom.

#### Long Function / Long Method
- Flag functions exceeding 30 lines.
- Look for comment blocks that divide a long function into labeled sections — each section is a candidate for extraction.
- Look for functions that do setup, execution, AND cleanup in the same body.

#### Deep Nesting
- Flag any code path nested more than 3 levels (if inside if inside for inside try).
- Common patterns: nested callbacks, nested conditionals for validation, nested loops.
- Refactoring: early return / guard clause, extract method, promise chaining, inversion of conditions.

#### God Object / God Class
- Flag classes with more than 15 methods or 10 fields.
- Flag classes whose methods span unrelated concerns (e.g., a `User` class that also handles email sending and payment processing).
- Refactoring: extract class, move method, introduce service objects.

#### Duplicate Code
- Flag verbatim or near-verbatim code blocks appearing more than once.
- Flag structural duplication: two switch statements with the same case structure.
- Flag parallel class hierarchies (two class trees that must be updated in lockstep).
- Refactoring: extract function, template method pattern, parameterize by function.

#### Primitive Obsession
- Flag function signatures with 3+ consecutive primitives of the same type (e.g., `(string, string, string, boolean)`).
- Flag repeated field access patterns like `user.firstName + ' ' + user.lastName` scattered across the codebase.
- Flag raw strings used as type-discriminating values (e.g., `status === 'active'` repeated everywhere).
- Refactoring: introduce value object, replace primitive with object, define enum/const map.

#### Feature Envy
- Flag methods that call methods or access fields on another object more than on their own object.
- Symptom: a method in `OrderService` that does most of its work on `Cart` fields.
- Refactoring: move method to the class it is "envious" of.

#### Long Parameter List
- Flag functions with more than 4 parameters.
- Especially flag when parameters are positional primitives (easy to pass in wrong order).
- Refactoring: introduce parameter object, use builder pattern, use named options object.

#### Shotgun Surgery
- Flag when a single logical change requires modifications in many unrelated files.
- Symptom: adding a new field requires changes in the model, serializer, DTO, validator, and mapper separately.
- Refactoring: inline class, combine related logic, introduce a single transformation boundary.

#### Inappropriate Intimacy
- Flag when a class or module directly accesses the internal fields or private methods of another.
- Refactoring: move field, replace direct access with a method on the owning class.

#### Dead Code
- Flag unreachable code after unconditional returns.
- Flag commented-out code blocks (remove them; version control tracks history).
- Flag exported functions with no internal or external callers (requires user confirmation before suggesting removal).

#### Magic Numbers and Strings
- Flag literal numbers or strings whose meaning is not self-evident from context.
- Refactoring: extract constant with a descriptive name.

### Step 4 — Assign Severity

For each finding:

- **CRITICAL**: The smell actively causes bugs, makes the code nearly impossible to modify safely, or represents a fundamental design error.
- **HIGH**: The smell significantly increases change cost or defect rate. Address in current sprint.
- **MEDIUM**: The smell increases friction but is manageable short-term. Add to backlog.
- **LOW**: Minor readability improvement. Address opportunistically.

### Step 5 — Compose the Refactoring Report

```
## Refactoring Advisory — [filename]

### Overall Assessment
[2-3 sentence summary: is this file a localized issue or a symptom of broader design
problems? What is the single most important change to make?]

### Findings

#### [SEVERITY] [Smell Name] — [function/class name, line range]
**Symptom:** [What you observed]
**Why it matters:** [Concrete consequence: harder to test, prone to this type of bug, etc.]
**Refactoring:** [Named technique]

Before:
```[language]
[Minimal before snippet showing the problem]
```

After:
```[language]
[Minimal after snippet showing the fix]
```

---

[Repeat for each finding]

### Priority Refactoring Plan
[Ordered list of changes. Earlier items must not break later ones.]
1. [First change — why this unblocks everything else]
2. ...
```

### Step 6 — Ask Before Applying

After presenting the report, ask:

> "Which of these would you like me to apply? I can tackle them one at a time or in a batch. I recommend starting with [#1] because [reason]."

Do not use `Edit` or `Write` tools until the user confirms which refactorings to apply.

When applying a refactoring:
1. Apply it in isolation — one smell at a time.
2. Preserve all existing behavior. Do not add features during refactoring.
3. After applying, note what tests (if any) should be run to verify no regression.

---

## Examples

### Example 1 — Long Function with Deep Nesting

**Input (JavaScript):**
```js
async function processOrder(order) {
  if (order) {
    if (order.items && order.items.length > 0) {
      let total = 0;
      for (const item of order.items) {
        if (item.inStock) {
          if (item.discount) {
            total += item.price * (1 - item.discount);
          } else {
            total += item.price;
          }
        }
      }
      if (total > 0) {
        const payment = await chargeCard(order.card, total);
        if (payment.success) {
          await sendConfirmationEmail(order.email);
          await updateInventory(order.items);
          return { success: true, orderId: payment.id };
        } else {
          return { success: false, error: payment.error };
        }
      }
    }
  }
  return { success: false, error: 'Invalid order' };
}
```

**Report excerpt:**

```
#### HIGH — Deep Nesting — processOrder() (lines 1–28)
**Symptom:** 5 levels of nesting. The main logic is buried under guard conditions.
**Why it matters:** Extremely difficult to trace execution paths. Adding a new
condition (e.g., fraud check) requires understanding all parent conditions first.
**Refactoring:** Guard Clause (early return) + Extract Function

Before: [as above]

After:
```js
async function processOrder(order) {
  if (!order?.items?.length) return { success: false, error: 'Invalid order' };

  const total = calculateTotal(order.items);
  if (total === 0) return { success: false, error: 'Invalid order' };

  const payment = await chargeCard(order.card, total);
  if (!payment.success) return { success: false, error: payment.error };

  await Promise.all([
    sendConfirmationEmail(order.email),
    updateInventory(order.items),
  ]);
  return { success: true, orderId: payment.id };
}

function calculateTotal(items) {
  return items
    .filter(item => item.inStock)
    .reduce((sum, item) => sum + item.price * (1 - (item.discount ?? 0)), 0);
}
```
```

### Example 2 — Primitive Obsession

**Input (TypeScript):**
```ts
function createUser(
  firstName: string,
  lastName: string,
  email: string,
  role: string,
  isActive: boolean
) { ... }
```

**Report excerpt:**
```
#### MEDIUM — Primitive Obsession / Long Parameter List — createUser() (line 1)
**Symptom:** 5 positional parameters, 3 of which are strings. Callers can easily
swap firstName and lastName, or lastName and email, with no type error.
**Refactoring:** Introduce Parameter Object

After:
```ts
interface CreateUserParams {
  firstName: string;
  lastName: string;
  email: string;
  role: 'admin' | 'member' | 'viewer'; // replace magic strings with union
  isActive: boolean;
}

function createUser(params: CreateUserParams) { ... }
```
```

---

## Anti-patterns

1. **Applying all refactorings in one large edit.** Refactorings interact. Extracting a function changes the scope of variables used in the extraction. Apply one refactoring at a time and verify it before proceeding to the next.

2. **Changing behavior while refactoring.** Refactoring is strictly behavior-preserving. If you notice a bug while refactoring, note it separately and do not fix it in the same change. Mixing bug fixes and refactoring makes both harder to review.

3. **Flagging everything as HIGH or CRITICAL.** Severity inflation causes engineers to ignore the report. A file with 10 CRITICAL findings is not being reviewed seriously. Reserve CRITICAL for smells that are actively causing or will soon cause defects.

4. **Suggesting patterns that are heavier than the problem warrants.** A function with a 5-item switch does not need the Strategy pattern. Match the solution complexity to the problem complexity. Over-engineering is itself a code smell.

5. **Reporting smells without explaining the concrete consequence.** "This function is too long" is not actionable. "This function is too long — the three distinct phases (validation, computation, persistence) cannot be tested independently, and a bug in persistence forces re-reading the entire function" is actionable.
