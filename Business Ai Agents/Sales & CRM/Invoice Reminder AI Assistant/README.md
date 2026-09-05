# Invoice Reminder Assistant

An n8n workflow that monitors incoming Gmail messages, identifies
invoice-related emails, extracts and normalizes invoice information with
AI, calculates payment timing, tracks invoice status, logs invoice data
to Google Sheets, and sends status-based notifications.

## Overview

The **Invoice Reminder Assistant** is designed to automate a common
billing workflow:

1.  Monitor incoming email.
2.  Determine whether the email is an invoice.
3.  Label valid invoice emails in Gmail.
4.  Use an AI Agent to extract invoice information.
5.  Normalize and validate the AI response in a Code node.
6.  Calculate the due date when necessary.
7.  Calculate the number of days until or past the due date.
8.  Determine whether the invoice is **Paid**, **Upcoming**, or
    **Overdue**.
9.  Log invoice information to Google Sheets.
10. Route the invoice based on its status.
11. Send the appropriate notification.
12. Send an additional Slack alert for overdue invoices.

The workflow combines **AI extraction** with **deterministic JavaScript
business logic**, allowing the AI to handle unstructured email content
while the Code node handles calculations and status decisions.

------------------------------------------------------------------------

## Workflow Architecture

``` text
                         ┌─────────────────────┐
                         │     Gmail Trigger   │
                         │     invoiceEmail     │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │  Classify Invoice   │
                         │   Text Classifier   │
                         └──────────┬──────────┘
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                    Invoice                Non-Invoice
                         │                     │
                         ▼                     ▼
                ┌────────────────┐      ┌──────────────┐
                │ Add Gmail      │      │ non-invoice  │
                │ Invoice Label  │      │    NoOp      │
                └───────┬────────┘      └──────────────┘
                        │
                        ▼
                ┌────────────────┐
                │    AI Agent    │
                │ Invoice Parser │
                └───────┬────────┘
                        │
                        ▼
                ┌────────────────┐
                │   parseData     │
                │ Code / Normalize│
                └───────┬────────┘
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
       ┌──────────────┐    ┌────────────────┐
       │ routeStatus  │    │   logInvoice   │
       │    Switch    │    │  Google Sheets │
       └──────┬───────┘    └────────────────┘
              │
       ┌──────┼───────────────┐
       │      │               │
       ▼      ▼               ▼
     Paid   Upcoming        Overdue
       │      │               │
       ▼      ▼               ├───────────────┐
   paidMessage sendReminder   │               │
                              ▼               ▼
                         sendReminder   overdueSlackAlert
```

------------------------------------------------------------------------

## Core Workflow Flow

### 1. Receive Invoice Email

**Node:** `invoiceEmail`

The workflow begins with a Gmail Trigger that polls for new email
messages every minute.

The trigger provides information such as:

-   Sender name
-   Sender email address
-   Subject
-   Email body
-   Gmail message ID
-   Gmail thread ID

The sender metadata is also used later to populate the customer name.

------------------------------------------------------------------------

### 2. Classify the Email

**Node:** `Classify Invoice`

The Text Classifier determines whether the incoming message is:

-   `invoice`
-   `non-invoice`

### Invoice

An email that contains, references, or communicates a bill or payment
request for goods or services. It may contain invoice details such as:

-   Invoice number
-   Amount due
-   Invoice date
-   Due date
-   Vendor information
-   Payment instructions
-   Attached invoice documentation

### Non-Invoice

An email that does not contain a bill or payment request.

Examples include:

-   General inquiries
-   Quotations
-   Purchase discussions
-   Receipts
-   Payment confirmations
-   Delivery updates
-   Support messages
-   Newsletters
-   Other business communications

Non-invoice messages are sent to the `non-invoice` NoOp branch and are
not processed further.

------------------------------------------------------------------------

### 3. Label the Invoice

**Node:** `addLabel: Invoice`

When the classifier identifies the email as an invoice, the workflow
applies the configured Gmail Invoice label.

This provides a simple way to organize invoice-related messages in
Gmail.

------------------------------------------------------------------------

## 4. AI Invoice Analysis

**Node:** `AI Agent`

The AI Agent analyzes the invoice email and returns structured JSON.

The workflow uses **Gemini** as the configured AI language model for the
Agent, while **DeepSeek** is connected to the Text Classifier.

