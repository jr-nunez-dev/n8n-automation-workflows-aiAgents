# 🟢 AI Lead Generation Automation

## 📌 Description

**AI Lead Generation Automation** is an n8n-powered prospecting workflow that automatically discovers local businesses from Google Maps, processes their business information with an AI agent, generates personalized outreach icebreakers, stores qualified lead data in Google Sheets, and sends an automated email notification when new prospects are ready for outreach.

The workflow is designed as a lightweight lead-hunting pipeline for sales, marketing, agencies, freelancers, and automation consultants who need a repeatable way to turn a business category into a structured prospect list.

---

## 🎯 Problem

Manual lead generation is repetitive and time-consuming: a person has to search Google Maps, open business profiles, copy contact information, normalize phone numbers, review ratings, inspect operating hours, write personalized opening messages, organize everything into a spreadsheet, and then notify the sales team that the leads are ready.

This workflow removes most of that repetitive preparation by connecting lead discovery, AI enrichment, data storage, and notification into one automated process.

---

## 💡 Solution

The workflow uses a Google Sheets control row as the trigger, checks whether the requested lead-hunting job is marked **RUN**, uses **Apify's Google Maps Scraper** to discover businesses in Metro Manila, limits the result set to three leads, processes each lead with an AI agent using GPT-4.1 through OpenRouter, saves the structured results to a Google Sheets lead database, and emails the team a dynamic notification containing the newly discovered companies.

---

## 👥 Who Will Use This?

This workflow is useful for:

- Lead generation agencies
- Digital marketing agencies
- Automation consultants
- Freelancers
- Real estate agencies
- B2B sales teams
- Business development teams
- Local service providers
- Appointment-setting teams
- SaaS sales teams
- Recruitment and sourcing teams
- Anyone building a Google Maps-based prospecting system

---

# ⚙️ Workflow Overview

The automation follows this general pipeline:

```text
Google Sheets — Lead Status
          ↓
       RUN? Check
      ↙           ↘
   RUN             STOP
    ↓                ↓
Apify Google Maps   Done/Null
    ↓
Limit to 3 Leads
    ↓
Process One Lead at a Time
    ↓
Aggregate Lead Data
    ↓
AI Agent + GPT-4.1
    ↓
Wait 1.5 Seconds
    ↓
Parse AI JSON
    ↓
Save / Update Lead in Google Sheets
    ↓
Loop Until Finished
    ↓
Generate Email Notification
    ↓
Send Gmail Notification

After the job completes:
Lead Status → Done
```

---

# 🧩 Workflow Components

| Node | Purpose |
|---|---|
| `getLeadStatus` | Watches the Google Sheets control sheet for the lead-hunting job |
| `RUN?` | Determines whether the workflow should execute |
| `getLeads` | Runs the Apify Google Maps scraper |
| `limitsTo3` | Restricts the scraped result set to a maximum of three leads |
| `1item/Loops` | Processes leads one at a time and controls the workflow loop |
| `aggregateData` | Aggregates lead data before AI processing |
| `AI Agent` | Extracts, transforms, summarizes, and enriches raw lead data |
| `gpt-4.1` | Provides the language model used by the AI Agent through OpenRouter |
| `1.5secs` | Adds a 1.5-second delay between AI processing stages |
| `parseLeadData` | Converts the AI agent's JSON string into usable n8n JSON |
| `saveLeadData` | Appends or updates the structured lead in Google Sheets |
| `generateEmailNotif` | Creates a dynamic notification subject and message |
| `sendEmailNotif` | Sends the completed lead notification through Gmail |
| `setUpdate` | Prepares the control record with `Status = Done` |
| `Done` | Updates the control sheet after completion |
| `Done/Null` | Safely terminates executions that are not marked for running |

---

# 🔄 Detailed Workflow Process

## 1. Google Sheets Trigger — `getLeadStatus`

The workflow starts by monitoring a Google Sheets document.

