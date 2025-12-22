# Autonomous Sales Engine ADR

**Status:** Accepted  
&nbsp;
**Deciders:** José Thomaz  
&nbsp;
**Date:** 2025-12-18  

**Technical Story:** Design and implementation of a one-click, agentic autonomous sales engine capable of ingesting brand and competitor emails, generating a strategy Blueprint, producing outbound sales emails, and executing mass sends safely.

---

## Context and Problem Statement

We are building an AI-native sales engine where a user provides a brand, competitors, and email access, then receives generated outbound sales emails with minimal setup.  
The challenge is orchestrating ingestion, reasoning, and generation steps reliably without overengineering early-stage infrastructure or falsely claiming real-time intelligence.

The core question is:  
**How do we build a production-grade, agentic pipeline that is simple to ship, observable, and evolvable—without premature complexity?**

---

## Decision Drivers

- One-click UX (no manual multi-step flows)
- Agentic reasoning from v1
- Clear separation between orchestration and business logic
- Safe handling of side effects (email sending)
- Easy debugging and observability
- Acceptable to investors and senior engineers
- Avoid overengineering (but not a hack)

---

## Considered Options

1. Custom queue + stateless workers + DB-driven stages
2. DAG orchestration frameworks (Dagster / Prefect)
3. Fully real-time, streaming-based architecture
4. Hybrid orchestration with Prefect limited to orchestration only

---

## Decision Outcome

**Chosen option:** *Hybrid orchestration using Prefect strictly for orchestration*, combined with stateless Python workers and explicit storage layers.

This option best balances speed to ship, clarity, agentic reasoning, and long-term flexibility while avoiding orchestration overreach.

---

## Final Architecture

```
┌──────────────┐
│   Frontend   │
│ (React / TS) │
└──────┬───────┘
       │ OAuth + POST /campaign
       ▼
┌─────────────────────┐
│     Core API        │
│   (FastAPI)         │
│ - stores campaign   │
│ - stores tokens     │
└──────┬──────────────┘
       │ webhook trigger
       ▼
┌──────────────────────────┐
│        Prefect           │
│   (ORCHESTRATION ONLY)   │
│                          │
│  Flow = campaign_flow    │
│                          │
│  1) ingest_brand_emails  │
│  2) ingest_competitors   │
│  3) extract_features     │
│  4) generate_blueprint   │
└──────┬───────────────────┘
       │ writes artifacts
       ▼
┌──────────────────────────────────────────┐
│               Storage                    │
│  Postgres | S3 | pgvector                │
└──────────────────────────────────────────┘

(after approval)

┌────────────────────────┐
│   Send Workers         │
│ (rate-limited)         │
└─────────┬──────────────┘
          ▼
      ESP (SES / SendGrid)
```

### Detailed Diagram

<img width="8269" height="9465" alt="image" src="https://github.com/user-attachments/assets/0c19c07c-7d84-49ee-aca6-8328c6ebdb38" />

---

## Why Prefect (and How It Is Used)

Prefect is used **only** as an orchestration engine:

- Triggered via webhook from Core API
- Executes a predefined `campaign_flow`
- Tracks progress and failures
- Exposes run state for frontend polling

Prefect **does NOT**:
- Contain business logic
- Perform reasoning
- Replace agents or workers

All intelligence lives in Python code, not in Prefect graphs.

---

## Pipeline Breakdown

### 1. Campaign Creation
- Frontend submits brand + competitors
- Core API stores campaign + OAuth tokens
- Core API triggers Prefect via webhook

### 2. Brand Email Ingestion
- Batch ingestion from Gmail/Outlook
- Raw emails → S3
- Normalized emails → Postgres

### 3. Competitor Email Ingestion
- Inbox seeding
- Public sources
- Third-party tools (if available)
- Heuristics over structure and cadence

No hard performance metrics required.

### 4. Feature Extraction
- Deterministic, batch computation
- Lengths, cadence, CTAs, follow-ups
- Stored in Postgres

### 5. Blueprint Generation (Agentic)
- Aggregates brand + competitor patterns
- Applies conservative constraints
- LLM used only for synthesis
- Outputs a structured Blueprint

### 6. Email Generation
- Existing generation service
- Blueprint-enforced
- Stored and previewed
- Not auto-sent

### 7. Mass Send (Post-Approval)
- Separate send workers
- Rate-limited
- Idempotent
- Kill-switch protected
- I suggest a tool like [MailReach](https://www.mailreach.co) to help with handling domain warmup and spam checks

---

## Agent Model (Key Principle)

> **Agents reason. Workers execute. Prefect orchestrates.**

- Agents are stateless Python functions
- Inputs/outputs are structured
- LLMs are used only for reasoning
- Side effects happen in workers

---

## Pros and Cons of the Options

### Option 1: Custom Queue + Workers

**Good**
- Maximum control
- Clean separation

**Bad**
- More infra to build
- More operational overhead early

---

### Option 2: Heavy DAG Frameworks Everywhere

**Good**
- Visual pipelines
- Built-in retries

**Bad**
- Encourages logic in orchestration
- Slows iteration
- Harder agent evolution

---

### Option 3: Real-Time Streaming

**Good**
- Sounds impressive

**Bad**
- Not needed
- Hard to debug
- Overkill for UX

---

### Option 4: Prefect for Orchestration Only (Chosen)

**Good**
- Fast to ship
- Observable
- Investor-safe
- Evolvable

**Bad**
- Requires discipline not to misuse Prefect

---

## Deep-dive in the details

If you wish to understand more about some specific parts of this system design, check the following links:

- [What Makes a Good Sales Email?](https://github.com/josethz00/autonomous-sales-engine-adr/blob/main/GOOD_SALES_EMAIL.md)
- [Database Indexing and Blueprint](https://github.com/josethz00/autonomous-sales-engine-adr/blob/main/DATABASE_INDEXING_AND_BLUEPRINT.md)
- [Limitations and Edge Cases](https://github.com/josethz00/autonomous-sales-engine-adr/blob/main/LIMITATIONS_AND_EDGE_CASES.md)

## Links

- [Prefect](https://www.prefect.io/)
- [FastAPI](https://fastapi.tiangolo.com/)
- [PostgreSQL](https://www.postgresql.org/)
- [pgvector](https://github.com/pgvector/pgvector)
- [AWS SQS](https://aws.amazon.com/sqs/)
- [AWS S3](https://aws.amazon.com/s3/)
- [AWS SES](https://aws.amazon.com/ses/)
