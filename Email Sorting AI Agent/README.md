# 📧 Email Sorting AI Agent

An AI-powered email triage and response-assistance workflow built with **n8n**, **Gmail**, **OpenRouter**, **Google Gemini**, **DeepSeek**, and **Telegram**.

The workflow automatically monitors incoming Gmail messages, classifies them into business-relevant categories, applies Gmail labels, and uses specialized AI agents to generate context-aware response drafts. High-priority emails receive additional treatment through an AI-generated Telegram alert and an email response draft.

> **Workflow Type:** AI Email Automation
> **Platform:** n8n
> **Status:** Inactive / Ready for Configuration
> **Primary Use Case:** Automated email classification, prioritization, labeling, and AI-assisted response drafting

---

## Overview

Managing a busy inbox often requires manually reading, categorizing, prioritizing, labeling, and responding to every incoming message.

This workflow automates that process by combining **rule-based routing through AI classification** with **specialized AI agents**.

When a new email arrives:

1. Gmail detects the incoming message.
2. The email content is passed to an AI classifier.
3. The classifier determines the appropriate category.
4. The workflow applies the corresponding Gmail label.
5. The categorized email is routed to a specialized AI agent.
6. The agent analyzes the email and generates an appropriate response.
7. High-priority emails receive additional escalation handling.
8. The generated response is parsed and converted into a clean data structure.
9. A Gmail draft is created in the original email thread.
10. High-priority emails additionally trigger a Telegram alert.

The workflow therefore acts as an **AI-powered inbox assistant rather than an autonomous email sender**.

---

## Key Features

* 📥 Automatic Gmail email monitoring
* 🧠 AI-powered email classification
* 🏷️ Automatic Gmail label assignment
* 🚨 High-priority email detection
* 🤖 Specialized AI agents for different email types
* ✍️ AI-generated response drafts
* 💬 Telegram alerts for urgent emails
* 🧵 Gmail thread-aware draft creation
* 🧹 Structured parsing of AI-generated JSON
* 🔀 Category-based workflow routing
* 🧩 Modular architecture using specialized processing branches
* 🔐 Credential placeholders for safe workflow sharing

---

# Architecture

```text
                         ┌─────────────────────┐
                         │    Gmail Trigger    │
                         │   New Email         │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │   Email Classifier  │
                         │      AI Router      │
                         └──────────┬──────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
              ▼                     ▼                     ▼
      ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
      │ High Priority │     │Customer       │     │Finance /      │
      │               │     │Support        │     │Billing        │
      └───────┬───────┘     └───────┬───────┘     └───────┬───────┘
              │                     │                     │
              ▼                     ▼                     ▼
      ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
      │ High Priority │     │ Customer      │     │ Finance       │
      │ Agent         │     │ Support Agent │     │ Agent         │
      └───────┬───────┘     └───────┬───────┘     └───────┬───────┘
              │                     │                     │
              ▼                     └──────────┬──────────┘
      ┌───────────────┐                         │
      │ Parse High    │                         ▼
      │ Priority      │                ┌─────────────────┐
      └───────┬───────┘                │ Parse Email     │
              │                        │ Message         │
              │                        └────────┬────────┘
              │                                 │
              ├──────────────┐                  │
              ▼              ▼                  ▼
      ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
      │   Telegram   │ │ Gmail Draft  │ │ Gmail Draft  │
      │    Alert     │ │              │ │              │
      └──────────────┘ └──────────────┘ └──────────────┘

                         Personal /
                       Out of Scope
                              │
                              ▼
                         ┌─────────┐
                         │  NoOp   │
                         └─────────┘
```

---

# Email Classification

The workflow uses the n8n **Text Classifier** node to categorize incoming emails.

The classifier receives:

* Sender email
* Email subject
* Email body

The configured classification categories are:

| Category                    | Purpose                                                                                                   |
| --------------------------- | --------------------------------------------------------------------------------------------------------- |
| **High Priority**           | Immediate action, time-sensitive updates, executive escalations, system outages, or critical issues       |
| **Customer Support**        | Product usage questions, troubleshooting, technical bugs, account access problems, and feature requests   |
| **Finance/Billing**         | Invoices, receipts, subscriptions, billing updates, expenses, taxes, and payment issues                   |
| **Personal / Out of Scope** | Personal correspondence, social updates, internal banter, or messages outside automated business handling |

The classifier has four connected processing branches, with an additional unused output available in the node configuration.

---

# AI Models

