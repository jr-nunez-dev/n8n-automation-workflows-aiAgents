# AI Shipment Delay Root-Cause Explainer

---

# Workflow Objective

The objective of this workflow is to automate shipment delay analysis.

Instead of manually reviewing shipment records, AI identifies delay causes, prioritizes urgency, and prepares executive reports.

---

# Workflow Stages

---

# 1. Trigger

Starts the workflow.

Can be:

- Manual
- Scheduled
- Webhook

---

# 2. Read Shipment Data

Reads shipment records from Google Sheets.

Each shipment contains:

- Shipment ID
- Origin
- Destination
- Carrier
- Delay Hours
- Weather Flag
- Customs Flag
- Inventory Flag

---

# 3. AI Logistics Analyst

The first AI Agent.

Responsibilities:

- Determine Root Cause
- Explain Delay
- Recommend Next Action
- Assign Urgency
- Confidence Score
- Contributing Factors

Example Output

```json
{
  "shipment_id":"SHP-1004",
  "root_cause":"Weather Delay",
  "urgency":"High",
  "confidence":90,
  "contributing_factors":[
      {
          "factor":"Weather",
          "weight":60
      },
      {
          "factor":"Customs",
          "weight":40
      }
  ]
}
```

---

# 4. Output Parser

Ensures the AI always returns structured JSON.

---

# 5. Urgency Routing

Business rules classify shipments into:

High

Medium

Low

---

## High

- Slack Alert
- Google Sheets Log

---

## Medium

- Google Sheets Log

---

## Low

- Archive

---

# 6. Merge Results

Rejoins every shipment after routing.

Purpose:

Generate complete reporting.

---

# 7. Compress Data

A JavaScript node calculates KPIs.

Generated Metrics

- Total Shipments
- High Count
- Medium Count
- Low Count
- Average Confidence
- Weather %
- Customs %
- Inventory %
- Carrier %
- Top Delay Cause
- Highest Risk Shipments

This reduces token consumption before sending data to the Executive AI.

---

# 8. Executive AI

Instead of reading all shipments again, the Executive AI only receives KPIs.

Responsibilities:

- Manager Briefing
- Executive Summary
- Operational Trends
- Recommendations
- Next 24-Hour Priorities

---

# 9. Executive Output Parser

Ensures structured management reports.

---

# 10. Slack Executive Report

Posts a professional daily logistics summary.

---

# AI Architecture

## AI Agent #1

Role:

Logistics Analyst

Tasks:

- Root Cause Analysis
- Confidence Score
- Contributing Factors
- Recommended Action

---

## AI Agent #2

Role:

Senior Logistics Operations Manager

Tasks:

- Read KPIs
- Generate Executive Summary
- Produce Recommendations
- Highlight Operational Trends

---

# Workflow Benefits

✅ Eliminates manual shipment review

✅ Standardizes incident reporting

✅ Prioritizes critical shipments

✅ Improves communication

✅ Produces executive-ready reports

✅ Optimizes AI token usage

---

# Future Improvements

- Live Carrier APIs
- Weather API Integration
- Customs API
- Power BI Dashboard
- Airtable Integration
- PostgreSQL Data Warehouse
- Predictive Delay Forecasting
- Email Escalations
- Historical Trend Analytics

---

# Author

Built with ❤️ using **n8n**, **GPT-4.1**, **Google Sheets**, and **Slack**.