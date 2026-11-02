# 🕸️ Chatbot Web Scraper

## 📌 Description

**Chatbot Web Scraper** is an n8n webhook workflow that reads a public web page, returns an AI summary with topics and suggested questions, and then answers follow-up questions using only that page's text.

---

## 🎯 Problem

Reading a long web page to find its main points and specific answers takes time, and general chatbots often answer from memory instead of from the page itself.

---

## 💡 Solution

One guarded webhook with two actions: `scrape_page` safely fetches the page (with robots.txt respect and a Google-snippets fallback) and summarizes it, while `ask_page` answers questions from the page text only and keeps only quotes that really appear on the page.

---

## 🧰 Tools

n8n, Cloudflare Turnstile, Google Gemini, SerpAPI, LangChain (n8n AI nodes), Gmail, JavaScript (Code node), Webhook

---

## ✨ Features

Single webhook with two actions, Forgiving URL cleaning (plain URL, Markdown link, URL in a sentence, bare domain), Public-website-only address vetting, Blocking of localhost and private addresses, Robots.txt respect, No redirect following, Direct page fetch with an identified bot user agent, HTML to clean text conversion, Main-content detection, Reference-section trimming, 60,000-character content cap, Google snippets fallback through SerpAPI, AI page summary with topics, Suggested questions including one the page cannot answer, Follow-up question answering from page text only, Source quotes verified against the page, Honest "Not on this page" results, Conversation history support, Human check with Cloudflare Turnstile, Hashed IP rate limiting, Email rate limiting, Tool-less AI agents, Retry on AI failure, Structured JSON output with auto-fix, Server-side output validation, Optional result email, HTML-escaped email output, No page or question storage, Consistent HTTP 200 responses for the page, Email sent after the response

---

## 👥 Who Will Use This?

- Researchers and students reading long articles
- Content and SEO teams reviewing competitor pages
- Support teams checking documentation pages
- Developers building page-aware chatbots
- Automation consultants and agencies
- Anyone showing a safe, guarded web-scraping AI demo

---

# ⚙️ Workflow Overview

```text
Webhook (POST)
      ↓
Clean input
      ↓
Input valid? ── no ──► Respond: invalid_input
      ↓ yes
Switch on action
 ├── scrape_page
 │     ↓
 │   Verify Turnstile ─► Human check passed? ── no ──► Respond: blocked
 │     ↓
 │   Rate limit: scrape ─► Within limit? ── no ──► Respond: rate_limited
 │     ↓
 │   Vet address ─► URL ok? ── no ──► Respond: page error
 │     ↓
 │   Fetch robots.txt ─► Robots check ─► Robots ok? ── no ──► Respond: page error
 │     ↓
 │   Fetch page ─► Page text ─► Page readable?
 │                                 ├── yes ─────────────────────────┐
 │                                 └── no ─► Try Google snippets?   │
 │                                              ├── yes ─► SerpAPI ─► Snippets to text ─► Snippets usable?
 │                                              │                                          ├── yes ─┐
 │                                              │                                          └── no ─► Respond: page error
 │                                              └── no ─► Respond: page error              │
 │                                                                                         ▼
 │                                                              Page ready ◄───────────────┘
 │     ↓
 │   Summary Agent (Gemini + Summary Parser)
 │     ↓
 │   Validate summary ─► Summary ok? ── no ──► Respond: error
 │     ↓ yes
 │   Respond: page read ─► Email requested? ─► Build email ─► Send result email
 │
 └── ask_page
       ↓
     Rate limit: ask ─► Within limit? ── no ──► Respond: rate_limited
       ↓
     Answer Agent (Gemini + Answer Parser)
       ↓
     Validate answer ─► Answer ok? ── no ──► Respond: error
       ↓ yes
     Respond: answer
```

---

# 🧩 Workflow Components

