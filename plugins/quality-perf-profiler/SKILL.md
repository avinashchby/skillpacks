---
name: quality-perf-profiler
description: >
  Identifies performance problems in code through static analysis, covering backend
  issues (N+1 queries, missing indexes, synchronous blocking, memory leaks, inefficient
  algorithms), frontend issues (unnecessary re-renders, expensive computed values,
  large bundle contributions, layout thrashing), and general issues (O(n²) loops,
  unbounded caches, inefficient data structures). Triggered by phrases like
  "performance", "optimize", "why is this slow", "perf audit", "this is too slow",
  "reduce latency", "memory leak", "bundle size", or "too many re-renders". Produces
  a prioritized report with estimated impact, root cause analysis, and concrete
  before/after fixes. Does not require profiler output — operates from code alone,
  though will ask for profiler data if available.
version: 1.0.0
---

# Quality Performance Profiler

## When to Use

Activate this skill when the user says any of the following:
- "performance"
- "optimize"
- "why is this slow"
- "perf audit"
- "this is too slow"
- "reduce latency"
- "memory leak"
- "bundle size"
- "too many re-renders"
- "high CPU usage"
- "this is timing out"
- "reduce database load"

Also activate when the user shares code and mentions slow response times, high memory usage, or user-facing lag, even without using specific performance vocabulary.

Do NOT activate for:
- Algorithmic design questions that haven't been implemented yet — discuss trade-offs first.
- Infrastructure-level performance (server sizing, CDN, load balancers) — those require ops context not available in code.

---

## Instructions

### Step 1 — Gather Context

1. Use `Read` to load the full file under review.
2. Ask the user (if not already stated): "Do you have profiler output, slow query logs, or browser performance traces? If so, share them — they will make the analysis more precise." Proceed without waiting if the user has already indicated they don't.
3. Identify the execution context:
   - **Backend**: language runtime, DB driver/ORM, caching layer
   - **Frontend**: framework (React, Vue, Svelte, etc.), bundler, rendering model (SSR, CSR, hydration)
   - **General**: scripting, data pipeline, CLI tool
4. Identify the hot path — the code executed most frequently or under the highest load.

### Step 2 — Backend Performance Analysis

#### N+1 Query Detection
- Flag any loop that contains a DB query (ORM call, raw SQL, Redis call).
- Flag ORM lazy-loading patterns: accessing a relationship field inside a loop without an explicit eager-load.
- Flag sequential `await`s on DB calls inside a loop that could be batched.

Remediation patterns:
- SQL: Use a JOIN or a single `WHERE id IN (...)` query.
- ORM: Use `include`/`eager_load`/`prefetch_related` / `with` (depends on ORM).
- Multiple independent async calls: Use `Promise.all` / `asyncio.gather` / goroutine fan-out.

#### Missing or Inefficient Indexes
- Flag `WHERE` clauses on columns that are likely unindexed (foreign keys, email, status fields, timestamps used for sorting).
- Flag `ORDER BY` without a corresponding index on the sort column.
- Flag `LIKE '%prefix%'` patterns (leading wildcard defeats index).
- Flag `SELECT *` — retrieves unused columns, increases I/O and network transfer.
- Note: confirm with `EXPLAIN` output if available.

#### Synchronous Blocking in Async Contexts
- **Node.js**: Flag `fs.readFileSync`, `crypto.pbkdf2Sync`, `JSON.parse` on large payloads, CPU-heavy loops on the main thread.
- **Python asyncio**: Flag synchronous I/O inside `async def` (e.g., `requests.get`, `open()`, `time.sleep`). Should use `aiohttp`, `aiofiles`, `asyncio.sleep`.
- **Go**: Flag blocking syscalls outside goroutines in hot paths.

#### Memory Leaks
- Flag unbounded data structures that grow on every request: module-level arrays/maps that are appended to but never cleared.
- Flag event listeners registered inside request handlers without corresponding `removeEventListener`.
- Flag closures that capture large objects and are stored in long-lived caches.
- Flag stream operations that buffer entire payloads: `.text()`, `.json()` on large response bodies when streaming would suffice.
- Flag timers (`setInterval`, background goroutines) that reference objects and are never cancelled.

