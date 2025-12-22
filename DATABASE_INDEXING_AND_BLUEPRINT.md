# Database, Indexing, and Blueprint

The system separates **raw data storage**, **queryable structured data**, and **semantic retrieval** to balance cost, flexibility, correctness, and performance.

---

## Core Tables (Postgres)

### `emails`
- One row per ingested email (brand or competitor)
- Stores normalized fields used for filtering, aggregation, and analysis

Columns:
- `id`
- `campaign_id`
- `source ('brand' | 'competitor')`
- `provider ('gmail' | 'outlook' | 'ses' | ...)`
- `thread_id`
- `subject`
- `body`
- `sent_at`
- `metadata`
- `created_at`

To guarantee idempotency, we can use `sent_at` , `campaign_id`  and `provider`  to ensure an email is not sent more than once. The Lambda function will check in the database the state of this email to ensure exact once delivery.

Indexes:
- `(campaign_id)`
- `(source, campaign_id)`
- `(sent_at)`
- GIN index on `metadata` for flexible filtering

---

### `email_features`
- Deterministic, extracted signals (no LLM output)

Columns:
- `word_count`
- `cta_type`
- `follow_up_index`
- `has_personalization`
- `structure_type`
- `cadence_bucket` (e.g. same-day, 2–3 days, 5–7 days)

Indexes:
- `(campaign_id)`
- `(cta_type)`
- `(follow_up_index)`
- `(cadence_bucket)`

Enables queries like:
> “Show competitor emails with soft CTAs, sent as step 2, with a 3–5 day follow-up cadence.”

---

## Embeddings (pgvector)

### `email_embeddings`
- Stores semantic representations of emails

Columns:
- `email_id`
- `embedding VECTOR`

Indexes:
- Approximate Nearest Neighbor algorithms for indexing: `ivfflat` or `hnsw` index on `embedding` . Preferably, HNSW, for consistent results and better read performance.

Used for:
- Semantic similarity and clustering
- Competitor pattern discovery
- Supporting Blueprint synthesis

Embeddings **complement structured filters**, even though they never replace deterministic features or indexes.

---

## Object Storage (S3)

S3 is used as the **raw / bronze layer**, following data engineering best practices:

Stored artifacts:
- Raw Gmail / Outlook API payloads
- Raw CSV files, for the cases where the user manually uploads files instead of connecting via OAuth
- Original email HTML
- Ingestion logs and debug artifacts

Benefits:
- Enables retries, reprocessing, and backfills
- Preserves original source-of-truth data
- Keeps Postgres lean and cost-efficient

S3 is **never queried directly** for analytics.  
Postgres remains the system of record for all analysis and learning.

---

## Blueprint Representation (Dos, Don’ts, Cadence)

The **Blueprint** is derived from Postgres data and stored as a structured artifact:

Includes:
- **Dos** (e.g. preferred CTA types, opening styles)
- **Don’ts** (e.g. hard meeting asks on first touch)
- **Cadence rules** (e.g. follow-up spacing ranges)
- **Structural constraints** (length, sections, tone)

Blueprints are:
- Human-readable
- Deterministic
- Enforced downstream during generation and sending

---

## Why This Works

- Postgres = fast, filterable, debuggable
- JSONB = flexible without schema churn
- pgvector = semantic understanding, where it adds value
- S3 = cheap, replayable raw storage

Result: a dataset that is both **queryable by rules** and **searchable by meaning**, while remaining safe, idempotent, and production-ready.
