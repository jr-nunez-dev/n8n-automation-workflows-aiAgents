# AI Customer Support Triage & Auto-Reply Agent

An n8n workflow that automatically reads incoming support emails, classifies them, looks up real information (order status or company policy), drafts a response, and either auto-sends it or escalates to a human — depending on how confident the AI actually is.

Built with: **n8n**, **Claude Sonnet 5** (AI Agent brain), **Supabase + pgvector** (knowledge base / RAG), **Google Gemini Embeddings**, **Gmail**, and **Slack**.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Repo Structure](#repo-structure)
- [Main Workflow — Node by Node](#main-workflow--node-by-node)
- [Subworkflow 1: Mock Order API](#subworkflow-1-mock-order-api)
- [Subworkflow 2: Knowledge Base Ingestion](#subworkflow-2-knowledge-base-ingestion)
- [Setup Guide](#setup-guide)
- [Q&A — Understanding This Workflow](#qa--understanding-this-workflow)
- [Key Concepts: RAG & Vectors](#key-concepts-rag--vectors)
- [Known Limitations / Notes](#known-limitations--notes)

---

## Overview

This project automates the first line of customer support. Instead of a human reading every incoming email to figure out what it's about, this workflow does that automatically:

1. Polls a Gmail inbox for new support emails
2. Uses an **AI Agent** (Claude Sonnet 5) to read and classify each email
3. Gives the Agent **tools** to look up real information — order status via an API, and policy/FAQ answers via a searchable knowledge base (RAG)
4. The Agent scores its own confidence in the answer
5. High-confidence, non-urgent emails get an **automatic reply**
6. Low-confidence, urgent, or ambiguous emails get **escalated to a human via Slack**, with a pre-drafted reply ready for review
7. Spam gets labeled and set aside
8. Every processed email is logged to a Google Sheet as an audit trail
9. If any step fails, a Slack alert fires so nothing silently falls through the cracks

---

## Architecture

```
                                  ┌─────────────────────┐
                                  │   Gmail Trigger      │
                                  │   (Poll Emails)      │
                                  └──────────┬───────────┘
                                             │
                                  ┌──────────▼───────────┐
                                  │  Edit Fields          │
                                  │  (Get Data)           │
                                  └──────────┬───────────┘
                                             │
                        ┌────────────────────▼────────────────────┐
                        │              AI Agent                    │
                        │        (Claude Sonnet 5)                 │
                        │                                          │
                        │  Tools:                                  │
                        │   • Supabase Vector Store (KB / RAG)     │
                        │   • HTTP Request (Order Lookup)          │
                        │  Memory:                                 │
                        │   • Simple Memory (per sender)           │
                        └────────────────────┬────────────────────┘
                                             │
                                  ┌──────────▼───────────┐
                                  │  Parse Agent Output   │
                                  │  (Code node)          │
                                  └──────────┬───────────┘
                                             │
                                  ┌──────────▼───────────┐
                                  │       Switch          │
                                  └──┬────────┬────────┬──┘
                              urgent │   spam │        │ default
                     ┌───────────────┘        │        └───────────────┐
                     │                        │                        │
          ┌──────────▼─────────┐   ┌──────────▼─────────┐   ┌──────────▼─────────┐
          │  Slack Escalation   │   │  Gmail Label Spam    │   │        IF           │
          │  (Urgent)           │   │                       │   │  confidence >= 0.8  │
          └──────────┬─────────┘   └──────────┬─────────┘   └──┬────────────────┬───┘
                     │                        │           true │           false │
                     │                        │      ┌─────────▼───────┐ ┌───────▼──────────┐
                     │                        │      │  Gmail Reply     │ │ Slack Escalation  │
                     │                        │      │  (Auto-send)     │ │ (Low Confidence)  │
                     │                        │      └─────────┬───────┘ └───────┬──────────┘
                     └────────────────────────┴────────────────┴────────────────┘
                                             │
                                  ┌──────────▼───────────┐
                                  │        Merge          │
                                  └──────────┬───────────┘
                                             │
                                  ┌──────────▼───────────┐
                                  │  Google Sheets         │
                                  │  (Log Ticket)          │
                                  └────────────────────────┘

     Error output on any node → Slack Error Notification
```

---

## Repo Structure

```
/
├── main-workflow.json           # The primary support triage workflow
├── mock-order-api.json          # Standalone workflow simulating an order/CRM API
├── kb-ingestion.json            # Standalone workflow that loads policy docs into Supabase
├── sql/
│   └── support_kb_documents.sql # Table + match function for the knowledge base
└── README.md                    # This file
```

---

## Main Workflow — Node by Node

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | **Poll Emails** | Gmail Trigger | Watches the support inbox, fires on every new email |
| 2 | **Get Data** | Edit Fields (Set) | Extracts clean `sender`, `subject`, `body`, `messageId` from the raw Gmail payload |
| 3 | **AI Agent** | AI Agent | Core reasoning node — classifies the email, calls tools, drafts a reply, outputs structured JSON |
| 4 | **Claude Sonnet-5** | Chat Model | The language model powering the Agent |
| 5 | **Memory** | Simple Memory | Keeps conversation context per customer (keyed by sender email) |
| 6 | **Supabase Vector Store** | Tool (Retrieve) | RAG tool — semantic search over the company knowledge base for policy/FAQ questions |
| 7 | **Embeddings Google Gemini** | Embeddings | Converts the customer's question into a vector for similarity search |
| 8 | **Orders (HTTP Request Tool)** | Tool | Looks up order status by order ID or email, via the Mock Order API |
| 9 | **Parse Agent Output** | Code | Converts the Agent's raw JSON-text output into real fields; strips markdown code fences if present |
| 10 | **Switch** | Switch | Routes based on `classification`: `urgent` / `spam` / default |
| 11 | **IF (Confidence Check)** | IF | For non-urgent/non-spam emails, checks whether `confidence >= 0.8` |
| 12 | **Set (action_taken)** | Set (×4) | Tags each branch with what actually happened, for logging |
| 13 | **Gmail Reply** | Gmail | Sends the AI-drafted reply automatically (high-confidence path only) |
| 14 | **Slack Escalation (Urgent)** | Slack | Alerts a human immediately for urgent emails |
| 15 | **Slack Escalation (Low Confidence)** | Slack | Alerts a human when the AI isn't confident enough to auto-send |
| 16 | **Gmail Label Spam** | Gmail | Tags spam emails with a label instead of replying |
| 17 | **Merge** | Merge | Combines all 4 branches back into a single stream |
| 18 | **Log Ticket** | Google Sheets | Appends every processed email (classification, confidence, action, timestamp) to an audit log |
| 19 | **Error Notification** | Slack (error branch) | Fires only if a node fails, so a broken run doesn't silently drop a customer email |

---

## Subworkflow 1: Mock Order API

**File:** `mock-order-api.json`

**Role:** A stand-in for a real order/CRM system (Shopify, WooCommerce, internal API, etc.). Since this project doesn't connect to a live e-commerce backend, this subworkflow simulates one — behaving exactly like a real API would, so the main workflow's HTTP Request Tool has something real to call during development and testing.

| Node | Purpose |
|---|---|
| **Webhook** (`GET /orders`) | Listens for a request with `order_id` or `email` as query parameters — the same contract a real order API would use |
| **Code** | Holds a hardcoded mock order list and performs the actual lookup (by exact order ID, or all orders matching an email) |
| **Respond to Webhook** | Returns the matched order(s), or an error message, as JSON |

**Why it matters:** because the interface (query params in → JSON out) matches a real API, swapping this out for a production order system later requires changing only the URL in the HTTP Request Tool — nothing else in the main workflow needs to change.

---

## Subworkflow 2: Knowledge Base Ingestion

**File:** `kb-ingestion.json`

**Role:** The "loading dock" for the RAG knowledge base. This is how company policy text (refund policy, shipping policy, cancellation policy, etc.) actually gets into Supabase in a searchable form. It's run manually, on demand — whenever policies are added or updated — separate from the always-on main workflow.

| Node | Purpose |
|---|---|
| **Manual Trigger** | Run by hand whenever policy content changes |
| **Source content** (file or Set node) | The raw policy text to be ingested |
| **Default Data Loader** | Wraps the raw text into a "document" object the pipeline can process |
| **Recursive Character Text Splitter** | Breaks long policy text into smaller overlapping chunks, so retrieval later can return the *specific relevant paragraph* rather than one giant blob |
| **Embeddings Google Gemini** | Converts each text chunk into a vector |
| **Supabase Vector Store (Insert mode)** | Writes each chunk + its vector into the `support_kb_documents` table |

**Why it's separate:** ingestion (writing to the knowledge base) and retrieval (searching it) are different concerns run at different times — you update the knowledge base occasionally, but query it on every single support email.

---

## Setup Guide

1. **Import all 3 workflows** into your n8n instance.
2. **Credentials needed:**
   - Gmail (OAuth2)
   - Slack (Bot Token, `chat:write` + `chat:write.public` + `channels:read` scopes)
   - Supabase (Project URL + service_role key)
   - Google Gemini API key (for embeddings)
   - Anthropic API key (for Claude Sonnet 5)
   - Google Sheets (OAuth2)
3. **Set up the Supabase table** — run `sql/support_kb_documents.sql` in the Supabase SQL Editor.
4. **Activate the Mock Order API workflow** first, and copy its production webhook URL into the main workflow's HTTP Request Tool node.
5. **Run the KB Ingestion workflow** once to populate the knowledge base with your policy content.
6. **Activate the main workflow.**
7. Send a test email to the connected inbox to verify the full pipeline.

---

## Q&A — Understanding This Workflow

### 1. What is this workflow?

It's an AI Customer Support Triage & Auto-Reply Agent — a system that watches a support inbox, reads each incoming email, figures out what the customer needs, looks up real information (order status or policy details), decides whether it can confidently answer on its own, and either replies automatically or hands it to a human — while logging everything.

### 2. Why do companies need this workflow?

- Response time is one of the biggest drivers of customer satisfaction in support.
- Support volume tends to scale faster than headcount, especially for growing businesses.
- Most support emails are repetitive ("where's my order," "what's your refund policy") — easy questions that don't need a human but still eat a human's time today.
- Customers email at all hours; a human team doesn't work 24/7 — this does.

### 3. What problem did it solve?

Support inboxes typically fail in one of two ways: they're **slow** (everything waits in a queue for manual triage) or **inconsistent** (different agents answer the same question differently). This workflow removes the triage bottleneck specifically — it doesn't replace judgment on hard cases (that's what escalation is for), it removes the busywork of reading, classifying, looking up basic facts, and drafting a first response.

### 4. What improvements did it bring?

- **Speed** — routine questions get answered in seconds instead of hours.
- **Consistency** — every reply is grounded in the same source of truth (the knowledge base), not whichever agent happened to remember the policy correctly.
- **Smart escalation, not blind automation** — the confidence score means uncertain cases still reach a human, with full context and a draft already prepared.
- **Auditability** — the ticket log surfaces patterns over time (common questions, how often the AI needed help).
- **Safety net** — the error-handling branch means a failure becomes a Slack alert instead of a silently dropped email.

### 5. Explain every node, the process, and what it's for (including the two subworkflows)

See [Main Workflow — Node by Node](#main-workflow--node-by-node), [Subworkflow 1: Mock Order API](#subworkflow-1-mock-order-api), and [Subworkflow 2: Knowledge Base Ingestion](#subworkflow-2-knowledge-base-ingestion) above for the full breakdown.

In short: **Poll Emails → Get Data → AI Agent** (with Memory, a Supabase RAG tool for policy questions, and an HTTP Request tool for order lookups) **→ Parse Agent Output → Switch/IF routing → Gmail Reply / Slack Escalation / Gmail Spam Label → Merge → Log Ticket**, with an error branch feeding a Slack alert. The **Mock Order API** subworkflow simulates a real order system so the Orders tool has something to call. The **KB Ingestion** subworkflow is the one-time/occasional process that loads policy text into Supabase so the RAG tool has something to search.

---

## Key Concepts: RAG & Vectors

**RAG (Retrieval-Augmented Generation)** is a technique for giving an AI accurate, current information it wasn't trained on, instead of relying on it to "remember" facts — which it can't, for private company data, and shouldn't be trusted to guess. Before answering, the AI first *retrieves* relevant real documents, then *generates* its answer grounded in what it just found. In this workflow, instead of asking Claude to guess a refund policy, the Supabase Vector Store tool first finds the actual policy text, and Claude answers based on that — which is why the Agent correctly says "I can't confirm specifics" rather than making something up when the knowledge base has no matching content.

**Vectors / embeddings** are how "search by meaning" works instead of "search by exact keyword." An embedding model (Google Gemini here) converts text into a long list of numbers representing its meaning — similar meanings produce similar number patterns, even with completely different wording. A customer asking "my item showed up broken, can I get money back?" doesn't literally contain the words "refund policy," but its embedding vector ends up mathematically close to the refund policy chunk's vector, because they mean similar things. Supabase's `pgvector` extension compares these vectors efficiently (via the `<=>` distance operator) to find the closest matches, which are then handed to Claude to build a grounded answer.

---

## Known Limitations / Notes

- Order and CRM data is currently **mocked** — swap the Mock Order API's URL in the HTTP Request Tool for a real order system when ready for production.
- The knowledge base currently holds a small, manually-entered policy set — intended as a proof of concept, not exhaustive documentation.
- Token usage is not currently logged per-email; check your model provider's usage dashboard for cost tracking.
- Auto-reply confidence threshold (`0.8`) is a starting point — tune based on observed accuracy over time.
