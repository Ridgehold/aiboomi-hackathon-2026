# Ridgehold

**The AI account-growth OS for B2B AI and SaaS**

Built at AIBoomi Startup Weekend · June 2026 · by Yash Kothari & Srinivasan R
# AccMan.ai

**An agentic AI copilot for B2B account managers who own churn prevention, renewals, and expansion.**

AccMan.ai is a Claude/ChatGPT-style chat interface purpose-built for renewal and
account managers. Its north star is **Net Revenue Retention (NRR)**: it unifies
everything known about a customer account, proactively surfaces what's at risk
(or ripe for expansion), explains *why it matters*, recommends concrete next
steps — and, unlike a passive chatbot, **takes action** on the account manager's
behalf through a bounded set of tools.

---

## The Problem

A B2B account manager (AM) owns a portfolio of accounts and is measured on
retention and growth. The data they need to do that job is scattered across
six+ systems:

- **CRM** (Salesforce/HubSpot) — stakeholders, ownership, deal history
- **Ticketing** (Zendesk/Jira/Intercom) — support load and sentiment
- **Product** — usage, adoption, seat utilization
- **Contracts** — buried PDFs with the renewal date and commercial terms
- **Notes** — human, email, Slack, and CRM threads
- **The outside world** — news, SEC filings, social, reviews

The result: AMs spend most of their time *gathering context* instead of acting,
churn signals get noticed too late, and the work (drafting the outreach, logging
the follow-up, prepping the renewal brief) still has to be done by hand.

**AccMan.ai collapses that into one copilot.** It ingests the data, runs LLM
analysis over it, raises an actionable inbox notification when something changes,
and opens a chat where an agent briefs the AM and does the busywork.

---

## What It Does

| Capability | How |
|------------|-----|
| **Unified account intelligence** | An LLM digests each account's firmographics, stakeholders, notes, usage, tickets, contracts, and signals into a **health score (0–100)**, **churn-risk** rating, summary, and prioritized **recommended actions**. |
| **Proactive signal triage** | External signals (news, SEC filings, social, web, reviews) are fetched and interpreted by the LLM into an **inbox notification** — "why this matters" for *this* account, with a severity and category. No fixed scenario catalog; brand-new signal types need no code change. |
| **Contract understanding** | Upload a contract PDF → PyMuPDF extracts the text → the LLM extracts structured commercial/legal terms (type, dates, value, renewal) into JSONB. |
| **Agentic chat** | Click a notification (or start a chat) and an agent grounds itself in the account context, investigates with read tools, renders rich UI widgets, and executes internal write actions — then tells the AM what it did. |
| **Custom skills** | Admins author markdown playbooks (e.g. "renewal-playbook"). The agent sees a catalog of them and loads a skill's full instructions on demand — Claude-style progressive disclosure. |

---

## Tech Approach

### AI Model

| | |
|---|---|
| **Primary model** | **OpenAI GPT-5.5** (deep-reasoning default for chat streaming) |
| **Lightweight model** | **GPT-5.5-mini** for cheaper, lower-stakes calls |
| **Interface** | OpenAI function-calling (tool use) + streaming completions |
| **Prompt management** | **Langfuse** — prompts live in Langfuse (versioned), not hardcoded; no local fallbacks in production |
| **Observability** | Langfuse tracing via OpenTelemetry auto-instrumentation of the OpenAI SDK (token usage, latency, traces per tenant/account) |

### The five LLM pipelines

The product is not one prompt — it's a set of specialized LLM jobs, each with its
own system prompt (under [`shared/prompts/templates/`](shared/prompts/templates)):

1. **Account intelligence** — takes a full JSON digest of one account and returns
   health, churn risk, summary, drivers, and signals.
2. **Signal enrichment** — summarizes a single external signal and scores its
   sentiment + relevance to renewal/churn/expansion.
3. **Signal triage** — turns a raw signal + account snapshot into an actionable
   inbox notification (title, "why it matters", category, severity). Reasons from
   evidence rather than a fixed signal taxonomy.
4. **Contract extraction** — turns raw PDF text into structured contract terms.
5. **Chat copilot** — the agentic system prompt that drives the conversation.

### The agentic loop (the centerpiece)

Chat is **not** a single-shot completion. It runs a bounded **agent loop** in the
FastAPI data plane ([`chatapi/app/services/llm.py`](chatapi/app/services/llm.py)):
the model may call tools, the server executes them and feeds results back, and the
model continues until it produces a final turn — this is what lets the agent
*act*, not just suggest. Three classes of tool are advertised to the model
([`chatapi/app/services/tools.py`](chatapi/app/services/tools.py)):

- **Read** — `get_account_details`, `list_accounts`, `get_usage_history`,
  `get_stakeholders`, `get_signals`, `get_contracts`. Executed server-side; the
  result is fed back to the model so it can investigate on demand.
- **Render** — `render_account_card`, `render_recommended_actions`,
  `render_table`, `render_chart`, `render_email_draft`, `render_brief`,
  `render_tracking`. Short-circuited and forwarded to the client as rich UI
  widgets streamed over SSE.
- **Write** — `create_followup`, `complete_followup`, `save_draft`,
  `schedule_meeting`, `mark_notification_actioned`. These mutate tenant state —
  the agent takes real internal actions on the AM's behalf.

