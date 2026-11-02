# 🟢 CSV Data Validator

## 📌 Description

**CSV Data Validator** is an n8n webhook workflow that checks an uploaded CSV against column rules using plain deterministic code, returns a cleaned file and a detailed issue list, and adds a short AI-written summary on top.

---

## 🎯 Problem

Messy CSV files with bad emails, broken dates, duplicate rows and inconsistent formats take a long time to check by hand and are easy to get wrong.

---

## 💡 Solution

A webhook receives the parsed CSV from a web page, passes it through a human check and rate limit, validates it with a fixed Code node, lets an AI explain the results from counts only, and returns everything as JSON with an optional email copy.

---

## 🧰 Tools

n8n, Cloudflare Turnstile, Groq, LangChain (n8n AI nodes), Gmail, JavaScript (Code node), Webhook

---

## ✨ Features

Webhook-based API for a web front end, Human check with Cloudflare Turnstile, Rate limiting before any work runs, Deterministic validation (same file, same answer), Seven column types (text, email, phone, date, number, integer, url), Required and unique column rules, Automatic whitespace trimming, Date standardization to YYYY-MM-DD with DMY or MDY input, Thousands-separator cleanup for numbers, Duplicate row detection with optional removal, Column-count mismatch detection, Error and warning severity levels, Cleaned CSV output, AI summary and top fixes without sending cell values, Graceful fallback when the AI fails, Optional result email, No file storage by design, Safe error responses with clear codes

---

## 👥 Who Will Use This?

- Data analysts and operations teams cleaning exports before import
- Marketing and sales teams preparing contact lists
- Developers who need a validation API behind an upload form
- Automation consultants and agencies
- Anyone showing a safe, practical AI-assisted data-quality demo

---

# ⚙️ Workflow Overview

```text
Webhook (POST)
      ↓
Turnstile Verify ──── fail ──► 403 Blocked
      ↓ pass
Rate Limit Check ──── over ──► 429 Rate Limited
      ↓ allowed
validate-csv (Code) ─ error ─► invalid_input (HTTP 200)
      ↓
AI Summary (Groq + Output Parser)
      ↓
merge ─────────────── error ─► error (HTTP 200)
      ↓
Respond OK (JSON to the page)
      ↓
Build Email ──► Send Email (Gmail, optional)
```

---

# 🧩 Workflow Components

| Node | Purpose |
|---|---|
| `Webhook` | Receives the CSV payload (POST) and restricts allowed origins |
| `turnstile-verify` | Verifies the Cloudflare Turnstile token |
| `turnstile ok?` | Continues on success, otherwise sends the 403 response |
| `rate-limit (replace me)` | Placeholder for the rate-limit nodes; must output `{ allowed: true \| false }` |
| `within limit?` | Continues if allowed, otherwise sends the 429 response |
| `validate-csv` | Code node that guards input and runs all validation and cleaning |
| `AI summary` | LLM chain that writes a short summary and top fixes |
| `Groq Chat Model` | Language model (temperature 0.3) behind the AI summary |
| `Output parser` | Forces the AI to return `{ aiSummary, topFixes }` |
| `merge` | Combines validation result and AI output into the final response |
| `Respond OK` | Returns the result to the page (HTTP 200) |
| `Respond 403 (human check)` | Returns `blocked` when Turnstile fails |
| `Respond 429 (rate limit)` | Returns `rate_limited` when the limit is exceeded |
| `Respond invalid_input` | Returns `invalid_input` (HTTP 200) so the page can show the message |
| `Respond error` | Returns a generic `error` if merging fails |
| `build email` | Builds an HTML result email, or nothing if the address is empty or invalid |
| `Send email` | Sends the result email through Gmail |
| `Note: ...` sticky notes | Setup, Gate, Validate, AI and Email guidance on the canvas |

---

# 🔄 Detailed Workflow Process

## 1. Trigger: `Webhook`

A POST webhook with `responseMode: responseNode`, so the workflow decides when and how to reply. Allowed origins are limited to the demo site and `http://localhost:5173`.

