# Kesar & Co. — Automated Support Copilot

**Author:** Raghuveer Singh  
**Source Repository:** [raghu4350/kesar-project](https://github.com/raghu4350/kesar-project.git)  
**Audit Sheet (Read-Only):** [Kesar Support Google Sheet](https://docs.google.com/spreadsheets/d/1dNM176hfNtVme6NZ-Yh7Ctni7wPnjRAEj1ajw87qdao/edit?usp=sharing)

---

## Overview

This project implements an end-to-end customer support automation pipeline built for **Kesar & Co.**, a D2C Ayurvedic skincare brand. Incoming support emails arrive at a webhook endpoint, pass through injection sanitization and structured intent classification, get validated against deterministic privacy and date-window policies, and generate brand-consistent draft replies alongside deduplicated support tickets and an 18-column audit trail in Google Sheets.

- **Stack & Model:** **n8n Cloud** (exported workflow blueprints included), **Google Gemini 1.5 Flash** via Google AI Studio (`temperature: 0.0` for structured JSON extraction, `temperature: 0.2` for reply generation), and deterministic JavaScript code nodes.
- **Audit Log Link:** [Kesar Support Google Sheet](https://docs.google.com/spreadsheets/d/1dNM176hfNtVme6NZ-Yh7Ctni7wPnjRAEj1ajw87qdao/edit?usp=sharing) *(view access enabled)*
- **Build Effort:** ~5.5 hours covering system architecture, prompt hardening, edge-case validation, and documentation.
- **Verified Capabilities:**
  - Schema-enforced JSON classification with zero policy hallucination outside `faq.md`.
  - Prompt-injection detection and neutralization (verified on email `E12`).
  - Sender-to-order email privacy verification (verified on email `E07`).
  - Order-level ticket deduplication across follow-up threads (verified on `E04` and `E21`).
  - Hindi and Hinglish intent recognition plus deterministic return-window math.
  - Automated 6-point CI/CD safety evaluation suite via `scripts/run_evals.py`.
- **Known Boundary:** Underspecified messages lacking any order ID or customer metadata (such as email `E18`) cannot be linked to an order automatically and are flagged for human triage.

---

## System Architecture

### Execution Flow

1. **Ingest (`POST /support-email`)** — Webhook receives an email JSON payload.
2. **Sanitize & Guard** — Code node normalizes fields and scans for jailbreak phrases (`SYSTEM NOTE`, `ignore previous instructions`).
3. **Context Retrieval (Parallel)** — Fetches current records from `orders.json` and policy rules from `faq.md`.
4. **Structured Classification** — Gemini 1.5 Flash (`temperature: 0.0`) maps the message to `schemas/llm_output.schema.json`.
5. **Deterministic Business Rules** — Code node checks sender ownership, resolves order records, and computes elapsed days against reference date `2026-11-20`.
6. **Intent Switch Routing** — Dispatches across five dedicated branches:
   - `order_status` → Composes tracking or delay updates in brand voice.
   - `product_query` → Answers strictly from FAQ content; escalates medical claims.
   - `return_refund` → Checks the 7-day unopened eligibility window → creates/updates ticket + drafts response.
   - `complaint` → Checks the 48-hour damaged/leaking/wrong-item window → opens high-priority ticket + drafts response.
   - `other` → Separates spam, collaboration pitches, and GST inquiries → logs or escalates.
7. **Ticket Deduplication** — Queries existing open tickets by `order_id`; updates an open record if present, otherwise creates a new ticket.
8. **Audit & Response** — Formats an 18-column log entry, appends it to Google Sheets (`Audit Log`), and returns an `HTTP 200` JSON confirmation.

```
[POST /support-email]
        │
        ▼
[Normalize & Guardrails] ──► (Parallel Fetch: orders.json + faq.md)
        │
        ▼
[Gemini Intent Classifier (schemas/llm_output.schema.json)]
        │
        ▼
[Order Lookup & Deterministic Rules (Privacy + Date Math vs 2026-11-20)]
        │
        ▼
[Intent Switch]
   ├── order_status   ──► Tracking / Delay Draft
   ├── product_query  ──► FAQ-Grounded Draft (Medical -> Escalate)
   ├── return_refund  ──► 7-Day Unopened Check ──► Ticket + Draft
   ├── complaint      ──► 48h Damage Check     ──► High-Priority Ticket + Draft
   └── other          ──► Filter Spam / Collab / GST ──► Log or Escalate
        │
        ▼
[Ticket Deduplication (Match open ticket by order_id)]
        │
        ▼
[18-Col Audit Formatter] ──► [Google Sheets Append] + [HTTP 200 JSON Reply]
```

### Component Breakdown

| Stage | Responsibility | Design Rationale |
|---|---|---|
| **Webhook Trigger** | Accepts individual email payloads from `send_test_emails.py`. | Mirrors real-time production inbox webhooks. |
| **Normalize & Guardrails** | Cleans input text and catches prompt injection markers (`SYSTEM NOTE`, `ignore previous instructions`). | Isolates untrusted user content before any prompt reaches the LLM. |
| **Order & FAQ Fetch** | Loads `orders.json` and `faq.md` dynamically per run. | Separates knowledge/data state from workflow logic like read-only APIs. |
| **Classify & Extract Intent** | Outputs strict JSON validated against `llm_output.schema.json`. | Zero-temperature structured output keeps downstream branching predictable. |
| **Order Lookup & Rules** | Matches by `order_id` or email, enforces privacy, computes `days_since_delivery` vs `2026-11-20`. | Keeps arithmetic and authorization out of probabilistic LLM reasoning. |
| **Intent Router** | Splits execution into 5 isolated domain paths. | Lets policy checks, priority flags, and templates evolve independently. |
| **Reply Drafter** | Writes 3–6 sentence warm, concise replies signed off as Team Kesar. | Enforces `brand_voice.md` tone and mandatory `faq.md` medical disclaimers. |
| **Ticket Deduplication** | Updates matching open `order_id` tickets rather than spawning duplicates. | Keeps multi-email customer issues consolidated in a single thread. |
| **Audit Logger** | Writes complete execution metadata to Google Sheets. | Gives operations teams full traceability and a human-review queue. |

---

## Local Setup & Reproduction

### Prerequisites
- An [n8n Cloud](https://n8n.io) workspace (or self-hosted Docker instance on n8n v1.x+).
- A Google account authorized to read/write Google Sheets.
- A Google Gemini API key from [Google AI Studio](https://aistudio.google.com).

### Importing & Configuring the Workflow
1. In n8n, open **Workflows → Add Workflow → ⋯ (top-right menu) → Import from File**.
2. Load either:
   - `workflow/kesar_copilot_workflow.json` *(standard deterministic pipeline)*, or
   - `workflow/kesar_copilot_ai_agent.json` *(agentic variant equipped with LangChain tools `get_order` and `search_faq`)*.
3. Open the **Google Sheets (Audit Log)** node, connect **Google Sheets OAuth2 API** via Google sign-in, and set `Document` to your target sheet URL.
4. Set `GEMINI_API_KEY` in the workflow environment or node credentials.

### Running Test Emails & Safety Evals

```bash
# Smoke-test a small subset first
python scripts/send_test_emails.py --url <YOUR_N8N_WEBHOOK_URL> --only E01,E02,E03

# Replay the full 25-email test suite (5s pacing between requests)
python scripts/send_test_emails.py --url <YOUR_N8N_WEBHOOK_URL>

# Execute the automated CI/CD safety and policy evaluation checks
python scripts/run_evals.py
```

---

## Prompt Engineering

Two separated, versioned prompts drive the language model stages:

1. **Intent Classification & Entity Extraction** — [prompts/classification_prompt.md](prompts/classification_prompt.md)  
   Runs on Gemini 1.5 Flash at `temperature: 0.0`, returning raw JSON conforming strictly to `schemas/llm_output.schema.json`.
2. **Customer Reply Generation** — [prompts/agent_reply_prompt.md](prompts/agent_reply_prompt.md)  
   Runs on Gemini 1.5 Flash at `temperature: 0.2` under the Kesar & Co. Customer Specialist persona. Constrained by `brand_voice.md` to 3–6 sentences, zero emojis, zero filler, warm honesty, and language matching the customer.

---

## Core Design Decisions & Guardrails

- **Low-Confidence Escalation:** Whenever classification confidence drops below `0.70` or intent cannot be resolved, `needs_human` is set to `TRUE` in the audit sheet and automated reply drafting is suppressed.
- **Follow-Up Ticket Consolidation:** Before opening a ticket for a `return_refund` or `complaint`, the workflow searches existing open tickets by `order_id`. Matches trigger an `UPDATE` action that appends the new email ID (e.g., `E04, E21`) and summarizes the follow-up inside `notes`.
- **Strict PII & Email Verification:** Inside `order_lookup_and_rules.js`, the incoming `from.email` is checked against `order.customer_email`. Any mismatch (such as `E07`) raises `privacy_violation`, blocks order data exposure, and drafts a neutral privacy notice.
- **Single Source of Truth (`data/faq.md`):** Prompts prohibit inventing shipping regions (such as international delivery outside India), product attributes, or medical cure claims; anything absent from `data/faq.md` is escalated to a human agent.
- **Deterministic Policy Windows:** Using `2026-11-20` as the scenario reference date, both the 7-day return window and 48-hour damage reporting window are calculated in JavaScript via `Math.floor(diffMs / 86400000)` to eliminate LLM calendar errors.

---

## Scaling Strategy (1,000 Emails / Day)

At 1,000 emails/day (~42/hour average, ~150/hour peak):

### Primary Bottlenecks
- **API Throttling:** Free-tier limits (Gemini at 15 RPM; Google Sheets at 60 requests/minute per user) will reject burst traffic during peak hours.
- **Spreadsheet Concurrency Limits:** Using Google Sheets as transactional storage leads to row-lock contention, deduplication race conditions, and high write latency.

### Production Upgrade Path
- **Decoupled Ingestion Queue:** Place RabbitMQ, Redis Streams, or AWS SQS between the webhook gateway and worker consumers to smooth traffic spikes with controlled concurrency and exponential backoff retries.
- **Relational Storage (PostgreSQL):** Move tickets and audit logs out of Google Sheets into PostgreSQL indexed on `order_id` and `customer_email`, using ACID transactions for race-free ticket deduplication.
- **Semantic Caching & Tiered Routing:** Cache frequent FAQ answers in Redis and handle verified order-status lookups with direct SQL queries or smaller models, reserving full LLM reasoning for nuanced complaints.

---

## Product Thinking *(≤150 words each)*

### 1. Auto-Send vs. Draft Mode
> Auto-send fits strictly deterministic, low-risk interactions: verified `order_status` lookups where sender email matches the order record, and straightforward `product_query` questions answered directly from `faq.md` with confidence `≥ 0.90`. Every message involving money (refunds, return approvals), damaged or leaking shipments, policy exceptions, `angry` sentiment, or confidence `< 0.90` must stay in draft mode. In skincare, customer trust is unforgiving—an incorrect refund denial or fabricated product promise causes lasting brand harm, while a brief human check prevents costly mistakes.

### 2. Cost Estimate at 1,000 Emails / Day
> - **Token Footprint:** ~400 input + ~100 output tokens for classification; ~1,000 input + ~250 output tokens for FAQ-grounded drafting (**~1,400 input / ~350 output per email**).
> - **Gemini 1.5 Flash Rates:** \$0.075 per 1M input tokens; \$0.30 per 1M output tokens.
> - **Per-Email Cost:**
>   - Input: $1,400 \times \$0.000000075 = \$0.000105$
>   - Output: $350 \times \$0.00000030 = \$0.000105$
>   - Total: **\$0.00021 / email** (~₹0.0175 INR)
> - **Monthly Run Rate (30,000 emails):** $30,000 \times \$0.00021 = \mathbf{\$6.30\text{/month}}$ (~₹525 INR/month).
> - Even doubling capacity for retries and tool-call overhead keeps monthly LLM spend under **\$13/month**.

### 3. Success Metric & Kill Switch
> - **Week 1 North Star:** **Draft Acceptance Rate without Edits (DAR)**—the share of AI-generated drafts agents send untouched or with minimal tweaks. Hitting `≥ 80%` confirms the system cuts handling time instead of shifting effort into rewrites.
> - **Kill Switch Trigger:** Immediate shutdown of automated routing upon any **Zero-Tolerance Safety Violation**: leaking customer PII to an unverified address, authorizing an unvalidated refund, or generating an ungrounded medical claim stating a product cures a skin condition.

### 4. Failure Experience & Damage Control
> Misclassifications or awkward phrasing make customers feel unheard or brushed off by automation. Three safeguards limit fallout:
> 1. Launch in draft-only mode so human agents catch mistakes before delivery.
> 2. Enforce conservative phrasing in any auto-replies—stick strictly to verified facts and always close with a human escape hatch: *"If this doesn't fully answer your question, simply reply to this email and our team will personally assist you."*
> 3. Route any follow-up reply received within 4 hours straight to senior human support, bypassing automation.

### 5. What Must Stay Human-Only (and Why)
> **Adverse skin reactions and medical escalations must remain 100% manual.** When someone reports burning, rashes, or an allergic reaction to a serum, automated templates feel dismissive and expose the business to serious risk. Physical reaction cases demand genuine human care, immediate escalation to product quality teams for warehouse batch tracing, and coordination with dermatologists to protect customer safety and meet consumer-protection obligations.

---

## Use of AI Tools in Development

- **AI-Assisted Acceleration:** Leveraged Claude and Gemini to refine edge-case definitions in `llm_output.schema.json`, probe regex filters with prompt-injection test cases, and scaffold initial JavaScript helpers for date arithmetic and ticket deduplication.
- **Manual Engineering Decisions:**
  - Replaced prompt-only rules with hard-coded JavaScript enforcement for email privacy (`from_email !== customer_email`) and policy day calculations.
  - Architected the two-stage lookup/update deduplication flow to keep Google Sheets ticket history unified.
  - Tuned the `0.70` / `0.90` confidence thresholds and defined the zero-tolerance kill-switch criteria.

---

## Roadmap (With One Additional Week)

1. **LangChain ReAct Tooling:** Evolve static HTTP fetches into dynamic agent tool bindings with memory across prior customer conversations.
2. **Vector RAG Catalog Search:** Index the complete product catalog in Qdrant or Pinecone so responses cite exact ingredient lists, certifications, and usage steps.
3. **Omnichannel Support Intake:** Add webhook adapters for WhatsApp Business API and Shopify Inbox to unify threads per customer.
4. **Real-Time Slack / Zendesk Escalations:** Push `sentiment: angry` and damaged-order alerts into `#support-escalations` with one-click agent approval actions.
