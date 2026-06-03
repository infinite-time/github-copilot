---
marp: true
theme: default
paginate: true
title: "GitHub Copilot: Tokens & Model Selection"
style: |
  section {
    font-size: 21px;
    padding: 48px 56px;
    justify-content: flex-start;
  }
  section.lead { justify-content: center; }
  h1 { font-size: 38px; }
  h2 { font-size: 30px; margin-bottom: 0.4em; }
  p, li { line-height: 1.35; margin: 0.25em 0; }
  ul, ol { margin: 0.3em 0; }
  pre { font-size: 0.72em; line-height: 1.3; margin: 0.4em 0; }
  code { font-size: 0.95em; }
  table { font-size: 0.82em; }
  th, td { padding: 3px 8px; }
  blockquote { font-size: 0.9em; margin: 0.4em 0; }
---

# GitHub Copilot
## Tokens, Token Sizing & Model Selection

*Why your model choice now shows up directly on the bill.*

---

## Why this matters now

As of **June 1, 2026**, GitHub Copilot moved from *premium requests* to **usage-based billing**.

- Every interaction is billed by **token consumption** — input, output, and cached tokens.
- Token totals are converted into **GitHub AI Credits**, where **1 AI credit = $0.01 USD**.
- Cost of any interaction = **(which model) × (how many tokens)**.

> Code completions / next-edit suggestions are **not** billed in credits — they stay unlimited on paid plans. Billing applies to **Chat, agents, code review, CLI, etc.**

This session is about understanding the two levers you control: **tokens** and **model**.

---

## Agenda

1. What is a token? What is "token size"?
2. Code examples — how token counts vary
3. The three token buckets: input, output, cached
4. Context windows
5. How model selection changes token cost
6. Model categories & current pricing
7. Which model for which task (with code + output differences)
8. Decision trees: GPT family & Claude family
9. Practical ways to cut token usage

---

## What is a token?

A **token** is the unit a language model reads and writes. It is *not* a word and *not* a character — it's a chunk produced by the model's tokenizer (typically sub-word "byte-pair encoding").

Rough rules of thumb for English text:

| Unit | Approx. tokens |
|------|----------------|
| 1 token | ~4 characters |
| 1 token | ~0.75 words |
| 100 tokens | ~75 words |
| 1,000 tokens | ~750 words (~1.5 pages) |

**Code is denser than prose** — punctuation, indentation, camelCase, and operators each tend to cost extra tokens, so code often runs closer to **3 characters per token**.

---

## "Token size" — two meanings to keep separate

People say "token size" to mean different things. Be precise:

1. **Token count of a payload** — *how many tokens* a given prompt, file, or response is.
   *e.g. "This 200-line file is ~2,400 tokens."*

2. **Context window size** — the *maximum* tokens a model can hold in one request (input + output combined).
   *e.g. "This model has a 200K-token context window."*

This deck mostly means **(1)** when we say "how big is this in tokens," and calls out **(2)** explicitly as "context window."

---

## Code example: how token count varies (1/2)

The same logic in different forms produces very different token counts.

**A — Tiny snippet (~12 tokens)**
```python
def add(a, b):
    return a + b
```

**B — Same idea, verbose + typed + docstring (~70–80 tokens)**
```python
def add(first_value: int, second_value: int) -> int:
    """Return the sum of two integers.

    Args:
        first_value: The first addend.
        second_value: The second addend.
    """
    return first_value + second_value
```

Naming, type hints, and docstrings are *real tokens* in every request.

---

## Code example: how token count varies (2/2)

**C — A whole file pasted as context (~1,800–2,500 tokens)**
```text
# 250 lines of a service module, imports, comments, and tests
# pasted into chat as "here's my file, fix the bug"
```

**D — A repo-wide agent task (tens of thousands of tokens)**
```text
"Refactor the auth module and update all call sites"
→ agent reads many files, plans, edits, re-reads → 30k–100k+ tokens
```

**Takeaway:** the *prompt you type* is rarely the expensive part. The **files, context, and agent tool-loops** dominate token usage.