Expected request body:

```json
{
  "action": "validate_csv",
  "turnstileToken": "<token>",
  "fileName": "contacts.csv",
  "notifyEmail": "optional@example.com",
  "headers": ["name", "email", "joined"],
  "rows": [["Ana", "ana@example.com", "03/15/2025"]],
  "rules": [
    { "index": 1, "type": "email", "required": true, "unique": true }
  ],
  "options": {
    "trimWhitespace": true,
    "removeDuplicateRows": false,
    "dateOrder": "MDY"
  }
}
```

---

## 2. Human Check: `turnstile-verify` and `turnstile ok?`

The token is sent to Cloudflare's `siteverify` endpoint (`neverError` is on, so a bad token does not crash the run). If `success` is not `true`, the workflow answers **403 blocked** and stops.

---

## 3. Rate Limit: `rate-limit (replace me)` and `within limit?`

The rate-limit node is a **placeholder** (No-Op) that must be replaced with your own rate-limit logic. The next node passes the request when `allowed !== false`; otherwise it answers **429 rate_limited**.

Both gates run before any validation or AI work, so abusive traffic never reaches the AI.

---

## 4. Validation: `validate-csv`

A deterministic Code node (Run Once for All Items). The same file always produces the same answer.

### Input guard

| Limit | Value |
|---|---|
| Max rows | 500 |
| Max columns | 20 |
| Max characters per cell | 1000 |
| Allowed `action` | `validate_csv` |

Any violation throws `invalid_input`, which exits through the node's error output.

### Column types

`text`, `email`, `phone`, `date`, `number`, `integer`, `url`

### Checks and issue types

| Issue type | Severity | Meaning |
|---|---|---|
| `required_missing` | error | Required value is empty |
| `invalid_email` | error | Not a valid email address |
| `invalid_phone` | error | Not 7 to 15 digits or has invalid characters |
| `invalid_date` | error | Not a real calendar date |
| `invalid_number` | error | Not a valid number or integer |
| `invalid_url` | error | Not a valid http(s) address |
| `duplicate_value` | error | Value in a `unique` column appears more than once |
| `column_count` | error | Row length differs from the header |
| `whitespace` | warning | Extra spaces at the start or end (auto-trimmed if enabled) |
| `duplicate_row` | warning | Identical to an earlier row (removed if enabled) |
| `format_fixed` | warning | Auto-standardized (dates to `YYYY-MM-DD`, thousands separators removed) |

### Output

```json
{
  "summary": {
    "rowsChecked": 0,
    "rowsWithErrors": 0,
    "cleanRows": 0,
    "issuesTotal": 0,
    "autoFixed": 0,
    "duplicatesRemoved": 0
  },
  "byType": [{ "type": "invalid_email", "count": 3, "severity": "error" }],
  "issues": [],
  "truncated": false,
  "cleaned": { "headers": [], "rows": [] }
}
```

The issue list is capped at **500** items (`truncated` tells the page if more existed). Cell values in issues are cut to 80 characters.

---

## 5. AI Explanation: `AI summary`

An LLM chain using **Groq** (temperature 0.3) with a structured output parser. It receives:

- Row count and summary totals
- Issue counts by type
- Up to 15 example issues containing **row, column, type and suggestion only, never cell values**

The system prompt tells the model the checks are already correct, to use only the given numbers, and to treat all input as data.

### AI output schema

```json
{
  "aiSummary": "2 to 3 plain sentences on how clean the file is",
  "topFixes": ["Up to 4 short, specific actions, most important first"]
}
```

The node uses **Continue on error**, so if the AI fails the workflow still returns the validation with an empty summary.

---

## 6. Merge and Response: `merge` and `Respond OK`

`merge` combines the validation result with the AI output (empty summary and fixes if the AI failed, top fixes capped at 5) and wraps it as `{ ok: true, result: {...} }`. `Respond OK` returns it with HTTP 200.

### Response codes

