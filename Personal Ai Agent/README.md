# Multi-Agent AI Personal Assistant

A modular multi-agent AI assistant built with **n8n** that uses an orchestrator architecture to intelligently route user requests to specialized AI agents. Each agent is responsible for a specific domain—Email, Research, Web Scraping, Calendar Management, Note Taking, and Google Drive Operations—allowing the assistant to perform complex productivity tasks while maintaining a clear separation of responsibilities.

The workflow demonstrates how multiple AI agents can collaborate under a central orchestrator, enabling scalable, maintainable, and enterprise-ready AI automation.

---

## Quick Facts

| Item | Value |
|------|------|
| **Workflow Type** | Multi-Agent AI Automation |
| **Primary Purpose** | AI Productivity Assistant |
| **Architecture** | Orchestrator + Specialized Agents |
| **Trigger** | Telegram Bot |
| **AI Pattern** | Multi-Agent System |
| **Memory** | Persistent AI Memory |
| **Reasoning** | Think Tool |
| **Programming** | n8n AI Agents |
| **Integrations** | Gmail, Google Calendar, Google Drive, Google Sheets, Google Search, Google Maps, Google News, YouTube |
| **Communication Platform** | Telegram |
| **Development Mode** | Fully Modular Multi-Agent Architecture |

---

# 1. Overview

Modern AI assistants often become difficult to maintain because every capability is placed inside a single AI agent. As the number of available tools grows, the model becomes increasingly complex, slower to reason, and harder to scale.

This workflow solves that problem by implementing a **Multi-Agent Architecture**.

Instead of giving one AI access to every available tool, a central **Orchestrator Agent** receives user requests from Telegram, understands the user's intent, and delegates the task to the most appropriate specialized agent.

Each specialized agent has a dedicated responsibility:

- Email Management
- Research
- Web Scraping
- Calendar Management
- Note Management
- Google Drive Management

Because every agent focuses on only one domain, prompts remain smaller, reasoning becomes more accurate, and the system can easily grow by adding additional agents without redesigning the entire workflow.

The orchestrator also maintains conversational memory and reasoning capabilities, allowing it to determine which agent should execute a task before returning a final response back to the user via Telegram.

```
Telegram
    │
    ▼
Receive Message
    │
    ▼
Preprocess Request
    │
    ▼
        Orchestrator
             │
     ┌───────┼────────┬────────┬────────┬─────────┐
     ▼       ▼        ▼        ▼        ▼         ▼
 Email   Research  Scraper  Calendar  Notes   Drive
 Agent     Agent     Agent     Agent    Agent    Agent
     │       │        │        │        │         │
     └─────────────── Results ─────────────────────┘
                     │
                     ▼
              Telegram Response
```

---

# 2. What Problem Does It Solve?

Traditional AI assistants often rely on a single model that has access to every available tool. As more capabilities are added, prompt complexity increases, response quality decreases, and maintaining the system becomes increasingly difficult.

This workflow separates responsibilities into specialized AI agents coordinated by a central orchestrator.

Instead of one agent deciding how to perform every task, each specialist focuses on its own expertise while the orchestrator determines which agent should handle the user's request.

This architecture provides better scalability, easier maintenance, improved reasoning accuracy, and allows new capabilities to be added without affecting existing agents.

---

# 3. Benefits

- Modular AI architecture
- Easier maintenance and scalability
- Faster reasoning through specialized prompts
- Better task delegation
- Reduced prompt complexity
- Persistent conversational memory
- Centralized orchestration
- Easy integration of new AI agents
- Enterprise-ready architecture
- Clean separation of responsibilities

---

# 4. Who Will Use It?

This workflow is ideal for users who need a centralized AI assistant capable of managing multiple productivity tasks.

Typical users include:

- Knowledge Workers
- Software Engineers
- Project Managers
- Researchers
- Business Owners
- Executive Assistants
- Students
- Operations Teams
- Productivity Enthusiasts

---

# 5. Workflow Breakdown

## `TG Message` — Telegram Trigger

Serves as the entry point of the workflow.

Whenever a user sends a message to the Telegram bot, the workflow is automatically triggered and the message is forwarded into the orchestration pipeline.

---

## `Get Message`

Extracts and normalizes the Telegram payload.

This node prepares the incoming message so the orchestrator receives a consistent request structure regardless of Telegram's raw webhook format.

---

