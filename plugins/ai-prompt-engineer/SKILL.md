---
name: ai-prompt-engineer
description: >
  Analyzes, restructures, and optimizes LLM prompts for token efficiency, output
  consistency, and model performance. Identifies waste (redundant instructions, vague
  language, missing constraints), rewrites prompts with structured output formats, and
  produces before/after token counts and quality assessments. Trigger phrases: "optimize
  prompt", "improve prompt", "reduce tokens", "prompt engineering", "rewrite prompt",
  "my prompt is too long", "prompt not working", "better system prompt", "structured
  output prompt", "make prompt cheaper".
version: 1.0.0
---

# Prompt Engineer

## When to Use

Activate this skill when the user:
- Pastes a prompt and says "optimize", "improve", "clean up", or "shorten" it
- Complains about inconsistent LLM outputs from an existing prompt
- Wants to add structured output (JSON, XML, markdown tables) to an existing prompt
- Asks to reduce token costs without degrading quality
- Says a prompt "isn't working" or produces wrong formats

Do NOT activate for general writing assistance unrelated to LLM prompts.

---

## Instructions

### Step 1 — Audit the incoming prompt

When a prompt is provided, perform a structured audit across five dimensions before writing a single word of the rewrite:

**A. Token waste analysis**
- Count approximate tokens (1 token ≈ 4 chars for English)
- Identify: repeated instructions, filler phrases ("Please", "I want you to", "As an AI"), restatements of the obvious, overly verbose examples
- Flag each waste category with a severity: HIGH / MEDIUM / LOW

**B. Ambiguity scan**
- Find any instruction where two reasonable LLMs would behave differently
- Flag: missing output format spec, no length constraint, undefined terms, contradictory rules

**C. Structure assessment**
- Is the prompt flat prose or structured with sections?
- Does it separate system context, task, constraints, and output format?
- Are few-shot examples present and correctly formatted?

**D. Output format evaluation**
- Does the prompt specify the exact output format (JSON schema, markdown, plain text)?
- If JSON is needed, is the schema provided inline?
- Are success and failure output formats both defined?

**E. Model-specific fit**
- If model is known (Claude, GPT-4, Gemini), flag any anti-patterns for that model
- Claude: prefers XML tags for structure, does not need "You are a helpful assistant"
- GPT-4: benefits from explicit "step by step" instructions, handles JSON mode
- Gemini: handles long context well, prefers bullet constraints over prose

Present the audit as a short table before the rewrite.

### Step 2 — Produce a rewrite plan

Before writing the optimized prompt, state:
1. What structural changes will be made (adding sections, XML tags, etc.)
2. What content will be removed and why
3. What will be added (output schema, constraints, etc.)
4. Estimated token reduction (as a percentage)

Get confirmation if the changes are significant (>30% restructuring).

### Step 3 — Rewrite the prompt

Apply these rules in order:

**Rule 1 — Structure before prose**
Use XML tags for Claude, markdown headers for most others:
```xml
<system>You are a billing support agent for Acme Corp.</system>
<task>Classify the customer message into one of the provided categories.</task>
<constraints>
- Respond only with the JSON object. No preamble.
- If the message fits multiple categories, pick the most specific one.
</constraints>
<output_format>
{"category": "<string>", "confidence": <0.0–1.0>, "reason": "<one sentence>"}
</output_format>
```

**Rule 2 — Cut filler ruthlessly**
Remove without replacement:
- "Please", "Kindly", "I would like you to"
- "As an AI language model"
- "Sure! Here is..." (output preamble instructions — just define the format)
- Restating the task in the examples section

**Rule 3 — Inline the output schema**
If JSON output is required, always include the exact schema in the prompt:
```
Output format (strict JSON, no markdown fences):
{
  "summary": "string, max 2 sentences",
  "sentiment": "positive | negative | neutral",
  "action_required": boolean
}
```

**Rule 4 — Add explicit constraints for every dimension that matters**
- Length: "Respond in 1-3 sentences." not "Be concise."
- Uncertainty: "If you are unsure, set confidence below 0.5 and explain in reason."
- Refusal: "If the input is not a billing question, return category: 'out_of_scope'."

**Rule 5 — Fix few-shot examples**
- Examples must use the exact same output format as defined in the prompt
- Include at least one edge case example, not just the happy path
- Remove examples that demonstrate the wrong behavior (a common source of bugs)