The control sheet is expected to contain at least:

| Field | Example |
|---|---|
| `Lead` | Real Estate Agencies |
| `Status` | RUN |

The trigger is configured to check the sheet weekly at **7:00 AM**.

The `Lead` value becomes the search term used by the Google Maps scraper.

For example:

```text
Lead = Real Estate Agencies
```

becomes a search request equivalent to:

```text
Real Estate Agencies in Metro Manila, Philippines
```

This makes the workflow category-driven instead of requiring the search term to be hardcoded inside the scraping node.

---

# ▶️ 2. Run Control — `RUN?`

The workflow checks the value of the `Status` field.

The condition effectively evaluates:

```javascript
$json.Status.toUpperCase() === "RUN"
```

### If Status = RUN

The workflow continues into lead generation.

### If Status is anything else

The workflow terminates through:

```text
Done/Null
```

This gives the spreadsheet a simple manual control mechanism.

For example:

```text
Lead: Real Estate Agencies
Status: RUN
```

starts the process.

After completion, the workflow changes the status to:

```text
Done
```

---

# 🗺️ 3. Google Maps Lead Discovery — `getLeads`

The workflow uses **Apify** and the **Google Maps Scraper (compass/crawler-google-places)** actor to discover businesses.

The scraper dynamically receives the Google Sheets `Lead` value.

Configured search structure:

```text
[Lead] in Metro Manila, Philippines
```

The scraper is configured for:

```text
maxCrawledPlacesPerSearch: 10
```

and has a configured maximum total charge of:

```text
$0.50
```

per Apify execution.

The workflow therefore combines:

```text
Google Sheets category
        ↓
Dynamic Google Maps search
        ↓
Business discovery
```

This allows the same automation to be reused for different industries.

Example categories:

```text
Real Estate Agencies
Dental Clinics
Accounting Firms
Law Firms
Digital Marketing Agencies
Construction Companies
Property Management Companies
Insurance Agencies
Recruitment Agencies
IT Support Companies
```

---

# 🔢 4. Lead Limit — `limitsTo3`

Although the scraper can crawl up to 10 places, the workflow intentionally limits the downstream dataset to **three leads**.

This is useful for:

- Controlled testing
- Reducing AI processing costs
- Preventing oversized lead batches
- Keeping notification emails concise
- Testing personalized outreach quality
- Creating a small, reviewable prospect batch

The limit can be increased later if the workflow is ready for higher-volume operation.

---

# 🔁 5. One-Lead-at-a-Time Processing — `1item/Loops`

The workflow uses a batching/loop mechanism to process leads individually.

Conceptually:

```text
Lead 1 → AI → Save
Lead 2 → AI → Save
Lead 3 → AI → Save
```

rather than attempting to process the entire dataset as one large operation.

This makes the workflow easier to control and allows each business to receive its own AI-generated enrichment.

After each lead is saved, the workflow returns to the loop until the available lead items are exhausted.

---

# 🤖 6. AI Lead Processing — `AI Agent`

The AI Agent is the main enrichment layer.

It receives raw JSON scraped from Google Maps and transforms it into a standardized lead record.

The agent performs two major jobs:

### A. Data Extraction and Formatting

The AI extracts:

- Business name
- Primary category
- Full address
- Website
- Phone number
- Google Maps rating
- Review count
- Average rating
- Opening hours

### B. Personalized Outreach Generation

The AI also creates a short, personalized **icebreaker** for the business.

The workflow instructs the AI to make the message:

- Warm
- Professional
- Conversational
- Authentic
- Personalized
- Suitable for an owner or management team

The AI is specifically instructed to avoid robotic sales language and generic buzzwords.

It can use contextual information such as:

- Business location
- Review score
- Business category
- Services
- Other information contained in the scraped profile

This transforms raw Google Maps data into information that is more immediately useful for outreach.

---

# 🧠 7. LLM — `gpt-4.1`

