# Kesar & Co. Support Copilot — Raghuveer Singh

> Template guide: copy into `README.md` for submission and complete every section concisely (bullet points welcome).

## 1. Executive Summary

- **Solution Overview (2–3 lines):**
- **Platform & Model Stack:**
- **Total Time Invested (hours):**
- **Working Scope & Known Gaps:**

## 2. System Architecture

Include a full-canvas screenshot alongside a brief node-by-node breakdown.

![workflow](./screenshots/workflow.png)

| Node / Stage | Responsibility | Design Rationale |
|---|---|---|
| | | |

## 3. Reproducibility & Setup

1. **Required Accounts / Credentials** *(never include live keys)*:
2. **Importing the Workflow Blueprint:**
3. **Configuring Environment Variables & Secrets:**
4. **Running the 25-Email Test Suite:**

## 4. Prompt Specifications

Document your final classification and reply-drafting prompts along with model parameters (model ID, temperature, output format).

## 5. Architecture Decisions & Assumptions

- **Low-Confidence Handling:**
- **Ticket Deduplication Logic:**
- **Customer Email Privacy Enforcement:**
- **FAQ Knowledge Grounding:**
- **Additional Assumptions:**

## 6. Scaling to 1,000 Emails / Day

Detail expected bottlenecks (rate limits, storage contention, latency, error recovery) and the architectural upgrades needed to handle 1,000 messages daily.

## 7. Product Thinking *(≤150 words per answer)*

1. **Auto-Send vs. Draft:** Which messages can send automatically, which require human review, and where is your confidence threshold?
2. **Cost Projection:** Estimate per-email and monthly LLM spend at 1,000 emails/day with full token math.
3. **Success Metric & Kill Switch:** What week-1 KPI tracks success, and what condition triggers an immediate shutoff?
4. **Failure Containment:** What happens for the customer when automation gets it wrong, and how do you limit harm?
5. **What Stays Manual:** Name one workflow step that should never be automated and justify why.

## 8. Use of AI Assistants

Summarize which AI tools assisted your build (prompting, debugging, code snippets) versus where you applied manual engineering judgment.

## 9. Roadmap (With One More Week)

-
