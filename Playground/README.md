# 🧪 Playground Workflows

A collection of interactive n8n workflows built for the Testarosa Playground, where each page lets visitors try a safe, guarded AI automation live: validating a CSV, triaging an incident, classifying a support ticket, reading a web page or checking an invoice.

This folder serves as a growing library of demo-ready workflows that show how to put AI behind a webhook responsibly, with human checks, rate limits, input cleaning, rule-based validation and optional email results.

---

## What You'll Find

Workflows in this category include:

- 📊 CSV Data Validator: checks a CSV with plain code and lets AI explain the results
- 🚨 AI Incident Analyzer: turns an alert or log into a structured incident triage
- 🎫 AI Support Ticket Classifier: classifies, prioritizes and routes customer tickets
- 🕸️ Chatbot Web Scraper: reads a public web page and answers questions from it
- 🧾 AI Invoice Assistant: extracts invoice fields and checks them with fixed rules

As the repository grows, additional playground workflows will be added, and existing ones may be updated.

---

## Current Workflows

| Workflow | What it does | AI model | Key safeguards |
|---|---|---|---|
| CSV Data Validator | Validates and cleans uploaded CSV rows, returns an issue list, a cleaned file and an AI summary | Groq | Deterministic validation, AI sees counts only (never cell values), no file storage |
| AI Incident Analyzer | Returns severity, likely causes, safe first checks and an escalation draft from a pasted alert or log | Groq | Injection flagging, severity floors, destructive commands removed |
| AI Support Ticket Classifier | Returns category, priority, sentiment, routing, a draft reply and a human-review flag | Groq | Tool-less agent, forced human review for refunds and angry customers |
| Chatbot Web Scraper | Reads a public page, summarizes it and answers follow-up questions from the page text only | Google Gemini | Public-sites-only address check, robots.txt respected, source quotes verified |
| AI Invoice Assistant | Extracts invoice fields, runs rule checks and recommends approve, review or reject | Groq | AI extracts only, rules decide, changed-bank-details detection |

---

## Shared Design

Every playground workflow follows the same pattern:

- 🌐 Webhook trigger with limited allowed origins
- 🧹 Input cleaning and validation before anything else runs
- 🤖 Human check with Cloudflare Turnstile
- ⏱️ Rate limiting with workflow static data
- 🧠 AI step with no tools, treating user content as data
- ✅ Output validation in code, so the AI never has the final say
- 📬 Optional result email, sent after the page already has its answer
- 🔒 No user data stored by the workflow

---

## Common Integrations

Depending on the workflow, integrations may include:

- Cloudflare Turnstile
- Groq
- Google Gemini
- Gmail
- SerpAPI
- Webhooks
- REST APIs
- LangChain (n8n AI nodes)

---

## Purpose

The goal of this category is to demonstrate practical, safe-by-design AI automations that anyone can try from a web page, and to show how to combine AI with fixed rules, rate limiting and human review.

Each workflow is self-contained and includes:

- 📄 A detailed README
- 📦 A cleaned n8n workflow (.json)
- 🖼️ A workflow canvas screenshot

---

## Status

🚧 This category is continuously expanding as new playground workflows are developed, and existing workflows may be updated.