| Node | Purpose |
|---|---|
| `Webhook` | Receives the request (POST) and limits allowed origins |
| `Clean input` | Cleans and validates both actions, forgives messy URLs, reads the caller IP |
| `Input valid?` | Sends bad input to the `invalid_input` response |
| `Switch on action` | Routes to `scrape_page` or `ask_page` |
| `Verify Turnstile` | Verifies the Cloudflare Turnstile token (scrape only) |
| `Human check passed?` | Continues on success, otherwise sends `blocked` |
| `Rate limit: scrape` | Counts scrape requests per hashed IP, overall, and emails |
| `Within limit?` | Continues if under the limit, otherwise sends `rate_limited` |
| `Vet address` | Blocks anything that is not a normal public website |
| `URL ok?` | Sends blocked addresses to `Respond: page error` |
| `Fetch robots.txt` | Reads the site's robots.txt (5 second timeout) |
| `Robots check` | Stops if the path is disallowed for all agents |
| `Robots ok?` | Routes disallowed pages to `Respond: page error` |
| `Fetch page` | Downloads the page (10 second timeout, no redirects) |
| `Page text` | Converts HTML to plain text and decides if a fallback is worth trying |
| `Page readable?` | Continues if text was extracted |
| `Try Google snippets?` | Uses the fallback only for blocked, busy, broken or empty pages |
| `SerpAPI: site search` | Gets Google results for `site:host/path` |
| `Snippets to text` | Builds page text from same-site result snippets |
| `Snippets usable?` | Continues if enough snippet text was found |
| `Page ready` | No-Op merge point for both reading routes |
| `Gemini Flash` | Language model (temperature 0.4) behind both agents and parsers |
| `Summary Agent` | Writes the summary, topics and suggested questions |
| `Summary Parser` | Forces the summary JSON schema (auto-fix on) |
| `Validate summary` | Checks and caps the summary fields, builds the result |
| `Summary ok?` | Routes unusable summaries to `Respond: error` |
| `Respond: page read` | Returns `{ ok: true, result }` for a scrape |
| `Email requested?` / `Build email` / `Send result email` | Optional HTML-escaped result email through Gmail |
| `Rate limit: ask` / `Within limit? (ask)` | Per-IP and overall limit for questions |
| `Answer Agent` | Answers a question from the page text only |
| `Answer Parser` | Forces the answer JSON schema (auto-fix on) |
| `Validate answer` | Keeps only source quotes that appear on the page |
| `Answer ok?` | Routes unusable answers to `Respond: error` |
| `Respond: answer` | Returns `{ ok: true, result }` for a question |
| `Respond: invalid_input` / `blocked` / `rate_limited` / `page error` / `error` | Friendly error replies (all HTTP 200) |
| `Note: ...` sticky notes | Setup, URL handling, limits, network safety, fallback and error codes |

---

# 🔄 Detailed Workflow Process

## 1. Trigger: `Webhook`

A POST webhook with `responseMode: responseNode`. Allowed origins: the demo site and `http://localhost:5173`. One webhook serves both actions.

Scrape request:

```json
{
  "action": "scrape_page",
  "source": "testarosa-playground",
  "url": "https://example.com/article",
  "notifyEmail": "optional@example.com",
  "turnstileToken": "<token>"
}
```

Ask request (the page sends the text back with every question, so nothing is stored):

```json
{
  "action": "ask_page",
  "source": "testarosa-playground",
  "url": "https://example.com/article",
  "content": "<page text returned by scrape_page>",
  "question": "What is the main argument?",
  "history": [{ "role": "user", "text": "..." }, { "role": "assistant", "text": "..." }]
}
```

---

## 2. Input Cleaning: `Clean input` and `Input valid?`

| Field | Rule |
|---|---|
| `source` | Must be `testarosa-playground` |
| `url` | Cleaned, then must look like `http(s)://host.tld`, cut to 300 characters |
| `notifyEmail` | Optional, lowercased, max 254 characters. A bad address is ignored and the page is still read |
| `content` (ask) | Must not be empty, `<page>` tags removed, cut to 60,000 characters |
| `question` (ask) | 3 to 300 characters |
| `history` (ask) | Last 6 turns, each cut to 1000 characters, `<page>` tags removed |

The URL cleaner accepts a plain URL, a Markdown link, a URL inside a sentence, or a bare domain such as `example.com` (https is added). It removes invisible characters and trailing punctuation. The caller IP is read from `cf-connecting-ip`, `x-forwarded-for` or `x-real-ip`.