| Code | HTTP | When |
|---|---|---|
| `ok: true` | 200 | Validation completed |
| `blocked` | 403 | Human check failed |
| `rate_limited` | 429 | Too many runs |
| `invalid_input` | 200 | File or rules failed the input guard |
| `error` | 200 | Merge or processing failed |

---

## 7. Optional Email Copy: `build email` and `Send email`

This branch runs **after** the page already has its answer, so a slow or failed email never delays the demo.

- `build email` returns **no items** if `notifyEmail` is empty or not a valid address, so nothing is sent.
- Otherwise it builds an HTML email with the verdict, AI summary, stats, issues by type, top fixes and the first 10 issues. All values are HTML-escaped.
- `Send email` sends it through Gmail (sender name "Testarosa - Playground", attribution off).

---

# 🔌 Integrations

## n8n
Runs the webhook, branching, Code nodes, AI chain and responses.

## Cloudflare Turnstile
Human verification before any processing.

## Groq
Language model for the AI summary.

## Gmail
Sends the optional result email.

---

# 🔐 Credentials & Configuration

Before activating, set up:

| Item | Where |
|---|---|
| Turnstile secret | Node `turnstile-verify` (replace `<PUT_CREDS_HERE>`) |
| Rate limit logic | Replace node `rate-limit (replace me)` |
| Groq credential | Node `Groq Chat Model` |
| Gmail credential | Node `Send email` |
| Allowed Origins | Node `Webhook` > Options |
| Front-end variable | Put the **Production URL** in `VITE_CSV_WEBHOOK_URL` |

---

# 🛠️ Configuration Checklist

- [ ] Import the workflow JSON into n8n (Workflows > Import from File).
- [ ] Add the Turnstile secret to `turnstile-verify`.
- [ ] Replace the rate-limit placeholder with your own logic.
- [ ] Confirm the rate-limit step outputs `{ allowed: true | false }`.
- [ ] Attach the Groq credential to `Groq Chat Model`.
- [ ] Attach the Gmail credential to `Send email`.
- [ ] Check the Webhook's Allowed Origins.
- [ ] Turn execution data saving OFF in workflow Settings.
- [ ] Optionally set an error workflow in Settings.
- [ ] Activate the workflow.
- [ ] Copy the Production URL into `VITE_CSV_WEBHOOK_URL`.
- [ ] Test with a small CSV, an invalid file and a blocked token.

---

# ⚠️ Important Configuration Notes

### Rate limit is a placeholder
Until it is replaced, every request that passes Turnstile is allowed through.

### Execution data is not stored on purpose
Uploaded rows must not be saved, so execution data saving should be OFF. Verify this setting after import.

### Validation logic exists in two places
The same function runs in the web page's preview mode. If you change one, keep the other in sync.

### Response codes
Validation and processing problems return HTTP 200 with `ok: false`, so the page can show the message. Only the human check and rate limit use 403 and 429.

### AI is advisory
The checks are plain code and are the source of truth. The summary is a suggestion and can be empty.

---

# 🚀 How to Run

1. Open the web page and choose a CSV file.
2. Set column types and rules (required, unique) and cleaning options.
3. Pass the human check and submit.
4. The page sends the parsed rows to the webhook.
5. The workflow verifies, rate-limits, validates and summarizes.
6. The page shows the summary, issues, top fixes and cleaned CSV download.
7. If an email was entered, a copy of the result is sent afterward.

Quick test with curl (a real Turnstile token is required):

```bash
curl -X POST "<PRODUCTION_URL>" \
  -H "Content-Type: application/json" \
  -d '{"action":"validate_csv","turnstileToken":"<token>","headers":["email"],"rows":[["bad-email"]],"rules":[{"index":0,"type":"email"}]}'
```

---

# 🧯 Troubleshooting

| Symptom | Likely cause |
|---|---|
| 403 `blocked` | Missing or invalid Turnstile token or secret |
| 429 `rate_limited` | Rate-limit logic returned `allowed: false` |
| `invalid_input` | Over 500 rows, 20 columns or 1000 characters per cell, wrong `action`, or unknown column type |
| Empty AI summary | AI node failed or the Groq credential is missing; validation still works |
| CORS error in browser | Page origin is not in the Webhook's Allowed Origins |
| No email received | Address empty or invalid, or Gmail credential not attached |

