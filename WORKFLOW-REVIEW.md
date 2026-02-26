# n8n Workflow Review: "News Daily - team@riterz.com V2"

## Workflow Architecture Summary

```
PATH 1 — "OPEN AI"
Schedule Trigger (Mon/Wed 04:00 CET)
  → Date & Time
    → AI Agent (GPT-4.1)
        ├── Tool: Perplexity (search recency: week)
        └── Tool: Microsoft Outlook → team@riterz.com, j.steffen@hohle-gasse.ch

PATH 2 — "CLAUDE"
Schedule Trigger1 (Mon/Wed 04:00 CET)
  → Date & Time1
    → AI Agent1 (Claude Sonnet 4.5)
        ├── Tool: Perplexity1 (search recency: week)
        └── Tool: Microsoft Outlook1 → myriam.michel@riterz.com only
```

Both paths use **identical prompts**. Both fire on the **same cron schedule**.

---

## Critical Issues

### 1. Claude Extended Thinking is DISABLED

**Severity: High — directly reduces output quality**

The Anthropic Chat Model node has `"thinking": false`. Extended thinking is Sonnet 4.5's headline feature — it lets the model reason through complex analytical tasks before generating output. For a multi-section commercial intelligence report that requires synthesising research, evaluating deal structures, and generating honest stress-tests, this is exactly the kind of task where thinking makes a huge difference.

**Fix:** Set `"thinking": true` in the Anthropic Chat Model options. Optionally set a thinking budget (e.g., `"thinkingBudget": 10000` tokens) to balance cost vs depth.

### 2. Disconnected Language Model on Claude Path

**Severity: High — possible wiring bug**

The `OpenAI Chat Model1` node (GPT-4.1) exists in the Claude path but its `ai_languageModel` connection array is **empty** (`[]`). Meanwhile, `Anthropic Chat Model` correctly connects to `AI Agent1`. This means the OpenAI Chat Model1 node is an orphan — it's doing nothing but could cause confusion. It looks like a leftover from when you copied the OpenAI path.

**Fix:** Delete the `OpenAI Chat Model1` node entirely from the Claude path.

### 3. No Error Handling

**Severity: High — silent failures send nothing or send broken content**

Neither path has any error handling. If Perplexity's API is down, rate-limited, or returns garbage, the agent will either:
- Hallucinate the entire report (no real data), or
- Fail silently and send no email, with no one being notified

**Fix:** Add an Error Trigger node connected to a fallback notification (e.g., a simple email or Slack message saying "Intel briefing failed — manual review needed"). n8n's built-in error workflow or a try/catch pattern with an IF node works here.

### 4. Recipient Mismatch — Claude Path is Still in Test Mode

**Severity: Medium — likely intentional but worth flagging**

- OpenAI path → `team@riterz.com, j.steffen@hohle-gasse.ch` (full distribution)
- Claude path → `myriam.michel@riterz.com` only (single recipient)

If you're A/B testing, this is fine. But once the Claude path is validated, update recipients to match. Consider adding a shared variable/expression for the recipient list so you only change it in one place.

---

## Structural Improvements

### 5. Single Perplexity Call is a Bottleneck

The prompt asks the agent to gather "3–5 meaningful signals" across EU, APAC, US/UAE covering deal structures, digital overlays, ROI cases, brand budgets, measurement tech, regulatory changes, and retail media crossover. That's a lot to extract from one Perplexity query.

**Improvement:** Use a **multi-step research pattern**:

```
Schedule Trigger
  → Date & Time
    → AI Agent (Step 1: Research)
        └── Tool: Perplexity (called 2-3 times with targeted queries)
    → AI Agent (Step 2: Write & Send)
        └── Tool: Microsoft Outlook
```

Or, instruct the agent explicitly in the prompt to make **multiple Perplexity calls** with different focused queries (e.g., one for deal signals, one for measurement/ROI, one for brand budget shifts). The current setup allows this since the agent can call the tool multiple times — but the prompt doesn't guide it to do so.

**Add to prompt:**
```
When using perplexity, make at least 2-3 separate searches with different
focused queries rather than one broad search. For example:
- Search 1: Recent sports sponsorship deals, digital inventory, modular packages
- Search 2: Sponsorship ROI measurement, attribution technology, brand spend shifts
- Search 3: [Region-specific] EU/APAC sports digital activation news
```

### 6. No Quality Gate Before Sending

There's no validation between content generation and email send. The agent generates HTML and immediately sends it.