### AI Responsibilities

The AI Agent is responsible for:

-   Confirming whether the message is an invoice
-   Validating invoice information
-   Assigning a confidence score
-   Extracting invoice fields
-   Identifying payment information
-   Identifying an explicit due date or payment terms
-   Returning structured JSON

### Extracted Invoice Fields

``` json
{
  "invoiceNumber": "...",
  "vendorName": "...",
  "customerName": "...",
  "description": "...",
  "amountDue": 0,
  "currency": "...",
  "invoiceDate": "YYYY-MM-DD",
  "dueDate": "YYYY-MM-DD",
  "paymentTerms": "...",
  "senderEmail": "...",
  "subject": "..."
}
```

The AI is instructed to return `null` when information is unavailable
rather than inventing data.

------------------------------------------------------------------------

## 5. Parse and Normalize the AI Response

**Node:** `parseData`

The Code node converts the AI response into a predictable n8n JSON
structure.

### Processing performed

The Code node:

1.  Retrieves the AI output.
2.  Removes accidental Markdown code fences.
3.  Parses the JSON.
4.  Retrieves the customer name directly from Gmail metadata.
5.  Normalizes the confidence score.
6.  Extracts invoice information.
7.  Uses the AI-provided due date when available.
8.  Attempts to calculate a due date from payment terms when no explicit
    due date exists.
9.  Calculates the number of days until or past the due date.
10. Determines the final invoice status.
11. Returns a clean n8n item.

### Customer Name Handling

The workflow intentionally retrieves the customer name from Gmail
metadata:

``` javascript
const customerName =
  $('invoiceEmail').item.json.from?.value?.[0]?.name ?? null;
```

This avoids relying on the AI to infer a customer name that is already
available from the email metadata.

------------------------------------------------------------------------

## 6. Invoice Status Logic

The Code node determines the final status using payment information and
the calculated due date.

### Paid

``` text
payment.isPaid = true
```

Result:

``` text
Paid
```

### Upcoming

The invoice is unpaid and the due date has not passed.

``` text
daysUntilDue >= 0
```

Result:

``` text
Upcoming
```

### Overdue

The invoice is unpaid and the due date has passed.

``` text
daysUntilDue < 0
```

Result:

``` text
Overdue
```

### Days Until Due

The workflow stores the date difference as:

``` text
Positive number → days remaining
Negative number → days overdue
```

For example:

``` text
10  → 10 days remaining
-5  → 5 days overdue
```

------------------------------------------------------------------------

## 7. Log Invoice Data

**Node:** `logInvoice`

Invoice information is written to Google Sheets using an **Append or
Update** operation.

The invoice number is used as the matching column, allowing an existing
invoice record to be updated rather than blindly creating duplicate
records.

### Logged Fields

  Google Sheets Column   Source
  ---------------------- ----------------------------------------
  Invoice number         `invoice.invoiceNumber`
  Customer               `invoice.customerName`
  Vendor                 `invoice.vendorName`
  Description            `invoice.description`
  Invoice Date           `invoice.invoiceDate`
  Due Date               `invoice.dueDate`
  Status                 `status.value`
  Amount                 `invoice.amountDue + invoice.currency`

This creates a simple invoice tracking layer outside Gmail.

------------------------------------------------------------------------

## 8. Status-Based Routing

**Node:** `routeStatus`

The Switch node routes invoices according to:

``` text
Paid
Upcoming
Overdue
```

### Paid

Routes to:

``` text
paidMessage
```

The workflow replies to the original Gmail thread with a payment
confirmation message.

### Upcoming

Routes to:

``` text
sendReminder
```

The workflow sends a billing reminder containing the invoice information
and remaining days.

### Overdue

Routes to:

``` text
sendReminder
overdueSlackAlert
```

Overdue invoices trigger both:

-   Email notification
-   Slack alert

This provides a stronger escalation path for invoices that have passed
their due date.

------------------------------------------------------------------------

# Notification System

## Paid Notification

**Node:** `paidMessage`

The paid notification is an HTML email that dynamically uses fields from
the parsed invoice JSON.

It includes:

-   Customer name
-   Invoice number
-   Vendor
-   Description
-   Invoice date
-   Due date
-   Amount paid
-   Invoice status

The message is sent as a reply to the original email thread.

