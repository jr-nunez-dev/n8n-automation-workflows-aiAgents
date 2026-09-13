# 🟢 AI Lead Generation & Enrichment

## 📌 Description

**AI Lead Generation & Enrichment** is an n8n-powered prospecting workflow that automatically discovers LinkedIn profiles matching a target **Position** and **Location**, evaluates each candidate with an AI Scoring Agent, enriches high-priority leads with verified contact emails, generates personalized outreach emails, stores everything in a live Google Sheets database, and automatically notifies the team through Slack and Gmail the moment a lead is ready for outreach.

The workflow is designed as a lightweight, form-driven sourcing pipeline for recruiters, sales teams, agencies, and automation consultants who need a repeatable way to turn a job title and location into a structured, pre-qualified prospect list.

---

## 🎯 Problem

Manual LinkedIn sourcing is repetitive and time-consuming: a person has to search LinkedIn, open dozens of profiles, judge whether each candidate is actually worth pursuing, hunt for a working email address, draft a personalized opening message, record everything in a spreadsheet, and then notify the right people once something urgent is found.

This workflow removes most of that repetitive preparation by connecting lead discovery, AI qualification, contact enrichment, data storage, and notification into one automated process.

---

## 💡 Solution

The workflow uses a simple n8n form as its trigger, captures a **Position** and **Location**, uses **Apify's LinkedIn Profile Search Scraper** to discover matching profiles, limits the result set to ten leads for controlled processing, evaluates each lead individually with an AI Scoring Agent using GPT-4.1 through OpenRouter, enriches every lead with a verified email address via a second Apify actor, saves the fully structured record to a Google Sheets lead database, routes each lead by its AI classification (HOT / WARM / COLD), drafts a personalized outreach email for every HOT lead, and notifies the team through Slack (HOT leads) and Gmail (WARM leads).

---

## 👥 Who Will Use This?

This workflow is useful for:

- Recruiters and talent acquisition teams
- Sales development representatives (SDRs/BDRs)
- Business development teams
- Recruitment and sourcing agencies
- Automation consultants
- Freelance recruiters and lead-gen consultants
- Growth and RevOps teams
- Agency owners and founders
- Anyone building a LinkedIn-based prospecting or candidate-sourcing system

---

# ⚙️ Workflow Overview

The automation follows this general pipeline:

```text
Form Submission (Position, Location)
          ↓
   Apify LinkedIn Search
          ↓
    Parse Raw Lead Data
          ↓
     Limit to 10 Leads
          ↓
  Process One Lead at a Time
          ↓
    Aggregate Lead Data
          ↓
  AI Scoring Agent + GPT-4.1
          ↓
     Parse AI JSON
          ↓
      Map Lead Fields
          ↓
  Apify Email Enrichment
          ↓
   Save Lead to Google Sheets
          ↓
   Loop Until Finished
          ↓
     Switch (HOT / WARM / COLD)
    ↙            ↓            ↘
  HOT           WARM          COLD
   ↓              ↓             ↓
Generate AI      Email Team   No Further
Email + Update    (Gmail)      Action
Sheet + Slack
Notification
```

---

# 🧩 Workflow Components