The workflow uses multiple AI models for different responsibilities.

## DeepSeek

The email classification stage uses:

```text
deepseek/deepseek-v4-flash-0731
```

through OpenRouter.

Its responsibility is to classify incoming emails into the predefined categories.

## Google Gemini

The specialized AI agents use:

```text
google/gemini-2.5-flash
```

through OpenRouter.

Gemini powers:

* High Priority Agent
* Customer Support Agent
* Finance Agent

This separation allows the classifier and response-generation stages to use different models according to their respective roles.

---

# Workflow Components

## 1. Gmail Trigger

**Node:** `emailReceived`

The workflow begins with a Gmail Trigger configured to poll for new messages every minute.

The trigger retrieves a maximum of one result per polling cycle and passes the incoming email into the classification stage.

The email data used throughout the workflow includes information such as:

```text
Message ID
Thread ID
Sender Name
Sender Email
Subject
Body
Date
```

---

## 2. Email Classifier

**Node:** `Email Classifier`

The Text Classifier analyzes the incoming email using the following structure:

```text
From: sender@example.com
Subject: Email Subject
Body: Email Body
```

It then routes the message to one of the four categories:

```text
High Priority
Customer Support
Finance/Billing
Personal / Out of Scope
```

---

# 3. Gmail Labeling

Once an email has been classified, the workflow applies a corresponding Gmail label.

### High Priority

**Node:** `High Priority`

Adds the configured Gmail label to the incoming message.

### Customer Support

**Node:** `Customer Support`

Adds the configured Customer Support Gmail label.

### Finance/Billing

**Node:** `Finance/BIlling`

Adds the configured Finance/Billing Gmail label.

### Personal / Out of Scope

**Node:** `Personal / Out of Scope`

Uses a No Operation node.

This branch intentionally stops automated processing for emails that do not require business handling.

---

# 4. High Priority Processing

High-priority messages receive the most extensive processing path.

```text
High Priority
      │
      ▼
High Priority Agent
      │
      ▼
parseHighPriorityMessage
      │
      ├──────────────► Gmail Draft
      │
      └──────────────► Telegram Alert
```

The High Priority Agent is designed to behave like a proactive executive assistant.

It generates **two outputs**:

### Telegram Alert

A concise alert containing:

* Sender
* Subject
* Situation summary
* Required action

The generated message uses Telegram HTML formatting.

Example structure:

```text
🚨 HIGH PRIORITY EMAIL

From: Sender Name
Subject: Subject Line

Summary:
Brief explanation of the situation.

Action Needed:
Required immediate action.
```

### Email Draft

The agent also creates a professional response draft that:

* Acknowledges the issue
* Describes immediate action
* Sets expectations for an update
* Uses a natural communication style

The agent is explicitly instructed to avoid robotic or generic corporate language.

---

# 5. High Priority Response Parser

**Node:** `parseHighPriorityMessage`

The AI agent returns JSON as a string.

This Code node:

1. Retrieves the AI output.
2. Removes Markdown JSON code fences.
3. Parses the string into an object.
4. Extracts:

```text
telegram_alert
email_draft
```

5. Passes those fields to downstream nodes.

This creates a clean interface between the AI generation stage and the communication stage.

---

# 6. Telegram Escalation

**Node:** `Send a text message`

High-priority alerts are sent through Telegram.

The Telegram node receives:

```text
$json.telegram_alert
```

The workflow therefore provides an additional real-time notification channel for urgent email events.

---

# 7. Customer Support Agent

**Node:** `Customer Support Agent`

Customer Support emails are passed to a specialized AI agent.

Its role is to act as an empathetic customer support specialist.

The agent is designed to:

* Understand the customer's specific issue
* Provide troubleshooting guidance
* Investigate account or system information when necessary
* Ask clarifying questions when important information is missing
* Produce concise and friendly responses

The agent returns:

```json
{
  "email_draft": "..."
}
```

rather than automatically sending the response.

---

# 8. Finance Agent

**Node:** `Finance Agent`

Finance/Billing messages are handled by a dedicated AI agent.

Its responsibilities include:

* Acknowledging invoices
* Handling payment notifications
* Addressing billing inquiries
* Identifying missing financial information
* Suggesting appropriate next steps
* Preparing professional finance-related responses

Potential missing information may include:

```text
Invoice Number
PO Number
W-9
Payment Reference
```

The resulting output is an email draft.

---

# 9. Email Response Parser

**Node:** `parseEmailMessage`

