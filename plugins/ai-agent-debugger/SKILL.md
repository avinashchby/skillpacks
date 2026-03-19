---
name: ai-agent-debugger
description: >
  Diagnoses misbehaving LLM agent workflows by analyzing tool call traces, identifying
  infinite loops, detecting hallucinated tool arguments, and producing actionable fixes.
  Works with LangChain, LangGraph, CrewAI, AutoGen, and custom agent frameworks.
  Produces a structured debug report with root-cause analysis and a minimal reproduction
  case. Trigger phrases: "debug agent", "agent stuck", "agent loop", "trace agent",
  "agent not working", "agent keeps calling", "agent won't stop", "tool call loop",
  "agent failed", "why is my agent", "agent crashes".
version: 1.0.0
---

# Agent Debugger

## When to Use

Activate this skill when the user:
- Pastes an agent trace, log output, or error and asks why the agent failed
- Describes an agent that runs but never terminates or calls the same tool repeatedly
- Reports an agent that ignores tool results and re-hallucinates the same arguments
- Has an agent that crashes partway through a multi-step task
- Wants to add observability/breakpoints to a working agent before it breaks in production

Do NOT activate for general Python/TypeScript debugging unrelated to agent control flow.

---

## Instructions

### Step 1 — Collect the trace

Ask the user to provide as much of the following as possible:

1. **Tool call log** — The sequence of `(tool_name, input_args, output)` tuples. This
   is the primary diagnostic artifact.
2. **Framework and version** — LangChain, LangGraph, CrewAI, AutoGen, custom, etc.
3. **Model and version** — GPT-4o, Claude 3.5 Sonnet, etc.
4. **Agent configuration** — System prompt, available tools, max_iterations setting
5. **Error output** — Full stack trace if the agent crashed
6. **Expected behavior** — What the agent was supposed to do

If the user cannot provide a trace, ask them to add logging first (see Step 6).

### Step 2 — Parse the trace into a structured sequence

Before diagnosing, normalize the trace into a canonical table:

```
Step | Tool Called         | Key Inputs                    | Output Summary         | Issue?
-----|---------------------|-------------------------------|------------------------|-------
1    | search_web          | query="capital of France"     | "Paris"                | —
2    | search_web          | query="capital of France"     | "Paris"                | LOOP
3    | search_web          | query="capital of France"     | "Paris"                | LOOP
4    | <finish>            | —                             | —                      | NEVER REACHED
```

Look for these patterns during normalization:
- **Repeated identical calls** — same tool, same args, ≥2 times
- **Repeated calls with incremental drift** — same tool, slightly mutated args (sign of
  a confused search loop)
- **Tool result ignored** — the model calls a tool, receives output, then calls the
  same tool with different args that don't incorporate the prior result
- **Premature termination** — agent calls `<finish>` before all sub-tasks are complete
- **Tool argument hallucination** — args reference IDs, names, or values never seen
  in any prior tool output

### Step 3 — Identify the root cause category

Classify the failure into one of these root causes:

**RC-1: Missing termination condition**
The agent has no clear signal for when to stop. The system prompt says "research X"
but never says "stop when you have 3 sources" or "call finish_task when done."
Fix: Add explicit completion criteria to the system prompt.

**RC-2: Tool result not injected into context**
The framework is configured incorrectly and tool outputs are being dropped before the
next LLM call. The model is reasoning without the data it collected.
Fix: Verify the message history includes `role: tool` entries after each call.

**RC-3: Ambiguous tool selection**
Two tools have overlapping descriptions and the model oscillates between them.
Fix: Rewrite tool descriptions to be mutually exclusive; add "Use this tool ONLY when
X. Do not use it for Y (use <other_tool> instead)."

**RC-4: Context window overflow mid-task**
Long tasks accumulate tool outputs that exceed the model's context window. The model
starts hallucinating earlier tool results it can no longer see.
Fix: Add a summarization step every N tool calls; use streaming context compression.

**RC-5: Hallucinated arguments**
The model generates IDs, file paths, or values that were never in any tool output.
Usually caused by an under-specified tool schema that allows the model to invent values.
Fix: Constrain argument types in the tool schema; add validation that rejects synthetic IDs.