**Rule 6 — Remove model persona unless required**
"You are a helpful, harmless, honest AI" wastes tokens. Only include persona if it
materially changes the output (e.g., "You are a senior security engineer reviewing
code for vulnerabilities" — the domain expertise does change behavior).

### Step 4 — Deliver the comparison report

Always deliver:

```
## Audit Results

| Dimension        | Issue Found                            | Severity |
|------------------|----------------------------------------|----------|
| Token waste      | 12 filler phrases removed              | HIGH     |
| Ambiguity        | No output format specified             | HIGH     |
| Structure        | Flat prose → XML sections added        | MEDIUM   |
| Output format    | JSON schema added inline               | HIGH     |
| Model fit        | Removed "As an AI" persona             | LOW      |

## Token Count
Before: ~480 tokens
After:  ~210 tokens
Reduction: 56%

## Optimized Prompt
[full rewrite here]

## What Changed and Why
- [bullet list of each change with a one-line rationale]
```

### Step 5 — Offer test variations (optional)

If the user wants to test the rewrite, offer to generate 3 test inputs that cover:
1. The typical happy path
2. An edge case (empty input, ambiguous input)
3. An adversarial case (input designed to break format compliance)

Provide expected outputs for each so the user can validate the new prompt manually
or in an eval harness.

---

## Examples

### Example 1 — Verbose Classification Prompt

**Before (user-provided):**
```
You are a helpful AI assistant. I would like you to please help me classify customer
support tickets. When you receive a ticket, you should read it carefully and then
determine what category it belongs to. The categories are: billing, technical,
account, and general. Please make sure to be accurate and helpful. Thank you!
```
Approximate tokens: ~65

**Audit findings:**
- "You are a helpful AI assistant" — LOW waste, provides no domain signal
- "I would like you to please help me" — HIGH waste, pure filler
- "read it carefully" — LOW, implied
- No output format — HIGH ambiguity
- No examples — MEDIUM gap

**After (optimized):**
```xml
<task>Classify the customer support ticket into exactly one category.</task>
<categories>billing | technical | account | general</categories>
<constraints>
- Output only the category name. No explanation unless confidence is below 0.7.
- If confidence is below 0.7, output: {"category": "<name>", "note": "<reason for uncertainty>"}
</constraints>
```
Approximate tokens: ~45. Reduction: 31%.

---

### Example 2 — System Prompt for a Coding Assistant

**Before:**
```
You are an expert software engineer with 20 years of experience in all programming
languages. You are helpful, harmless, and honest. When I ask you a coding question,
please provide a detailed and thorough answer. Make sure to explain your code with
comments. Always write clean, maintainable code. If you don't know something, say so.
```
Approximate tokens: ~70

**After:**
```xml
<system>You are a senior software engineer. Prioritize correctness, then readability.</system>
<constraints>
- Include inline comments only for non-obvious logic.
- If multiple approaches exist, briefly state the tradeoff before showing code.
- If a question is ambiguous, ask one clarifying question before answering.
- Acknowledge uncertainty explicitly rather than guessing.
</constraints>
```
Approximate tokens: ~52. Reduction: 26%.
Key change: Replaced vague "clean, maintainable" with actionable constraints. Removed
the "20 years" persona — it provides no measurable output improvement.

---

### Example 3 — Adding Structured Output to an Existing Prompt

**User input:** "Here's my summarization prompt. I need the output to be JSON so I can parse it in my app."

**Before:**
```
Summarize the following article in 2-3 sentences, highlighting the main point and
any action items mentioned.
```

**After:**
```
Summarize the article below.

Output format (strict JSON, no markdown fences or extra text):
{
  "summary": "string, 2-3 sentences covering the main point",
  "action_items": ["string"] // empty array if none
}

Article:
{{article}}
```

**Why the `{{article}}` placeholder matters:** Always separate the static instruction
from the dynamic input with a clear template variable. This prevents prompt injection
where the article content accidentally looks like an instruction.

---

## Anti-patterns

1. **Optimizing for token count at the expense of output quality.** Token reduction is a
   means, not an end. If removing context causes the model to produce wrong category
   labels 20% more often, the optimization is a net loss. Always validate that the
   rewritten prompt matches or exceeds the original on a sample of real inputs before
   deploying.

2. **Using natural language to specify output format.** "Respond with a JSON object
   containing the category and confidence" is ambiguous — is the key "category" or
   "Category"? Is confidence a float or a string? Always provide a literal example
   of the exact expected JSON, including key names and value types.

3. **Adding more examples instead of fixing the base instruction.** Few-shot examples
   are a patch for an ambiguous instruction. If you need 5 examples to get consistent
   output, the instruction itself is broken. Fix the instruction first; then use 1-2
   examples only for edge cases.

4. **Over-constraining with contradictory rules.** "Always explain your reasoning" and
   "Respond in one sentence" cannot both be satisfied. Audit the constraint list for
   conflicts before finalizing. When rules conflict, make one explicitly subordinate:
   "Respond in one sentence unless the answer requires a code block."

5. **Treating prompt optimization as one-time work.** A prompt optimized for GPT-4-turbo
   in January 2024 may perform differently on GPT-4o or Claude 3.5. Mark each prompt
   with the model version it was tuned for and re-audit when the model changes. Include
   the model name and date in a comment at the top of production system prompts.