#### Connection and Resource Management
- Flag DB connections or HTTP clients created per-request instead of using a pool.
- Flag file handles opened without guaranteed close (missing `finally`, `defer`, `with`, `using`).
- Flag missing connection pool sizing for high-concurrency scenarios.

#### Inefficient Serialization
- Flag repeated JSON serialization of the same object in a loop.
- Flag serialization of full objects when only a subset of fields is needed.

### Step 3 — Frontend Performance Analysis

#### Unnecessary Re-renders (React / Vue / similar)
- Flag components that receive object/array props created inline in the parent render: `<Comp items={[1,2,3]} />` creates a new array on every render.
- Flag expensive computations inside render functions without memoization (`useMemo`, `computed`).
- Flag event handler functions created inline without `useCallback` when passed to memoized children.
- Flag context providers whose value object is created inline — every consumer re-renders on every provider render.
- Flag missing `key` props on list items, or keys that use array index (causes full re-render on reorder).
- Flag components that subscribe to a large store slice but only use a small part of it.

#### Expensive Computed Values
- Flag regex patterns compiled inside render (move outside component or memoize).
- Flag `Date` parsing or number formatting inside render loops without memoization.
- Flag `.filter().map().reduce()` chains run on every render on large arrays — suggest `useMemo`.

#### Layout Thrashing
- Flag interleaved DOM reads and writes in JavaScript: reading `offsetHeight`, then writing style, then reading again — each read forces a synchronous layout.
- Remediation: batch all reads first, then all writes. Use `requestAnimationFrame` for visual updates.

#### Bundle Size
- Flag default imports from large libraries when named imports would enable tree-shaking: `import _ from 'lodash'` vs `import { debounce } from 'lodash'`.
- Flag synchronous imports of code used only after user interaction — suggest `React.lazy` / dynamic `import()`.
- Flag large assets (images, fonts) referenced without optimization hints.
- Flag polyfills included for browsers already covered by the browserslist config.

### Step 4 — General / Algorithmic Analysis

#### Algorithm Complexity
- Flag nested loops over the same collection: O(n²). Identify whether a Map/Set lookup would reduce to O(n).
- Flag repeated `.find()`, `.indexOf()`, `.includes()` inside loops: O(n) per iteration = O(n²) total.
- Flag sort called on every render or every request on data that doesn't change — sort once and cache.
- Flag recursive functions without memoization on overlapping subproblems.

#### Data Structure Selection
- Flag using an array where lookups by key are frequent → use Map or object.
- Flag using a Map where a Set would suffice (only membership check needed).
- Flag appending to the front of an array in a loop → use a deque or reverse once at the end.

#### String and I/O Operations
- Flag string concatenation in a loop: `str += chunk` is O(n²). Use array join or a builder.
- Flag repeated parsing of the same config file, template, or regex on every call.

### Step 5 — Compose the Performance Report

```
## Performance Audit — [filename]

### Severity Scale
- P0 (Critical): Will cause outages, OOM crashes, or >10× slowdown under load.
- P1 (High): Significant latency or memory impact at production scale.
- P2 (Medium): Noticeable performance cost, worth fixing in next sprint.
- P3 (Low): Minor optimization, fix opportunistically.

### Findings

#### [P-level] [Issue Type] — [function/component, line range]
**Root Cause:** [What is actually happening and why it is slow]
**Impact:** [Estimated: e.g., "1 DB query per item → 100 queries for a 100-item list"]
**Fix:**

Before:
```[language]
[Minimal code showing the problem]
```

After:
```[language]
[Minimal code showing the fix]
```

**Verification:** [How to confirm the fix worked: EXPLAIN output, React DevTools, etc.]

---

### Estimated Impact Summary
| Finding | Type | Estimated Gain |
|---------|------|---------------|
| ...     | N+1  | ~99 fewer queries per page load |

### Recommended Fix Order
[Explain why this order — P0s first, but also note if one fix is a prerequisite for another]
```

---

## Examples

### Example 1 — N+1 Query in Node.js / Sequelize

**Input:**
```js
async function getOrdersWithItems(userId) {
  const orders = await Order.findAll({ where: { userId } });
  for (const order of orders) {
    order.items = await OrderItem.findAll({ where: { orderId: order.id } });
  }
  return orders;
}
```