---

## Counting tokens yourself

You can measure before you send. Example using OpenAI's `tiktoken`:

```python
import tiktoken

enc = tiktoken.get_encoding("o200k_base")  # GPT-5 family encoding

snippet = """def add(a, b):
    return a + b
"""

tokens = enc.encode(snippet)
print(len(tokens))        # ~12
print(tokens[:5])         # the integer token IDs
```

Different model families use **different tokenizers**, so the *same* text can have slightly different counts on GPT vs Claude vs Gemini. Treat counts as **estimates**, not exact billing.

---

## The three token buckets

Every billable interaction is metered in up to three categories:

| Bucket | What it is | Relative cost |
|--------|-----------|---------------|
| **Input** | Everything sent *to* the model: your prompt, files, chat history, system instructions | Base rate |
| **Output** | Everything the model *generates*: code, explanations, tool calls | **Usually 4–6× the input rate** |
| **Cached input** | Repeated context the model reuses across turns | **Heavily discounted (~10× cheaper)** |

**Implication:** long, chatty answers cost more than long prompts. Asking for "just the diff" instead of "the whole file rewritten" directly cuts the most expensive bucket.

---

## Output is the expensive bucket — example

Using **GPT-5.4** rates ($2.50 input / $15.00 output per 1M tokens):

| Scenario | Input | Output | Cost |
|----------|-------|--------|------|
| Ask: "fix this 2K-token file, return only the changed lines" | 2,000 | 300 | $0.0095 |
| Ask: "fix this 2K-token file, rewrite the whole thing" | 2,000 | 2,000 | $0.035 |

Same input, same fix — the second costs **~3.7× more** purely because you asked for more **output tokens**.

> Prompting for minimal, targeted output is one of the cheapest optimizations available.

---

## Context windows

The **context window** is the hard ceiling on tokens per request (input + output together).

- Larger windows let you paste more code / more files, but **you pay for every token you put in there**.
- Some models apply a **long-context surcharge** past a threshold:
  - GPT-5.4 standard pricing applies to prompts **≤ 272K tokens**.
  - Gemini Pro tiers' listed pricing applies to prompts **≤ 200K tokens**.

**Mental model:** the context window is the size of the desk. Tokens are what you pile on it. A bigger desk doesn't make the paper free.

---

## How model selection changes token cost

Two requests with **identical** token counts can cost very differently depending on the model.

Example: an interaction consuming **10K input + 2K output tokens**.

| Model | Category | Approx. cost |
|-------|----------|--------------|
| GPT-5 mini | Lightweight | ~$0.0065 |
| GPT-4.1 | Versatile | ~$0.036 |
| Claude Sonnet 4.6 | Versatile | ~$0.060 |
| Claude Opus 4.8 | Powerful | ~$0.100 |
| GPT-5.5 | Powerful | ~$0.110 |

*Same work, ~17× cost spread.* The model is the **biggest single multiplier** on your bill.

---

## Model categories

GitHub groups models into three tiers. This is the most useful lens for choosing:

| Category | Built for | Cost | Examples |
|----------|-----------|------|----------|
| **Lightweight** | Fast, cheap, high-volume, simple tasks | $ | GPT-5 mini, GPT-5.4 nano/mini, Gemini Flash, Claude Haiku 4.5 |
| **Versatile** | The everyday default — good balance | $$ | GPT-4.1, GPT-5.2, Claude Sonnet 4.6, Raptor mini |
| **Powerful** | Hard reasoning, large refactors, agents | $$$ | GPT-5.5, GPT-5.x-Codex, Claude Opus 4.x, Gemini Pro |

**Auto model selection** lets Copilot pick a tier based on prompt complexity — a safe default if you don't want to think about it per-prompt.

---

## Current per-token pricing — OpenAI & Anthropic

Per **1M tokens** (1 AI credit = $0.01). Source: GitHub Copilot docs, June 2026.

**OpenAI**