**RC-6: Infinite retry on recoverable error**
A tool returns an error and the agent retries with identical inputs indefinitely.
Fix: Implement exponential backoff with a max retry count; add a dead-letter step.

**RC-7: Planner/executor desync (multi-agent)**
In multi-agent setups, the planner assigns a task the executor cannot complete with
its available tools, causing the executor to loop trying to work around the gap.
Fix: Validate the task decomposition against available tools before dispatch.

### Step 4 — Produce the debug report

Structure the report as follows:

```markdown
## Agent Debug Report

### Trace Summary
[normalized step table from Step 2]

### Root Cause
**RC-X: [Name]**
[2-3 sentence explanation referencing specific steps in the trace]

### Evidence
- Step N: [specific observation that confirms the root cause]
- Step M: [secondary evidence]

### Fix

#### Immediate (system prompt change)
[Paste the exact system prompt addition or replacement]

#### Code fix (if framework misconfiguration)
[Minimal code diff]

#### Validation
How to confirm the fix worked: [specific thing to check in the next trace]
```

### Step 5 — Provide the code fix

For the most common frameworks:

**LangGraph — adding max_iterations guard:**
```python
from langgraph.graph import StateGraph, END

def should_continue(state: AgentState) -> str:
    if state["iterations"] >= 15:
        # Force termination — prevents runaway loops
        return "end"
    if state.get("final_answer"):
        return "end"
    return "continue"

graph.add_conditional_edges("agent", should_continue, {
    "continue": "tools",
    "end": END,
})
```

**LangChain — adding tool result validation:**
```python
from langchain.tools import tool
from pydantic import BaseModel, field_validator

class SearchInput(BaseModel):
    query: str
    max_results: int = 5

    @field_validator("query")
    @classmethod
    def query_not_empty(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("query cannot be empty — agent is likely looping")
        return v

@tool(args_schema=SearchInput)
def search_web(query: str, max_results: int = 5) -> str:
    """Search the web for current information."""
    ...
```

**Adding trace logging to any agent (framework-agnostic):**
```python
import json
from datetime import datetime

class AgentTracer:
    def __init__(self):
        self.steps: list[dict] = []
        self._seen_calls: set[tuple] = set()

    def record(self, tool: str, inputs: dict, output: str) -> None:
        call_key = (tool, json.dumps(inputs, sort_keys=True))
        is_duplicate = call_key in self._seen_calls
        self._seen_calls.add(call_key)
        self.steps.append({
            "step": len(self.steps) + 1,
            "tool": tool,
            "inputs": inputs,
            "output_preview": output[:200],
            "timestamp": datetime.utcnow().isoformat(),
            "is_duplicate": is_duplicate,
        })
        if is_duplicate:
            raise RuntimeError(
                f"LOOP DETECTED: tool '{tool}' called with identical args at step {len(self.steps)}. "
                f"Previous call at step {self._find_first(call_key)}. Halting agent."
            )

    def _find_first(self, key: tuple) -> int:
        for i, s in enumerate(self.steps[:-1]):
            if (s["tool"], json.dumps(s["inputs"], sort_keys=True)) == key:
                return i + 1
        return -1
```

### Step 6 — Adding observability when no trace exists

If the user has no trace, provide them with framework-specific logging setup before
debugging can proceed:

**LangChain:**
```python
from langchain.callbacks import StdOutCallbackHandler
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    callbacks=[StdOutCallbackHandler()],
    verbose=True,
    max_iterations=20,         # always set this
    handle_parsing_errors=True,
)
```

**LangGraph:**
```python
# Stream intermediate states for inspection
for event in graph.stream(inputs, config={"recursion_limit": 25}):
    for node, output in event.items():
        print(f"[{node}] {json.dumps(output, default=str)[:300]}")
```

**AutoGen:**
```python
config = {
    "timeout": 120,
    "seed": 42,
}
# Enable conversation history logging
assistant.initiate_chat(
    user_proxy,
    message=task,
    max_turns=20,   # always cap turns
)
print(assistant.chat_messages)
```

---

## Examples

### Example 1 — Identical Tool Call Loop

**User provides this trace:**
```
Step 1: search_web(query="OpenAI GPT-5 release date") → "No results found"
Step 2: search_web(query="OpenAI GPT-5 release date") → "No results found"
Step 3: search_web(query="OpenAI GPT-5 release date") → "No results found"
[agent exceeds max_iterations=10 and crashes]
```

