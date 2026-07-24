# On-Call Handoff Summary Generator

An n8n workflow that automatically transforms raw on-call incident logs into a structured handoff summary using an LLM, evaluates the operational risk for the incoming shift, archives every incident for auditing, and routes the completed handoff report to the appropriate Slack notification channel. This eliminates manual shift documentation while ensuring the next on-call engineer receives a consistent, actionable summary before taking ownership of production systems.

---

## Quick Facts

| Item | Value |
|------|------|
| **Workflow Type** | AI Operations Automation |
| **Primary Purpose** | Generate AI-powered on-call handoff summaries |
| **Trigger** | Manual Trigger (Mock Data) |
| **AI Pattern** | Basic LLM Chain |
| **LLM** | Gemini 2.5 Flash |
| **Output Format** | Structured JSON (Output Parser) |
| **Programming** | JavaScript (Code Node) |
| **Integrations** | Slack, Google Sheets |
| **Risk Levels** | Chill • Watch Closely • Buckle Up |
| **Development Mode** | Uses mock incident data (replace with monitoring platform for production) |

---

# 1. Overview

This workflow automates one of the most repetitive tasks performed by Site Reliability Engineers (SREs), DevOps Engineers, and IT Operations teams: writing end-of-shift handoff notes.

Instead of manually reading incident logs and writing summaries, the workflow receives a completed shift's incident history, archives every individual incident into Google Sheets, and sends the complete shift log to an LLM configured as an **On-Call Handoff Assistant**.

The model analyzes the incidents and produces a structured handoff report containing:

- Shift overview
- Chronological incident summary
- Outstanding issues that still require attention
- Overall operational risk assessment
- Markdown handoff summary

The workflow then categorizes the next shift into one of three operational risk levels:

- **Chill**
- **Watch Closely**
- **Buckle Up**

Depending on the predicted risk level, the completed handoff summary is automatically delivered to Slack, allowing the incoming engineer to immediately understand the current production environment before starting their shift.

At the same time, every incident is normalized and archived into Google Sheets, creating a searchable operational history for reporting, auditing, and post-incident reviews.

```
Manual Trigger
      │
      ▼
Mock Shift Data
      ├────────────────────────────┐
      │                            │
      ▼                            ▼
Split Incidents             Basic LLM Chain
      │                            │
Map Incident Fields      Structured Output Parser
      │                            │
Google Sheets Log      Determine Risk Level
                                   │
                                   ▼
                             Route by Risk
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
                 Chill     Watch Closely   Buckle Up
                        Slack Notification
```

---

# 2. What Problem Does It Solve?

Operations teams frequently rely on manually written shift notes, Slack conversations, spreadsheets, or verbal updates when handing production systems over to the next engineer.

As production environments grow larger and incidents become more frequent, manually reviewing every alert becomes increasingly difficult. Important unresolved issues can easily be forgotten, recurring incidents may go unnoticed, and incoming engineers often spend valuable time reconstructing what happened during the previous shift instead of immediately responding to ongoing operational concerns.

This workflow removes that manual documentation process.

Instead of writing handoff notes from scratch, engineers only need to provide the shift's incident log. The workflow automatically summarizes everything that happened, identifies unresolved or recurring issues, determines how risky the next shift is likely to be, archives every incident for future reference, and delivers a clean handoff summary directly into Slack.

---

# 3. Benefits

- **Standardized handoff documentation** — every shift follows the same reporting structure regardless of who was on call.
- **Reduced manual work** — engineers spend less time writing summaries and more time resolving incidents.
- **Improved operational awareness** — unresolved issues and recurring production problems are clearly highlighted.
- **Faster shift transitions** — incoming engineers receive a concise summary instead of reviewing dozens of alerts manually.
- **Historical incident archive** — every incident is stored in Google Sheets for reporting, auditing, and trend analysis.
- **Consistent AI decision-making** — operational risk is evaluated using predefined criteria rather than subjective judgment.
- **Scalable architecture** — the mock incident source can easily be replaced with PagerDuty, Grafana, Datadog, ServiceNow, Prometheus, or any monitoring platform without redesigning the workflow.

---

# 4. Who Will Use It?

This workflow is designed for teams responsible for monitoring and maintaining production systems.

Typical users include:

- Site Reliability Engineers (SRE)
- DevOps Engineers
- IT Operations Teams
- Network Operations Centers (NOC)
- Managed Service Providers (MSP)
- Platform Engineering Teams
- Engineering Managers overseeing on-call rotations

---

# 5. Node-by-Node Breakdown

## `mockTrigger` — Manual Trigger

Starts the workflow during development and testing.

Since this repository is designed using mock data, the workflow begins with a Manual Trigger. In production environments this node would typically be replaced with a Scheduler, Webhook, PagerDuty Trigger, ServiceNow Trigger, or another monitoring platform capable of initiating the workflow automatically after each completed shift.