------------------------------------------------------------------------

## Upcoming / Overdue Email Notification

**Node:** `sendReminder`

The same HTML template supports both:

-   `Upcoming`
-   `Overdue`

The presentation changes according to the status.

### Upcoming

The notification communicates that:

-   The invoice is approaching its due date.
-   Payment should be scheduled.
-   The billing team should review the invoice.

### Overdue

The notification communicates that:

-   The invoice has passed its due date.
-   Immediate billing attention is required.
-   The payment status should be verified.
-   Follow-up or escalation may be necessary.

The template dynamically changes the header, status presentation,
wording, and recommended action based on:

``` javascript
$json.status.value
```

------------------------------------------------------------------------

## Slack Overdue Alert

**Node:** `overdueSlackAlert`

Overdue invoices generate a Slack alert containing dynamic invoice
information.

The alert includes:

-   Invoice number
-   Vendor
-   Customer
-   Description
-   Amount due
-   Invoice date
-   Due date
-   Current status
-   Days overdue
-   Recommended action

The days overdue value is calculated from the negative `daysUntilDue`
value:

``` javascript
Math.abs($json.status.daysUntilDue)
```

This makes the Slack message easier to read for billing teams.

------------------------------------------------------------------------

# Data Structure

After the `parseData` Code node, the workflow produces a normalized
structure similar to:

``` json
{
  "category": "Invoice",
  "validation": {
    "isValidInvoice": true,
    "confidenceScore": 10,
    "confidenceLevel": "Very High"
  },
  "invoice": {
    "invoiceNumber": "INV-2026-1045",
    "vendorName": "ABC Services",
    "customerName": "Johnrex Nuñez",
    "description": "IT Infrastructure Support Services",
    "amountDue": 2500,
    "currency": "USD",
    "invoiceDate": "2026-09-01",
    "dueDate": "2026-09-15",
    "paymentTerms": null,
    "senderEmail": "sender@example.com",
    "subject": "Invoice INV-2026-1045 — ABC Services"
  },
  "payment": {
    "isPaid": false
  },
  "status": {
    "value": "Upcoming",
    "daysUntilDue": 10
  }
}
```

The exact values depend on the incoming email.

------------------------------------------------------------------------

# Node Reference

  -----------------------------------------------------------------------
  Node                    Type                    Purpose
  ----------------------- ----------------------- -----------------------
  `invoiceEmail`          Gmail Trigger           Detect incoming emails

  `Classify Invoice`      Text Classifier         Separate invoices from
                                                  non-invoices

  `non-invoice`           NoOp                    End non-invoice branch

  `addLabel: Invoice`     Gmail                   Apply Invoice label

  `AI Agent`              AI Agent                Analyze and extract
                                                  invoice data

  `deepseek`              OpenRouter Chat Model   Language model for
                                                  classification

  `gemini`                OpenRouter Chat Model   Language model for
                                                  invoice analysis

  `parseData`             Code                    Parse, normalize,
                                                  calculate, and
                                                  determine status

  `logInvoice`            Google Sheets           Create/update invoice
                                                  tracker record

  `routeStatus`           Switch                  Route
                                                  Paid/Upcoming/Overdue

  `paidMessage`           Gmail                   Send paid confirmation

  `sendReminder`          Gmail                   Send Upcoming/Overdue
                                                  email

  `overdueSlackAlert`     Slack                   Alert billing team
                                                  about overdue invoices
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# Credentials and Configuration

Before importing or activating the workflow, configure the required
credentials.

## Gmail

Required for:

-   Gmail Trigger
-   Gmail label operation
-   Paid notification
-   Billing reminder

The exported workflow intentionally contains credential placeholders.

Replace the placeholder Google credential references with your own n8n
credential.

------------------------------------------------------------------------

## OpenRouter

Required for:

-   DeepSeek model
-   Gemini model

The workflow currently references:

``` text
deepseek/deepseek-v4-flash
```

and:

``` text
google/gemini-3-flash-preview
```

Make sure the corresponding OpenRouter credential is configured in n8n.

------------------------------------------------------------------------

## Google Sheets

Required for:

``` text
logInvoice
```

Configure:

-   Google Sheets credential
-   Spreadsheet
-   Invoice worksheet
-   Matching column

The current workflow uses:

``` text
Invoice number
```

as the matching column.

------------------------------------------------------------------------

## Slack

Required for:

``` text
overdueSlackAlert
```

Configure:

-   Slack credential
-   Destination channel

The exported workflow currently contains a configured channel reference
for the `n8n-automations` channel.

Use your own Slack channel if deploying this workflow in another
environment.

------------------------------------------------------------------------

# Google Sheets Structure

The Invoice worksheet should contain columns corresponding to the
workflow mappings:

``` text
Invoice number
Customer
Vendor
Description
Amount
Invoice Date
Due Date
Status
```

Example:

  -------------------------------------------------------------------------------------------------------
  Invoice number  Customer   Vendor     Description           Amount Invoice Date Due Date     Status
  --------------- ---------- ---------- ---------------- ----------- ------------ ------------ ----------
  INV-2026-1045   Johnrex    ABC        IT                  2500 USD 2026-09-01   2026-09-15   Upcoming
                  Nuñez      Services   Infrastructure                                         
                                        Support Services                                       

  -------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

# Example Scenarios

## Scenario 1 --- Upcoming Invoice

Input:

``` text
Invoice Date: 2026-09-01
Due Date: 2026-09-15
Paid: No
```

Result:

``` json
{
  "status": {
    "value": "Upcoming",
    "daysUntilDue": 10
  }
}
```

Actions:

``` text
Google Sheets → Update invoice
        ↓
Upcoming → Billing email reminder
```

------------------------------------------------------------------------

## Scenario 2 --- Overdue Invoice

Input:

``` text
Invoice Date: 2026-09-01
Due Date: 2026-09-01
Paid: No
```

If the current date is after the due date, the workflow produces:

``` json
{
  "status": {
    "value": "Overdue",
    "daysUntilDue": -5
  }
}
```

Actions:

``` text
Google Sheets → Update invoice
        ↓
Overdue
   ├── Billing email
   └── Slack alert
```

------------------------------------------------------------------------

## Scenario 3 --- Paid Invoice

Input:

``` text
Paid: Yes
```

Result:

``` json
{
  "status": {
    "value": "Paid",
    "daysUntilDue": null
  }
}
```

Action:

``` text
Google Sheets → Update invoice
        ↓
Paid → Payment confirmation email
```

------------------------------------------------------------------------

## Scenario 4 --- Non-Invoice Email

Example:

``` text
Subject: Request for quotation
```

Result:

``` text
Classify Invoice
       ↓
Non-Invoice
       ↓
NoOp
```

No invoice extraction, tracking, or billing notification occurs.

------------------------------------------------------------------------

# Error Handling and Data Safety

The `parseData` node validates that an AI response exists and can be
parsed as JSON.

If the AI response is missing:

``` text
AI response is empty or missing.
```

If the AI returns invalid JSON:

``` text
Invalid JSON returned by AI
```

The AI instructions also require unavailable information to be
represented as:

``` json
null
```

rather than invented values.

This reduces the risk of silently creating incorrect invoice records.

------------------------------------------------------------------------

# Design Principles

## AI for Unstructured Data

AI is used where interpretation is required:

``` text
Email
  ↓
Classification
  ↓
Invoice extraction
```

This is useful because invoice emails may vary in wording and structure.

## Code for Deterministic Logic

The Code node handles deterministic operations:

``` text
Dates
+
Days calculation
+
Payment state
+
Invoice status
```

This makes business logic more predictable than relying entirely on
AI-generated calculations.

## Metadata over Inference

Customer information is retrieved from Gmail metadata where available
instead of asking the AI to infer it from the email body.

## Centralized Status Routing

All status decisions are normalized before reaching the Switch node:

``` text
Paid
Upcoming
Overdue
```

This keeps downstream notification logic simple.

## Separate Processing from Presentation

The workflow separates:

``` text
AI extraction
      ↓
Data normalization
      ↓
Business logic
      ↓
Storage
      ↓
Notification
```

This makes the workflow easier to maintain and extend.

------------------------------------------------------------------------

# Current Workflow Limitations

The following points should be considered before production deployment.

### 1. Gmail polling

The Gmail Trigger is configured to poll every minute. This is suitable
for the current automation but should be evaluated against the desired
production notification frequency.

### 2. Payment confirmation source