---

## 3. Human Check (scrape only): `Verify Turnstile` and `Human check passed?`

The token is sent to Cloudflare `siteverify` (continues on error). If `success` is not `true`, the workflow answers `blocked`. Turnstile runs on `scrape_page` only, because a token works once; `ask_page` is protected by its rate limit instead.

---

## 4. Rate Limits: `Rate limit: scrape`, `Rate limit: ask`

Counters live in the workflow's static data. The caller IP is **hashed** before it is stored, and the raw IP is dropped from the data afterward.

| Limit | Scrape (in code) | Ask (in code) |
|---|---|---|
| Per visitor per hour | 50 | 30 |
| Overall per hour | 100 | 150 |
| Emails per address per day | 50 | n/a |
| Emails per day overall | 100 | n/a |

- Over a request limit: `rate_limited` response.
- Over an email limit: the page is still read, only the email is skipped.
- Static data persists only for **active** (production) runs.
- Numbers are at the top of the two Rate limit nodes.

---

## 5. Address Vetting: `Vet address` and `URL ok?`

Only normal public websites are allowed. The node rejects:

- Usernames or passwords in the address
- Ports other than 80 and 443
- `localhost`, `.local` and `.internal` hosts
- Private and loopback IPv4 ranges, link-local and shared ranges
- Hosts with no dot, and raw numeric IPs

Otherwise it outputs a clean URL, origin, host and path. No `new URL()` is used, so it works in every Code-node sandbox.

---

## 6. Robots.txt: `Fetch robots.txt`, `Robots check`, `Robots ok?`

The site's `/robots.txt` is read first (no redirects, 5 second timeout). If it disallows the requested path for `User-agent: *`, the workflow answers `robots_disallowed` and does not fetch the page. A missing or unreadable robots.txt does not block the request. The check is a simple parser, so review it if you need full robots.txt rules such as `Allow` lines or named agents.

---

## 7. Page Fetch and Text: `Fetch page`, `Page text`, `Page readable?`

The page is fetched with the user agent `TestarosaDemoBot/1.0`, no redirects, and a 10 second timeout.

| Result | Behavior |
|---|---|
| 2xx with text | HTML converted to plain text (scripts, styles, nav, footer, header, forms and aside removed), main or article area preferred, reference sections trimmed, capped at 60,000 characters |
| Redirect (3xx) | `fetch_failed`, asks for the final address, no fallback |
| 401, 403, 429, 451, 5xx or timeout | `fetch_failed`, fallback allowed |
| Other errors such as 404 | `fetch_failed`, no fallback |
| Under 80 characters of text | `no_content` (may need JavaScript or a login), fallback allowed |

The result also holds the title, meta description, word count and a `truncated` flag.

---

## 8. Google Snippets Fallback: `Try Google snippets?`, `SerpAPI: site search`, `Snippets to text`, `Snippets usable?`

Only when the page could not be read directly and a retry makes sense. SerpAPI searches `site:host/path` (10 results, 60 second timeout). The workflow keeps up to 8 same-site results and builds text from their titles, snippets and links. It needs at least 80 characters, otherwise it answers `no_content`.

Because snippets are short, the summary is shallower and says so ("Read from Google search snippets...").

---

## 9. AI Summary: `Summary Agent`, `Validate summary`

A tool-less n8n AI Agent with Gemini, a structured output parser (auto-fix) and up to 3 tries with a 1.5 second wait. The page text is wrapped in `<page>` tags and treated as data, never instructions.

```json
{
  "summary": "2 to 3 plain sentences about what the page is",
  "topics": ["2 to 6 short topic labels"],
  "suggestedQuestions": ["3 or 4 questions, one closely related but NOT answered by the page"]
}
```

`Validate summary` rejects empty summaries, caps lengths (summary 800 characters, 6 topics, 4 questions) and builds the result with URL, title, description, word count, `truncated` flag and the page content. Failures go to `Respond: error`.

---

## 10. Scrape Response and Optional Email: `Respond: page read`, `Email requested?`, `Build email`, `Send result email`