**Root cause:** RC-1 + RC-6. The agent has no fallback when search returns empty,
and retries identically without a backoff or alternate strategy.

**Fix (system prompt addition):**
```
If a search returns no results, try ONE alternative query reformulation.
If the reformulated search also returns no results, respond to the user explaining
that the information is not available in your search index and suggest they check
the source directly.
```

**Fix (tool-level):**
```python
@tool
def search_web(query: str, attempt: int = 1) -> str:
    """Search the web. attempt tracks retry count (max 2)."""
    if attempt > 2:
        return "SEARCH_EXHAUSTED: No results after 2 attempts. Do not retry."
    result = _do_search(query)
    if not result:
        return f"No results for: {query}. Suggest reformulating. (attempt {attempt}/2)"
    return result
```

---

### Example 2 — Multi-Agent Planner/Executor Desync

**User describes:** "My planner agent decomposes a task and sends 'scrape the competitor
pricing page' to my executor, but the executor only has tools for reading local files.
It keeps trying file_read with random paths."

**Root cause:** RC-7. The planner doesn't know the executor's tool inventory.

**Fix:**
```python
EXECUTOR_TOOL_MANIFEST = """
Available executor tools:
- file_read(path: str) → read a local file
- file_write(path: str, content: str) → write a local file
- run_python(code: str) → execute Python code in sandbox

NOT available: web scraping, HTTP requests, browser automation.
"""

planner_system_prompt = f"""
You decompose tasks into subtasks for an executor agent.
CRITICAL: Only assign tasks the executor can complete with its available tools.
{EXECUTOR_TOOL_MANIFEST}
If a task requires tools not listed above, mark it as BLOCKED and explain what
capability is missing instead of delegating it.
"""
```

---

### Example 3 — Context Window Overflow

**User reports:** "My research agent works fine for small tasks but after 30 minutes it
starts repeating searches it already did and contradicting itself."

**Root cause:** RC-4. Tool outputs accumulate, and the model loses access to earlier steps.

**Fix — add periodic summarization:**
```python
async def compress_history(messages: list[dict], llm) -> list[dict]:
    """Summarize tool results older than the last 5 steps to save context."""
    if len(messages) < 20:
        return messages

    old_messages = messages[:-10]
    recent_messages = messages[-10:]

    summary_prompt = (
        "Summarize the following agent research steps into a compact fact list. "
        "Preserve all specific facts, numbers, and source names. "
        "Discard reasoning and intermediate thoughts.\n\n"
        + "\n".join(str(m) for m in old_messages)
    )
    summary = await llm.ainvoke(summary_prompt)
    return [
        {"role": "system", "content": f"Prior research summary:\n{summary.content}"},
        *recent_messages,
    ]
```

---

## Anti-patterns

1. **Debugging without a trace.** Asking "why is my agent looping?" without a tool call
   log is like debugging a crash without a stack trace. Always instrument the agent to
   emit `(tool, inputs, output)` tuples before attempting any diagnosis. Guessing at
   root cause from a description alone produces wrong fixes.

2. **Setting `max_iterations` to a high number to "fix" loops.** Raising the limit from
   10 to 100 does not fix a loop — it just makes it 10x more expensive before it crashes.
   Always fix the underlying cause (missing termination condition, bad tool schema, etc.)
   and keep `max_iterations` as a last-resort safety net, not a primary fix.

3. **Adding more tools to fix a stuck agent.** When an agent can't complete a task, the
   instinct is to add more tools. Usually the agent is stuck because the existing tools
   have ambiguous descriptions or return formats the model can't parse. Fix the existing
   tools before adding new ones.

4. **Ignoring the `handle_parsing_errors` flag.** In LangChain, if the model outputs
   malformed JSON, the default behavior silently retries, consuming budget. Always set
   `handle_parsing_errors=True` and log parsing failures — they are a leading indicator
   of prompt or output format issues.

5. **Multi-agent debugging without isolating each agent.** In a planner + executor setup,
   test the planner in isolation (feed it a fixed task, check its plan) and the executor
   in isolation (feed it a fixed plan step, check its tool calls) before testing the
   integrated system. Debugging both at once makes it impossible to localize the fault.
