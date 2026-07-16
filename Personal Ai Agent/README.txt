====================================================
MY PERSONAL AI AGENT
An n8n-based multi-agent AI swarm with Telegram interface
====================================================

OVERVIEW
--------
This is a personal AI assistant built as a multi-agent "swarm" in n8n.
A single Orchestrator agent receives messages via Telegram and delegates
tasks to specialized sub-agents, each exposed to it as a callable tool.

The system can: send/manage email, search the web, read webpages, manage
a Google Calendar, save/recall personal notes, and manage Google Drive
files — all through natural conversation in a Telegram chat.


ARCHITECTURE
------------
Telegram Trigger -> Get Message -> Orchestrator -> Telegram Response

The Orchestrator has:
  - A chat model (brain)
  - A Memory node (conversation context, keyed by Telegram chat ID)
  - A Think tool (mandatory reasoning step before delegating)
  - 6 sub-agents wired in as Tools:
      1. Email Agent
      2. Research Agent
      3. Scraper Agent
      4. Calendar Agent
      5. Notes Agent
      6. Drive Agent

Each sub-agent is itself a mini AI Agent node with its own brain, its
own system prompt, and its own set of tools (see AGENTS.md for full
detail on each one).


SUB-AGENTS AT A GLANCE
-----------------------
Email Agent      -> Send, Get, Get Many, Reply, Delete, Label email
                     (Gmail) + Contacts lookup (Google Sheet)

Research Agent   -> Google Search, Google Maps, Google News (via
                     SerpAPI) + YouTube video search (native node)

Scraper Agent    -> Scrape Webpage (Jina AI Reader, via HTTP Request)

Calendar Agent   -> Create, Get, Get Many, Update, Delete events
                     (Google Calendar)

Notes Agent      -> Save, Search, Update, Delete personal notes
                     (Google Sheets)

Drive Agent      -> Search Files/Folders, Download, Update, Move,
                     Share, Delete (Google Drive)


REQUIREMENTS
------------
- n8n instance (self-hosted or cloud), workflow set to Active
- Telegram Bot (created via BotFather) + Telegram credential in n8n
- Google account with the following APIs enabled:
    - Gmail API
    - Google Calendar API
    - Google Sheets API
    - Google Drive API
- SerpAPI account/key (Search, Maps, News)
- YouTube Data API v3 key (Google Cloud Console — free tier)
- Jina AI Reader (no key required for basic use)
- A chat-capable LLM credential (OpenAI, Anthropic, etc.) for each
  agent's "brain" node


SETUP NOTES
-----------
1. All Google-based tools (Gmail, Calendar, Sheets, Drive) can share
   the same OAuth credential.

2. Memory persistence: the Orchestrator's Memory node Session ID must
   be set to the Telegram chat ID (not left default), or the agent
   will lose conversation context between messages:
       {{ $('Telegram Trigger').item.json.message.chat.id }}

3. The workflow must be set to ACTIVE (not just "Listen for test
   event") for persistent, real-world conversation behavior via the
   live Telegram bot.

4. Dates should be generated via n8n expressions, not left for the AI
   to infer — e.g. {{ $now.format('yyyy-MM-dd') }} — otherwise the
   model may hallucinate an incorrect date (e.g. from training data).

5. Any Google Sheets "match/key column" or Google Drive "Fields"
   parameter should be set manually (not AI-controlled) — these are
   structural parameters, not conversational ones, and letting the
   AI set them can cause tool errors and iteration-limit failures.


KNOWN ISSUES FIXED DURING DEVELOPMENT
--------------------------------------
- Untitled calendar events -> caused by the "Summary" field not being
  present in the Create Event node by default; had to be manually
  added via "Additional Fields."
- YouTube tool 400 errors -> caused by scraping Google Search HTML
  directly instead of using a real API; fixed by switching to the
  native YouTube node + YouTube Data API v3.
- Notes Agent "Could not find column for key 'id'" -> caused by a
  Sheets column mapping mismatch; fixed by manually mapping to the
  sheet's actual header names.
- Drive Agent "fields.join is not a function" -> caused by the
  Search node's "Fields" parameter being left AI-controlled (model
  passed a string, node expected an array); fixed by setting Fields
  manually to [All].
- Max iteration errors -> in both cases above, were a downstream
  symptom of a tool call failing repeatedly; resolved once the
  underlying tool config was fixed.


FULL DOCUMENTATION
-------------------
See AGENTS.md in this repo for the complete system prompt, tool
descriptions, and routing rules for the Orchestrator and every
sub-agent.


LICENSE / USAGE
----------------
Personal project — shared for reference and portfolio purposes.
Feel free to fork and adapt for your own use.