**Report excerpt:**
```
#### P0 — N+1 Query — getOrdersWithItems() (lines 1–6)
**Root Cause:** `OrderItem.findAll` is called inside a loop — one query per order.
**Impact:** For a user with 50 orders, this executes 51 queries (1 + 50).
  Under 100 concurrent users, that is 5,100 queries per second from this endpoint alone.
**Fix:**

Before: [as above]

After:
```js
async function getOrdersWithItems(userId) {
  return Order.findAll({
    where: { userId },
    include: [{ model: OrderItem, as: 'items' }],
  });
}
```
// Single JOIN query regardless of order count.

**Verification:** Enable Sequelize query logging and confirm exactly 1 query is emitted.
```

### Example 2 — Unnecessary Re-render in React

**Input:**
```jsx
function UserList({ users }) {
  return (
    <ul>
      {users.map((user, index) => (
        <UserCard
          key={index}
          user={user}
          onDelete={() => deleteUser(user.id)}
        />
      ))}
    </ul>
  );
}
```

**Report excerpt:**
```
#### P2 — Unnecessary Re-render + Unstable Keys — UserList (lines 1–10)

**Root Cause (1):** `key={index}` means React cannot distinguish reordered items.
  On any list mutation (delete, reorder), all cards after the mutation point re-render
  and lose local state (e.g., open dropdowns, focus position).
**Fix (1):** Use `key={user.id}`.

**Root Cause (2):** `onDelete={() => deleteUser(user.id)}` creates a new function
  reference on every UserList render. If UserCard is wrapped in React.memo,
  the memo is defeated on every parent render.
**Fix (2):** Wrap with `useCallback` or pass `user.id` as a prop and define the
  handler inside UserCard.

After:
```jsx
const UserList = React.memo(function UserList({ users, onDelete }) {
  return (
    <ul>
      {users.map(user => (
        <UserCard key={user.id} user={user} onDelete={onDelete} />
      ))}
    </ul>
  );
});
```
```

### Example 3 — O(n²) Algorithm

**Input (Python):**
```python
def find_duplicates(items: list[str]) -> list[str]:
    duplicates = []
    for i, item in enumerate(items):
        for j, other in enumerate(items):
            if i != j and item == other and item not in duplicates:
                duplicates.append(item)
    return duplicates
```

**Report excerpt:**
```
#### P1 — O(n²) Algorithm — find_duplicates() (lines 1–7)
**Root Cause:** Nested loop compares every pair. For 10,000 items: 100M comparisons.
  The inner `item not in duplicates` is also O(n), making this O(n³) in the worst case.
**Fix:** Use a Counter (single pass, O(n)):

```python
from collections import Counter

def find_duplicates(items: list[str]) -> list[str]:
    counts = Counter(items)
    return [item for item, count in counts.items() if count > 1]
```
```

---

## Anti-patterns

1. **Optimizing without measuring first.** Always ask the user if profiler output is available before recommending fixes. Static analysis can identify candidates, but actual profiler data shows which candidates are on the hot path. Optimizing a cold path is wasted effort and adds complexity.

2. **Recommending memoization for everything.** `useMemo` and `useCallback` have overhead. They are beneficial when the memoized computation is genuinely expensive or when referential stability is needed for a downstream `React.memo`. Wrapping every value in `useMemo` is itself a performance anti-pattern.

3. **Suggesting premature parallelism.** `Promise.all` over 500 DB queries is not a fix for an N+1 — it fires 500 parallel queries and will overwhelm the connection pool. The fix is to batch into a single query. Parallelism is appropriate for independent external calls (2–5 at a time), not for unbounded fan-out.

4. **Reporting estimated gains with false precision.** "This will make the page 73.4% faster" is not credible from static analysis alone. Use ranges and qualifiers: "~50–100 fewer queries per request", "eliminates the dominant O(n²) term for n > 1000". Overstating impact undermines trust in the report.

5. **Suggesting index additions without noting the write overhead trade-off.** Every index speeds reads and slows writes. For write-heavy tables, always note: "Confirm this table is not write-heavy before adding the index. If it is, evaluate a covering index or query rewrite instead."
