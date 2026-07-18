# Inbound Lead Qualifier & BANT Scorer Agent

An n8n workflow that automatically reads inbound lead emails, scores them using the **BANT** framework (Budget, Authority, Need, Timeline) via an LLM, and routes them to the right next action — a sales alert with an auto-booked calendar slot, a CRM log entry, or silent disqualification — with zero manual triage.

---

## 1. Overview

This workflow watches a Gmail inbox for inbound leads, extracts the sender's details and message, enriches it with company context, and feeds it to an LLM configured as a **BANT scoring engine**. The model returns a structured score (0–25 per dimension, 0–100 total) and a qualification tier: **Hot**, **Warm**, or **Cold**.

Based on that tier, the workflow automatically:
- **Hot** → posts a Slack alert to the sales channel and books a discovery call on Google Calendar at the next real open slot
- **Warm** → logs the lead with full BANT detail to a Google Sheet for manual follow-up / nurture
- **Cold** → silently discards, no action taken

A validation step also guards against malformed LLM output, routing failures to a Slack error alert instead of breaking the run.

```
Gmail Trigger → Normalize Fields → Enrich → BANT Scoring (LLM) → Validate
                                                                     │
                        ┌────────────────────────────────────────────┤
                        ▼ valid                                ▼ invalid
                  Route by Tier                            Slack Error Alert
                        │
        ┌───────────────┼───────────────┐
        ▼ Hot            ▼ Warm            ▼ Cold
   Slack Alert       Log to Sheet      Ignore
        │
   Find Open Calendar Slot
        │
   Book Discovery Call
        │
        └──────────────┬──────────────┘
                        ▼
                 Merge & Log Outcome
```

---

## 2. What Problem Does It Solve?

Sales and growth teams get inbound interest through email — contact forms, replies to campaigns, cold outreach responses — but not every inquiry is worth a rep's time. Manually reading every email to judge budget, authority, urgency, and fit doesn't scale, and it delays the leads that *are* worth acting on fast.

This workflow removes that manual triage step. It reads the email, judges lead quality against a consistent framework (BANT), and acts on that judgment immediately — so sales reps only spend time on leads that are actually qualified, and hot leads get a calendar invite before the trail goes cold.

---

## 3. Benefits

- **Consistency** — every lead is scored against the same four criteria, removing rep-to-rep or day-to-day variance in how "hot" gets judged.
- **Speed** — hot leads get a Slack ping and a calendar invite within seconds of the email arriving, instead of sitting in an inbox until someone checks it.
- **No lost leads** — every inbound email is logged with its score and reasoning, so nothing falls through the cracks even if it's not immediately actionable.
- **Low cost to run** — uses a cheap, fast LLM (DeepSeek via OpenRouter) for scoring rather than a large, expensive model, since this task doesn't need deep reasoning — just consistent judgment.
- **Fails safely** — a validation step catches malformed AI output before it reaches routing logic, alerting a human rather than silently misrouting or crashing.
- **Extensible** — enrichment, scoring criteria, and routing destinations (Slack, Sheets, Calendar) are all modular and swappable for your own CRM/stack.

---

## 4. What's Its Use?

- **Sales teams** fielding inbound interest from a contact form or shared inbox who want automatic lead triage before a human ever looks at it.
- **Founders / small teams** without a dedicated SDR who need qualified leads surfaced immediately rather than manually checked.
- **Agencies or consultancies** handling client inquiries at volume, where a consistent qualification bar matters more than individual judgment calls.
- **Anyone building an AI agent swarm in n8n** looking for a lead-qualification sub-agent that can be called by an orchestrator alongside other agents (support, research, calendar, etc.).

---

## 5. Node-by-Node Breakdown

### `Get Leads` — Gmail Trigger
Polls the connected Gmail inbox every minute for new mail (`maxResults: 1`). This is the entry point of the workflow — every run starts with one new inbound email.

### `Set Leads` — Set/Edit Fields
Normalizes the raw Gmail payload into clean, workflow-friendly fields:
- `name`, `email` — pulled from the sender's `from` header
- `company` — derived from the sender's email domain (e.g. `acme.com` from `jane@acme.com`) as a lightweight fallback signal
- `subject`, `message` — the email subject and body text
- `date&time` — formatted send timestamp

This decouples the rest of the workflow from Gmail's raw field names, so the trigger source could later be swapped (webhook, CRM, etc.) without touching downstream nodes.

### `Set Mock Enrichment` — Set/Edit Fields
Adds placeholder company enrichment data — `employees`, `industry`, `funding_stage` — normally sourced from a service like Clearbit or Apollo. During development/testing this is hardcoded so the BANT scorer's use of enrichment data can be validated without needing a live API key. **Replace this node with a real HTTP Request to an enrichment API for production use.**

### `Basic LLM Chain` — LangChain Chain node
The core scoring step. Sends a single prompt (lead message + enrichment data) to the connected LLM and expects one structured JSON response back. A **Basic LLM Chain** is used instead of a full **AI Agent** node because this task doesn't need tools, memory, or multi-step reasoning — just one input, one judged output — which also makes native output-format enforcement possible (see below).