The current status logic depends on the `payment.isPaid` value produced
by the AI response. A production implementation should ideally verify
payment status against a trusted accounting, ERP, payment, or invoice
system rather than relying only on email content.

### 3. Payment-term parsing

The Code node can calculate a due date from a numeric payment term when
no explicit due date exists. The current implementation is best suited
to simple terms such as:

``` text
Net 15
Net 30
Net 45
```

Complex payment terms should be handled with additional parsing rules.

### 4. Notification recipient

The current reminder email node contains a placeholder recipient:

``` text
user@example.com
```

Replace this with the intended billing recipient or a dynamic recipient
before activation.

### 5. AI model availability

The workflow references specific OpenRouter models. Model availability
and naming can change, so verify the configured models before
deployment.

------------------------------------------------------------------------

# Recommended Production Improvements

Potential future enhancements include:

-   Connect to an accounting or ERP system for verified payment status.
-   Process invoice attachments such as PDF invoices.
-   Add OCR/document extraction for scanned invoices.
-   Add vendor-specific payment rules.
-   Add configurable reminder thresholds.
-   Send reminders at configurable intervals.
-   Add escalation levels for invoices overdue by 1, 7, 14, or 30+ days.
-   Add a payment link where supported.
-   Track notification history.
-   Add duplicate invoice detection beyond invoice number.
-   Add currency normalization.
-   Add timezone-aware date handling.
-   Add an approval workflow for high-value invoices.
-   Add dashboards and invoice aging analytics.
-   Add error notifications for failed AI parsing or downstream
    services.

------------------------------------------------------------------------

# Suggested Future Architecture

A more advanced version could evolve into:

``` text
Gmail / Email
      ↓
Invoice Classification
      ↓
Document / Attachment Extraction
      ↓
AI Invoice Extraction
      ↓
Data Validation
      ↓
Invoice Database / ERP
      ↓
Payment Verification
      ↓
Status Calculation
      ↓
Invoice Aging
      ↓
┌─────────────┬─────────────┬─────────────┐
│    Paid     │  Upcoming   │   Overdue   │
└─────────────┴─────────────┴─────────────┘
       ↓              ↓              ↓
 Confirmation     Reminder       Escalation
                                   ↓
                              Slack / Email
```

This would transform the current workflow from a simple invoice reminder
automation into a more complete **Accounts Payable / Invoice Operations
Assistant**.

------------------------------------------------------------------------

# Workflow Status

**Current implementation:** Functional workflow blueprint

### Implemented

-   [x] Gmail invoice intake
-   [x] Invoice/non-invoice classification
-   [x] Gmail invoice labeling
-   [x] AI invoice extraction
-   [x] AI confidence scoring
-   [x] Invoice JSON parsing
-   [x] Customer metadata extraction
-   [x] Due-date calculation
-   [x] Days-until-due calculation
-   [x] Paid status
-   [x] Upcoming status
-   [x] Overdue status
-   [x] Google Sheets invoice logging
-   [x] Status-based routing
-   [x] Paid email notification
-   [x] Upcoming email notification
-   [x] Overdue email notification
-   [x] Overdue Slack alert

------------------------------------------------------------------------

# Repository / Portfolio Description

## Short Description

> An n8n-powered Invoice Reminder Assistant that uses AI to classify and
> extract invoice data, calculates payment deadlines, tracks invoice
> status, logs records to Google Sheets, and automatically notifies
> billing teams about upcoming and overdue payments.

## Key Technologies

-   **n8n**
-   **Gmail**
-   **OpenRouter**
-   **DeepSeek**
-   **Gemini**
-   **JavaScript**
-   **Google Sheets**
-   **Slack**
-   **AI Agents**
-   **Workflow Automation**

## Skills Demonstrated

-   Workflow orchestration
-   Event-driven automation
-   AI-powered information extraction
-   Structured JSON processing
-   JavaScript in n8n Code nodes
-   Date calculations
-   Conditional routing
-   Gmail automation
-   Google Sheets integration
-   Slack notifications
-   Dynamic HTML email generation
-   Business-rule implementation
-   Data normalization
-   Error validation
-   Human-in-the-loop-ready notification design

------------------------------------------------------------------------

# License

This project is intended as a personal automation and portfolio project.
Adapt the workflow, credentials, integrations, and business rules to
your own environment before production use.