> **Safety by design:** there is deliberately **no tool that sends outbound
> communications.** The agent can only *draft* emails for the AM to review and
> send themselves. Internal records (follow-ups, drafts, meetings, notification
> status) are reversible, so the agent executes those on its own initiative.

### Data feeding the AI

The LLM context is built per session from tenant-scoped data
([`chatapi/app/services/context.py`](chatapi/app/services/context.py)) — a
portfolio-level summary cached on the session, plus on-demand account detail.
The underlying domain (Django models):

| Domain | Models |
|--------|--------|
| **Accounts** | `Account`, `Stakeholder`, `AccountNote` (human/email/Slack/CRM), `ProductUsage`, `SupportTicket` |
| **Contracts** | `Contract` (PDF + PyMuPDF text + LLM-extracted terms) |
| **Intelligence** | `AccountIntelligence` (health/churn/summary), `RecommendedAction` |
| **Signals** | `Signal` (news/SEC/social/web/review), `Notification` (AM inbox) |
| **Conversations** | `ChatSession`, `ChatMessage`, `ChatAttachment` |
| **Integrations** | `Integration` (Salesforce/HubSpot/Zendesk/Jira/Intercom), `SyncRun` |
| **Skills** | `Skill` (admin-authored agent playbooks) |

Flexible LLM outputs land in **JSONB** columns; all primary keys are **UUID7**
(time-ordered) for better index locality.

### Async regeneration pattern

Every LLM-generated artifact (account intelligence, signal enrichment, contract
extraction) carries a `pending | generating | completed | failed` **status** and
is produced by a **Procrastinate** worker task
(`generate_account_intelligence_task`, `enrich_signal_task`,
`extract_contract_terms_task`, `run_integration_sync_task`). The previous content
**stays visible** while a new generation runs; the frontend polls with backoff
and shows a "Regenerating…" badge — never a blank skeleton.

---

## Architecture

A **dual-process backend** sharing a single PostgreSQL database. Django owns the
schema and migrations; FastAPI is the async data plane for chat.

```
                         ┌──────────────────────────┐
                         │   React 19 SPA (Vite)     │
                         │   Chakra UI v3 · SSE      │
                         └────────────┬─────────────┘
              REST (/api/v1)          │        SSE chat stream
        ┌───────────────────┐        │        ┌──────────────────────┐
        │  Django + DRF     │◄───────┴───────►│  FastAPI (async)     │
        │  Management plane │                 │  Data plane          │
        │  auth · admin ·   │                 │  agent loop · tools ·│
        │  uploads · CRUD · │                 │  SSE streaming       │
        │  owns migrations  │                 │  (OpenAI GPT-5.5)    │
        └─────────┬─────────┘                 └──────────┬───────────┘
                  │                                      │
        ┌─────────▼──────────┐              ┌────────────▼───────────┐
        │ Procrastinate      │              │  PostgreSQL 18         │
        │ worker (LLM jobs)  │─────────────►│  schema-per-tenant     │
        │ same image         │              │  + task queue          │
        └────────────────────┘              └────────────────────────┘
```

- **Django** = management plane: JWT auth (15-min access tokens, silent refresh),
  admin, file uploads, CRUD, schema ownership.
- **FastAPI** = data plane: SSE chat streaming, LLM orchestration, the agent loop.
- **Procrastinate** = PostgreSQL-native task queue (no separate broker) for the
  async LLM regeneration jobs.
- **Multi-tenant**: schema-per-tenant Postgres; every query is tenant-scoped via
  `SET LOCAL search_path`.
- **Response envelope** `{ data, error, meta }` on every API endpoint.

### Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 19 · Vite · Chakra UI v3 · TypeScript |
| Backend (management) | Django 6 + DRF |
| Backend (streaming) | FastAPI (async) |
| Database | PostgreSQL 18 (Django owns migrations) |
| Task queue | Procrastinate (PostgreSQL-native) |
| LLM | OpenAI GPT-5.5 / GPT-5.5-mini |
| Prompts + tracing | Langfuse |
| PDF parsing | PyMuPDF (`fitz`) |
| Export | python-docx · WeasyPrint |
| Deployment | Docker / Docker Compose (local) · Railway (prod) |


## AI Usage

GPT-5.5 powers the full Ridgehold stack — conversational interface, skill and agent execution, reasoning across account context, and artifact generation.

All AI outputs are grounded in account records. Confidence labels accompany every recommendation. Signal rules are explainable and auditable. Nothing is auto-sent — human approval is required at every customer-facing step.

---

## Product Demo

A screen recording of the live prototype is included as part of the submission, demonstrating the full loop:

> Signal Detected → Account Context → Recommended Actions → Draft Artifact → Tracked to Closure

The demo uses the Champion Exit scenario (Maverick Retail Group) as the primary flow, with Usage Spike and Legacy Pricing Gap as supporting signals.

### Submission Files

| File | Description |
|---|---|
| `ridgehold_demo_v2.gif` | Demo GIF showing the full product loop |
| `Ridgehold_Pitch_Deck.pptx` | Pitch deck presented at AIBoomi Startup Weekend |
| `Ridgehold_AI_Impact_Statement.docx` | AI Impact Statement covering models, data provenance, guardrails, and expected outcomes |

---

## Team

| Name | Role |
|---|---|
| Yash Kothari | Product |
| Srinivasan R | Engineering |

---

*AIBoomi Startup Weekend · June 2026*