Both the Customer Support and Finance branches eventually converge into this parser.

The parser:

1. Reads the raw AI output.
2. Removes Markdown code fences.
3. Parses the JSON.
4. Extracts the `email_draft` field.
5. Sends the clean draft to Gmail.

This allows multiple AI agents to share the same downstream response-generation infrastructure.

---

# 10. Gmail Draft Creation

**Node:** `draftEmail`

The workflow does **not automatically send AI-generated responses**.

Instead, it creates a Gmail draft.

The draft uses:

```text
Subject:
Re: Original Subject
```

and preserves the original Gmail thread using the original `threadId`.

The generated message is inserted using:

```text
$json.email_draft
```

This provides a human-in-the-loop approval layer before any response is sent.

---

# Human-in-the-Loop Design

One of the most important design choices in this workflow is that AI does not directly send email responses.

Instead:

```text
Incoming Email
      ↓
AI Analysis
      ↓
AI Response Generation
      ↓
Gmail Draft
      ↓
Human Review
      ↓
Manual Send
```

This reduces the risk of an AI agent sending an inappropriate response automatically.

The exception is the high-priority Telegram notification, which is designed as an internal alert rather than a customer-facing response.

---

# Node Inventory

The workflow contains the following functional nodes:

| Node                       | Type                  | Purpose                      |
| -------------------------- | --------------------- | ---------------------------- |
| `emailReceived`            | Gmail Trigger         | Detect incoming emails       |
| `Email Classifier`         | Text Classifier       | Categorize emails            |
| `deepseek-v4`              | OpenRouter Chat Model | Classification model         |
| `High Priority`            | Gmail                 | Apply priority label         |
| `Customer Support`         | Gmail                 | Apply support label          |
| `Finance/BIlling`          | Gmail                 | Apply finance label          |
| `Personal / Out of Scope`  | NoOp                  | Stop non-business processing |
| `High Priority Agent`      | AI Agent              | Analyze urgent emails        |
| `Customer Support Agent`   | AI Agent              | Generate support responses   |
| `Finance Agent`            | AI Agent              | Generate finance responses   |
| `parseHighPriorityMessage` | Code                  | Parse urgent AI JSON         |
| `parseEmailMessage`        | Code                  | Parse standard AI JSON       |
| `draftEmail`               | Gmail                 | Create response draft        |
| `Send a text message`      | Telegram              | Send urgent alert            |
| `gemini`                   | OpenRouter Chat Model | Power specialized agents     |

The workflow also contains sticky-note documentation groups for:

* Classify Email
* Label Email
* Generate AI Response
* Parse AI Response
* Send Response

These visual groups document the workflow's major processing stages.

---

# Processing Flow

## Standard Email

```text
Gmail
  │
  ▼
Email Classifier
  │
  ├── Customer Support
  │        │
  │        ▼
  │   Gmail Label
  │        │
  │        ▼
  │   Customer Support Agent
  │        │
  │        ▼
  │   Parse Email
  │        │
  │        ▼
  │   Gmail Draft
  │
  └── Finance/Billing
           │
           ▼
      Gmail Label
           │
           ▼
      Finance Agent
           │
           ▼
      Parse Email
           │
           ▼
      Gmail Draft
```

---

## High Priority Email

```text
Gmail
  │
  ▼
Email Classifier
  │
  ▼
High Priority
  │
  ▼
Gmail Label
  │
  ▼
High Priority Agent
  │
  ▼
Parse AI Response
  │
  ├──────────────► Telegram Alert
  │
  └──────────────► Gmail Draft
```

---

## Personal / Out of Scope

```text
Gmail
  │
  ▼
Email Classifier
  │
  ▼
Personal / Out of Scope
  │
  ▼
NoOp
```

No AI response is generated for this category.

---

# Data Flow

The workflow passes email information between nodes using n8n expressions.

Important fields include:

```text
id
threadId
subject
text
date
from.value[0].name
from.value[0].address
```

AI agents receive the original email context so they can generate responses based on the actual message rather than only its classification.

For example:

```text
ID
Thread ID
Subject
Sender Name
Sender Email
Body
Date
```

are passed into the specialized agents.

---

# Credentials

The exported workflow intentionally contains placeholder credential references.

Before running the workflow, configure:

### Gmail

A Gmail OAuth2 credential is required for:

* Gmail Trigger
* Gmail label operations
* Gmail draft creation

The exported workflow uses placeholders such as:

```text
YOUR_GOOGLE_CREDENTIAL_ID
```

