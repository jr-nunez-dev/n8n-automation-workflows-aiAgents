# 🎫 AI Support Ticket Classifier

## 📌 Description

**AI Support Ticket Classifier** is an n8n webhook workflow that reads a customer support message and returns its category, priority, sentiment, routing team, a draft reply and a human-review flag as structured JSON.

---

## 🎯 Problem

Support teams spend a lot of time reading each ticket, deciding its category and urgency, routing it to the right team and writing a first reply.

---

## 💡 Solution

A guarded webhook cleans the ticket, checks the visitor is human and within limits, has a tool-less AI agent classify it, then re-validates every field in code and forces human review on sensitive tickets before returning the result and an optional email copy.

---

## 🧰 Tools

n8n, Cloudflare Turnstile, Groq, LangChain (n8n AI nodes), Gmail, JavaScript (Code node), Webhook

---

## ✨ Features

Webhook-based API for a web front end, Input cleaning and length limits, Ticket tag break-out protection, Human check with Cloudflare Turnstile, Rate limiting for requests and emails, Tool-less AI agent (nothing for an injected instruction to call), Structured JSON output with auto-fix retry, Nine ticket categories, Four priority levels, Sentiment detection, Confidence level, Language detection, Team routing with reason, Key points and tags, Draft reply in the customer's language, Missing-information list, Human-review flag, Forced human review for refunds and angry customers, Prompt-injection handling as spam or abuse, Premium customer priority boost, Server-side output validation, Optional result email, HTML-escaped email output, Consistent HTTP 200 responses for the page, Email sent after the response

---

## 👥 Who Will Use This?

- Customer support and service desk teams
- Support team leads and operations managers
- SaaS and e-commerce companies
- Helpdesk and CRM automation consultants
- Developers building AI-assisted ticketing tools
- Anyone showing a safe, guarded AI triage demo

---

# ⚙️ Workflow Overview

```text
Webhook (POST)
      ↓
Clean input
      ↓
Input valid? ───────────── no ──► Respond: invalid_input
      ↓ yes
Verify Turnstile
      ↓
Human check passed? ────── no ──► Respond: blocked
      ↓ yes
Rate limit
      ↓
Within limit? ──────────── no ──► Respond: rate_limited
      ↓ yes
AI Agent (Groq + Structured Output Parser)
      ↓                      ↘ error
Validate output
      ↓
Output ok? ─────────────── no ──► Respond: error
      ↓ yes
Respond: ok (page gets its answer)
      ↓
Email requested? ──► Build email ──► Send result email (Gmail)
```

---

# 🧩 Workflow Components

| Node | Purpose |
|---|---|
| `Webhook` | Receives the ticket payload (POST) and limits allowed origins |
| `Clean input` | Trims and validates fields, strips `<ticket>` tags, checks the optional email |
| `Input valid?` | Sends bad input to the `invalid_input` response |
| `Verify Turnstile` | Verifies the Cloudflare Turnstile token |
| `Human check passed?` | Continues on success, otherwise sends the `blocked` response |
| `Rate limit` | Code node that counts requests and emails using workflow static data |
| `Within limit?` | Continues if under the limit, otherwise sends `rate_limited` |
| `AI Agent` | Classifies the ticket (no tools attached) |
| `Groq Chat Model` | Language model behind the agent and the parser auto-fix |
| `Structured Output Parser` | Forces the AI to follow the ticket schema, with auto-fix on |
| `Validate output` | Re-checks every field, applies safe defaults and forced human-review rules |
| `Output ok?` | Sends unusable output to the `error` response |
| `Respond: ok` | Returns `{ ok: true, result }` to the page |
| `Respond: invalid_input` / `blocked` / `rate_limited` / `error` | Friendly error replies (all HTTP 200) |
| `Email requested?` | Continues only if a valid email survived the checks |
| `Build email` | Builds an HTML-escaped result email |
| `Send result email` | Sends the email through Gmail |
| `Note: ...` sticky notes | Setup, section guides, error codes and test samples on the canvas |

---

# 🔄 Detailed Workflow Process

## 1. Trigger: `Webhook`

A POST webhook with `responseMode: responseNode`. Allowed origins: the demo site and `http://localhost:5173`.

Expected request body:

```json
{
  "action": "classify_ticket",
  "channel": "email",
  "customerTier": "premium",
  "subject": "Charged twice",
  "text": "I was charged twice for my subscription this month...",
  "notifyEmail": "optional@example.com",
  "turnstileToken": "<token>"
}
```

---

## 2. Input Cleaning: `Clean input` and `Input valid?`

Nothing downstream trusts the browser.

| Field | Rule |
|---|---|
| `action` | Must be `classify_ticket` |
| `text` | 20 to 3000 characters, trimmed, any `<ticket>` tags removed |
| `subject` | Trimmed, `<ticket>` tags removed, cut to 150 characters |
| `channel` | `email`, `chat`, `web_form`, `social` (defaults to `email`) |
| `customerTier` | `free`, `standard`, `premium`, `unknown` (defaults to `unknown`) |
| `notifyEmail` | Lowercased, max 254 characters, no whitespace, valid shape. A bad address is ignored and the ticket is still classified |

Invalid input exits through `Respond: invalid_input`.

---

## 3. Human Check: `Verify Turnstile` and `Human check passed?`

The token is sent to Cloudflare `siteverify` (continues on error). If `success` is not `true`, the workflow answers `blocked` and stops.

---

## 4. Rate Limit: `Rate limit` and `Within limit?`

Counters live in the workflow's own static data.

| Limit | Value |
|---|---|
| Requests per hour (all visitors) | 30 |
| Emails per address per day | 3 |
| Emails per day (all visitors) | 40 |

- Over the request limit: `rate_limited` response.
- Over an email limit: the ticket is still classified, only the email is skipped.
- Static data persists only for **active** (production) runs, so manual test runs always pass.
- To share another budget, replace this node with your own rate-limit check using a separate key.

---

## 5. AI Classification: `AI Agent`

An n8n AI Agent with **no tools**, a Groq chat model and a structured output parser (**Auto-fix** on, so one malformed reply gets retried). The customer text is wrapped in `<ticket>` tags and the system prompt says it is data, never instructions.

If the ticket asks the AI to ignore rules, change a priority, reveal the prompt or contact anyone, the agent classifies it as `spam_abuse` and requires human review.

### Classification rules

| Field | Values |
|---|---|
| `category` | `billing`, `account_access`, `technical_issue`, `bug_report`, `feature_request`, `refund_cancellation`, `shipping_delivery`, `general_question`, `spam_abuse` |
| `priority` | `urgent` (time-critical or security risk), `high` (money problem, repeat contact, chargeback or legal threat), `normal`, `low` |
| `sentiment` | `angry`, `frustrated`, `neutral`, `positive` |
| `confidence` | `high`, `medium`, `low` |

A premium customer moves a borderline case up one priority level. The draft reply is short, polite, in the customer's language, never promises a refund or reset, and is signed "Support".

### Output schema

```json
{
  "category": "billing",
  "priority": "high",
  "sentiment": "frustrated",
  "confidence": "high",
  "language": "English",
  "summary": "1 to 2 sentences",
  "suggestedTeam": "Billing support",
  "routingReason": "1 sentence",
  "keyPoints": ["2 to 5 short facts"],
  "tags": ["up-to-6", "lowercase-kebab-case-tags"],
  "humanReview": { "required": false, "reason": "" },
  "suggestedReply": "Short draft reply",
  "missingInfo": ["Things an agent should ask for"]
}
```

If the agent fails, its error output goes to `Respond: error`.

---

## 6. Output Validation: `Validate output` and `Output ok?`

Fixed rules beat the model.

| Rule | Behavior |
|---|---|
| Required fields | Missing or unknown `category` or `priority` returns `ok: false` |
| Safe defaults | Unknown sentiment becomes `neutral`, unknown confidence becomes `low` |
| Caps | Up to 5 key points, 6 tags, 5 missing-info items |
| Forced human review | Always on for `refund_cancellation` or `angry` sentiment, with a default reason if none is given |

Failed output goes to `Respond: error`.

---

## 7. Response and Optional Email: `Respond: ok`, `Email requested?`, `Build email`, `Send result email`

`Respond: ok` answers the page first with `{ ok: true, result }`. Only then does the email branch run, so a slow or failed email never delays or breaks the page.