**Prompt logic:** scores the lead 0–25 on each of Budget, Authority, Need, and Timeline, sums them into a 0–100 total, and assigns a tier: `Hot` (≥70), `Warm` (40–69), or `Cold` (<40).

### `DeepSeek` — OpenRouter Chat Model (sub-node)
The language model powering the scoring step, accessed via OpenRouter. DeepSeek was chosen for its low cost and strong structured-output reliability — this task doesn't need a large, expensive model since it's a single, well-defined judgment call rather than open-ended reasoning.

### `Structured Output Parser` — LangChain Output Parser (sub-node)
Enforces a strict JSON Schema on the LLM's response: each BANT dimension must be a number between 0–25, `total` between 0–100, and `tier` must be exactly one of `Hot`, `Warm`, or `Cold`. If the model's output doesn't match, n8n automatically retries the call — this is what keeps scoring reliable across constantly varying email content.

### `Validate Score` — IF node
A safety check after scoring: confirms `tier` came back non-empty (i.e. parsing actually succeeded). 
- **True** → continues to tier routing
- **False** → routes to `Error Alert` instead, so a parsing failure never silently breaks the workflow or misroutes a lead

### `Split Tier` — Split Out
Extracts the nested `output.tier` field into a flat, top-level field so it can be evaluated cleanly by the routing logic that follows.

### `Route Tier` — Switch
The routing brain of the workflow. Reads the flattened tier field and sends the lead down one of three branches: `Hot`, `Warm`, or `Cold`.

### `Hot Lead Alert` — Slack
Posts an immediate notification to the sales Slack channel for any Hot lead: name, company, total score, the model's reasoning, and contact email.

### `Search Calendar` — Google Calendar (Get All)
Pulls upcoming events from the connected calendar for the next few days, so the workflow can find a genuinely free slot rather than guessing a fixed time.

### `Find Open Slot` — Code
A JavaScript/Luxon function that scans forward from the current time (skipping weekends and outside working hours) and returns the next 30-minute window that doesn't overlap with an existing event. Built with explicit timezone handling (`Asia/Manila`) to avoid the common pitfall of date math silently running in UTC and producing wrong meeting times.

### `Set Meeting for Hot Lead` — Google Calendar (Create Event)
Books the discovery call at the slot found above, invites the lead by email, and includes the AI's BANT reasoning in the event description — so whoever takes the call already has context before it starts.

### `Get Warm Details` — Set/Edit Fields
Consolidates the lead's identifying info and BANT output into a clean object, ready to be logged, for any lead that scored in the Warm range.

### `Warm Leads Log` — Google Sheets (Append)
Writes Warm leads to a dedicated sheet tab for manual follow-up or a future nurture sequence — visibility into leads that aren't urgent but shouldn't be ignored either.

### `Ignore Cold Leads` — No-Op
The Cold branch's terminus. Deliberately does nothing — Cold leads are judged not worth acting on, so the workflow simply ends here for that branch.

### `Merge Output` — Merge
Joins the Hot and Warm branches back into a single stream once their branch-specific actions (Slack/Calendar or Sheets logging) are complete, so both can share one final logging step downstream.

### `Get Output Data` — Set/Edit Fields
Builds the final record — name, email, full BANT output, and tier — used for the consolidated outcome log.

### `Log Lead Outcome` — Google Sheets (Append)
Writes every Hot and Warm lead's final outcome to a shared "merged data" sheet, giving a single place to review the whole funnel and check whether the BANT prompt is well-calibrated over time.

### `Error Alert` — Slack
Fires only when `Validate Score` catches a failure. Posts the failing node, the lead's email (if available), the error message, and a timestamp to Slack — so a bad AI response gets a human's attention immediately instead of vanishing.

---

## Setup Requirements

| Service | Used for | Credential needed |
|---|---|---|
| Gmail | Trigger — reading inbound leads | Gmail OAuth2 |
| OpenRouter | LLM access (DeepSeek) for scoring | OpenRouter API key |
| Slack | Hot lead alerts + error alerts | Slack API token |
| Google Calendar | Checking availability + booking calls | Google Calendar OAuth2 |
| Google Sheets | Logging Warm leads + final outcomes | Google Sheets OAuth2 |

## Known Limitations / Next Steps

- `Set Mock Enrichment` uses hardcoded placeholder data — swap for a real enrichment API (Clearbit, Apollo, or a scraper) before production use.
- Cold leads are currently discarded with no log entry — consider logging them too for full funnel visibility.
- The Hot Lead Slack alert fires slightly before the calendar event is actually created (same run, different steps) — for stricter accuracy, reorder so the alert fires after booking is confirmed.
- Error handling is intentionally lightweight (one validation checkpoint) rather than per-node error wiring, to keep the workflow readable — expand this if you need stricter observability.