`Respond: page read` answers the page first with `{ ok: true, result }`. Only then does the email branch run, so a slow or failed email never delays the page. The email shows title, URL, word count, summary, topics and suggested questions. All text from the page or the model is HTML-escaped. The email is sent with Gmail as "Testarosa - Playground".

---

## 11. Question Answering: `Answer Agent`, `Validate answer`, `Respond: answer`

A tool-less AI Agent (same Gemini model, auto-fix parser, up to 3 tries). The prompt contains the last turns of the conversation, the page text and the question. The agent must use **only** the page text and set `found: false` if the answer is not there.

```json
{
  "answer": "Short direct answer",
  "found": true,
  "sources": ["Up to 3 passages copied word for word from the page"]
}
```

`Validate answer` keeps only source quotes that really appear on the page (compared ignoring case and spacing). If the model claims `found: true` but no valid quote remains, `found` becomes `false`, so the page shows **Not on this page**.

---

## Response codes

All responses use **HTTP 200** because the page expects it.

| Code | When |
|---|---|
| `ok: true` | Page read, or question answered |
| `invalid_input` | Bad address, question or source |
| `blocked` | Turnstile failed |
| `rate_limited` | Over the hourly cap |
| `blocked_url` | Address is not a normal public website |
| `robots_disallowed` | The site's robots.txt disallows the page |
| `fetch_failed` | The site did not answer, redirected or returned an error |
| `no_content` | Almost no readable text, and the Google fallback found little |
| `error` | The AI reply was unusable or the agent failed |

---

# 🔌 Integrations

## n8n
Runs the webhook, guards, branching, fetching, AI agents and responses.

## Cloudflare Turnstile
Human verification before scraping.

## Google Gemini
Language model for the summary and answers.

## SerpAPI
Google result snippets as a fallback when a page cannot be fetched.

## Gmail
Sends the optional result email.

---

# 🔐 Credentials & Configuration

| Item | Where |
|---|---|
| Google Gemini credential | Node `Gemini Flash` |
| Turnstile **secret** key (not the site key) | Node `Verify Turnstile` (replace `<PUT_CREDS_HERE>`) |
| SerpAPI key | Node `SerpAPI: site search`, `api_key` parameter (replace `<PUT_CREDS_HERE>`) |
| Gmail credential | Node `Send result email` |
| Allowed Origins | Node `Webhook` > Options |
| Front-end variable | Put the **production** URL (`/webhook/...`) in `VITE_SCRAPER_WEBHOOK_URL`, then redeploy |
| Rate-limit numbers | Top of the two `Rate limit` nodes |

---

# 🛠️ Configuration Checklist

- [ ] Import the workflow JSON into n8n (Workflows > Import from File).
- [ ] Attach the Gemini credential to `Gemini Flash`.
- [ ] Add the Turnstile secret key to `Verify Turnstile`.
- [ ] Add the SerpAPI key to `SerpAPI: site search`, or remove the fallback branch.
- [ ] Attach the Gmail credential to `Send result email`.
- [ ] Check the Webhook Allowed Origins.
- [ ] Review the rate-limit numbers in both Rate limit nodes.
- [ ] Block n8n's outbound traffic to private ranges at network level (see notes).
- [ ] Optionally set an error workflow in Settings.
- [ ] Test the cases in the table below.
- [ ] Activate the workflow.
- [ ] Copy the production URL into `VITE_SCRAPER_WEBHOOK_URL` and redeploy.

---

# ⚠️ Important Configuration Notes

### Network safety before going public
A hostname can still resolve to a private address through DNS, and the address check cannot see that. If your n8n host can reach private services, **block outbound traffic from n8n to private ranges and to the cloud metadata address `169.254.169.254` at network level**.

### Redirects are not followed
Users must enter the final address. This keeps the address check meaningful.

### Page content returns to the browser
The scrape result includes the page text (up to 60,000 characters), and the page sends it back with each question. This is why nothing needs to be stored, but it makes requests larger.

### Keep the source-quote check
`Validate answer` is what turns "the AI says it found it" into "the page really contains it". Do not remove it.