| Model | Category | Input | Cached | Output |
|-------|----------|-------|--------|--------|
| GPT-5 mini *(included)* | Lightweight | $0.25 | $0.025 | $2.00 |
| GPT-4.1 *(included)* | Versatile | $2.00 | $0.50 | $8.00 |
| GPT-5.4 | Versatile | $2.50 | $0.25 | $15.00 |
| GPT-5.x-Codex | Powerful | $1.75 | $0.175 | $14.00 |
| GPT-5.5 | Powerful | $5.00 | $0.50 | $30.00 |

**Anthropic** *(also has a separate cache-write cost)*

| Model | Category | Input | Cached | Output |
|-------|----------|-------|--------|--------|
| Claude Haiku 4.5 | Versatile | $1.00 | $0.10 | $5.00 |
| Claude Sonnet 4.6 | Versatile | $3.00 | $0.30 | $15.00 |
| Claude Opus 4.8 | Powerful | $5.00 | $0.50 | $25.00 |

---

## Which model for which task

| Task | Good fit | Why |
|------|----------|-----|
| Autocomplete, boilerplate, simple Q&A | **Lightweight** (GPT-5 mini, Haiku 4.5) | Fast & cheap; quality difference is negligible here |
| Everyday coding, explain code, write tests | **Versatile** (GPT-4.1, Sonnet 4.6, GPT-5.2) | Best balance of quality and cost |
| Multi-file refactor, debugging tricky logic, architecture | **Powerful** (Opus 4.x, GPT-5.5) | Deeper reasoning earns its higher token rate |
| Long agentic / coding-agent runs | **Codex / Opus** | Strong tool-use & long-horizon planning |
| Large-context review across many files | **Pro / large-window models** | Mind the long-context surcharge |

Rule: **start cheap, escalate only when the cheap model fails.**

---

## Code example: a simple task (use Lightweight)

**Prompt:** *"Write a function to check if a string is a palindrome."*

A lightweight model nails this on the first try:

```python
def is_palindrome(s: str) -> bool:
    cleaned = "".join(c.lower() for c in s if c.isalnum())
    return cleaned == cleaned[::-1]
```

There is **no quality benefit** to spending a Powerful model here. The output is essentially identical — you'd just pay 10–15× more per token for the same answer.

✅ **GPT-5 mini / Claude Haiku 4.5** is the right call.

---

## Code example: a hard task (use Powerful)

**Prompt:** *"This concurrent cache has a race condition under high load. Find and fix it."*

```python
class Cache:
    def __init__(self):
        self._data = {}

    def get_or_compute(self, key, compute):
        if key not in self._data:          # check
            self._data[key] = compute()    # ...and set — not atomic!
        return self._data[key]
```

Here the model's **reasoning depth changes the answer** — this is where a Powerful model pays off.

---

## How output differs by model (illustrative)

Same race-condition prompt, representative behavior:

**Lightweight model** — pattern-matches "add a lock," may miss the double-check:
```python
import threading
class Cache:
    def __init__(self):
        self._data, self._lock = {}, threading.Lock()
    def get_or_compute(self, key, compute):
        with self._lock:                 # correct but serializes ALL reads
            if key not in self._data:
                self._data[key] = compute()
            return self._data[key]
```

**Powerful model** — reasons about contention and uses double-checked locking + per-key locks, *and explains the trade-off* (lock granularity, thundering herd). More output tokens, but a materially better answer.

> The difference shows up on **hard, ambiguous, or multi-step** problems — rarely on simple ones.

---

## How to choose a model — a repeatable process

Run every task through four questions, top to bottom. Stop at the first "yes."

1. **Is there one obvious, verifiable answer?** (boilerplate, rename, format, simple regex)
   → **Lightweight.** Spending more buys nothing.
2. **Is it everyday coding needing some judgment?** (a feature, a test suite, explain a file)
   → **Versatile.** This is your default — most work lands here.
3. **Does correctness hinge on multi-step reasoning or subtle edge cases?** (concurrency, security, tricky algorithms)
   → **Powerful.**