| Node | Purpose |
|---|---|
| `On form submission` | Captures the target `Position` and `Location` from the user |
| `Get leads` | Runs the Apify LinkedIn Profile Search Scraper |
| `Parse Leads Data` | Normalizes raw scraped profile fields into a consistent structure |
| `10 Limits` | Restricts the lead set to a maximum of ten profiles |
| `Loop Over Items` | Processes leads one at a time and controls the scoring/enrichment loop |
| `Aggregate Data` | Aggregates the current lead's data before AI scoring |
| `Scoring Agent` | AI Agent that classifies the lead as HOT, WARM, or COLD |
| `GPT-4.1` | Language model (via OpenRouter) powering the Scoring Agent |
| `Parse Data` | Converts the Scoring Agent's JSON string into usable n8n JSON |
| `Map Data` | Combines the original profile fields with the AI scoring results |
| `Look For Email` | Runs a second Apify actor to enrich the lead with a verified email |
| `Enrich Data` | Merges the enrichment result with the full lead record |
| `Save Data` | Appends or updates the lead in the Google Sheets database |
| `Switch` | Routes each lead down the HOT, WARM, or COLD path |
| `COLD Leads` | No-operation endpoint — cold leads receive no further action |
| `Generate Email Alert` | Builds a team notification listing all WARM leads |
| `Send Email Alert` | Sends the WARM lead digest via Gmail |
| `For Each HOT Lead` | Processes HOT leads one at a time for email generation |
| `Aggregate` | Aggregates a single HOT lead's data before email generation |
| `Email Agent Generator` | AI Agent that drafts a personalized outreach email |
| `GPT` | Language model (via OpenRouter) powering the Email Agent Generator |
| `Parse Sample Email` | Converts the Email Agent's JSON string into usable n8n JSON |
| `Map Email Data` | Prepares the subject/body/id fields for the sheet update |
| `Update HOT Leads` | Updates the existing lead row with the generated email draft |
| `Generate Slack Message` | Builds a summary of all HOT lead email drafts |
| `Send Executive Message` | Posts the HOT lead summary to a Slack channel |

---

# 🔄 Detailed Workflow Process

## 1. Form Trigger — `On form submission`

The workflow starts with a simple n8n form containing two required fields:

| Field | Example |
|---|---|
| `Position` | Digital Marketing Manager |
| `Location` | Metro Manila |

These two values drive the entire search and become the workflow's only required input — no code or spreadsheet setup is needed to kick off a new sourcing run.

---

# 🔎 2. Lead Discovery — `Get leads`

The workflow uses **Apify's LinkedIn Profile Search Scraper** (`harvestapi/linkedin-profile-search`) to find matching profiles.

Configured request:

```javascript
{
  searchQuery: Position,
  locations: [Location],
  maxItems: 15,
  profileScraperMode: "Full + email search"
}
```

The actor run is capped at a maximum charge of:

```text
$0.50 per execution
```

This keeps a single sourcing run predictable and low-cost while still returning a healthy batch of candidate profiles.

---

# 🧹 3. Raw Data Normalization — `Parse Leads Data`

LinkedIn profile data returned by Apify is nested and inconsistent between profiles. This Code node flattens it into a clean, predictable shape:

```javascript
{
  id,
  fullName,
  linkedinUrl,
  openToWork,
  headline,
  location,
  currentRole,
  currentCompany,
  about,
  topSkills
}
```

This standardized structure is what every downstream node (AI scoring, enrichment, Google Sheets) relies on.

---

# 🔢 4. Lead Limit — `10 Limits`

Although the scraper can return up to 15 profiles, the workflow intentionally limits downstream processing to **10 leads**.

This is useful for:

- Controlled testing
- Reducing AI and enrichment processing costs
- Preventing oversized batches
- Keeping HOT lead review manageable
- Testing scoring accuracy before scaling up

The limit can be increased once the workflow is validated for production use.

---

# 🔁 5. One-Lead-at-a-Time Processing — `Loop Over Items`

The workflow uses a batching/loop mechanism to score and enrich leads individually:

```text
Lead 1 → Score → Enrich → Save
Lead 2 → Score → Enrich → Save
Lead 3 → Score → Enrich → Save
```

rather than attempting to process the entire batch as one operation. This keeps each AI evaluation isolated and reliable, and ensures every lead — regardless of eventual ranking — is enriched and recorded before routing decisions are made.

Once every lead has passed through, the loop's "done" output releases the full, scored batch to the `Switch` node for routing.

---

# 🤖 6. AI Lead Scoring — `Scoring Agent`

The Scoring Agent is the core qualification layer. It receives a single lead's aggregated profile JSON and evaluates it against strict, explainable criteria.

### Scoring Criteria

- **HOT** — `openToWork` **must** be `true`, and the headline or current role directly matches the target skill needs, with a clear, verified skill set.
- **WARM** — `openToWork` is `true` but the role/skills are only generically aligned, **or** `openToWork` is `false` but the candidate has elite, high-value skills worth nurturing.
- **COLD** — `openToWork` is `false` with no clear skill alignment, **or** key profile fields are missing, making evaluation unreliable.

