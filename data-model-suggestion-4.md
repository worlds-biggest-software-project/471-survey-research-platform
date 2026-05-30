# Data Model Suggestion 4: Multi-Engine Polyglot Architecture (PostgreSQL + ClickHouse + Meilisearch + Redis)

> Project: Survey & Research Platform (Candidate #471)
> Generated: 2026-05-26

## Overview

This model uses purpose-built storage engines for each distinct workload in the survey platform, rather than forcing a single database to serve all access patterns. The fundamental observation is that a survey research platform has four fundamentally different data access patterns, each with different performance characteristics:

1. **Transactional OLTP** (survey design, panel management, user management) — row-oriented reads and writes with strong consistency requirements. Best served by PostgreSQL.

2. **High-volume append-only ingestion with real-time aggregation** (response collection, answer storage, live dashboards) — millions of writes per hour during large survey campaigns, with concurrent aggregation queries. Best served by a columnar analytical database like ClickHouse.

3. **Full-text search and theme extraction** (open-text response searching, survey content search, theme analysis indexing) — fuzzy matching, relevance scoring, and faceted filtering across unstructured text. Best served by a search engine like Meilisearch or Elasticsearch.

4. **Real-time counters, session state, and cache** (live response counts, quota tracking, rate limiting, survey session state) — sub-millisecond reads and writes with TTL-based expiration. Best served by Redis.

Rather than compromising on any of these workloads, this architecture gives each engine its optimal data model and query language, connected by a Change Data Capture (CDC) pipeline that keeps all engines consistent.

## Technology Stack

| Component | Technology | Role |
|-----------|-----------|------|
| **Transactional Store** | PostgreSQL 17 | Source of truth for surveys, users, orgs, panels, experiments, compliance |
| **Analytical Store** | ClickHouse 24+ | Response storage, cross-tabulation, statistical analysis, conjoint/MaxDiff scoring |
| **Search Engine** | Meilisearch 1.x | Full-text search on survey content, open-text responses, panel member search |
| **Cache & Real-Time** | Redis 7+ (with Redis Stack for time-series) | Live counters, quota tracking, session state, rate limiting, pub/sub for live dashboards |
| **CDC Pipeline** | Debezium + Apache Kafka (or NATS JetStream) | Streams changes from PostgreSQL to ClickHouse, Meilisearch, and Redis |
| **Object Storage** | S3-compatible (MinIO for self-hosted) | File uploads, report PDFs, long-term response archives (Parquet) |
| **Vector Store** | pgvector extension on PostgreSQL | Embedding storage for AI-powered theme extraction and semantic search on open-text responses |
| **ORM / Query Layer** | Prisma (PostgreSQL), @clickhouse/client, meilisearch-js, ioredis | Type-safe access to each engine |

---

## Architecture Diagram

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                    API Layer                             │
                    │        (REST / GraphQL / WebSocket / MCP)               │
                    └─────────┬──────────┬──────────┬──────────┬─────────────┘
                              │          │          │          │
                    ┌─────────▼────┐ ┌───▼─────┐ ┌─▼────────┐ ┌▼───────────┐
                    │  PostgreSQL  │ │ClickHouse│ │Meilisearch│ │   Redis    │
                    │  (OLTP)      │ │ (OLAP)   │ │ (Search)  │ │ (Cache+RT) │
                    └──────┬───────┘ └──▲───────┘ └──▲────────┘ └──▲────────┘
                           │            │            │             │
                    ┌──────▼────────────┴────────────┴─────────────┘
                    │              CDC Pipeline (Debezium → Kafka)
                    └───────────────────────────────────────────────
```

**Data flow:**
- All writes go to PostgreSQL (source of truth)
- Debezium captures changes from PostgreSQL WAL
- Kafka topics distribute changes to ClickHouse, Meilisearch, and Redis consumers
- Read queries are routed to the optimal engine based on the query type

---

## PostgreSQL Schema (Transactional OLTP)

PostgreSQL stores the source-of-truth for all entities. The schema is similar to a hybrid relational+JSONB model (Suggestion 3), but responses are stored temporarily here and then streamed to ClickHouse for analytics.

```sql
-- ============================================================
-- CORE ENTITIES (identical to suggestions 1/3 — abbreviated here)
-- ============================================================

CREATE TABLE organizations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL UNIQUE,
    plan_tier           VARCHAR(50) NOT NULL DEFAULT 'free',
    data_residency      VARCHAR(10) NOT NULL DEFAULT 'us',
    settings            JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMPTZ
);

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               VARCHAR(255) NOT NULL UNIQUE,
    email_verified      BOOLEAN NOT NULL DEFAULT FALSE,
    password_hash       VARCHAR(255),
    full_name           VARCHAR(255) NOT NULL,
    preferences         JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_memberships (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role                VARCHAR(50) NOT NULL DEFAULT 'member',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);