# Central AI Layer

## `Orchestrator`

The Orchestrator is the brain of the entire workflow.

Its responsibilities include:

- Understanding user intent
- Selecting the appropriate specialist
- Maintaining conversation flow
- Delegating tasks
- Combining tool outputs
- Returning the final response

Unlike traditional AI assistants that directly execute every task, the orchestrator functions as an intelligent dispatcher responsible for coordinating all available agents.

---

## `Orchestrator Brain`

Provides the Large Language Model responsible for the orchestrator's reasoning and decision-making.

The model determines which specialized agent should receive the user's request.

---

## `Memory`

Stores conversational context between interactions.

This enables follow-up questions without requiring the user to repeat previous information.

Example:

> User:
> "Show tomorrow's meetings."

Later:

> "Move the first one to Friday."

The assistant remembers which meeting is being referenced.

---

## `Think`

Provides additional reasoning capabilities before delegation.

Rather than immediately selecting an agent, the orchestrator can analyze complex requests, decompose multiple tasks, and determine the optimal execution strategy.

---

# Email Agent

Responsible for all Gmail-related operations.

Its available tools include:

- Send Email
- Get Email
- Search Email
- Reply to Email
- Delete Email
- Label Email
- Read Contacts

The Email Agent isolates all email functionality from the rest of the assistant, keeping prompts focused and reducing unnecessary tool selection.

---

# Research Agent

Handles internet research and information gathering.

Available tools include:

- Google Search
- Google Maps
- Google News
- YouTube Search

This agent specializes in collecting publicly available information rather than performing productivity tasks.

---

# Scraper Agent

Responsible for extracting content from websites.

Its primary tool performs webpage scraping, allowing the assistant to summarize articles, retrieve structured information, or collect webpage content for further analysis.

Keeping scraping isolated prevents unrelated agents from handling web extraction tasks.

---

# Calendar Agent

Handles scheduling and calendar management.

Supported operations include:

- Create Event
- Update Event
- Delete Event
- Get Event
- Search Events

This agent acts as the user's scheduling assistant while remaining completely independent from email and note management.

---

# Notes Agent

Responsible for personal note management using Google Sheets.

Available capabilities include:

- Save Note
- Search Note
- Update Note
- Delete Note

Using Google Sheets provides a lightweight cloud-based knowledge repository that can easily be expanded or replaced by dedicated databases.

---

# Drive Agent

Handles Google Drive file management.

Supported operations include:

- Search Files
- Download Files
- Upload Files
- Move Files
- Share Files
- Delete Files

This allows the assistant to function as a lightweight document management system capable of organizing and retrieving files on behalf of the user.

---

## `Response`

After the selected agent completes its task, the orchestrator generates a natural language response and sends the final answer back to the user through Telegram.

This ensures users interact with a single AI assistant despite multiple specialized agents working behind the scenes.

---

# Setup Requirements

| Service | Used For | Credential Needed |
|----------|----------|------------------|
| Telegram Bot API | User Interface | Telegram Bot Token |
| Google Gmail API | Email Operations | OAuth2 |
| Google Calendar API | Calendar Management | OAuth2 |
| Google Drive API | File Management | OAuth2 |
| Google Sheets API | Notes Database | OAuth2 |
| Google Search API | Internet Search | API Key |
| Google Maps API | Location Search | API Key |
| Google News API | News Retrieval | API Key |
| YouTube Data API | Video Search | API Key |
| LLM Provider | Orchestration & Agent Reasoning | API Key |

---

# Known Limitations / Next Steps

- The current architecture routes requests to a single specialist at a time. Future versions could support parallel multi-agent collaboration for complex tasks.
- Long-term memory is currently limited and could be enhanced using a vector database such as Pinecone, Qdrant, or Chroma.
- Additional security layers should be added before allowing destructive operations such as deleting files or emails.
- Agent execution history is not currently logged for auditing and debugging purposes.
- Human approval checkpoints could be introduced for sensitive operations like sending emails or deleting documents.

---

# Future Improvements

- Add Finance Agent
- Add Coding Agent
- Add Database Agent
- Add Travel Planning Agent
- Add HR Agent
- Multi-Agent Collaboration
- Vector Memory Integration
- Voice Assistant Support
- Image Understanding Agent
- Workflow Monitoring Dashboard
- Automatic Agent Performance Metrics
- Self-improving Agent Selection
