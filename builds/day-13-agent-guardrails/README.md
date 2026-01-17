# Day 13: Stop Your AI Agent From Making Things Up: 3 Guardrails That Actually Work

## Output
- **Blog Post**: [Stop Your AI Agent From Making Things Up: 3 Guardrails That Actually Work](https://blog.avnishyadav.com/2026/01/ai-agent-guardrails-hallucinations-memory-validation.html)

## TL;DR
AI agents without guardrails will hallucinate, forget context, and return outputs that break your workflows. Use these three guardrails in production: **(1) JSON schema + allowlist validation** for every AI response, **(2) memory window limits + “state summaries”** to prevent drift, **(3) input sanitization + fact-check gates** so bad inputs don’t become confident lies. I wasted two weeks debugging an agent that invented support categories before I added these checks.

---

## Why Guardrails Aren’t Optional

Think of guardrails like seatbelts. You don’t build them because you’re pessimistic. You build them because production is unpredictable.

Without guardrails, you’ll see three predictable failures:
- **Hallucinations:** plausible-but-fake facts (“refund in 2 days”)
- **Memory drift:** the agent contradicts earlier context as the chat grows
- **Format errors:** JSON breaks, fields change, downstream nodes crash

I learned this while building a customer support categorization agent. It worked in testing… then started inventing categories like “urgent billing escalation” that didn’t even exist in our CRM.

---

## Guardrail 1: Structured Output Validation (Schema + Allowlist)

Your first rule: **never trust raw AI text**. Treat the AI like an API that can return invalid responses. Define the format, then validate it.

**In n8n:** AI Node → Function Node (validate) → only then continue.

### What to validate
- Is it valid JSON?
- Are all required fields present?
- Are values constrained to allowed lists?
- Are numbers in correct ranges?
- Does it contain prohibited claims/phrases?

Here’s a practical validator you can paste into an n8n **Function** node:

```javascript
// n8n Function node: Validate AI JSON output (no external libs)
const validCategories = ["billing", "technical", "account", "general"];
const validPriorities = ["low", "medium", "high", "critical"];

function fail(error, raw) {
  return [{
    json: {
      valid: false,
      error,
      raw
    }
  }];
}

function ok(data) {
  return [{
    json: {
      valid: true,
      ...data
    }
  }];
}

const raw = $input.first().json.ai_response;

let parsed;
try {
  parsed = JSON.parse(raw);
} catch (e) {
  return fail(`Invalid JSON: ${e.message}`, raw);
}

// Required fields
const required = ["category", "priority", "confidence"];
for (const key of required) {
  if (!(key in parsed)) return fail(`Missing field: ${key}`, raw);
}

// Allowlist validation
if (!validCategories.includes(parsed.category)) {
  return fail(`Invalid category: ${parsed.category}`, raw);
}
if (!validPriorities.includes(parsed.priority)) {
  return fail(`Invalid priority: ${parsed.priority}`, raw);
}

// Range validation
if (typeof parsed.confidence !== "number" || parsed.confidence < 0 || parsed.confidence > 1) {
  return fail(`Invalid confidence: ${parsed.confidence}`, raw);
}

// Pass through validated output
return ok({ data: parsed });
```

This single node eliminates a huge chunk of “AI randomness”. If the model tries to return “super urgent”, your workflow blocks it immediately.

**Bonus tip:** log every failure (input + raw output + error) to a Google Sheet. Those failures become your prompt improvements and new allowlist entries.

---

## Guardrail 2: Memory Window Management (Stop Context Drift)

Memory drift happens because models have limited context windows. As the conversation grows, older details get dropped or diluted. That’s not a bug — it’s physics.

The fix is simple: **limit the window** and keep a **state summary**.

### My practical memory strategy
- Keep only the last **N** messages (example: 10–20)
- Maintain a compact **“User State” JSON** (preferences, last action, constraints)
- Send the state summary every time, even when trimming older messages

Example function to trim history (works with chat-style arrays):

```javascript
// n8n Function node: Trim conversation window
const maxMessages = 15; // adjust per model
const history = $input.first().json.history || []; // [{role, content}, ...]
const system = history.find(m => m.role === "system");
const msgs = history.filter(m => m.role !== "system");

const trimmed = msgs.length <= maxMessages ? msgs : msgs.slice(-maxMessages);

return [{
  json: {
    history: system ? [system, ...trimmed] : trimmed
  }
}];
```

### The “state summary” pattern (highly recommended)

Instead of forcing the model to remember everything, store stable facts in a small object:

```json
{
  "user_state": {
    "customer_tier": "pro",
    "refund_policy_days": "5-7",
    "preferred_language": "en",
    "last_ticket_id": "SUP-10921"
  }
}
```

You can store that in Sheets/Postgres and inject it into the prompt every run. This is how you prevent “it forgot after 3 messages”.

---

## Guardrail 3: Input Sanitization + Fact-Check Gates

Bad inputs create confident lies. If the user gives a wrong order ID, many agents will “helpfully” invent details. Your job is to stop that upstream.

### Sanitize inputs
- Trim whitespace, normalize casing
- Extract IDs with regex (order IDs, ticket IDs)
- Normalize dates (“next Friday” → ISO date if possible)
- Remove dangerous prompt-injection text when using tools

### Fact-check gates
Before calling the AI for policy/transactional answers, verify against known sources:
1. Extract order ID (Function node)
2. Lookup order in DB/Sheets (HTTP Request / DB node)
3. If not found → ask user to confirm ID (no AI guesswork)
4. If found → AI can explain next steps using validated facts

This single “exists check” can drop real-world errors massively. It turns hallucination-prone tasks into deterministic workflows.

---

## Putting It Together: A Production-Ready n8n Flow

**Workflow outline:**
1. **Trigger:** new message (WhatsApp/Email/Discord/Webhook)
2. **Sanitize input:** clean text + extract IDs
3. **Fact-check:** verify IDs / policies via DB or source-of-truth
4. **Load memory:** last N messages + user_state JSON
5. **AI node:** generate structured JSON output
6. **Validate output:** schema + allowlists + range checks
7. **Confidence gate:** if confidence < 0.7 → human review queue
8. **Update memory:** append message + update user_state
9. **Respond:** send final message / trigger action

Notice the order: **facts → AI → validation**. Not AI first.

---

## Common Mistakes (That Keep Agents Broken)

- **Validating too late:** Validate immediately after the AI node.
- **Not logging failures:** Every failure is data. Log input + model output + validator error into a sheet.
- **Ignoring confidence:** If the model says it’s uncertain, believe it.
- **Using the same strictness for everything:** Match guardrails to the cost of being wrong.

---

## The Before/After Reality

**Before guardrails:** confident lies, missing context, broken JSON, downstream crashes.
**After guardrails:** invalid outputs get blocked, chats stay coherent, workflows fail safely.

Guardrails don’t limit your agent. They make it usable.