4. **Is it a long, autonomous, multi-file agent run?**
   → **Codex / Opus.**

**Cost check before you commit:** *"Would the cheaper tier plausibly get this right?"* If yes, start there and escalate only on failure.

---

## Selection in practice — five scenarios

| You're about to… | Pick | Why |
|------------------|------|-----|
| Rename a variable across one file | Lightweight | Mechanical; no reasoning needed |
| Add a REST endpoint to an existing controller | Versatile | Pattern-following with light judgment |
| Generate a full test suite with edge cases | Versatile → Powerful | Versatile usually fine; escalate if coverage is shallow |
| Debug an intermittent production race condition | Powerful | Needs hypothesis-forming across states |
| Migrate a 40-file module to a new API | Codex / Opus (agent) | Long-horizon, many files, tool use |
| Draft a commit message | Lightweight | Short, low-stakes, cheap |

The pattern: **most rows are not "Powerful."** Reserve the expensive tier for the rows that actually need it.

---

## Output impact: writing unit tests

**Prompt:** *"Write tests for this discount function."*

```python
def apply_discount(price, pct):
    return price * (1 - pct / 100)
```

**Lightweight** — covers the happy path and stops:

```python
def test_apply_discount():
    assert apply_discount(100, 10) == 90
```

**Powerful** — parametrizes, hits boundaries, and *flags the missing validation*:

```python
import pytest

@pytest.mark.parametrize("price, pct, expected", [
    (100, 0, 100),    # no discount
    (100, 100, 0),    # full discount
    (100, 10, 90),    # typical
])
def test_valid(price, pct, expected):
    assert apply_discount(price, pct) == pytest.approx(expected)

def test_negative_pct_is_unhandled():
    # surfaces a DESIGN gap: the function never validates input
    assert apply_discount(100, -5) == 105  # silently inflates price!
```

> The Powerful model didn't just test — it found a bug in the code under test.

---

## Output impact: performance / SQL

**Prompt:** *"Make this query faster."*

```sql
SELECT * FROM orders WHERE DATE(created_at) = '2026-06-01';
```

**Lightweight** — trims columns, misses the real problem:

```sql
SELECT id, customer_id, total
FROM orders WHERE DATE(created_at) = '2026-06-01';  -- still can't use an index
```

**Powerful** — spots that wrapping the column in `DATE()` makes the predicate *non-sargable* (defeats any index), and rewrites to a range scan:

```sql
SELECT id, customer_id, total
FROM orders
WHERE created_at >= '2026-06-01'
  AND created_at <  '2026-06-02';
-- + recommends: CREATE INDEX idx_orders_created_at ON orders (created_at);
```

Same prompt — the cheaper model fixed *style*, the powerful model fixed the **query plan**.

---

## Output impact: a vague bug report

**Prompt:** *"Users get logged out randomly. Here's the session middleware."*

**Lightweight** — latches onto the most literal lever:

> "Increase the session timeout from 30 to 120 minutes."

**Powerful** — treats "randomly" as a clue and forms ranked hypotheses:

> 1. **Load balancer without sticky sessions** — requests hit nodes that don't share session state (most likely given "random").
> 2. **Token-refresh race** — concurrent tabs invalidate each other's refresh token.
> 3. **Clock skew** across nodes expiring JWTs early.
>
> "Before changing timeouts, check whether logouts correlate with which node served the request."

> On **ambiguous** problems, the tier difference is *diagnostic quality*, not syntax.

---

## Escalation in action

The selection process isn't one-shot — it's a cheap-first loop.

```text
1. Start: Sonnet 4.6 / GPT-4.1  (Versatile)
   Prompt: "Fix the failing test in checkout."
   -> proposes a fix. You run tests. Still red on a boundary case.

2. Re-prompt same model with the failure output.
   -> second attempt, still misses the off-by-one in date handling.

3. ESCALATE: Opus 4.8 / GPT-5.5  (Powerful)
   -> identifies a timezone/midnight-boundary bug the
      cheaper model kept pattern-matching past. Tests pass.
```