The AI Agent is connected to an OpenRouter Chat Model node configured as:

```text
gpt-4.1
```

The model is responsible for interpreting the scraped business information and generating the structured lead enrichment.

Architecture:

```text
Google Maps Raw JSON
        ↓
      AI Agent
        ↓
     GPT-4.1
        ↓
Structured Lead JSON
```

---

# ⏱️ 8. Processing Delay — `1.5secs`

A **1.5-second Wait node** is placed after the AI Agent.

This creates a small processing pause before the workflow parses and stores the AI result.

The delay can also be useful when building workflows that interact with external services and when testing execution behavior.

---

# 🧹 9. AI Output Parsing — `parseLeadData`

The AI Agent returns its structured result as a JSON string.

The Code node converts that string into an actual JavaScript object.

The core operation is conceptually:

```javascript
const rawOutput = $input.first().json.output;
const parsedData = JSON.parse(rawOutput);

return [{ json: parsedData }];
```

This transforms:

```text
JSON string
```

into:

```text
n8n JSON object
```

so the following Google Sheets node can map each field individually.

---

# 📊 10. Lead Database — `saveLeadData`

The structured lead is stored in Google Sheets using an **Append or Update** operation.

The workflow maps the AI output into the following fields:

| Google Sheets Column | AI Output |
|---|---|
| Company | `title` |
| Category | `categoryName` |
| Address | `address` |
| Website | `website` |
| Phone Number | `phoneUnformatted` |
| Score | `totalScore` |
| Ratings | `reviewsCount` |
| Opening Hours | `openingHours` |
| Icebreaker | `icebreaker` |

The workflow uses:

```text
Company
```

as its matching column.

This means an existing company can be updated instead of blindly creating another row.

That provides a basic level of duplicate prevention and data maintenance.

---

# 📱 Phone Number Normalization

The AI is instructed to convert Philippine phone numbers beginning with:

```text
+63
```

into the local:

```text
0
```

format.

Example:

```text
+639661832059
```

becomes:

```text
09661832059
```

This makes phone numbers more convenient for local outreach and downstream processing.

---

# ⭐ Review Information

The workflow preserves the business's Google Maps rating information.

The AI is instructed to produce:

```text
Total Score
```

and a combined review summary such as:

```text
96 reviews (Avg: 4.9/5 stars)
```

This gives the sales user useful social-proof context before contacting the business.

---

# 🕒 Opening Hours Formatting

The AI converts raw opening-hour information into concise human-readable text.

Example:

```text
Mon-Fri: 9 AM - 5 PM | Sat-Sun: Closed
```

This makes the data easier to use during manual outreach.

---

# ✍️ Personalized Icebreaker

One of the most valuable features of the workflow is the AI-generated outreach icebreaker.

Instead of producing:

```text
Hi, I noticed your business online.
```

the workflow attempts to use specific business information.

Potential context includes:

- Business name
- Category
- Location
- Review performance
- Services
- Other profile information

The intended output is a short **1–2 sentence** personalized opener that can be used as the starting point for sales outreach.

---

# 🔄 11. Loop Completion

After a lead is saved, the workflow returns to:

```text
1item/Loops
```

The loop continues until all selected leads have been processed.

Once the final lead has been handled, the workflow exits the lead-processing loop and continues to notification generation.

---

# 📧 12. Notification Generation — `generateEmailNotif`

After the lead batch has been processed, the workflow creates a dynamic notification email.

The Code node:

1. Collects all company names.
2. Counts the number of newly processed leads.
3. Generates a dynamic subject line.
4. Creates a formatted company list.
5. Produces the final email body.

The subject dynamically changes depending on the number of leads.

Conceptually:

```text
🚀 New Leads Alert: 3 fresh prospects ready!
```

The email then tells the team that the latest companies have been added to the lead pipeline.

---

# 📬 13. Gmail Notification — `sendEmailNotif`

The generated notification is sent using Gmail.