CREATE TABLE workspaces (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- SURVEYS — JSONB design document (same as Suggestion 3)
-- ============================================================

CREATE TABLE surveys (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    workspace_id        UUID REFERENCES workspaces(id) ON DELETE SET NULL,
    title               VARCHAR(500) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    survey_type         VARCHAR(50) NOT NULL DEFAULT 'standard',
    default_language    VARCHAR(10) NOT NULL DEFAULT 'en',
    question_count      INTEGER NOT NULL DEFAULT 0,
    response_limit      INTEGER,
    starts_at           TIMESTAMPTZ,
    closes_at           TIMESTAMPTZ,
    published_at        TIMESTAMPTZ,
    created_by          UUID NOT NULL REFERENCES users(id),
    design              JSONB NOT NULL DEFAULT '{}'::jsonb,
    settings            JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_surveys_org ON surveys (organization_id);
CREATE INDEX idx_surveys_status ON surveys (status);
CREATE INDEX idx_surveys_design ON surveys USING GIN (design jsonb_path_ops);

-- ============================================================
-- SURVEY VERSIONS (frozen snapshots)
-- ============================================================

CREATE TABLE survey_versions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    version_number      INTEGER NOT NULL,
    design_snapshot     JSONB NOT NULL,
    settings_snapshot   JSONB NOT NULL,
    published_by        UUID NOT NULL REFERENCES users(id),
    published_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (survey_id, version_number)
);

-- ============================================================
-- DISTRIBUTION CHANNELS
-- ============================================================

CREATE TABLE distribution_channels (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    channel_type        VARCHAR(30) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    unique_link_slug    VARCHAR(100) UNIQUE,
    channel_config      JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE email_invitations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id          UUID NOT NULL REFERENCES distribution_channels(id) ON DELETE CASCADE,
    recipient_email     VARCHAR(255) NOT NULL,
    token               VARCHAR(100) NOT NULL UNIQUE,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
    sent_at             TIMESTAMPTZ,
    opened_at           TIMESTAMPTZ,
    clicked_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PANELS & RESPONDENTS
-- ============================================================

CREATE TABLE panels (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    panel_type          VARCHAR(30) NOT NULL DEFAULT 'internal',
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    member_count        INTEGER NOT NULL DEFAULT 0,
    custom_attribute_schema JSONB NOT NULL DEFAULT '[]'::jsonb,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE panel_respondents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    panel_id            UUID NOT NULL REFERENCES panels(id) ON DELETE CASCADE,
    email               VARCHAR(255),
    external_id         VARCHAR(255),
    first_name          VARCHAR(255),
    last_name           VARCHAR(255),
    gender              VARCHAR(20),
    birth_year          INTEGER,
    country             VARCHAR(3),
    region              VARCHAR(100),
    language            VARCHAR(10),
    consent_given       BOOLEAN NOT NULL DEFAULT FALSE,
    consent_given_at    TIMESTAMPTZ,
    opt_out             BOOLEAN NOT NULL DEFAULT FALSE,
    total_surveys_sent  INTEGER NOT NULL DEFAULT 0,
    total_surveys_completed INTEGER NOT NULL DEFAULT 0,
    quality_score       DECIMAL(5,2),
    custom_attributes   JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_respondents_panel ON panel_respondents (panel_id);
CREATE INDEX idx_respondents_email ON panel_respondents (email);
CREATE INDEX idx_respondents_country ON panel_respondents (country);
CREATE INDEX idx_respondents_custom ON panel_respondents USING GIN (custom_attributes jsonb_path_ops);

-- ============================================================
-- RESPONSES IN POSTGRESQL (staging table — streamed to ClickHouse)
-- ============================================================
-- Responses are written here first for transactional consistency,
-- then CDC streams them to ClickHouse. PostgreSQL retains responses
-- for 90 days; ClickHouse is the long-term analytical store.

CREATE TABLE survey_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id),
    survey_version      INTEGER NOT NULL,
    channel_id          UUID REFERENCES distribution_channels(id),
    respondent_id       UUID REFERENCES panel_respondents(id),
    external_user_id    VARCHAR(255),
    status              VARCHAR(30) NOT NULL DEFAULT 'in_progress',
    language            VARCHAR(10) NOT NULL DEFAULT 'en',
    ip_address          INET,
    user_agent          TEXT,
    geo_country         VARCHAR(3),
    geo_region          VARCHAR(100),
    duration_seconds    INTEGER,
    is_test             BOOLEAN NOT NULL DEFAULT FALSE,
    weight              DECIMAL(10,6) DEFAULT 1.0,
    answers             JSONB NOT NULL DEFAULT '{}'::jsonb,
    started_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Only keep recent partitions in PostgreSQL (older data lives in ClickHouse)
CREATE TABLE survey_responses_2026_q2 PARTITION OF survey_responses
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE survey_responses_2026_q3 PARTITION OF survey_responses
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');

CREATE INDEX idx_pg_responses_survey ON survey_responses (survey_id);
CREATE INDEX idx_pg_responses_status ON survey_responses (status);

-- ============================================================
-- EXPERIMENTS
-- ============================================================

CREATE TABLE experiments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    hypothesis          TEXT,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    allocation_method   VARCHAR(30) NOT NULL DEFAULT 'random',
    experiment_config   JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE experiment_groups (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id       UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    group_name          VARCHAR(100) NOT NULL,
    group_type          VARCHAR(30) NOT NULL,
    allocation_pct      DECIMAL(5,2) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE experiment_assignments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id       UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    group_id            UUID NOT NULL REFERENCES experiment_groups(id) ON DELETE CASCADE,
    response_id         UUID NOT NULL,
    assigned_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (experiment_id, response_id)
);

-- ============================================================
-- LONGITUDINAL STUDIES
-- ============================================================

CREATE TABLE longitudinal_studies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    panel_id            UUID REFERENCES panels(id),
    study_config        JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE study_waves (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    study_id            UUID NOT NULL REFERENCES longitudinal_studies(id) ON DELETE CASCADE,
    survey_id           UUID NOT NULL REFERENCES surveys(id),
    wave_number         INTEGER NOT NULL,
    wave_label          VARCHAR(100),
    launched_at         TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (study_id, wave_number)
);

-- ============================================================
-- COMPLIANCE (GDPR, audit)
-- ============================================================

CREATE TABLE consent_records (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    respondent_id       UUID REFERENCES panel_respondents(id) ON DELETE SET NULL,
    response_id         UUID,
    consent_type        VARCHAR(50) NOT NULL,
    consent_given       BOOLEAN NOT NULL,
    consent_text        TEXT NOT NULL,
    ip_address          INET,
    legal_basis         VARCHAR(50) NOT NULL DEFAULT 'consent',
    given_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    withdrawn_at        TIMESTAMPTZ
);

CREATE TABLE data_deletion_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    requester_email     VARCHAR(255) NOT NULL,
    respondent_id       UUID,
    request_type        VARCHAR(30) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    organization_id     UUID NOT NULL,
    user_id             UUID,
    action              VARCHAR(100) NOT NULL,
    entity_type         VARCHAR(50) NOT NULL,
    entity_id           UUID,
    details             JSONB NOT NULL DEFAULT '{}'::jsonb,
    ip_address          INET,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_log_2026_q2 PARTITION OF audit_log
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE audit_log_2026_q3 PARTITION OF audit_log
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');

-- ============================================================
-- VECTOR EMBEDDINGS (pgvector)
-- ============================================================

CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE response_embeddings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    response_id         UUID NOT NULL,
    question_id         VARCHAR(100) NOT NULL,
    survey_id           UUID NOT NULL,
    text_content        TEXT NOT NULL,
    embedding           vector(1536) NOT NULL,  -- OpenAI ada-002 / Claude embedding dimension
    model_version       VARCHAR(100) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embeddings_survey ON response_embeddings (survey_id, question_id);

-- IVFFlat index for approximate nearest neighbor search
CREATE INDEX idx_embeddings_vector ON response_embeddings
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

---

## ClickHouse Schema (Analytical OLAP)

ClickHouse is the primary engine for all analytical queries: response aggregation, cross-tabulation, statistical analysis, NPS trending, conjoint utility estimation, and MaxDiff scoring. Its columnar storage and vectorized query engine handle billions of rows with sub-second query times.

```sql
-- ============================================================
-- CLICKHOUSE: RESPONSE FACTS
-- ============================================================
-- This is the primary analytical table. Each row represents one
-- completed survey response with flattened metadata.

CREATE TABLE response_facts (
    -- Identity
    response_id         UUID,
    survey_id           UUID,
    organization_id     UUID,
    survey_version      UInt32,

    -- Distribution
    channel_id          UUID,
    channel_type        LowCardinality(String),

    -- Respondent
    respondent_id       UUID,
    external_user_id    String,

    -- Response metadata
    status              LowCardinality(String),
    language            LowCardinality(String),
    geo_country         LowCardinality(String),
    geo_region          LowCardinality(String),
    duration_seconds    UInt32,
    is_test             UInt8,
    weight              Float64 DEFAULT 1.0,

    -- Respondent demographics (denormalized from panel)
    respondent_gender       LowCardinality(String) DEFAULT '',
    respondent_birth_year   UInt16 DEFAULT 0,
    respondent_country      LowCardinality(String) DEFAULT '',
    respondent_region       LowCardinality(String) DEFAULT '',
    respondent_language     LowCardinality(String) DEFAULT '',

    -- Experiment assignment
    experiment_id       UUID DEFAULT '00000000-0000-0000-0000-000000000000',
    experiment_group    LowCardinality(String) DEFAULT '',

    -- Timestamps
    started_at          DateTime64(3, 'UTC'),
    completed_at        DateTime64(3, 'UTC'),
    created_at          DateTime64(3, 'UTC'),

    -- Raw answer payload (stored for reprocessing)
    answers_json        String
)
ENGINE = MergeTree()
PARTITION BY (toYYYYMM(created_at), organization_id)
ORDER BY (organization_id, survey_id, created_at, response_id)
TTL created_at + INTERVAL 5 YEAR
SETTINGS index_granularity = 8192;

-- ============================================================
-- CLICKHOUSE: ANSWER FACTS (flattened, one row per question per response)
-- ============================================================
-- This is the workhorse table for cross-tabulation and statistical analysis.

CREATE TABLE answer_facts (
    -- Identity
    response_id         UUID,
    survey_id           UUID,
    organization_id     UUID,
    question_id         String,   -- question ID from the survey design
    question_code       LowCardinality(String),  -- researcher-assigned code (Q1, Q2, etc.)
    question_type       LowCardinality(String),

    -- Answer values (populated based on question type)
    selected_option_id  String DEFAULT '',
    selected_option_label String DEFAULT '',
    text_value          String DEFAULT '',
    numeric_value       Float64 DEFAULT 0,
    array_values        Array(String) DEFAULT [],
    rank_position       UInt8 DEFAULT 0,

    -- Timing
    duration_ms         UInt32 DEFAULT 0,

    -- Response metadata (denormalized for query efficiency)
    response_status     LowCardinality(String),
    response_language   LowCardinality(String),
    geo_country         LowCardinality(String),
    respondent_gender   LowCardinality(String) DEFAULT '',
    respondent_birth_year UInt16 DEFAULT 0,
    weight              Float64 DEFAULT 1.0,

    -- Experiment
    experiment_group    LowCardinality(String) DEFAULT '',

    -- Timestamp
    answered_at         DateTime64(3, 'UTC'),
    created_at          DateTime64(3, 'UTC')
)
ENGINE = MergeTree()
PARTITION BY (toYYYYMM(created_at), organization_id)
ORDER BY (organization_id, survey_id, question_id, created_at)
TTL created_at + INTERVAL 5 YEAR
SETTINGS index_granularity = 8192;

-- ============================================================
-- CLICKHOUSE: CONJOINT RESPONSES (one row per task choice)
-- ============================================================

CREATE TABLE conjoint_facts (
    response_id         UUID,
    survey_id           UUID,
    organization_id     UUID,
    design_id           String,
    task_number         UInt8,

    -- The chosen profile and its attribute levels
    chosen_profile_id   String,
    chosen_none         UInt8 DEFAULT 0,

    -- Attribute levels for each profile shown (denormalized)
    -- Using arrays: index 0 = profile 1, index 1 = profile 2, etc.
    profile_ids         Array(String),
    attribute_names     Array(String),
    -- For each profile, the level values as a nested array
    -- profile_levels[profile_idx][attribute_idx] = level_label
    profile_levels      Array(Array(String)),

    response_time_ms    UInt32,
    weight              Float64 DEFAULT 1.0,

    -- Demographics for segmented analysis
    respondent_gender   LowCardinality(String) DEFAULT '',
    respondent_birth_year UInt16 DEFAULT 0,
    geo_country         LowCardinality(String) DEFAULT '',

    created_at          DateTime64(3, 'UTC')
)
ENGINE = MergeTree()
PARTITION BY (toYYYYMM(created_at))
ORDER BY (organization_id, survey_id, design_id, response_id, task_number)
SETTINGS index_granularity = 8192;

-- ============================================================
-- CLICKHOUSE: MAXDIFF RESPONSES (one row per set choice)
-- ============================================================

CREATE TABLE maxdiff_facts (
    response_id         UUID,
    survey_id           UUID,
    organization_id     UUID,
    design_id           String,
    set_number          UInt8,

    -- Items shown in this set
    item_ids            Array(String),
    item_labels         Array(String),

    -- Chosen best and worst
    best_item_id        String,
    best_item_label     String,
    worst_item_id       String,
    worst_item_label    String,

    response_time_ms    UInt32,
    weight              Float64 DEFAULT 1.0,

    respondent_gender   LowCardinality(String) DEFAULT '',
    geo_country         LowCardinality(String) DEFAULT '',

    created_at          DateTime64(3, 'UTC')
)
ENGINE = MergeTree()
PARTITION BY (toYYYYMM(created_at))
ORDER BY (organization_id, survey_id, design_id, response_id, set_number)
SETTINGS index_granularity = 8192;

-- ============================================================
-- CLICKHOUSE: NPS DAILY AGGREGATES (materialized view)
-- ============================================================

CREATE MATERIALIZED VIEW nps_daily_mv
ENGINE = SummingMergeTree()
PARTITION BY (toYYYYMM(day))
ORDER BY (organization_id, survey_id, question_id, day)
AS
SELECT
    organization_id,
    survey_id,
    question_id,
    toDate(answered_at) AS day,
    countIf(numeric_value >= 9) AS promoters,
    countIf(numeric_value >= 7 AND numeric_value <= 8) AS passives,
    countIf(numeric_value <= 6) AS detractors,
    count() AS total_responses
FROM answer_facts
WHERE question_type = 'net_promoter_score'
GROUP BY organization_id, survey_id, question_id, day;

-- ============================================================
-- CLICKHOUSE: COMPLETION RATE AGGREGATES (materialized view)
-- ============================================================

CREATE MATERIALIZED VIEW completion_daily_mv
ENGINE = SummingMergeTree()
PARTITION BY (toYYYYMM(day))
ORDER BY (organization_id, survey_id, channel_type, day)
AS
SELECT
    organization_id,
    survey_id,
    channel_type,
    toDate(created_at) AS day,
    countIf(status = 'completed') AS completed,
    countIf(status = 'abandoned') AS abandoned,
    countIf(status = 'screened_out') AS screened_out,
    count() AS total_started,
    avgIf(duration_seconds, status = 'completed') AS avg_duration
FROM response_facts
GROUP BY organization_id, survey_id, channel_type, day;
```

### Example Analytical Queries in ClickHouse

```sql
-- Cross-tabulation: gender vs satisfaction for a survey
SELECT
    respondent_gender AS gender,
    selected_option_label AS satisfaction,
    count() AS count,
    round(count() * 100.0 / sum(count()) OVER (PARTITION BY respondent_gender), 1) AS pct
FROM answer_facts
WHERE survey_id = '...'
  AND question_code = 'Q2'
  AND response_status = 'completed'
GROUP BY gender, satisfaction
ORDER BY gender, satisfaction;

-- Weighted NPS calculation with confidence interval
SELECT
    countIf(numeric_value >= 9) AS promoters,
    countIf(numeric_value <= 6) AS detractors,
    count() AS n,
    round((sumIf(weight, numeric_value >= 9) - sumIf(weight, numeric_value <= 6))
          / sum(weight) * 100, 1) AS weighted_nps,
    -- Margin of error approximation (95% CI)
    round(1.96 * sqrt(
        (sumIf(weight, numeric_value >= 9) / sum(weight)) *
        (1 - sumIf(weight, numeric_value >= 9) / sum(weight)) / count()
        +
        (sumIf(weight, numeric_value <= 6) / sum(weight)) *
        (1 - sumIf(weight, numeric_value <= 6) / sum(weight)) / count()
    ) * 100, 1) AS margin_of_error
FROM answer_facts
WHERE survey_id = '...'
  AND question_type = 'net_promoter_score'
  AND response_status = 'completed';

-- Longitudinal NPS trend across study waves
SELECT
    sw.wave_label,
    nps.day,
    nps.promoters,
    nps.detractors,
    nps.total_responses,
    round((nps.promoters - nps.detractors) * 100.0 / nps.total_responses, 1) AS nps_score
FROM nps_daily_mv nps
-- Join with PostgreSQL study_waves via application layer
ORDER BY nps.day;

-- Response time analysis per question (identify problematic questions)
SELECT
    question_code,
    question_type,
    count() AS responses,
    round(avg(duration_ms) / 1000, 1) AS avg_seconds,
    round(quantile(0.5)(duration_ms) / 1000, 1) AS median_seconds,
    round(quantile(0.95)(duration_ms) / 1000, 1) AS p95_seconds
FROM answer_facts
WHERE survey_id = '...'
  AND response_status = 'completed'
GROUP BY question_code, question_type
ORDER BY avg_seconds DESC;
```

---

## Meilisearch Index Configuration

Meilisearch provides instant, typo-tolerant search for survey content and open-text responses.

```jsonc
// Index: surveys
// Populated from PostgreSQL surveys table via CDC
{
    "uid": "surveys",
    "primaryKey": "id",
    "searchableAttributes": ["title", "internal_name", "design_text"],
    "filterableAttributes": ["organization_id", "status", "survey_type", "created_at"],
    "sortableAttributes": ["created_at", "title"],
    "displayedAttributes": ["id", "title", "status", "survey_type", "question_count", "created_at"]
}

// Index: open_text_responses
// Populated from ClickHouse answer_facts WHERE question_type IN ('text_short', 'text_long')
{
    "uid": "open_text_responses",
    "primaryKey": "id",
    "searchableAttributes": ["text_value"],
    "filterableAttributes": ["survey_id", "question_id", "geo_country", "respondent_gender", "answered_at"],
    "sortableAttributes": ["answered_at"],
    "displayedAttributes": ["id", "survey_id", "question_id", "text_value", "answered_at", "geo_country"]
}

// Index: panel_members
// Populated from PostgreSQL panel_respondents via CDC
{
    "uid": "panel_members",
    "primaryKey": "id",
    "searchableAttributes": ["first_name", "last_name", "email"],
    "filterableAttributes": ["panel_id", "organization_id", "country", "gender", "consent_given", "quality_score"],
    "sortableAttributes": ["quality_score", "last_survey_at", "created_at"],
    "displayedAttributes": ["id", "first_name", "last_name", "email", "country", "quality_score"]
}
```

---

## Redis Data Structures

Redis serves four distinct roles: real-time counters, quota tracking, session state, and pub/sub for live dashboards.

```
# ============================================================
# REAL-TIME RESPONSE COUNTERS
# ============================================================

# Total response count per survey (used for dashboard hero number)
# Key: survey:{surveyId}:responses:{status}
# Type: String (atomic counter)
INCR survey:abc-123:responses:completed
INCR survey:abc-123:responses:abandoned

# Response count by channel
# Key: survey:{surveyId}:channel:{channelId}:count
# Type: String (atomic counter)
INCR survey:abc-123:channel:ch-456:count

# Response count by country (for live geo map)
# Key: survey:{surveyId}:geo
# Type: Hash
HINCRBY survey:abc-123:geo US 1
HINCRBY survey:abc-123:geo GB 1
HINCRBY survey:abc-123:geo AU 1

# ============================================================
# QUOTA TRACKING
# ============================================================

# Quota counter with atomic check-and-increment
# Key: quota:{quotaId}:count
# Type: String (atomic counter)
# Used with Lua script for atomic quota check:

--[[
    KEYS[1] = quota:{quotaId}:count
    ARGV[1] = target count
    Returns: 1 if within quota, 0 if quota full
]]--
local current = redis.call('GET', KEYS[1])
if current == false then current = 0 else current = tonumber(current) end
if current < tonumber(ARGV[1]) then
    redis.call('INCR', KEYS[1])
    return 1
else
    return 0
end

# ============================================================
# SURVEY SESSION STATE
# ============================================================

# Active respondent session (TTL 30 minutes)
# Key: session:{sessionId}
# Type: Hash
HSET session:sess-abc-123 response_id "resp-uuid"
HSET session:sess-abc-123 survey_id "survey-uuid"
HSET session:sess-abc-123 current_page 2
HSET session:sess-abc-123 started_at "2026-06-15T10:30:00Z"
HSET session:sess-abc-123 answers_buffer "{}"  -- partial answers before commit
EXPIRE session:sess-abc-123 1800

# ============================================================
# RATE LIMITING
# ============================================================

# API rate limit per organization (sliding window)
# Key: ratelimit:{orgId}:{minute}
# Type: String (counter with TTL)
INCR ratelimit:org-uuid:202606151030
EXPIRE ratelimit:org-uuid:202606151030 120

# ============================================================
# PUB/SUB FOR LIVE DASHBOARDS
# ============================================================

# Channel for real-time response notifications
# Subscribers: dashboard WebSocket connections
PUBLISH survey:abc-123:live '{"type":"response_completed","responseId":"resp-uuid","duration":287,"country":"US"}'

# ============================================================
# REDIS TIME-SERIES (Redis Stack)
# ============================================================

# Response rate time series (1-minute granularity)
TS.ADD survey:abc-123:ts:responses * 1 LABELS survey_id abc-123 metric responses
TS.RANGE survey:abc-123:ts:responses - + AGGREGATION count 60000
```

---

## CDC Pipeline Architecture

### Debezium Connector Configuration

```json
{
    "name": "survey-platform-postgres-connector",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.hostname": "postgres-primary",
        "database.port": "5432",
        "database.user": "debezium",
        "database.password": "${DEBEZIUM_PASSWORD}",
        "database.dbname": "survey_platform",
        "database.server.name": "survey",
        "table.include.list": "public.surveys,public.survey_responses,public.panel_respondents,public.consent_records,public.audit_log",
        "plugin.name": "pgoutput",
        "slot.name": "debezium_survey",
        "publication.name": "survey_cdc",
        "transforms": "route",
        "transforms.route.type": "io.debezium.transforms.ByLogicalTableRouter",
        "transforms.route.topic.regex": "survey\\.public\\.(.*)",
        "transforms.route.topic.replacement": "survey.cdc.$1"
    }
}
```

### Consumer Pipeline

```
PostgreSQL WAL
    │
    ▼
Debezium (captures INSERT/UPDATE/DELETE)
    │
    ▼
Kafka Topics:
    ├── survey.cdc.surveys         → Meilisearch indexer (survey search)
    ├── survey.cdc.survey_responses → ClickHouse ingester (flatten answers → answer_facts)
    │                               → Redis updater (increment counters)
    │                               → Meilisearch indexer (open-text search)
    ├── survey.cdc.panel_respondents → Meilisearch indexer (panel search)
    │                                → ClickHouse updater (respondent dimensions)
    ├── survey.cdc.consent_records → ClickHouse (compliance audit)
    └── survey.cdc.audit_log       → ClickHouse (long-term audit storage)
```

### Answer Flattening Consumer (Kafka → ClickHouse)

```typescript
// Pseudocode for the answer flattening consumer
async function processResponseEvent(event: DebeziumEvent): Promise<void> {
    const response = event.after;  // the new/updated row

    // 1. Insert into response_facts
    await clickhouse.insert('response_facts', {
        response_id: response.id,
        survey_id: response.survey_id,
        organization_id: await lookupOrgId(response.survey_id),
        survey_version: response.survey_version,
        channel_type: await lookupChannelType(response.channel_id),
        status: response.status,
        language: response.language,
        geo_country: response.geo_country,
        duration_seconds: response.duration_seconds,
        weight: response.weight,
        answers_json: JSON.stringify(response.answers),
        started_at: response.started_at,
        completed_at: response.completed_at,
        created_at: response.created_at,
    });

    // 2. Flatten answers into answer_facts (one row per question)
    const answers = response.answers as Record<string, any>;
    const surveyDesign = await lookupSurveyDesign(response.survey_id, response.survey_version);

    for (const [questionId, answer] of Object.entries(answers)) {
        const question = findQuestionInDesign(surveyDesign, questionId);

        if (answer.type === 'likert_matrix') {
            // Matrix questions produce one row per matrix row
            for (const [rowId, rowAnswer] of Object.entries(answer.rows)) {
                await clickhouse.insert('answer_facts', {
                    response_id: response.id,
                    survey_id: response.survey_id,
                    question_id: `${questionId}:${rowId}`,
                    question_code: `${question.code}_${rowId}`,
                    question_type: 'likert_matrix_row',
                    selected_option_id: rowAnswer.selectedOptionId,
                    numeric_value: rowAnswer.numericValue,
                    weight: response.weight,
                    // ... demographics, geo, etc.
                });
            }
        } else if (answer.type === 'conjoint_choice') {
            await clickhouse.insert('conjoint_facts', {
                response_id: response.id,
                survey_id: response.survey_id,
                design_id: answer.designId || '',
                task_number: answer.taskNumber || 0,
                chosen_profile_id: answer.chosenProfileId,
                response_time_ms: answer.responseTimeMs,
                weight: response.weight,
                // ... profile_levels denormalized
            });
        } else {
            // Standard single-value answer
            await clickhouse.insert('answer_facts', {
                response_id: response.id,
                survey_id: response.survey_id,
                question_id: questionId,
                question_code: question?.code || '',
                question_type: answer.type,
                selected_option_id: answer.selectedOptionId || '',
                selected_option_label: answer.selectedLabel || '',
                text_value: answer.textValue || '',
                numeric_value: answer.numericValue || 0,
                duration_ms: answer.durationMs || 0,
                weight: response.weight,
                // ... demographics, geo, etc.
            });
        }
    }

    // 3. Update Redis counters
    await redis.incr(`survey:${response.survey_id}:responses:${response.status}`);
    if (response.geo_country) {
        await redis.hincrby(`survey:${response.survey_id}:geo`, response.geo_country, 1);
    }

    // 4. Index open-text answers in Meilisearch
    const textAnswers = Object.entries(answers)
        .filter(([, a]) => ['text_short', 'text_long'].includes(a.type) && a.textValue);
    for (const [qId, answer] of textAnswers) {
        await meilisearch.index('open_text_responses').addDocuments([{
            id: `${response.id}:${qId}`,
            survey_id: response.survey_id,
            question_id: qId,
            text_value: answer.textValue,
            geo_country: response.geo_country,
            answered_at: answer.submittedAt || response.created_at,
        }]);
    }

    // 5. Publish to live dashboard channel
    await redis.publish(`survey:${response.survey_id}:live`, JSON.stringify({
        type: 'response_completed',
        responseId: response.id,
        duration: response.duration_seconds,
        country: response.geo_country,
    }));
}
```

---

## Semantic Search with pgvector

For AI-powered theme extraction, open-text responses are embedded using an LLM and stored in pgvector for semantic similarity search.

```sql
-- Find responses semantically similar to a theme
SELECT
    re.response_id,
    re.text_content,
    1 - (re.embedding <=> $1::vector) AS similarity
FROM response_embeddings re
WHERE re.survey_id = $2
  AND re.question_id = $3
  AND 1 - (re.embedding <=> $1::vector) > 0.7  -- similarity threshold
ORDER BY re.embedding <=> $1::vector
LIMIT 50;

-- Cluster responses by theme (application-level k-means on embeddings)
-- Step 1: Fetch all embeddings for a question
SELECT response_id, text_content, embedding
FROM response_embeddings
WHERE survey_id = $1 AND question_id = $2;

-- Step 2: Run k-means clustering in Python/TypeScript
-- Step 3: Store cluster assignments back for querying
```

---

## Pros and Cons

### Pros

1. **Best-in-class performance for every workload.** Each storage engine is optimised for its specific access pattern. ClickHouse handles cross-tabulation across millions of responses in milliseconds. Meilisearch provides instant typo-tolerant search. Redis delivers sub-millisecond counter updates. PostgreSQL ensures transactional consistency for design-time operations.

2. **Massive analytical scalability.** ClickHouse can handle billions of answer_facts rows with sub-second aggregation queries. This is orders of magnitude faster than trying to do cross-tabulation on PostgreSQL JSONB. For a platform targeting mid-market researchers who may run surveys with tens of thousands of responses, this ensures the analytics never become a bottleneck.

3. **Real-time dashboard without compromising OLTP.** Live response counters, geo maps, and NPS scores are served from Redis and ClickHouse materialized views. No analytical query ever touches the PostgreSQL primary. This eliminates the "live dashboard kills write performance" problem.

4. **Semantic search for theme extraction.** pgvector enables AI-powered open-text analysis that goes beyond keyword matching. Researchers can find thematically similar responses across thousands of free-text answers, enabling the "automated open-text theme analysis" feature described in the project README.

5. **Natural data lifecycle management.** Recent response data lives in PostgreSQL (for transactional needs) and ClickHouse (for analytics). Historical data in ClickHouse uses TTL policies for automatic tiering to cold storage. Old PostgreSQL partitions can be dropped once data is confirmed in ClickHouse.

6. **Search that actually works.** Meilisearch provides instant, typo-tolerant search for surveys, questions, and open-text responses. This is dramatically better than PostgreSQL `ILIKE` queries or even `tsvector` full-text search for the "search your responses" use case.

### Cons

1. **Highest operational complexity of all four suggestions.** Running PostgreSQL, ClickHouse, Meilisearch, Redis, Kafka, and Debezium in production requires significant DevOps expertise. Each engine has its own backup strategy, monitoring requirements, upgrade procedures, and failure modes. This is not suitable for a small team building an MVP.

2. **Data consistency is eventually consistent.** The CDC pipeline introduces latency between a response being written to PostgreSQL and it appearing in ClickHouse, Meilisearch, and Redis. Under normal conditions this is sub-second, but during pipeline failures or high load, the lag can extend to seconds or minutes. The system must be designed to handle temporary inconsistency gracefully.

3. **GDPR erasure must propagate across all engines.** When a data deletion request arrives, PII must be removed from PostgreSQL, ClickHouse, Meilisearch, Redis, pgvector, and any cached/archived data. Each engine has different deletion semantics (ClickHouse mutations are expensive; Meilisearch deletions require document IDs; Redis keys must be enumerated). A coordinated deletion pipeline is required.

4. **Debugging spans multiple systems.** When a response appears in PostgreSQL but not in the dashboard (which reads from ClickHouse/Redis), the developer must trace through the CDC pipeline, Kafka consumer logs, and multiple engine logs to find the failure point. Distributed tracing (OpenTelemetry) is essential but adds yet more infrastructure.

5. **Cost at small scale.** For an MVP with a few thousand responses per month, this architecture is dramatically over-engineered. The infrastructure cost (5+ services, Kafka cluster) is $500-2,000/month minimum, compared to $50-100/month for a single PostgreSQL instance.

6. **Schema synchronisation burden.** The same data appears in different forms across PostgreSQL, ClickHouse, and Meilisearch. When the survey design schema evolves, the CDC consumers, ClickHouse table schemas, and Meilisearch index configurations must all be updated in coordination.

---

## Migration and Scaling Considerations

### Phase 1: MVP — PostgreSQL Only (0-100K responses)

Do NOT deploy this architecture for the MVP. Start with Suggestion 3 (hybrid relational + JSONB on PostgreSQL). The polyglot architecture should be introduced incrementally as specific bottlenecks emerge.

### Phase 2: Add ClickHouse for Analytics (100K-10M responses)

When dashboard queries on PostgreSQL start exceeding acceptable latency (typically around 100K-500K responses in the JSONB model):

1. Deploy ClickHouse (single node or ClickHouse Cloud)
2. Build the response/answer flattening pipeline (can start as a cron job reading from PostgreSQL, no Kafka needed yet)
3. Redirect analytical queries (cross-tabulation, NPS trending, completion rates) to ClickHouse
4. Keep PostgreSQL as the write path and source of truth

### Phase 3: Add Meilisearch + Redis (10M-100M responses)

When search and real-time features become bottlenecks:

1. Deploy Meilisearch for survey search and open-text response search
2. Move live counters and session state to Redis
3. Introduce Kafka + Debezium for CDC (replacing the cron-based sync)
4. Add pgvector for semantic search on open-text responses

### Phase 4: Full Polyglot at Scale (100M+ responses)

1. ClickHouse cluster with multiple shards
2. Kafka cluster with replication
3. Multi-region deployment with data residency (EU ClickHouse + EU PostgreSQL)
4. S3 archival for old response data (Parquet format readable by ClickHouse external tables)
5. Add monitoring: Grafana dashboards for all engines, OpenTelemetry for distributed tracing, PagerDuty alerts for CDC pipeline lag

### Self-Hosted Deployment (Docker Compose)

For self-hosted customers who need data sovereignty, provide a simplified deployment:

```yaml
# docker-compose.yml (simplified — production would use Kubernetes)
services:
  postgres:
    image: postgres:17
    volumes: [pgdata:/var/lib/postgresql/data]
    environment:
      POSTGRES_DB: survey_platform

  clickhouse:
    image: clickhouse/clickhouse-server:24
    volumes: [chdata:/var/lib/clickhouse]

  meilisearch:
    image: getmeili/meilisearch:v1
    volumes: [msdata:/meili_data]

  redis:
    image: redis/redis-stack:7
    volumes: [redisdata:/data]

  kafka:
    image: confluentinc/cp-kafka:7.6
    # ... Kafka configuration

  debezium:
    image: debezium/connect:2.6
    # ... Debezium connector configuration

  api:
    image: survey-platform/api:latest
    depends_on: [postgres, clickhouse, meilisearch, redis, kafka]

volumes:
  pgdata:
  chdata:
  msdata:
  redisdata:
```

### Data Residency Implementation

For GDPR and data sovereignty requirements, each storage engine must respect data residency:

1. **PostgreSQL:** Separate instances per region (us.postgres, eu.postgres)
2. **ClickHouse:** Partition by organization_id; route queries to the correct regional cluster
3. **Meilisearch:** Separate indexes per region with tenant isolation
4. **Redis:** Separate Redis instances per region
5. **Kafka:** Regional topics (survey.us.cdc.responses, survey.eu.cdc.responses)

The application layer routes requests to the correct regional infrastructure based on the organization's `data_residency` setting.