### Additional Outputs

- **Icebreaker** — a 2–3 sentence, personalized outreach hook based on the candidate's headline, role, or location.
- **Confidence Level** — a 1–10 score reflecting how complete the candidate's profile data is.

---

# 🧠 7. LLM — `GPT-4.1`

The Scoring Agent is connected to an OpenRouter Chat Model node configured as:

```text
gpt-4.1
```

Architecture:

```text
Aggregated Lead JSON
        ↓
   Scoring Agent
        ↓
      GPT-4.1
        ↓
Structured Scoring JSON
```

---

# 🧹 8. AI Output Parsing — `Parse Data`

The Scoring Agent returns its result as a JSON string (sometimes wrapped in markdown code fences). This Code node safely strips and parses it:

```javascript
const rawOutput = inputData.text || inputData.output || inputData;
const cleanedText = rawOutput.replace(/```json\n?|\n?```/g, '').trim();
const parsed = JSON.parse(cleanedText);

return [{
  json: {
    leadRanking: parsed.leadRanking || 'COLD',
    reasoning: parsed.reasoning || '',
    icebreaker: parsed.icebreaker || '',
    confidenceLevel: Number(parsed.confidenceLevel) || 0
  }
}];
```

This guarantees the downstream Switch node always receives a valid `leadRanking`, defaulting safely to `COLD` if parsing ever fails.

---

# 🗂️ 9. Field Mapping — `Map Data`

This Set node reassembles the complete lead record by pulling the original profile fields (from `Aggregate Data`), the cleaned skill list (from `Parse Leads Data`), and the AI scoring results (from `Parse Data`) into a single, flat object ready for enrichment and storage.

---

# 📧 10. Email Enrichment — `Look For Email`

Every lead — regardless of ranking — is passed through a second Apify actor (`datadoping/linkedin-profile-scraper`) to look up a verified email address using the lead's LinkedIn URL:

```javascript
{
  "profiles": ["<linkedinUrl>"]
}
```

If no email can be found, the `Enrich Data` node marks the field as:

```text
Not Found
```

Enriching every lead up front (rather than only HOT leads) means the full dataset in Google Sheets is always outreach-ready, even if a WARM lead is later promoted for manual follow-up.

---

# 📊 11. Lead Database — `Save Data`

The structured lead is stored in Google Sheets using an **Append or Update** operation, matched on the `id` column.

| Column | Source |
|---|---|
| `id` | LinkedIn profile ID |
| `fullName` | Parsed profile data |
| `linkedinUrl` | Parsed profile data |
| `openToWork` | Parsed profile data |
| `headline` | Parsed profile data |
| `location` | Parsed profile data |
| `currentRole` / `currentCompany` | Parsed profile data |
| `about` / `topSkills` | Parsed profile data |
| `leadRanking` / `reasoning` / `icebreaker` / `confidenceLevel` | AI Scoring Agent |
| `email` / `urn` | Email enrichment |
| `subject` / `emailBody` | Filled in later for HOT leads |