The email contains:

- Number of new prospects
- Company names
- Confirmation that contact information is available
- Confirmation that icebreakers are ready
- Direction to begin outreach

This means the user does not need to continuously monitor the spreadsheet.

Instead:

```text
Lead generation completes
        ↓
Email notification arrives
        ↓
Sales/outreach begins
```

---

# ✅ 14. Completion Status — `setUpdate` + `Done`

The workflow updates the control record after the lead-generation process.

The status is changed to:

```text
Done
```

The `Lead` value is preserved.

The final Google Sheets update uses:

```text
Lead
```

as the matching field.

This creates a simple job-state lifecycle:

```text
RUN
 ↓
Processing
 ↓
Done
```

---

# 📋 Example Control Sheet

A simple control sheet can look like:

| Lead | Status |
|---|---|
| Real Estate Agencies | RUN |
| Dental Clinics | Done |
| Accounting Firms | Done |

When the desired category is set to `RUN`, the automation can execute the corresponding lead-hunting process.

---

# 📋 Example Output

The resulting `Leads Found` sheet can contain:

| Company | Category | Address | Website | Phone Number | Score | Ratings | Opening Hours | Icebreaker |
|---|---|---|---|---|---:|---|---|---|
| Example Realty | Real Estate Agency | Makati, Metro Manila | example.com | 09123456789 | 4.8 | 120 reviews (Avg: 4.8/5 stars) | Mon-Fri: 9 AM - 6 PM | Personalized AI-generated opener |

The exact values depend on what Google Maps and Apify return for the selected search.

---

# 🏗️ Architecture

```text
                    ┌──────────────────────┐
                    │    Google Sheets     │
                    │     Lead Status      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │        RUN?          │
                    └───────┬───────┬──────┘
                            │       │
                         RUN│       │STOP
                            ▼       ▼
                 ┌──────────────┐ ┌───────────┐
                 │    Apify     │ │ Done/Null │
                 │ Google Maps  │ └───────────┘
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Limit to 3   │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Lead Loop    │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Aggregate    │
                 │ Lead Data    │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │  AI Agent    │
                 │   GPT-4.1    │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Wait 1.5 sec │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Parse JSON   │
                 └──────┬───────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Save Lead    │
                 │ Google Sheets│
                 └──────┬───────┘
                        │
                        └──────► Loop
                                   │
                              Finished
                                   ▼
                         ┌─────────────────┐
                         │ Generate Email  │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │      Gmail      │
                         └─────────────────┘
```

---

# 🧠 AI Processing Schema

The AI Agent is designed to return:

```json
{
  "title": "",
  "categoryName": "",
  "address": "",
  "website": "",
  "phoneUnformatted": "",
  "totalScore": 0,
  "reviewsCount": "",
  "openingHours": "",
  "icebreaker": ""
}
```

This standardized structure makes the AI output predictable and easy to map into Google Sheets.

---

# 🔌 Integrations

## n8n

The workflow is orchestrated entirely through n8n.

Used for:

- Triggers
- Conditions
- Looping
- Data transformation
- AI orchestration
- Delays
- Google Sheets operations
- Gmail notifications

## Google Sheets

Used for two purposes:

1. **Lead Status** — controls whether a lead-generation job should run.
2. **Leads Found** — stores the enriched lead database.

## Apify

Used for Google Maps business discovery.

The workflow uses the Google Maps Scraper actor to find local businesses based on the requested category.

## OpenRouter

Provides the connection to the language model used by the AI Agent.

## GPT-4.1

Used to transform raw business information into structured data and generate personalized icebreakers.

## Gmail

Used to notify the sales/outreach team after a lead batch has been processed.

---

# 🔐 Credentials & Configuration

Before activating the workflow, configure the required credentials.

### Google Sheets

Required for:

- `getLeadStatus`
- `saveLeadData`
- `Done`

The imported workflow contains placeholder credential references such as:

```text
YOUR_GOOGLE_CREDENTIAL_ID
```

Replace these with the correct Google credential in your n8n instance.

### Apify

The workflow requires an Apify API credential.

Replace the placeholder credential configuration with your Apify account.

### OpenRouter

The AI Agent requires an OpenRouter credential for the GPT-4.1 model.

### Gmail

The Gmail node requires a connected Google account capable of sending the notification email.

---

# 🛠️ Configuration Checklist

Before using the workflow:

- [ ] Connect Google Sheets credentials.
- [ ] Connect Apify credentials.
- [ ] Connect OpenRouter credentials.
- [ ] Connect Gmail credentials.
- [ ] Configure the Google Sheets document.
- [ ] Configure the `Lead Status` sheet.
- [ ] Configure the `Leads Found` sheet.
- [ ] Replace placeholder spreadsheet IDs.
- [ ] Replace the placeholder notification email address.
- [ ] Confirm the Apify actor is available in your Apify account.
- [ ] Confirm the GPT-4.1 model is available through your OpenRouter account.
- [ ] Test the workflow with one lead category.
- [ ] Verify the AI JSON output.
- [ ] Verify Google Sheets mappings.
- [ ] Verify the Gmail notification.
- [ ] Change the control status to `RUN` only when ready for production execution.

---

# ⚠️ Important Configuration Notes

### Current Search Location

The scraper is currently configured specifically for:

```text
Metro Manila, Philippines
```

If you want to make the system geographically flexible, change the search expression so the location is also supplied dynamically.

### Current Lead Limit

The workflow limits processing to:

```text
3 leads
```

This is appropriate for testing and controlled batches.

For production-scale lead generation, increase the limit carefully and consider API cost, AI cost, processing time, and notification volume.

### Current Apify Crawl Limit

The scraper itself is configured for:

```text
10 places per search
```

while the downstream workflow processes a maximum of:

```text
3 leads
```

Therefore, the final lead count is intentionally smaller than the maximum crawl count.

### Duplicate Handling

The `Leads Found` Google Sheets node uses:

```text
Company
```

as its matching column.

This provides update behavior for matching company names, but company name alone is not a perfect global unique identifier. A production implementation could use a more reliable identifier such as a Google Maps place ID if available.

### AI Output Parsing

The `parseLeadData` Code node uses `JSON.parse()`.

Therefore, the AI Agent must return valid JSON matching the expected schema.

If the AI returns malformed JSON, the parsing step can fail.

---

# 🚀 How to Run

## Step 1 — Select a Lead Category

Open the `Lead Status` Google Sheet.

Example:

```text
Lead: Real Estate Agencies
```

## Step 2 — Set the Status

Change:

```text
Status: RUN
```

## Step 3 — Allow the Workflow to Execute

The trigger detects the control record and sends it through the `RUN?` condition.

## Step 4 — Discover Businesses

Apify searches Google Maps for:

```text
Real Estate Agencies in Metro Manila, Philippines
```

## Step 5 — Process Leads

The workflow limits the result set and processes each selected business.

## Step 6 — AI Enrichment

GPT-4.1 extracts the required information and generates an outreach icebreaker.

## Step 7 — Save Results

The enriched lead is stored in:

```text
Leads Found
```

## Step 8 — Receive Notification

Gmail sends the team a summary of the newly processed companies.

## Step 9 — Job Completion

The control record is updated to:

```text
Done
```

---

# 💰 Estimated Business Value

The current workflow is configured as a small controlled batch system, processing up to **3 leads per execution**.

Because the exact labor cost and manual research speed depend on the operator, the following numbers are estimates rather than measured results.

### Estimated Manual Work Per Lead

A reasonable manual lead-hunting process can involve:

- Finding the business
- Opening the profile
- Copying contact details
- Checking the website
- Reviewing ratings
- Recording opening hours
- Formatting the data
- Writing an initial personalized message
- Updating the spreadsheet