### Snippet fallback is shallower
Summaries made from Google snippets cover less than a direct read, and say so.

### Rate limit storage
Counters reset when the workflow is re-imported or deactivated. For a shared or persistent budget, use an external store.

### Sticky note numbers
The sticky note on the canvas lists different rate-limit numbers from the Code nodes. The Code nodes decide the real limits.

---

# 🚀 How to Run

1. Open the Chatbot Web Scraper page.
2. Enter a public web address (a plain URL, a Markdown link or just a domain works).
3. Optionally enter an email to receive the summary.
4. Pass the human check and submit.
5. The workflow vets the address, checks robots.txt, fetches the page and summarizes it.
6. The page shows the summary, topics and suggested questions.
7. Ask follow-up questions. Each answer comes only from the page text and shows its source quotes, or says it is not on the page.

### Test cases

| Test | Expected result |
|---|---|
| A normal article URL | Summary, topics and suggested questions |
| `example.com` (bare domain) | `https://` added, page read |
| A Markdown link or a URL inside a sentence | URL extracted and read |
| `http://localhost/...` or a private IP | `blocked_url` |
| A page disallowed in robots.txt | `robots_disallowed` |
| A page that returns 403 or needs JavaScript | Google snippets fallback, or `no_content` |
| A question the page does not answer | `found: false`, "Not on this page" |
| A page that contains instructions aimed at the AI | Ignored, page described normally |

---

# 🧯 Troubleshooting

| Symptom | Likely cause |
|---|---|
| `blocked` every time | Wrong Turnstile secret, or token missing |
| `rate_limited` | Over the hourly request limit |
| `blocked_url` | Local, private, numeric or non-standard-port address |
| `robots_disallowed` | The site asks scrapers not to read that path |
| `fetch_failed` with a redirect message | Enter the final address instead |
| `no_content` | Page needs JavaScript or a login, and the fallback found little |
| Fallback never works | SerpAPI key missing or out of credits |
| `error` every time | Check the agent execution, usually the Gemini credential |
| CORS error in browser | Page origin is not in the Webhook's Allowed Origins |
| No email received | Email invalid, email caps reached, or Gmail credential missing |

---

# 💰 Estimated Business Value

The numbers below are **estimates**, not measured results.

### Manual work per page

Opening a page, reading it, writing a short summary, listing topics and searching it for specific answers typically takes:

```text
5–10 minutes per page
```

The workflow returns a summary in seconds, and follow-up answers in seconds each. Net saving is estimated at 4–8 minutes per page.

### Estimated Impact

| Metric | Estimate |
|---|---:|
| Manual time per page | 5–10 min |
| Estimated Time Saved (per page) | 4–8 min |
| Pages processed per month (assumption) | 150 |
| **Estimated Time Saved (monthly)** | **~10–20 hours** |
| Illustrative labor value | $15–$30/hour |
| **Potential Savings (Monthly)** | **$150–$600** |

> **Note:** Savings depend on page length, how many follow-up questions are asked, labor cost and monthly volume. They do not include the time saved by quickly judging that a page is not relevant.

---

# 📈 Scalability & Recommended Improvements

- Handle JavaScript-rendered pages with a headless browser service
- Add PDF and document support
- Read several pages from one site and answer across them
- Add a persistent rate limit and an allow-list of domains
- Resolve hostnames and re-check the IP before fetching
- Use a fuller robots.txt parser (Allow rules, named agents, crawl delay)
- Cache results for popular pages
- Add citations with positions in the page
- Support more languages
- Add a model fallback if Gemini is unavailable

---

# 🔒 Data & Operational Considerations

- Only read pages you are allowed to read. Respect each site's terms of service and robots.txt.
- Page text and questions are sent to the AI provider (Google Gemini) for processing.
- When the fallback runs, the site address is sent to SerpAPI and Google results are used.
- The workflow does not store page text or questions in a database. Check the n8n execution-data settings if you do not want them kept in n8n.
- IP addresses are hashed before they are stored in the rate-limit data.
- Email content is HTML-escaped and only sent to an address the visitor typed.
- AI summaries can miss or misread things. Verify before relying on them.
- Monitor Gemini, SerpAPI, Gmail and n8n usage.