Matching on `id` prevents duplicate rows and allows the sheet to be safely updated as a lead moves through the pipeline (for example, once a HOT lead's email draft is generated).

---

# 🔀 12. Lead Routing — `Switch`

Once the full batch has been scored, enriched, and saved, each lead is routed based on its `leadRanking`:

```javascript
$json.leadRanking.toUpperCase() === "HOT"  → HOT path
$json.leadRanking.toUpperCase() === "WARM" → WARM path
$json.leadRanking.toUpperCase() === "COLD" → COLD path
```

---

# ❄️ 13. Cold Leads — `COLD Leads`

COLD leads are routed to a No-Operation node. No email generation, notification, or further processing occurs — the record simply remains in Google Sheets for reference.

---

# 🔥 14. Warm Leads — `Generate Email Alert` + `Send Email Alert`

WARM leads are bundled into a single team notification rather than receiving individual AI-generated outreach:

```javascript
subject = `New Warm Leads Identified (${count})`
emailBody = `Hi Team, ... Warm Leads: • Name 1 • Name 2 ...`
```

This is sent via Gmail to the configured team address, prompting a manual review and nurturing decision.

---

# ✉️ 15. Hot Lead Email Generation — `For Each HOT Lead` → `Email Agent Generator`

HOT leads are processed one at a time through a dedicated AI Agent that writes a fully personalized outreach email.

### Email Agent Guidelines

- **Tone:** Professional, warm, respectful — never robotic or transactional.
- **Structure:** Personalized subject line, warm introduction using the candidate's first name, a specific hook drawn from their headline/about/current role/icebreaker, a value proposition, and a low-pressure call to action.
- **Length:** Under 200 words, with clean paragraph breaks.

### Expected Output Schema

```json
{
  "subject": "<Personalized Subject Line>",
  "emailBody": "<Full Email Draft with line breaks>"
}
```

The `Parse Sample Email` Code node safely parses this JSON, and `Map Email Data` prepares it (with the lead's `id`) for the sheet update.

---

# 🗂️ 16. Updating the Record — `Update HOT Leads`

The `subject` and `emailBody` generated for each HOT lead are written back into the same Google Sheets row (matched on `id`), so the lead's full journey — from raw profile to ready-to-send email — lives in one place.

---

# 📢 17. Executive Notification — `Generate Slack Message` + `Send Executive Message`

Once every HOT lead in the batch has an email draft, the workflow compiles a summary:

```text
🚀 New Hot Lead Outreach Ready!

We have finalized personalized email drafts for 3 high-priority leads.

1. Candidate ID: `abc123`
📌 Subject: "..."
...
```

This is posted to a configured Slack channel (`n8n-automations`), giving the team immediate visibility into which HOT leads are ready for review and outreach.

---

# 🧠 AI Processing Schemas

### Scoring Agent Output

```json
{
  "leadRanking": "HOT | WARM | COLD",
  "reasoning": "Brief explanation for the lead rank",
  "icebreaker": "2-3 sentence personalized outreach message",
  "confidenceLevel": 0
}
```

### Email Agent Generator Output

```json
{
  "subject": "Personalized Subject Line",
  "emailBody": "Full email draft text with line breaks"
}
```

Both agents are instructed to output **only** valid JSON matching these schemas, which keeps downstream parsing predictable.

---

# 🔌 Integrations

## n8n

Orchestrates the entire workflow — form intake, looping, conditional routing, AI orchestration, and all third-party API calls.

## Apify

Used twice: once for LinkedIn profile discovery (`harvestapi/linkedin-profile-search`) and once for email enrichment (`datadoping/linkedin-profile-scraper`).

## OpenRouter

Provides the language models (GPT-4.1) used by both the Scoring Agent and the Email Agent Generator.

## Google Sheets

Acts as the lead database — storing profile data, AI scoring, enrichment results, and generated email drafts, with append-or-update logic to prevent duplicates.

## Gmail

Sends the WARM leads digest to the team.

## Slack

Sends the HOT leads outreach-ready summary to a designated channel.

---

# 🔐 Credentials & Configuration

Before activating the workflow, configure the required credentials.

### Apify

Required for `Get leads` and `Look For Email`. Replace the placeholder credential (`YOUR_CREDENTIAL_ID`) with your Apify account.

### OpenRouter

Required for `GPT-4.1` and `GPT` (the language models behind the Scoring Agent and Email Agent Generator).

### Google Sheets

Required for `Save Data` and `Update HOT Leads`. Replace the placeholder spreadsheet reference (`YOUR_GOOGLE_SHEET_ID`) with your own sheet.

### Gmail

Required for `Send Email Alert`. Replace the placeholder recipient (`user@example.com`) with your team's address.

### Slack

Required for `Send Executive Message`. Confirm access to the target channel (default: `n8n-automations`).

---

# 🛠️ Configuration Checklist

Before using the workflow:

- [ ] Connect Apify credentials.
- [ ] Connect OpenRouter credentials.
- [ ] Connect Google Sheets credentials.
- [ ] Connect Gmail credentials.
- [ ] Connect Slack credentials.
- [ ] Replace the placeholder Google Sheet ID.
- [ ] Confirm the `LinkedIn Leads` sheet/tab name matches.
- [ ] Replace the placeholder Gmail recipient address.
- [ ] Confirm the Slack channel is correct.
- [ ] Confirm both Apify actors are available in your Apify account.
- [ ] Confirm GPT-4.1 is available through your OpenRouter account.
- [ ] Test the workflow with a single Position/Location combination.
- [ ] Verify the AI scoring JSON output.
- [ ] Verify the Google Sheets column mappings.
- [ ] Verify Gmail and Slack notifications fire correctly.
- [ ] Increase the lead limit only after successful testing.

---

# ⚠️ Important Configuration Notes

### Current Lead Limit

The workflow limits processing to:

```text
10 leads
```

This is appropriate for testing. For production-scale sourcing, increase the limit carefully and consider Apify cost, AI token cost, and email/Slack notification volume.

### Enrichment Runs on Every Lead

The `Look For Email` enrichment step runs for **every** lead before the HOT/WARM/COLD switch, not only HOT leads. This keeps the full dataset outreach-ready but does mean enrichment cost applies to the entire batch, not just HOT candidates.

### Duplicate Handling

Both Google Sheets nodes match on the `id` column (the LinkedIn profile identifier). This provides reliable update behavior as long as Apify returns a consistent ID per profile.

### AI Output Parsing

Both `Parse Data` and `Parse Sample Email` rely on `JSON.parse()`. Both agents must return valid JSON matching their expected schema, or the parsing step will fail.

---

# 🚀 How to Run

## Step 1 — Submit the Form

Open the form and enter:

```text
Position: Digital Marketing Manager
Location: Metro Manila
```

## Step 2 — Lead Discovery

Apify searches LinkedIn for matching profiles and returns up to 15 results.

## Step 3 — Limit & Loop

The workflow limits the batch to 10 leads and processes them one at a time.

## Step 4 — AI Scoring

GPT-4.1 evaluates each lead and assigns a HOT, WARM, or COLD ranking with reasoning, an icebreaker, and a confidence score.

## Step 5 — Enrichment & Save

Each lead is enriched with a verified email (where available) and saved to Google Sheets.

## Step 6 — Routing

Once the batch is complete, leads are routed by ranking:

- **HOT** → personalized email drafted, sheet updated, Slack notification sent.
- **WARM** → bundled into a Gmail digest for the team.
- **COLD** → no further action.

## Step 7 — Review

Review HOT lead email drafts in Google Sheets or via the Slack summary before sending, and review WARM leads from the Gmail digest.

---

# 💰 Estimated Business Value

The current workflow is configured as a small, controlled batch system, processing up to **10 leads per execution**.

Because exact labor cost and manual research speed depend on the operator, the following numbers are estimates rather than measured results.

### Estimated Manual Work Per Lead

A reasonable manual sourcing process can involve:

- Searching LinkedIn for matching profiles
- Opening and reading each profile
- Judging availability and skill fit
- Searching for or guessing a contact email
- Writing an initial personalized message
- Recording everything in a spreadsheet

Estimated manual preparation:

```text
10–20 minutes per lead
```

For 10 leads:

```text
100–200 minutes per run (roughly 1.5–3.5 hours)
```

If the workflow runs a few times per week:

```text
~15–30 hours/month saved
```

### Estimated Labor Value

Using an illustrative productivity value of approximately:

```text
$15–$30/hour
```

the current configuration could represent approximately:

```text
$225–$900/month
```

in recovered labor value, plus the intangible benefit of faster response time on HOT leads.

### Estimated Current Savings

**Time Saved:** approximately **15–30 hours/month**

**Potential Labor Value Saved:** approximately **$225–$900/month**

These estimates increase substantially if the lead limit is raised and the workflow is run more frequently.

---

# 📈 Scalability Potential

The current workflow is intentionally conservative. A future production version could scale the system into:

```text
Position + Location
     ↓
LinkedIn Discovery
     ↓
Hundreds of Profiles
     ↓
Deduplication
     ↓
AI Scoring
     ↓
Email Enrichment + Verification
     ↓
CRM / ATS
     ↓
Automated Outreach Sequencing
     ↓
Response & Reply Tracking
```

Potential upgrades include:

- Dynamic multi-location or multi-role searching
- LinkedIn URN/ID-based deduplication
- Email verification before outreach
- CRM/ATS integration (HubSpot, Greenhouse, Lever, Airtable)
- Automated sending of HOT lead emails after human approval
- Follow-up sequencing for non-responders
- Telegram notifications alongside Slack/Gmail
- Lead status and pipeline-stage tracking
- Response and conversion analytics
- Configurable scoring thresholds per campaign

---

# 🧪 Example Use Case — Sales Prospecting

Suppose the form is submitted with:

```text
Position: Enterprise Account Executive
Location: Singapore
```

The workflow searches LinkedIn, discovers matching profiles, scores each one, and finds:

```text
3 HOT leads
4 WARM leads
3 COLD leads
```

The 3 HOT leads receive personalized email drafts and are saved to Google Sheets; the team is notified instantly in Slack. The 4 WARM leads are bundled into a single Gmail digest for manual review. The 3 COLD leads remain in the sheet with no further action.

The sales team opens Slack, reviews the HOT lead drafts, and begins outreach within minutes of the workflow finishing.

---

# 🔒 Data & Operational Considerations

This workflow relies on third-party services for lead discovery, AI processing, email delivery, and spreadsheet storage.

When deploying it for real-world sourcing or prospecting:

- Respect LinkedIn's, Apify's, and OpenRouter's applicable usage terms.
- Follow applicable privacy and data-protection laws when storing personal profile data.
- Follow email marketing and anti-spam requirements for any outbound outreach.
- Avoid collecting more personal information than necessary.
- Have a human review every AI-generated email before it is sent.
- Monitor Apify and OpenRouter usage and costs.
- Add opt-out handling where required.

The workflow itself does not include a complete email-verification or compliance layer.

---

# 📊 Current Workflow Characteristics

| Capability | Current Implementation |
|---|---|
| Trigger | n8n Form |
| Search Source | LinkedIn via Apify |
| Scraper Limit | 15 profiles |
| Processing Limit | 10 leads |
| Processing Method | One lead at a time |
| AI | n8n AI Agent (x2: Scoring + Email) |
| LLM | GPT-4.1 via OpenRouter |
| AI Output | Structured JSON |
| Email Enrichment | Apify (applied to all leads) |
| Lead Storage | Google Sheets |
| Duplicate Behavior | Append or update by `id` |
| Lead Routing | HOT / WARM / COLD Switch |
| HOT Lead Outcome | AI email draft + Sheet update + Slack alert |
| WARM Lead Outcome | Gmail digest to team |
| COLD Lead Outcome | No further action |

---

# 🏆 Key Features

- 📝 **Simple form-based intake** (Position + Location)
- 🔎 **Automated LinkedIn profile discovery**
- 🔢 **Configurable lead limit for controlled batches**
- 🔁 **One-lead-at-a-time processing loop**
- 🧠 **AI-powered lead scoring (HOT/WARM/COLD)**
- ✍️ **AI-generated personalized icebreakers**
- 📊 **Confidence scoring based on profile completeness**
- 📧 **Automatic email enrichment for every lead**
- 🔀 **Automatic routing by lead classification**
- ✉️ **AI-personalized outreach email generation for HOT leads**
- 🗂️ **Structured, append-or-update Google Sheets database**
- 📢 **Real-time Slack alerts for HOT leads**
- 📬 **Gmail digest notifications for WARM leads**
- ❄️ **Zero wasted effort on COLD leads**
- 💵 **Controlled scraping and AI budget**
- 📈 **Scalable architecture for larger sourcing campaigns**

---

# 🧱 Node Flow Summary

```text
On form submission
      │
      ▼
   Get leads
      │
      ▼
 Parse Leads Data
      │
      ▼
   10 Limits
      │
      ▼
 Loop Over Items
 ├── loop ──► Aggregate Data ──► Scoring Agent (GPT-4.1)
 │                                    │
 │                                    ▼
 │                              Parse Data
 │                                    │
 │                                    ▼
 │                               Map Data
 │                                    │
 │                                    ▼
 │                           Look For Email
 │                                    │
 │                                    ▼
 │                             Enrich Data
 │                                    │
 │                                    ▼
 │                              Save Data ──────► back to Loop Over Items
 │
 └── done ──► Switch
                ├── HOT ──► For Each HOT Lead
                │             ├── loop ──► Aggregate ──► Email Agent Generator (GPT)
                │             │                              │
                │             │                              ▼
                │             │                       Parse Sample Email
                │             │                              │
                │             │                              ▼
                │             │                        Map Email Data
                │             │                              │
                │             │                              ▼
                │             │                        Update HOT Leads ──► back to For Each HOT Lead
                │             │
                │             └── done ──► Generate Slack Message ──► Send Executive Message
                │
                ├── WARM ──► Generate Email Alert ──► Send Email Alert
                │
                └── COLD ──► COLD Leads (no action)
```

---

# 📦 Requirements

- n8n
- Apify account (with access to the LinkedIn search and email enrichment actors)
- OpenRouter account with access to GPT-4.1
- Google account
- Google Sheets
- Gmail
- Slack workspace with a configured channel

---

# 🔧 Recommended Production Improvements

For a more advanced production implementation, consider adding:

### 1. Email Verification

Verify enriched email addresses before they're used for outreach.

### 2. Lead Deduplication

Use the LinkedIn URN or public identifier as a stronger unique key across multiple campaign runs.

### 3. ATS/CRM Integration

Send qualified leads into:

- HubSpot
- Greenhouse
- Lever
- Salesforce
- Airtable

### 4. Automated Sending

After human approval, automatically send the AI-generated HOT lead emails and track opens/replies.

### 5. Multi-Campaign Support

Allow multiple Position/Location combinations to run in parallel with separate result tracking.

### 6. Follow-Up Sequencing

Add automated follow-up messages for HOT leads who don't respond within a set window.

### 7. Configurable Scoring Rules

Let the scoring criteria be adjusted per campaign (e.g., different skill requirements per role).

---

# 📌 Portfolio Value

This workflow demonstrates practical automation engineering rather than a simple single-purpose n8n flow. It combines:

```text
Form-Based Trigger Logic
+
Multi-Actor API Integration
+
Web Data Extraction & Enrichment
+
AI Agents (Scoring + Content Generation)
+
LLM Integration via OpenRouter
+
JSON Processing
+
Looping & Batch Control
+
Conditional Routing
+
Database-like Spreadsheet Storage
+
Multi-Channel Notification Automation
```

The project is particularly relevant to **recruiting automation, sales prospecting, AI workflow engineering, API integration, and business-process automation**.

---

# 📄 One-Sentence Project Summary

> **An AI-powered n8n lead-generation pipeline that discovers LinkedIn profiles by position and location, scores and enriches each candidate with GPT-4.1, generates personalized outreach emails for HOT leads, stores everything in Google Sheets, and automatically notifies the team through Slack and Gmail when new prospects are ready.**

---

# 📊 Estimated Impact

| Metric | Estimate |
|---|---:|
| Leads per execution | Up to 10 |
| Estimated manual time per lead | 10–20 min |
| Estimated time saved per run | 100–200 min |
| Estimated monthly time saved (a few runs/week) | **15–30 hours** |
| Estimated labor value saved | **$225–$900/month** |
| Apify maximum configured charge (lead search) | **$0.50/execution** |

> **Note:** Savings are estimates based on the current 10-lead configuration and illustrative manual labor assumptions. Actual savings depend on lead volume, manual research speed, labor cost, and the final production configuration.