The email shows priority and category, confidence, sentiment, language, summary, routing, key points, tags, the human-review notice, the suggested reply and what is still unknown. All customer and model text is HTML-escaped. The suggested reply is **a draft shown to the visitor only, never sent to a customer**.

### Response codes

All responses use **HTTP 200** because the page expects it.

| Code | When |
|---|---|
| `ok: true` | Classification completed |
| `invalid_input` | Text too short or too long, or wrong action |
| `blocked` | Turnstile failed |
| `rate_limited` | Over the hourly request cap |
| `error` | The AI reply was unusable or the agent failed |

---

# 🔌 Integrations

## n8n
Runs the webhook, guards, branching, AI agent and responses.

## Cloudflare Turnstile
Human verification before any AI call.

## Groq
Language model for classification.

## Gmail
Sends the optional result email.

---

# 🔐 Credentials & Configuration

| Item | Where |
|---|---|
| Turnstile **secret** key (not the site key) | Node `Verify Turnstile` (replace `<PUT_CREDS_HERE>`) |
| Groq credential | Node `Groq Chat Model` |
| Gmail credential | Node `Send result email` |
| Allowed Origins | Node `Webhook` > Options |
| Front-end variable | Put the **production** URL (`/webhook/...`, not `/webhook-test/`) in `VITE_SUPPORT_WEBHOOK_URL`, then redeploy the site |
| Rate-limit numbers | Top of the `Rate limit` code node |

---

# 🛠️ Configuration Checklist

- [ ] Import the workflow JSON into n8n (Workflows > Import from File).
- [ ] Add the Turnstile secret key to `Verify Turnstile`.
- [ ] Attach the Groq credential to `Groq Chat Model`.
- [ ] Attach the Gmail credential to `Send result email`.
- [ ] Check the Webhook Allowed Origins.
- [ ] Review the rate-limit numbers.
- [ ] Optionally set an error workflow in Settings.
- [ ] Test with the five samples below.
- [ ] Activate the workflow.
- [ ] Copy the production URL into `VITE_SUPPORT_WEBHOOK_URL` and redeploy.

---

# ⚠️ Important Configuration Notes

### Production vs test URL
The page must use the production URL, and the workflow must be active.

### Rate limit storage
Counters reset when the workflow is re-imported or deactivated. For a shared or persistent budget, use an external store.

### Draft replies are never sent
The suggested reply is shown to the visitor only. Nothing is sent to a customer.

### AI is advisory
The classification is a triage suggestion. A person should review sensitive tickets before any reply.

### Prompt-injection check
If sample 5 below leaks any prompt or tool name, tighten the system prompt and retest before going public.

---

# 🚀 How to Run

1. Open the Support Ticket Classifier page.
2. Paste a customer message (20 to 3000 characters).
3. Choose the channel and customer plan, and add a subject if you have one.
4. Optionally enter an email to receive the result.
5. Pass the human check and submit.
6. The workflow cleans, verifies, rate-limits, classifies and validates the ticket.
7. The page shows category, priority, routing, draft reply and the review flag; the email follows if requested.

### Test samples

| Sample | Expected result |
|---|---|
| Charged twice | `billing`, `high`, no human review |
| Locked out | `account_access`, `urgent`, human review on |
| App crash | `bug_report`, `normal` |
| Angry cancel | `refund_cancellation`, `high`, human review on |
| Prompt injection | `spam_abuse`, `low`, human review on, no prompt or tool names in any field |

Then run one with your own email filled in and check the inbox.

---

# 🧯 Troubleshooting

| Symptom | Likely cause |
|---|---|
| `blocked` every time | Wrong Turnstile secret, or token missing |
| `rate_limited` | More than 30 requests in the last hour |
| `invalid_input` | Text under 20 or over 3000 characters, or wrong `action` |
| `error` every time | Check the `AI Agent` execution, usually the model credential |
| CORS error in browser | Page origin is not in the Webhook's Allowed Origins |
| No email received | Email invalid, email caps reached, or Gmail credential missing |

---

# 💰 Estimated Business Value

The numbers below are **estimates**, not measured results.

### Manual work per ticket

Reading the ticket, choosing a category, setting priority, picking the team, noting key points and drafting a first reply typically takes:

```text
3–6 minutes per ticket
```