Estimated manual preparation:

```text
15–30 minutes per lead
```

For 3 leads:

```text
45–90 minutes per run
```

If the workflow runs approximately once per week:

```text
~3–6 hours/month saved
```

### Estimated Labor Value

Using an illustrative productivity value of approximately:

```text
$15–$25/hour
```

the current configuration could represent approximately:

```text
$45–$150/month
```

in recovered labor value.

### Estimated Current Savings

**Time Saved:** approximately **3–6 hours/month**

**Potential Labor Value Saved:** approximately **$45–$150/month**

These estimates increase substantially if the lead limit is increased and the automation is used for larger prospecting batches.

---

# 📈 Scalability Potential

The current workflow is intentionally conservative.

A future production version could scale the system into:

```text
Lead Category
     ↓
Location
     ↓
Google Maps Discovery
     ↓
Hundreds of Businesses
     ↓
Deduplication
     ↓
AI Enrichment
     ↓
Lead Scoring
     ↓
Email / Phone / Website Validation
     ↓
CRM
     ↓
Personalized Outreach
     ↓
Follow-Up Automation
```

Potential upgrades include:

- Dynamic city selection
- Dynamic country selection
- Multiple Google Maps searches
- Larger lead batches
- Google Maps place ID deduplication
- Email extraction
- Email verification
- Website scraping
- Company size detection
- Lead scoring
- ICP qualification
- CRM integration
- Automated outreach
- Follow-up sequences
- Slack notifications
- Telegram notifications
- Lead status tracking
- Campaign tracking
- Response tracking
- Appointment booking
- Conversion analytics

---

# 🧪 Example Use Case — Real Estate Lead Hunting

Suppose the control sheet contains:

```text
Lead: Real Estate Agencies
Status: RUN
```

The workflow searches:

```text
Real Estate Agencies in Metro Manila, Philippines
```

It discovers businesses, selects up to three leads, and processes them individually.

The AI then turns raw information into:

```text
Company
Category
Address
Website
Phone
Rating
Reviews
Opening Hours
Personalized Icebreaker
```

The final records are saved into the lead database.

The sales team then receives an email such as:

```text
🚀 New Leads Alert: 3 fresh prospects ready!
```

The team can open the spreadsheet and immediately begin outreach.

---

# 🔒 Data & Operational Considerations

This workflow relies on third-party services for business discovery, AI processing, email delivery, and spreadsheet storage.

When deploying it for real-world prospecting:

- Respect applicable Google Maps and Apify usage requirements.
- Follow applicable privacy and data-protection laws.
- Follow email marketing and anti-spam requirements.
- Avoid collecting unnecessary personal information.
- Validate contact information before outreach.
- Add opt-out handling where required.
- Monitor API usage and costs.
- Review AI-generated icebreakers before high-volume campaigns.

The workflow itself does not include a complete email-verification or compliance layer.

---

# 📊 Current Workflow Characteristics

| Capability | Current Implementation |
|---|---|
| Trigger | Google Sheets |
| Schedule | Weekly at 7:00 AM |
| Search Source | Google Maps via Apify |
| Search Location | Metro Manila, Philippines |
| Scraper Limit | 10 places |
| Processing Limit | 3 leads |
| Processing Method | One lead at a time |
| AI | n8n AI Agent |
| LLM | GPT-4.1 via OpenRouter |
| AI Output | Structured JSON |
| Phone Normalization | Philippine `+63` → `0` |
| Rating Extraction | Yes |
| Review Summary | Yes |
| Opening Hours | Yes |
| AI Icebreaker | Yes |
| Lead Storage | Google Sheets |
| Duplicate Behavior | Append or update by Company |
| Notification | Gmail |
| Completion Tracking | Google Sheets |
| Status Lifecycle | RUN → Done |

---

# 🏆 Key Features