### OpenRouter

An OpenRouter API credential is required for:

* DeepSeek
* Gemini

The exported workflow uses:

```text
YOUR_CREDENTIAL_ID
```

### Telegram

A Telegram API credential is required for the high-priority notification node.

The workflow also contains a configured Telegram chat ID that should be replaced with the intended destination before production use.

---

# Setup

## 1. Import the Workflow

Import the provided JSON workflow into your n8n instance.

## 2. Configure Gmail

Create or select a Gmail OAuth2 credential.

Connect it to:

```text
emailReceived
High Priority
Customer Support
Finance/BIlling
draftEmail
```

## 3. Configure OpenRouter

Create an OpenRouter credential and connect it to the AI model nodes.

The workflow expects:

```text
DeepSeek
Gemini
```

through OpenRouter.

## 4. Configure Telegram

Create a Telegram credential and connect it to:

```text
Send a text message
```

Update the destination chat ID as required.

## 5. Configure Gmail Labels

Create the required Gmail labels and make sure the label IDs configured in the Gmail nodes correspond to the labels in your Gmail account.

## 6. Review AI Prompts

Review the system prompts for:

* High Priority Agent
* Customer Support Agent
* Finance Agent

Customize:

* Organization terminology
* Response tone
* Escalation procedures
* Signature
* Support policies
* Finance policies

## 7. Test Before Activation

Send test emails representing each category:

```text
High Priority
Customer Support
Finance/Billing
Personal / Out of Scope
```

Verify that:

* Classification is correct
* Gmail labels are applied correctly
* AI responses are appropriate
* Gmail drafts appear in the correct thread
* Telegram alerts are generated for high-priority emails

## 8. Activate the Workflow

The exported workflow is currently configured as:

```text
active: false
```

so it should be treated as an imported, inactive workflow until credentials and configuration have been completed.

---

# Example Use Cases

## 🚨 Executive Escalation

An executive sends an email about a critical operational issue.

The workflow:

```text
Classifies → High Priority
Labels → High Priority
Analyzes → AI Agent
Alerts → Telegram
Creates → Gmail Draft
```

---

## 🛠️ Customer Technical Issue

A customer reports that a product feature is not working.

The workflow:

```text
Classifies → Customer Support
Labels → Customer Support
Analyzes → Customer Support Agent
Creates → Gmail Draft
```

---

## 💳 Invoice or Billing Request

A vendor sends an invoice.

The workflow:

```text
Classifies → Finance/Billing
Labels → Finance/Billing
Analyzes → Finance Agent
Creates → Gmail Draft
```

---

## 👤 Personal Email

A friend sends a personal message.

The workflow:

```text
Classifies → Personal / Out of Scope
      ↓
    NoOp
```

No automated response is generated.

---

# AI Agent Responsibilities

| Agent                      | Role                               | Output                       |
| -------------------------- | ---------------------------------- | ---------------------------- |
| **High Priority Agent**    | Executive / urgent email assistant | Telegram alert + email draft |
| **Customer Support Agent** | Customer support specialist        | Email draft                  |
| **Finance Agent**          | Finance & operations assistant     | Email draft                  |

Each agent has a dedicated system prompt and is designed around its specific business context rather than using a single generic email assistant.

---

# Design Principles

## 1. Specialized AI

Instead of using one general-purpose AI agent for every email, the workflow separates responsibilities.

```text
Urgent → Executive Assistant
Support → Customer Support Specialist
Finance → Finance Assistant
```

This makes the prompts more focused and easier to maintain.

---

## 2. Human Approval

AI generates drafts instead of automatically sending customer-facing emails.

This creates an approval checkpoint:

```text
AI → Draft → Human → Send
```

---

## 3. Modular Routing

The classifier acts as the central routing layer.

Each category has its own processing path, allowing individual branches to be modified without redesigning the entire workflow.

---

## 4. Structured AI Output

The AI agents are instructed to return JSON.

Code nodes then parse the JSON before passing the information to downstream nodes.

This creates a predictable interface between AI and automation components.

---

## 5. Urgency-Based Escalation

High-priority messages receive additional notification handling through Telegram.

This separates ordinary email assistance from urgent operational awareness.

---

# Error Considerations

The current workflow uses direct JSON parsing through Code nodes.

For example:

```javascript
JSON.parse(cleanJsonString)
```

If an AI model returns malformed JSON, the parsing node can fail.

For production deployment, consider adding:

* JSON validation
* Try/catch handling
* AI retry logic
* Invalid-output fallback
* Error notifications
* Execution logging
* Dead-letter handling

These improvements would make the workflow more resilient to unexpected model output.

---

# Security Considerations

Before sharing or deploying the workflow:

* Replace all placeholder credentials.
* Avoid exposing OAuth credentials.
* Avoid exposing Telegram destination IDs unnecessarily.
* Review AI prompts for sensitive business information.
* Review email data before sending it to external AI providers.
* Restrict Gmail permissions to the minimum required scope.
* Review OpenRouter/model data handling requirements.
* Do not commit secrets to Git repositories.

The exported workflow already uses placeholder credential identifiers rather than embedding actual credentials.

---

# Production Improvements

Potential future enhancements include:

### Advanced Classification

Add categories such as:

```text
Sales
HR
Legal
IT Operations
Marketing
Procurement
Spam
Internal
```

### Priority Scoring

Instead of only classifying into High Priority or other categories, calculate:

```text
Priority Score
Urgency
Business Impact
Sender Importance
SLA
```

### Confidence-Based Routing

Use AI confidence to determine whether:

```text
High Confidence → Automatic processing
Medium Confidence → Human review
Low Confidence → Manual triage
```

### Automated Follow-Up

Add reminders for drafts that remain unsent.

### Response Quality Validation

Introduce a second AI validation layer that checks:

* Accuracy
* Tone
* Missing information
* Sensitive content
* Policy violations

before creating the final draft.

### Observability

Add:

* Execution logging
* Classification statistics
* AI usage tracking
* Error tracking
* Response-generation metrics

---

# Technology Stack

| Technology        | Purpose                               |
| ----------------- | ------------------------------------- |
| **n8n**           | Workflow orchestration                |
| **Gmail**         | Email ingestion, labeling, and drafts |
| **OpenRouter**    | AI model gateway                      |
| **DeepSeek**      | Email classification                  |
| **Google Gemini** | Specialized AI agents                 |
| **Telegram**      | High-priority alerts                  |
| **JavaScript**    | AI output parsing and transformation  |

---

# Workflow Status

| Component              | Status                     |
| ---------------------- | -------------------------- |
| Gmail Trigger          | Configured                 |
| AI Classification      | Configured                 |
| Gmail Labeling         | Configured                 |
| High Priority Agent    | Configured                 |
| Customer Support Agent | Configured                 |
| Finance Agent          | Configured                 |
| AI Response Parsing    | Configured                 |
| Gmail Draft Creation   | Configured                 |
| Telegram Alerting      | Configured                 |
| Credentials            | **Requires configuration** |
| Production Activation  | **Pending**                |

The exported workflow itself is currently inactive.

---

# Workflow Philosophy

This project demonstrates a practical approach to **AI-assisted business automation**.

Rather than allowing an AI agent to independently control an inbox, the workflow separates the process into distinct stages:

```text
INGEST
   ↓
CLASSIFY
   ↓
LABEL
   ↓
SPECIALIZE
   ↓
GENERATE
   ↓
PARSE
   ↓
DRAFT / ALERT
   ↓
HUMAN REVIEW
```

This architecture provides a balance between **automation, specialization, and human oversight**.

The AI handles the repetitive cognitive work while the human retains final control over external email communication.

---

# Repository Structure

A recommended repository structure for this workflow is:

```text
email-sorting-ai-agent/
│
├── README.md
├── workflow/
│   └── email-sorting-ai-agent.json
│
├── docs/
│   ├── architecture.md
│   └── setup.md
│
└── screenshots/
    ├── workflow-overview.png
    ├── classification.png
    └── ai-agents.png
```

---

# Disclaimer

This workflow is designed as an **AI automation and portfolio project**.

The AI-generated responses should be reviewed by a human before being sent, particularly when emails involve:

* Financial decisions
* Legal matters
* Security incidents
* Customer account changes
* Executive communications
* Sensitive business information

AI-generated content should not be treated as authoritative without appropriate human verification.

---

# License

This project can be adapted for personal, educational, portfolio, or internal automation purposes.

If publishing the workflow publicly, ensure that:

* Credentials are removed.
* Private identifiers are removed.
* Telegram destinations are sanitized.
* Sensitive email examples are anonymized.
* Third-party service terms are respected.

---

## Built With

**n8n · Gmail · OpenRouter · DeepSeek · Google Gemini · Telegram · JavaScript**

> **Automate the inbox. Keep the human in control.**
