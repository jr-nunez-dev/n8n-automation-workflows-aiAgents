# 🚢 AI Shipment Delay Root-Cause Explainer

> An AI-powered logistics automation workflow built with **n8n** that analyzes delayed shipments, identifies root causes, prioritizes urgency, routes notifications, logs incidents, and generates executive logistics reports.

---

## 📌 Overview

Managing delayed shipments manually is time-consuming and prone to human error.

This workflow automates the entire process by combining AI reasoning, workflow automation, and business rules.

The workflow:

- Reads shipment records from Google Sheets
- Uses AI to determine why a shipment is delayed
- Assigns an urgency level
- Calculates confidence scores
- Identifies multiple contributing factors
- Sends Slack alerts for critical shipments
- Logs incidents into Google Sheets
- Generates executive logistics reports for management

---

## 🚀 Features

- 🤖 AI Root Cause Analysis
- 📊 Confidence Scoring
- 📈 Ranked Contributing Factors
- 🚨 High Urgency Slack Alerts
- 📑 Google Sheets Logging
- 📉 KPI Generation
- 📋 Executive Summary Generation
- 🔄 Multi-Agent AI Workflow
- ✅ Structured JSON Output
- ⚡ Low Token Optimized Architecture

---

## 🏗 Workflow

```text
Google Sheets
      │
      ▼
Read Shipment Records
      │
      ▼
AI Logistics Analyst
      │
      ▼
Determine

• Root Cause
• Confidence
• Urgency
• Recommended Action
• Contributing Factors

      │
      ▼
Switch (Urgency)

 ┌──────┬─────────┬─────────┐
 │      │         │
 ▼      ▼         ▼
High   Medium    Low
 │        │        │
Slack   Sheets   Archive
 │        │
 └────────┴────────┘
          │
          ▼
 Merge All Results
          │
          ▼
 Generate KPIs
          │
          ▼
 Executive AI
          │
          ▼
 Slack Executive Report
```

---

## 🛠 Tech Stack

| Technology | Purpose |
|------------|----------|
| n8n | Workflow Automation |
| GPT-4.1 Mini | Shipment Analysis |
| GPT-4.1 | Executive Reporting |
| Google Sheets | Data Source & Logs |
| Slack | Notifications |
| JavaScript | KPI Aggregation |
| JSON Schema | Output Validation |

---

## 📁 Project Structure

```
Trigger
   │
Google Sheets
   │
AI Logistics Analyst
   │
Switch (Urgency)
   ├── High
   ├── Medium
   └── Low
        │
Merge
   │
Compress Data
   │
Generate KPIs
   │
Executive AI
   │
Slack Report
```

---

## 📊 Sample Executive Report

```
🚢 DAILY SHIPMENT SUMMARY

Date: 2024-06-11

Total Shipments: 10

High: 3
Medium: 7
Low: 0

Average Confidence: 91%

Weather Related: 50%
Inventory Related: 40%
Carrier Related: 20%

━━━━━━━━━━━━━━━━━━

Weather continues to be the leading cause of delays.

Immediate attention should focus on inventory resolution and carrier coordination.

━━━━━━━━━━━━━━━━━━

Generated automatically by AI.
```

---

## 📚 Documentation

Detailed workflow documentation can be found in **DOCUMENTATION.md**.

---

## ⭐ Highlights

This project demonstrates:

- Multi-Agent AI
- AI Decision Making
- Workflow Automation
- Business Rule Routing
- KPI Aggregation
- Executive Reporting
- Slack Integration
- Google Sheets Automation
- Structured AI Outputs
- Token Optimization