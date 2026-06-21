# Ridgehold

**The AI account-growth operator for B2B AI and SaaS**

Built at AIBoomi Startup Weekend · June 2026 · by Yash Kothari & Srinivasan R

---

## What It Does

Ridgehold helps account managers (AMs) identify expansion opportunities, take timely action, and improve Net Revenue Retention (NRR) — without switching between five different tools.

It detects account signals (champion exits, usage spikes, pricing gaps, new hires, geographic expansion), reasons over account context, recommends the next best action, drafts the required artifact (email, executive brief, meeting agenda), and tracks progress to closure.

The result: AM teams move from reactive account management to proactive revenue motion.

---

## The Problem It Solves

Expansion revenue in B2B SaaS is no longer just relationship management — it's signal detection. But the signals that matter most are invisible until it's too late:

- A key champion leaves the company
- A customer raises a funding round and is ready to scale
- Usage drops quietly before a renewal conversation
- A new executive joins with no relationship to the AM

By the time AMs notice, the window has closed. Expansion turns into churn defence or lost opportunity.

Existing tools — CRM, CS platforms — store account data but don't drive action. Ridgehold is the AI action layer that sits on top, connecting signal to decision to execution.

---

## How It Works

Ridgehold runs a six-layer agent architecture:

| Layer | What It Does |
|---|---|
| Signal Detection | Detects stakeholder changes, usage spikes, entitlement gaps, renewal timelines |
| Notification Generation | Composes in-app alerts, routes to the right AM/CSM |
| Account Context Summarization | Summarizes health score, ARR, stakeholder map, renewal readiness |
| Action Recommendation | Ranks next-best actions grounded in account facts |
| Draft Output Generation | Generates emails, executive briefs, meeting agendas |
| State Management | Tracks follow-up, captures outcomes, closes the loop |

Every response is grounded in account records. Human approval is required before any customer-facing action is taken.

---

## Technology Stack

| Component | Technology |
|---|---|
| Frontend / UI | React, Chakra UI |
| Backend | Django, FastAPI |
| AI / LLM | GPT-5.5 (OpenAI) |
| Prototype Environment | Lovable |
| Data | Synthetic account data (demo) |

**Open Source Licenses:** Django (BSD 3-Clause) · FastAPI (MIT) · React (MIT) · Chakra UI (MIT)

---

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