---

## `mockData` — Code

Generates an entire mock on-call shift.

The node creates realistic operational data including:

- Shift time
- Assigned engineer
- Multiple production incidents
- Severity levels
- Current status
- Resolution actions

This allows the workflow to be demonstrated without requiring access to a live monitoring environment.

---

# Sub Workflow — Incident Logging

While the AI generates the handoff summary, another branch of the workflow simultaneously archives every incident.

This creates a permanent operational history independent of the AI summary.

---

## `getIncidents` — Split Out

Splits the incident array into individual items.

Instead of processing one large JSON object, each incident becomes its own workflow item, allowing every incident to be logged independently.

---

## `mapIncidents` — Set / Edit Fields

Normalizes every incident into a reporting-friendly structure.

Each incident receives:

- Generated Incident ID
- Severity
- System
- Summary
- Status
- Action Taken
- Timestamp

Creating this normalized structure decouples reporting from the original input format and allows future integrations without modifying downstream nodes.

---

## `logIncidents` — Google Sheets (Append)

Stores every incident into Google Sheets.

The sheet becomes a centralized incident archive that can later be used for:

- Operational reporting
- Trend analysis
- Compliance audits
- Postmortem investigations
- SLA reporting

---

## `Basic LLM Chain` — LangChain Chain

This is the intelligence behind the workflow.

The complete shift history is sent to Gemini along with instructions to produce a structured handoff containing:

- Shift overview
- Incident narrative
- Outstanding issues
- Operational recommendations
- Risk level
- Markdown handoff report

A **Basic LLM Chain** is intentionally used instead of an **AI Agent** because the workflow only requires one transformation from structured incident data into a structured summary.

The task does not require external tools, memory, or multi-step reasoning, making the Basic LLM Chain simpler, faster, and more reliable.

---

## `Gemini 2.5 Flash` — Chat Model *(Sub-node)*

Provides the Large Language Model responsible for generating the handoff report.

Gemini 2.5 Flash was selected because it provides fast inference, low operational cost, and excellent structured-output capabilities, making it well suited for operational summarization workflows.

---

## `SOP` — Structured Output Parser *(Sub-node)*

Validates the AI response against a predefined JSON schema.

The parser ensures every generated summary contains:

- `risk_level`
- `risk_reason`
- `summary_markdown`

This prevents malformed AI responses from reaching downstream routing logic.

---

## `routeRiskLevel` — Switch

Acts as the routing engine of the workflow.

It examines the AI-generated `risk_level` and directs the completed handoff toward one of three operational paths:

- Chill
- Watch Closely
- Buckle Up

This allows different notification strategies depending on the operational state of production.

---

## `Chill` — Slack

Posts the completed handoff summary when production is considered stable.

This indicates the incoming engineer should remain aware of the environment but no immediate action is expected.

---

## `Watch Closely` — Slack

Posts the summary when unresolved issues still require monitoring.

This risk level indicates that production remains healthy overall but certain systems should be watched carefully during the incoming shift.

---

## `Buckle Up` — Slack

Posts the summary when production is considered high risk.

This usually represents situations involving:

- Multiple unresolved incidents
- Recurring P1/P2 incidents
- Ongoing production instability
- Temporary fixes instead of permanent resolutions

The incoming engineer is immediately informed that active operational attention will likely be required.

---

# Setup Requirements

| Service | Used For | Credential Needed |
|----------|----------|------------------|
| Gemini API | AI Summary Generation | Gemini API Key |
| Slack | Handoff Notifications | Slack OAuth/API Token |
| Google Sheets | Incident Archive | Google Sheets OAuth2 |

---

# Known Limitations / Next Steps

- `mockData` currently uses hardcoded incident information and should be replaced with a production monitoring platform such as PagerDuty, Datadog, Grafana, ServiceNow, or Prometheus.
- The workflow currently routes notifications only through Slack. Additional integrations such as Microsoft Teams, Email, PagerDuty, or Jira could easily be added.
- Incident history is archived, but recurring incidents across multiple shifts are not yet correlated.
- AI-generated summaries depend on the quality of incoming incident data. More detailed telemetry and root-cause information would improve summary accuracy.
- Error handling is intentionally lightweight for readability. Production implementations should include retry logic, failure notifications, and per-node error workflows.

---

## Future Improvements

- Integrate with PagerDuty or Opsgenie
- Automatically generate Jira follow-up tickets for unresolved incidents
- Email shift summaries to engineering managers
- Store summaries in Notion or Confluence
- Correlate recurring incidents across multiple shifts
- Generate weekly operational health reports
- Add observability dashboards using Grafana or Looker Studio