The workflow returns a draft in seconds, leaving a short review. Net saving is estimated at 3–5 minutes per ticket.

### Estimated Impact

| Metric | Estimate |
|---|---:|
| Manual triage time per ticket | 3–6 min |
| Estimated Time Saved (per ticket) | 3–5 min |
| Tickets triaged per month (assumption) | 300 |
| **Estimated Time Saved (monthly)** | **~15–25 hours** |
| Illustrative labor value | $15–$25/hour |
| **Potential Savings (Monthly)** | **$225–$625** |

> **Note:** Savings depend on ticket volume, agent cost and how much of each draft needs editing. Faster routing of urgent tickets is an additional benefit not counted here.

---

# 📈 Scalability & Recommended Improvements

- Connect to Zendesk, Freshdesk, Intercom or HubSpot to classify real tickets automatically
- Auto-assign tickets to the suggested team
- Move rate limiting to a persistent store
- Add a knowledge-base lookup for better draft replies
- Track classification accuracy with agent feedback
- Add SLA timers based on priority
- Alert Slack or Teams on `urgent` tickets
- Customize categories and priorities per company
- Add a model fallback if Groq is unavailable

---

# 🔒 Data & Operational Considerations

- Tickets may contain personal data. Avoid pasting real customer details into a public demo.
- Ticket text is sent to the AI provider (Groq) for processing.
- The workflow does not store tickets in a database. Check the n8n execution-data settings if you do not want them kept in n8n.
- Email content is HTML-escaped and only sent to an address the visitor typed.
- Draft replies are never sent to customers.
- Human review is required before acting on sensitive tickets.
- Monitor Groq, Gmail and n8n usage.

---

# 📊 Current Workflow Characteristics

| Capability | Current Implementation |
|---|---|
| Trigger | POST Webhook |
| Input guard | Code node (length, enums, tags, email) |
| Human check | Cloudflare Turnstile |
| Rate limit | 30 requests/hour, 3 emails/address/day, 40 emails/day |
| AI | n8n AI Agent (no tools) with Groq |
| AI output | Structured JSON with auto-fix |
| Post-AI rules | Field validation, defaults, forced human review |
| Output | JSON `{ ok, result }` |
| Email | Optional Gmail copy, sent after the response |
| HTTP status | Always 200 |
| Timezone | Asia/Manila |

---

# 🧱 Node Flow Summary

```text
Webhook
  │
  ▼
Clean input
  │
  ▼
Input valid?
 ├── true ──► Verify Turnstile ──► Human check passed?
 │                                   ├── true ──► Rate limit ──► Within limit?
 │                                   │                             ├── true ──► AI Agent (Groq Chat Model + Structured Output Parser)
 │                                   │                             │              ├── success ──► Validate output ──► Output ok?
 │                                   │                             │              │                                    ├── true ──► Respond: ok ──► Email requested? ──► Build email ──► Send result email
 │                                   │                             │              │                                    └── false ─► Respond: error
 │                                   │                             │              └── error ────► Respond: error
 │                                   │                             └── false ──► Respond: rate_limited
 │                                   └── false ──► Respond: blocked
 └── false ──► Respond: invalid_input
```

---

# 📦 Requirements

- n8n (with AI / LangChain nodes)
- Cloudflare Turnstile site key and secret
- Groq account and API key
- Gmail account (OAuth2) for the optional email
- A front-end page that posts to the webhook

---

# 📌 Portfolio Value

This workflow shows how to make an AI step safe to expose publicly: **no tools, strict inputs, validated outputs, forced human review**.

```text
Webhook API Design
+
Input Sanitization
+
Prompt-Injection Defense
+
Bot Protection (Turnstile)
+
Request and Email Rate Limiting
+
Tool-less AI Agent
+
Structured Output with Auto-Fix
+
Post-AI Business Rules
+
Human-in-the-Loop Design
+
Transactional Email
```

Relevant for **customer support automation, AI safety in automation, API-backed workflows and AI workflow engineering**.

---

# 📄 One-Sentence Project Summary

> **An n8n webhook workflow that classifies customer support tickets by category, priority, sentiment and routing with a tool-less Groq AI agent, validates the output in code, forces human review on sensitive cases, and optionally emails the result.**