---

# 📊 Current Workflow Characteristics

| Capability | Current Implementation |
|---|---|
| Trigger | POST Webhook, two actions |
| Input guard | Code node (URL cleaning, source, lengths, email) |
| Human check | Cloudflare Turnstile (scrape only) |
| Rate limit | Hashed IP, overall and email counters in static data |
| Address safety | Public websites only, ports 80 and 443 |
| Fetching | Direct fetch, robots.txt first, no redirects |
| Fallback | SerpAPI Google snippets |
| Content cap | 60,000 characters |
| AI | Two tool-less n8n AI Agents with Google Gemini |
| AI output | Structured JSON with auto-fix and retries |
| Post-AI rules | Field caps, source quotes verified against the page |
| Output | JSON `{ ok, result }` |
| Email | Optional Gmail copy for scrapes, sent after the response |
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
 ├── false ──► Respond: invalid_input
 └── true ──► Switch on action
               ├── scrape_page ──► Verify Turnstile ──► Human check passed?
               │                     ├── false ──► Respond: blocked
               │                     └── true ──► Rate limit: scrape ──► Within limit?
               │                                    ├── false ──► Respond: rate_limited
               │                                    └── true ──► Vet address ──► URL ok?
               │                                                   ├── false ──► Respond: page error
               │                                                   └── true ──► Fetch robots.txt ──► Robots check ──► Robots ok?
               │                                                                  ├── false ──► Respond: page error
               │                                                                  └── true ──► Fetch page ──► Page text ──► Page readable?
               │                                                                                 ├── true ──► Page ready
               │                                                                                 └── false ──► Try Google snippets?
               │                                                                                                ├── false ──► Respond: page error
               │                                                                                                └── true ──► SerpAPI: site search ──► Snippets to text ──► Snippets usable?
               │                                                                                                               ├── true ──► Page ready
               │                                                                                                               └── false ──► Respond: page error
               │
               │     Page ready ──► Summary Agent (Gemini Flash + Summary Parser)
               │                      ├── success ──► Validate summary ──► Summary ok?
               │                      │                                      ├── true ──► Respond: page read ──► Email requested? ──► Build email ──► Send result email
               │                      │                                      └── false ──► Respond: error
               │                      └── error ────► Respond: error
               │
               └── ask_page ──► Rate limit: ask ──► Within limit? (ask)
                                  ├── false ──► Respond: rate_limited
                                  └── true ──► Answer Agent (Gemini Flash + Answer Parser)
                                                 ├── success ──► Validate answer ──► Answer ok?
                                                 │                                    ├── true ──► Respond: answer
                                                 │                                    └── false ──► Respond: error
                                                 └── error ────► Respond: error
```

---

# 📦 Requirements

- n8n (with AI / LangChain nodes)
- Cloudflare Turnstile site key and secret
- Google Gemini API credential
- SerpAPI account and key (for the fallback)
- Gmail account (OAuth2) for the optional email
- A front-end page that posts to the webhook
- Network rules that block n8n from reaching private ranges (recommended)

---

# 📌 Portfolio Value

This workflow shows how to let an AI read untrusted web content safely: **vet the address, respect robots.txt, treat the page as data, and verify every quote**.

```text
Webhook API Design (Multi-Action)
+
Input Sanitization
+
SSRF-Aware Address Vetting
+
Robots.txt Compliance
+
Web Page Fetching and HTML-to-Text
+
Fallback Data Source (SerpAPI)
+
Prompt-Injection Defense
+
Bot Protection and Hashed-IP Rate Limiting
+
Tool-less AI Agents with Structured Output
+
Grounded Answers with Verified Quotes
+
Transactional Email
```

Relevant for **web data extraction, retrieval-grounded AI, AI safety in automation, API-backed workflows and AI workflow engineering**.

---

# 📄 One-Sentence Project Summary

> **An n8n webhook workflow that safely reads a public web page, summarizes it with Google Gemini (with a Google-snippets fallback), and answers follow-up questions using only the page text with source quotes verified against the page.**
