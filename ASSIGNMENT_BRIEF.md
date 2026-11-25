# Take-Home Project Specification — AI Automation Intern
## Kesar & Co. Order-Support Copilot

Welcome to the **AI Automation Intern** take-home challenge. This exercise mirrors day-to-day automation engineering work: connecting an LLM to external tools, enforcing strict safety guardrails, and packaging everything cleanly enough for another engineer to reproduce.

Reliability, clear reasoning, and transparency about edge-case limits matter far more than surface complexity.

---

## Quick Facts

| Dimension | Expectation |
|---|---|
| **Effort Budget** | Roughly **4–6 hours** of focused work within a 4-day window (reserve 30–60 minutes initially for account setup). Cap total effort at 8 hours and submit your progress if you hit that mark. |
| **Submission Window** | **4 days** from receiving the invite (exact cutoff listed in your email). |
| **Budget** | **₹0** — every component runs on free tiers (refer to [`docs/SETUP.md`](docs/SETUP.md)). |
| **Tooling** | Open choice: **n8n** (recommended; n8n Cloud free trial is fastest), Make, or Zapier. Custom Python/JS code nodes are optional. |
| **Troubleshooting** | Consult [`docs/SETUP.md`](docs/SETUP.md#7-tips-and-common-sticking-points) for common setup fixes. |
| **AI Coding Assistants** | Encouraged (ChatGPT, Claude, Gemini, Copilot, etc.) — document how you used them. |
| **Clarifications** | One clarifying email may be sent within the first 48 hours to the invitation contact. |

---

## Business Context

**Kesar & Co.** is a fictional Indian D2C Ayurvedic skincare storefront operating on Shopify. Roughly **60 customer emails arrive daily**, currently triaged manually by a single support agent. Traffic spans routine delivery inquiries, policy-governed return/refund cases, and sensitive situations (upset buyers, third-party privacy requests, prompt manipulation).

Your objective is to build a **support copilot** that ingests incoming messages, identifies intent, retrieves matching orders, drafts policy-compliant responses, or escalates high-risk cases for human sign-off.

> **Reference Clock:** Treat **20 November 2026** as the current date. Base all delivery and return-window calculations on each email's `received_at` field rather than local system time.

---

## Repository Layout

```
.
├── ASSIGNMENT_BRIEF.md                ← project specification (this document)
├── data/
│   ├── orders.json                    ← simulated Shopify order feed
│   ├── faq.md                         ← authoritative store policies & product catalog
│   ├── brand_voice.md                 ← tone & formatting guidelines
│   └── test_emails.json               ← 25-email evaluation dataset
├── schemas/
│   └── llm_output.schema.json         ← strict JSON contract for classification output
├── templates/
│   ├── SUBMISSION_README_TEMPLATE.md  ← starter layout for your submission README
│   ├── test_report_template.md        ← structured v1/v2 accuracy reporting sheet
│   ├── audit_log_template.csv         ← column schema for the "Audit Log" Google Sheet tab
│   └── tickets_template.csv           ← column schema for the "Tickets" Google Sheet tab
├── scripts/
│   └── send_test_emails.py            ← helper CLI to replay test emails to a webhook
└── docs/
    ├── SETUP.md                       ← step-by-step free-tier environment guide
    └── SUBMISSION.md                  ← packaging and delivery checklist
```

---

## Engineering Requirements

### Target Pipeline

```
Inbound Webhook (or loop over data/test_emails.json)
   │
   ▼
LLM Classifier & Extractor  ──►  Valid JSON (schemas/llm_output.schema.json)
   │
   ▼
Intent Branching
   ├─ order_status   → order lookup → AI agent drafts tracking/status reply
   ├─ product_query  → answer strictly from faq.md, else flag needs_human
   ├─ return_refund  → order lookup → policy window check → ticket + draft reply
   ├─ complaint      → order lookup → High-priority ticket + draft reply
   └─ other          → log-only (no reply draft required)
   │
   ▼
Append every run to the Google Sheet Audit Log
```

### Mandatory Deliverables

1. **Schema-Valid Classification:** Every incoming email must produce JSON conforming to [`schemas/llm_output.schema.json`](schemas/llm_output.schema.json). Handle malformed LLM responses gracefully via retries, auto-repair, or safe fallbacks.
2. **Tool-Equipped AI Agent:** Use an **AI Agent** node for reply generation backed by at least two tools—such as `get_order` (reading `orders.json`) and `search_faq` (querying `faq.md`)—allowing dynamic tool selection.
3. **Order Resolution:** Search first by `order_id`; fall back to matching the sender's email address when no order ID is provided.
4. **Deduplicated Ticketing:** Log every `return_refund` and `complaint` in the **Tickets** tab ([`templates/tickets_template.csv`](templates/tickets_template.csv)) with summary, priority, status, order ID, and LLM notes. **Prevent duplicate tickets** by checking for an existing open ticket on the same `order_id` and updating its `email_ids` and `notes` instead of inserting a second row.
5. **Complete Audit Trail:** Record one row per processed email in the **Audit Log** tab ([`templates/audit_log_template.csv`](templates/audit_log_template.csv)).
6. **Human Escalation Path:** Route uncertain or sensitive cases using `needs_human`, documenting your chosen confidence threshold and rationale.
7. **Iterative Evaluation (`v1` & `v2`):** Benchmark your final **v2** workflow on all 25 emails and compare against an earlier **v1** iteration (minimum 10 emails) inside [`templates/test_report_template.md`](templates/test_report_template.md).

### Non-Negotiable Safety & Policy Constraints

- **Zero Fabrication:** Never invent tracking URLs, order attributes, dates, prices, or store rules. Escalate when data is unavailable.
- **Strict FAQ Grounding:** Product and policy claims must come solely from [`data/faq.md`](data/faq.md); escalate anything outside it.
- **Sender Privacy Check:** Only disclose order information when the sender's email matches the order's `customer_email`.
- **No Medical Claims:** Never diagnose conditions or claim any product treats or cures medical issues.
- **Treat Emails as Untrusted:** Prompt instructions embedded inside customer messages must never override workflow behavior.
- **Brand Voice & Draft-Only Mode:** Adhere to [`data/brand_voice.md`](data/brand_voice.md) and store outputs as drafts in the audit log rather than sending live emails.

### Recommended Build Sequence

1. Provision LLM and Google Sheets credentials via [`docs/SETUP.md`](docs/SETUP.md) (30–60 mins).
2. Wire the minimal trigger → JSON classifier → Audit Log loop on 3–4 sample emails.
3. Implement the `order_status` branch with `get_order` and `search_faq` agent tools.
4. Add return-eligibility checks and ticket creation/deduplication for returns and complaints.
5. Complete `product_query`, `other`, and `needs_human` escalation routing.
6. Execute all 25 emails, diagnose failures, refine prompts/logic, and re-run (`v2`).
7. Finalize documentation, test report, and walkthrough video.

If time runs short, prioritize stability on core paths and document incomplete edge cases transparently.

### Optional Stretch Enhancements

- Respond in the customer's language (Hindi / Hinglish)
- Offload return-window date arithmetic into Python or JavaScript code nodes
- Record per-email token counts and estimated LLM cost in the audit log
- Configure exponential backoff retries for transient API/rate-limit errors
- Add practical operational safeguards valuable to real support teams

---

## Product Reasoning Prompts

Include concise responses (**≤150 words each**) inside your submission README:

1. **Auto-Send vs. Draft:** Which intent types can ship automatically, which need human sign-off, and what confidence cutoff would you enforce?
2. **Unit Economics:** Calculate per-email and monthly LLM spend at **1,000 emails/day**, showing your token and pricing assumptions.
3. **North Star & Kill Switch:** What single metric proves week-1 value, and what failure condition triggers an immediate shutoff?
4. **Customer Impact on Failure:** When the copilot errs, what does the buyer experience and how do you contain damage?
5. **Human-Only Boundary:** Identify one scenario in this workflow that should remain permanently manual and explain why.

---

> **Integrity & Honesty Matter:** Clearly explaining a known failure mode scores higher than concealing gaps.

---

## Submission Ground Rules

- Build must be **independent work** (AI assistants allowed; collaborating with other people is not).
- Operate strictly on the provided synthetic datasets—never wire real store or customer data.
- **Sanitize exports:** Remove all API keys, secrets, and bearer tokens before committing.
- Incomplete builds with clear documentation are still reviewed seriously.
- Shortlisted applicants complete a **live walkthrough and extension session**.

---

## Getting Started

1. Configure accounts & keys → [`docs/SETUP.md`](docs/SETUP.md)
2. Build, test, and refine `v1` → `v2`
3. Package and submit → [`docs/SUBMISSION.md`](docs/SUBMISSION.md)