You spent Powerful tokens on **one** turn, not the whole session. That's the point of starting cheap: you only pay for depth when depth is actually required.

---

## When a bigger model does NOT help

Escalating is not free and not always better. A Powerful model gives **no meaningful uplift** when the task is:

- Boilerplate, scaffolding, or CRUD that follows an obvious pattern
- Formatting, linting fixes, or import organization
- Renaming / mechanical refactors with a single correct result
- Short, verifiable snippets you can eyeball in seconds
- Generating commit messages, docstrings, or simple README sections

For these, a bigger model often just produces **more words and more output tokens** for the same correct answer — paying a premium for verbosity.

> Bigger ≠ better. Bigger = *deeper reasoning*, which only pays off when the task needs it.

---

## Decision tree — GPT family

```text
START: What are you doing?
│
├─ Autocomplete / boilerplate / quick Q&A
│     └─► GPT-5 mini  (or GPT-5.4 nano)        [Lightweight, cheapest]
│
├─ Everyday coding / explain / tests / refactor one file
│     └─► GPT-4.1  (included) or GPT-5.2        [Versatile, balanced]
│
├─ Agentic coding / long autonomous coding sessions
│     └─► GPT-5.x-Codex                         [Powerful, tuned for code agents]
│
└─ Hardest reasoning / architecture / gnarly debugging
      └─► GPT-5.5                               [Powerful, highest cost]

Not sure? → Auto model selection lets Copilot pick the tier.
```

---

## Decision tree — Claude family

```text
START: What are you doing?
│
├─ High-volume, simple, latency-sensitive tasks
│     └─► Claude Haiku 4.5                      [cheapest Claude]
│
├─ Everyday coding / explain / tests / single-file refactor
│     └─► Claude Sonnet 4.6                     [the workhorse — best default]
│
├─ Multi-file refactor / deep debugging / agentic work
│     └─► Claude Opus 4.8                       [Powerful, deepest reasoning]
│
└─ Cost-sensitive but need >Haiku quality
      └─► Sonnet 4.6  (don't jump straight to Opus)

Rule of thumb: Sonnet for most work; reach for Opus only when Sonnet struggles.
```

---

## Cross-family quick guide

If you're choosing *between* families for the same job:

| Need | OpenAI pick | Anthropic pick |
|------|-------------|----------------|
| Cheapest viable | GPT-5 mini | Claude Haiku 4.5 |
| Balanced daily driver | GPT-4.1 / GPT-5.2 | Claude Sonnet 4.6 |
| Code-agent / autonomous | GPT-5.x-Codex | Claude Opus 4.8 |
| Maximum reasoning | GPT-5.5 | Claude Opus 4.8 |

Both families are strong; differences are usually **style and cost**, not capability, within the same tier. Pick the tier first, the family second.

---

## Practical ways to cut token usage

- **Match the model to the task** — biggest lever. Don't default to a Powerful model.
- **Ask for diffs, not full rewrites** — output tokens are the priciest bucket.
- **Trim context** — paste the relevant function, not the whole file/repo.
- **Lean on caching** — keep reused context stable so it hits the cached (discounted) rate.
- **Use Auto selection** for mixed workloads so simple prompts don't land on expensive models.
- **Keep code completions for completions** — they're free; don't open Chat for a one-line suggestion.

---

## Key takeaways

1. A **token** ≈ ¾ of a word; **code is denser**. "Token size" means either a payload's token count *or* a model's context window — keep them distinct.
2. Billing = **model × tokens**, split into **input / output / cached**, converted to AI Credits ($0.01 each).
3. **Output tokens are the expensive bucket** — prompt for concise, targeted answers.
4. **Model choice is the largest cost multiplier** — up to ~17× for identical work.
5. **Start cheap, escalate on failure.** Lightweight for simple, Versatile for daily, Powerful for hard.

---

# Questions?

**Tokens are the new unit of cost. Model choice is the new dial.**

Thank you.