---

# 💰 Estimated Business Value

The numbers below are **estimates**, not measured results.

### Manual work per file

Opening the CSV, scanning rows, checking emails, phones, dates and numbers, finding duplicates, fixing formats and re-saving typically takes:

```text
15–30 minutes per file
```

The workflow returns results in seconds to about a minute.

### Estimated Impact

| Metric | Estimate |
|---|---:|
| Manual time per file | 15–30 min |
| Estimated Time Saved (per file) | 15–30 min |
| Files checked per month (assumption) | 20 |
| **Estimated Time Saved (monthly)** | **5–10 hours** |
| Illustrative labor value | $15–$30/hour |
| **Potential Savings (Monthly)** | **$75–$300** |

> **Note:** Savings depend on file size, how messy the data is, labor cost and monthly volume. They grow with more files.

---

# 📈 Scalability & Recommended Improvements

- Replace the placeholder with a persistent rate limit
- Raise or remove the 500-row cap with batch processing
- Add custom regex or allowed-value rules per column
- Add country-specific phone validation
- Support Excel files and direct file upload
- Store results (without cell data) for usage analytics
- Add more AI-free auto-fixes such as case normalization
- Send results to Slack or Telegram
- Keep the page preview and workflow validation logic in one shared source

---

# 🔒 Data & Operational Considerations

- Uploaded rows are processed in memory and **not stored** when execution data saving is off.
- The AI never receives cell values, only counts, row numbers, column names, issue types and suggestions.
- Column names are sent to the AI, so avoid sensitive column headers.
- Email output is HTML-escaped and only sent to an address the visitor typed.
- Follow privacy and data-protection laws for any personal data users upload.
- Monitor Groq, Gmail and n8n usage.

---

# 📊 Current Workflow Characteristics

| Capability | Current Implementation |
|---|---|
| Trigger | POST Webhook |
| Human check | Cloudflare Turnstile |
| Rate limit | Placeholder (replace) |
| Validation | Deterministic Code node |
| Max size | 500 rows, 20 columns, 1000 characters per cell |
| Max issues returned | 500 |
| AI | Groq via LangChain chain + structured parser |
| AI input | Counts and up to 15 examples, no values |
| AI failure | Returns validation with empty summary |
| Output | JSON (summary, byType, issues, cleaned, aiSummary, topFixes) |
| Email | Optional Gmail copy, sent after the response |
| Storage | None |
| Timezone | Asia/Manila |

---

# 🧱 Node Flow Summary

```text
Webhook
  │
  ▼
turnstile-verify
  │
  ▼
turnstile ok?
 ├── true ──► rate-limit (replace me) ──► within limit?
 │                                          ├── true ──► validate-csv
 │                                          │              ├── success ──► AI summary (Groq Chat Model + Output parser)
 │                                          │              │                    │
 │                                          │              │                    ▼
 │                                          │              │                  merge
 │                                          │              │                    ├── success ──► Respond OK ──► build email ──► Send email
 │                                          │              │                    └── error ────► Respond error
 │                                          │              └── error ────► Respond invalid_input
 │                                          └── false ──► Respond 429 (rate limit)
 └── false ──► Respond 403 (human check)
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

This workflow shows a safe, production-minded AI pattern: **plain code decides, AI only explains**.

```text
Webhook API Design
+
Bot Protection (Turnstile)
+
Rate Limiting
+
Deterministic Data Validation
+
Input Guards
+
Privacy-Aware AI Summary
+
Graceful Failure Handling
+
Structured Output Parsing
+
Transactional Email
```

Relevant for **data quality, API-backed automation, AI workflow engineering and privacy-conscious design**.

---

# 📄 One-Sentence Project Summary

> **An n8n webhook workflow that validates and cleans uploaded CSV files with deterministic code, adds a privacy-safe Groq AI summary, and returns the results as JSON with an optional email copy.**