- 🔎 **Automated Google Maps Lead Discovery**
- 📍 **Location-specific prospecting**
- 🧠 **AI-powered lead enrichment**
- ✍️ **Personalized AI-generated icebreakers**
- ⭐ **Google rating and review extraction**
- 📞 **Philippine phone-number normalization**
- 🕒 **Opening-hours summarization**
- 📊 **Structured Google Sheets lead database**
- 🔄 **Append-or-update lead storage**
- 🔁 **One-lead-at-a-time processing**
- 🛑 **Spreadsheet-controlled execution**
- 📧 **Automatic Gmail lead notifications**
- ✅ **Automatic completion status updates**
- 💵 **Controlled scraping budget**
- 📈 **Scalable architecture for larger campaigns**

---

# 🧱 Node Flow Summary

```text
getLeadStatus
      │
      ├──────────────► setUpdate
      │                    │
      │                    ▼
      │                  Done
      │
      ▼
RUN?
 ├── FALSE ──► Done/Null
 │
 └── TRUE
       │
       ▼
    getLeads
       │
       ▼
  limitsTo3
       │
       ▼
  1item/Loops
       │
       ▼
 aggregateData
       │
       ▼
    AI Agent
       │
    GPT-4.1
       │
       ▼
    1.5secs
       │
       ▼
 parseLeadData
       │
       ▼
 saveLeadData
       │
       └──────────► 1item/Loops
                         │
                      Finished
                         ▼
                 generateEmailNotif
                         │
                         ▼
                   sendEmailNotif
```

---

# 📦 Requirements

- n8n
- Google account
- Google Sheets
- Gmail
- Apify account
- OpenRouter account
- Access to GPT-4.1 through OpenRouter

---

# 🔧 Recommended Production Improvements

For a more advanced production implementation, consider adding:

### 1. Email Extraction

Scrape the business website to identify publicly listed business emails.

### 2. Email Verification

Verify discovered email addresses before using them for outreach.

### 3. Lead Scoring

Score businesses based on:

```text
Industry Fit
Location
Reviews
Rating
Website Quality
Contact Availability
Company Size
```

### 4. CRM Integration

Send qualified prospects into:

- HubSpot
- Salesforce
- Pipedrive
- Airtable
- Supabase
- Other CRM systems

### 5. Automated Outreach

After human review, the system could generate and send personalized email campaigns.

### 6. Deduplication

Use a stronger unique identifier than company name.

### 7. Multi-Location Searching

Allow the control sheet to specify:

```text
Lead Category
Location
Lead Count
Status
```

making the workflow reusable across multiple markets.

---

# 📌 Portfolio Value

This workflow demonstrates practical automation engineering rather than a simple single-purpose n8n flow.

It combines:

```text
Trigger Logic
+
API Integration
+
Web Data Extraction
+
AI Agents
+
LLM Integration
+
JSON Processing
+
Looping
+
Data Normalization
+
Database-like Spreadsheet Storage
+
Notification Automation
+
State Management
```

The project is particularly relevant to **lead generation automation, sales operations, AI workflow engineering, API integration, and business-process automation**.

---

# 📄 One-Sentence Project Summary

> **An AI-powered n8n lead-generation pipeline that discovers local businesses through Google Maps, enriches their data with GPT-4.1, generates personalized outreach icebreakers, stores the results in Google Sheets, and automatically notifies the sales team when new prospects are ready.**

---

# 📊 Estimated Impact

| Metric | Estimate |
|---|---:|
| Leads per execution | Up to 3 |
| Estimated manual time per lead | 15–30 min |
| Estimated time saved per weekly run | 45–90 min |
| Estimated monthly time saved | **3–6 hours** |
| Estimated labor value saved | **$45–$150/month** |
| Apify maximum configured charge | **$0.50/execution** |

> **Note:** Savings are estimates based on the current 3-lead configuration and illustrative manual labor assumptions. Actual savings depend on lead volume, manual research speed, labor cost, and the final production configuration.