**Improvement:** Add a second lightweight AI step (or a Code node) that checks:
- Does the HTML contain all 8 required sections?
- Are there actual source URLs (not hallucinated)?
- Is the content length reasonable (not a stub or excessively long)?
- Does it contain any meta-instructions that leaked into the body?

This can be a simple Code node with regex checks, or a second quick LLM call with a focused validation prompt.

### 7. Both Paths Fire Simultaneously — No Comparison Mechanism

If you're A/B testing OpenAI vs Claude, both emails arrive at roughly the same time but go to different recipients. There's no structured way to compare quality.

**Improvement options:**
- Send both to the same test recipient with `[GPT]` / `[Claude]` prefixed in subject lines
- Add a Google Sheet node that logs: date, model used, section count, word count, source count — for tracking quality over time
- Or stagger them: run OpenAI on Monday, Claude on Wednesday, and compare

### 8. Date & Time Node — Dynamic Field Name Issue

The `outputFieldName` is set to `={{ $json['Readable date'] }}` which uses the *value* of `Readable date` as the *field name*. This means the output field name changes every run (e.g., `"Monday, 24 February 2026"`). This is unconventional and could cause downstream reference issues.

**Fix:** Set a static `outputFieldName` like `"Readable date"` and reference it consistently downstream.

---

## Prompt Improvements for Claude

### 9. The Prompt is Identical — But Claude and GPT Respond Best to Different Patterns

Claude responds particularly well to:
- **XML-tagged structure** for complex instructions
- **Explicit thinking guidance** ("Before writing each section, consider...")
- **Role depth** — Claude benefits from a detailed persona with values and judgement criteria, not just a job title
- **Negative examples** — "Do NOT do X" is very effective with Claude

The current prompt is written in a GPT-optimised style (flat numbered lists, imperative instructions). See the improved prompt below.

### 10. The Prompt Asks the Agent to Both Research AND Format AND Send in One Turn

This is a lot of cognitive load for a single agent call. The agent must:
1. Decide what to search for
2. Call Perplexity
3. Synthesise results
4. Generate complex HTML
5. Call the email tool

**Improvement:** Split into two agent calls chained together:
- **Agent 1 (Researcher):** Uses Perplexity, outputs structured JSON with signals, sources, and data points
- **Agent 2 (Writer):** Takes the structured data, generates the HTML report, sends via email

This separation means better research (Agent 1 focuses only on finding good data) and better writing (Agent 2 focuses only on formatting and analysis).

---

## Model Recommendation

### 11. Consider Claude Sonnet 4.6 or Opus 4.6

You're using `claude-sonnet-4-5-20250929`. The current latest models are:
- **Claude Sonnet 4.6** (`claude-sonnet-4-6`) — faster, better tool use, better instruction following
- **Claude Opus 4.6** (`claude-opus-4-6`) — strongest analytical reasoning, best for nuanced commercial analysis

For a commercial intelligence briefing where analytical quality matters more than cost, **Opus 4.6** would produce noticeably better stress-tests and comparative insights. If cost is a concern, Sonnet 4.6 is the better balance.

**Fix:** Update the model value to `claude-sonnet-4-6-20250514` or `claude-opus-4-6-20250527` (check n8n's model list for exact IDs available).

---

## Minor Issues

### 12. Hardcoded Webhook ID Shared Across Both Outlook Nodes
Both Outlook tool nodes share `webhookId: "0efee48c-0de7-48f8-870a-a4993811285d"`. This shouldn't cause issues since they use the same OAuth credential, but it's worth noting.

### 13. No Rate Limiting or Cooldown
If the workflow is manually triggered multiple times during testing, it will send duplicate emails. Consider adding a deduplication check or a flag.

### 14. Timezone Assumption
The cron runs at `0 4 * * 1,3` in `Europe/Paris` timezone (04:00 CET). This means the briefing covers "last 3-7 days" but always from the same time reference. This is fine but worth documenting.

---

## Priority Action List

| Priority | Action | Effort |
|----------|--------|--------|
| 1 | Enable extended thinking on Claude node | 1 min |
| 2 | Delete orphaned OpenAI Chat Model1 node | 1 min |
| 3 | Add error handling / failure notification | 15 min |
| 4 | Update to Claude Sonnet 4.6 or Opus 4.6 | 1 min |
| 5 | Add multi-query guidance to prompt | 5 min |
| 6 | Use Claude-optimised prompt (see companion file) | 10 min |
| 7 | Fix Date & Time outputFieldName to static value | 2 min |
| 8 | Update Claude path recipients when ready for production | 1 min |
| 9 | Add quality-gate Code node before email send | 20 min |
| 10 | Split into Research Agent + Writer Agent | 30 min |
