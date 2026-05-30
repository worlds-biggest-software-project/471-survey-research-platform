# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Survey & Research Platform (Candidate #471)
> Generated: 2026-05-26

## Overview

This model applies Event Sourcing and Command Query Responsibility Segregation (CQRS) to the survey platform domain. Every state change — a survey being created, a question being added, a response being submitted, a panel member being recruited — is captured as an immutable event in an append-only event store. The current state of any aggregate is derived by replaying its event stream. Separate read-model projections (materialized in PostgreSQL tables, Redis, or a search index) are optimised for specific query patterns: the survey builder UI, the response dashboard, the statistical analysis engine, and the compliance audit trail.

This approach is particularly well-suited to a survey research platform because:

1. **Survey design is inherently versioned.** Researchers need to know exactly what questions, logic, and options were active when each response was collected. Event sourcing provides a complete, tamper-proof history of every design change.
2. **Response collection is append-only by nature.** Responses are submitted and should never be silently modified. An event log makes this guarantee structural rather than policy-based.
3. **Compliance and auditability are first-class requirements.** GDPR, ISO 20252, and institutional review boards require full provenance of data collection. The event store IS the audit log.
4. **Analytics queries have fundamentally different access patterns from design-time writes.** CQRS lets you optimise read models for dashboard aggregation without compromising the write model's consistency.

## Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Event Store | PostgreSQL 17 (append-only tables with SERIALIZABLE isolation) | Familiar operations, transactional guarantees, partitioning support |
| Event Bus | Apache Kafka or NATS JetStream | Durable, ordered event streaming for projection rebuilds and cross-service communication |
| Write Model (Command Side) | Node.js / TypeScript with custom aggregate framework | Lightweight aggregate roots, command handlers, event emitters |
| Read Model (Query Side) | PostgreSQL (denormalized projections) + Redis (real-time counters) | Optimised materialized views for each UI screen |
| Search Projection | Elasticsearch or Meilisearch | Full-text search over survey content and open-text responses |
| Analytics Projection | ClickHouse or DuckDB | Columnar store for cross-tabulation, statistical analysis, and reporting |
| Snapshotting | PostgreSQL JSONB | Periodic aggregate snapshots to avoid replaying full event streams |

---

## Event Store Schema

### Core Event Store Table

```sql
-- ============================================================
-- EVENT STORE — THE SINGLE SOURCE OF TRUTH
-- ============================================================

CREATE TABLE event_store (
    -- Event identity
    event_id            UUID NOT NULL DEFAULT gen_random_uuid(),
    sequence_number     BIGSERIAL NOT NULL,  -- global ordering

    -- Aggregate identity
    aggregate_type      VARCHAR(100) NOT NULL,  -- e.g. 'Survey', 'Panel', 'Response'
    aggregate_id        UUID NOT NULL,
    aggregate_version   INTEGER NOT NULL,       -- per-aggregate version for optimistic concurrency

    -- Event metadata
    event_type          VARCHAR(200) NOT NULL,  -- e.g. 'SurveyCreated', 'QuestionAdded'
    event_data          JSONB NOT NULL,          -- the event payload
    event_metadata      JSONB NOT NULL DEFAULT '{}'::jsonb,  -- causation_id, correlation_id, user_id, ip, etc.

    -- Timestamps
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Constraints
    PRIMARY KEY (aggregate_type, aggregate_id, aggregate_version),
    UNIQUE (event_id),
    UNIQUE (sequence_number)
) PARTITION BY RANGE (created_at);

-- Quarterly partitions
CREATE TABLE event_store_2026_q1 PARTITION OF event_store
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE event_store_2026_q2 PARTITION OF event_store
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE event_store_2026_q3 PARTITION OF event_store
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE event_store_2026_q4 PARTITION OF event_store
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

-- Indexes for common access patterns
CREATE INDEX idx_event_store_aggregate ON event_store (aggregate_type, aggregate_id, aggregate_version);
CREATE INDEX idx_event_store_type ON event_store (event_type, created_at);
CREATE INDEX idx_event_store_sequence ON event_store (sequence_number);
CREATE INDEX idx_event_store_created ON event_store (created_at);
CREATE INDEX idx_event_store_correlation ON event_store USING GIN ((event_metadata -> 'correlation_id'));

-- ============================================================
-- AGGREGATE SNAPSHOTS
-- ============================================================

CREATE TABLE aggregate_snapshots (
    aggregate_type      VARCHAR(100) NOT NULL,
    aggregate_id        UUID NOT NULL,
    aggregate_version   INTEGER NOT NULL,
    snapshot_data       JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);

-- ============================================================
-- PROJECTION CHECKPOINTS (tracks which events each projection has consumed)
-- ============================================================

CREATE TABLE projection_checkpoints (
    projection_name     VARCHAR(200) NOT NULL PRIMARY KEY,
    last_sequence_number BIGINT NOT NULL DEFAULT 0,
    last_processed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    status              VARCHAR(30) NOT NULL DEFAULT 'running'
                        CHECK (status IN ('running', 'paused', 'rebuilding', 'error')),
    error_message       TEXT,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- DEAD LETTER QUEUE (events that failed projection processing)
-- ============================================================

CREATE TABLE projection_dead_letters (
    id                  BIGSERIAL PRIMARY KEY,
    projection_name     VARCHAR(200) NOT NULL,
    event_id            UUID NOT NULL,
    sequence_number     BIGINT NOT NULL,
    error_message       TEXT NOT NULL,
    retry_count         INTEGER NOT NULL DEFAULT 0,
    max_retries         INTEGER NOT NULL DEFAULT 5,
    next_retry_at       TIMESTAMPTZ,
    resolved            BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dead_letters_projection ON projection_dead_letters (projection_name, resolved);
```

---

## Aggregate Definitions & Event Types

### Survey Aggregate

The Survey aggregate governs the entire lifecycle of a survey from creation through design, publication, and closure.

```typescript
// Aggregate Root: Survey
// Stream: Survey-{surveyId}

interface SurveyAggregate {
  id: string;
  organizationId: string;
  version: number;
  status: 'draft' | 'scheduled' | 'active' | 'paused' | 'closed' | 'archived';
  title: string;
  pages: SurveyPage[];
  questions: Question[];
  logicRules: LogicRule[];
  settings: SurveySettings;
}
```

**Events:**

```sql
-- Survey lifecycle events
-- All stored in event_store with aggregate_type = 'Survey'

-- event_type: 'SurveyCreated'
-- event_data example:
{
    "surveyId": "uuid",
    "organizationId": "uuid",
    "workspaceId": "uuid",
    "title": "Customer Satisfaction Q2 2026",
    "surveyType": "standard",
    "defaultLanguage": "en",
    "createdBy": "uuid"
}

-- event_type: 'SurveyTitleUpdated'
{
    "surveyId": "uuid",
    "previousTitle": "Customer Satisfaction Q2 2026",
    "newTitle": "Customer Satisfaction Survey — Q2 2026",
    "updatedBy": "uuid"
}

-- event_type: 'SurveyPageAdded'
{
    "surveyId": "uuid",
    "pageId": "uuid",
    "title": "Demographics",
    "sortOrder": 1,
    "addedBy": "uuid"
}

-- event_type: 'QuestionAdded'
{
    "surveyId": "uuid",
    "pageId": "uuid",
    "questionId": "uuid",
    "questionType": "likert",
    "questionCode": "Q1",
    "title": "How satisfied are you with our product?",
    "isRequired": true,
    "sortOrder": 0,
    "options": [
        {"id": "uuid", "label": "Very Dissatisfied", "value": "1", "sortOrder": 0},
        {"id": "uuid", "label": "Dissatisfied", "value": "2", "sortOrder": 1},
        {"id": "uuid", "label": "Neutral", "value": "3", "sortOrder": 2},
        {"id": "uuid", "label": "Satisfied", "value": "4", "sortOrder": 3},
        {"id": "uuid", "label": "Very Satisfied", "value": "5", "sortOrder": 4}
    ],
    "addedBy": "uuid"
}

-- event_type: 'QuestionUpdated'
{
    "surveyId": "uuid",
    "questionId": "uuid",
    "changes": {
        "title": {"from": "How satisfied are you?", "to": "Overall, how satisfied are you with our product?"},
        "isRequired": {"from": false, "to": true}
    },
    "updatedBy": "uuid"
}

-- event_type: 'QuestionRemoved'
{
    "surveyId": "uuid",
    "questionId": "uuid",
    "questionSnapshot": { /* full question state at removal time */ },
    "removedBy": "uuid"
}

-- event_type: 'LogicRuleAdded'
{
    "surveyId": "uuid",
    "ruleId": "uuid",
    "sourceQuestionId": "uuid",
    "ruleType": "skip_to",
    "targetPageId": "uuid",
    "conditionOperator": "equals",
    "conditionValue": "5",
    "addedBy": "uuid"
}

-- event_type: 'SurveyTranslationAdded'
{
    "surveyId": "uuid",
    "language": "fr",
    "title": "Enquete de satisfaction client — T2 2026",
    "isRtl": false,
    "addedBy": "uuid"
}

-- event_type: 'SurveyPublished'
{
    "surveyId": "uuid",
    "publishedBy": "uuid",
    "designSnapshot": { /* complete survey design at publish time */ },
    "scheduledStart": "2026-06-01T00:00:00Z",
    "scheduledEnd": "2026-06-30T23:59:59Z"
}

-- event_type: 'SurveyClosed'
{
    "surveyId": "uuid",
    "closedBy": "uuid",
    "reason": "quota_reached",
    "totalResponses": 1247
}

-- event_type: 'SurveyArchived'
{
    "surveyId": "uuid",
    "archivedBy": "uuid"
}

-- event_type: 'ConjointDesignCreated'
{
    "surveyId": "uuid",
    "designId": "uuid",
    "designType": "choice_based",
    "numTasks": 10,
    "profilesPerTask": 3,
    "attributes": [
        {
            "id": "uuid",
            "name": "Price",
            "levels": [
                {"id": "uuid", "label": "$9.99"},
                {"id": "uuid", "label": "$14.99"},
                {"id": "uuid", "label": "$19.99"}
            ]
        },
        {
            "id": "uuid",
            "name": "Brand",
            "levels": [
                {"id": "uuid", "label": "Brand A"},
                {"id": "uuid", "label": "Brand B"},
                {"id": "uuid", "label": "Brand C"}
            ]
        }
    ],
    "createdBy": "uuid"
}

-- event_type: 'MaxDiffDesignCreated'
{
    "surveyId": "uuid",
    "designId": "uuid",
    "numSets": 12,
    "itemsPerSet": 5,
    "items": [
        {"id": "uuid", "label": "Easy to use"},
        {"id": "uuid", "label": "Fast performance"},
        {"id": "uuid", "label": "Low price"}
    ],
    "createdBy": "uuid"
}
```

### Response Aggregate

Each survey response is its own aggregate, ensuring that concurrent respondents do not conflict with one another.

```sql
-- Response lifecycle events
-- aggregate_type = 'Response'

-- event_type: 'ResponseStarted'
{
    "responseId": "uuid",
    "surveyId": "uuid",
    "channelId": "uuid",
    "respondentId": "uuid",
    "externalUserId": "ext-user-123",
    "sessionId": "sess-abc-123",
    "language": "en",
    "ipAddress": "203.0.113.42",
    "userAgent": "Mozilla/5.0 ...",
    "geoCountry": "AU",
    "geoRegion": "NSW",
    "surveyVersionAtStart": 14,
    "startedAt": "2026-06-15T10:30:00Z"
}

-- event_type: 'AnswerSubmitted'
{
    "responseId": "uuid",
    "questionId": "uuid",
    "questionType": "likert",
    "answer": {
        "selectedOptionId": "uuid",
        "numericValue": 4
    },
    "durationMs": 3200,
    "pageNumber": 1,
    "submittedAt": "2026-06-15T10:30:45Z"
}

-- event_type: 'AnswerUpdated' (respondent went back and changed answer)
{
    "responseId": "uuid",
    "questionId": "uuid",
    "previousAnswer": {
        "selectedOptionId": "uuid",
        "numericValue": 4
    },
    "newAnswer": {
        "selectedOptionId": "uuid",
        "numericValue": 5
    },
    "updatedAt": "2026-06-15T10:35:12Z"
}

-- event_type: 'OpenTextAnswerSubmitted'
{
    "responseId": "uuid",
    "questionId": "uuid",
    "questionType": "text_long",
    "answer": {
        "textValue": "The product is generally good but the onboarding process was confusing."
    },
    "durationMs": 15400,
    "submittedAt": "2026-06-15T10:31:22Z"
}

-- event_type: 'MatrixAnswerSubmitted'
{
    "responseId": "uuid",
    "questionId": "uuid",
    "questionType": "likert_matrix",
    "answer": {
        "rows": [
            {"matrixRowId": "uuid", "selectedOptionId": "uuid", "numericValue": 4},
            {"matrixRowId": "uuid", "selectedOptionId": "uuid", "numericValue": 3},
            {"matrixRowId": "uuid", "selectedOptionId": "uuid", "numericValue": 5}
        ]
    },
    "durationMs": 8700,
    "submittedAt": "2026-06-15T10:32:05Z"
}

-- event_type: 'RankingAnswerSubmitted'
{
    "responseId": "uuid",
    "questionId": "uuid",
    "questionType": "ranking",
    "answer": {
        "rankedOptions": [
            {"optionId": "uuid", "rank": 1},
            {"optionId": "uuid", "rank": 2},
            {"optionId": "uuid", "rank": 3}
        ]
    },
    "durationMs": 6100,
    "submittedAt": "2026-06-15T10:33:15Z"
}

-- event_type: 'ConjointChoiceMade'
{
    "responseId": "uuid",
    "designId": "uuid",
    "taskId": "uuid",
    "taskNumber": 3,
    "chosenProfileId": "uuid",
    "profiles": [
        {"profileId": "uuid", "attributes": {"Price": "$9.99", "Brand": "Brand A"}},
        {"profileId": "uuid", "attributes": {"Price": "$19.99", "Brand": "Brand B"}},
        {"profileId": "uuid", "attributes": {"Price": "$14.99", "Brand": "Brand C"}}
    ],
    "responseTimeMs": 4200,
    "submittedAt": "2026-06-15T10:34:00Z"
}

-- event_type: 'MaxDiffChoiceMade'
{
    "responseId": "uuid",
    "designId": "uuid",
    "setId": "uuid",
    "setNumber": 5,
    "items": ["uuid1", "uuid2", "uuid3", "uuid4", "uuid5"],
    "bestItemId": "uuid2",
    "worstItemId": "uuid4",
    "responseTimeMs": 3800,
    "submittedAt": "2026-06-15T10:34:30Z"
}

-- event_type: 'ResponseCompleted'
{
    "responseId": "uuid",
    "surveyId": "uuid",
    "totalDurationSeconds": 287,
    "questionsAnswered": 24,
    "questionsSkipped": 2,
    "completedAt": "2026-06-15T10:35:47Z"
}

-- event_type: 'ResponseScreenedOut'
{
    "responseId": "uuid",
    "surveyId": "uuid",
    "quotaId": "uuid",
    "screeningQuestionId": "uuid",
    "reason": "quota_full",
    "screenedOutAt": "2026-06-15T10:31:00Z"
}

-- event_type: 'ResponseAbandoned' (system-detected, e.g. no activity for 30 min)
{
    "responseId": "uuid",
    "surveyId": "uuid",
    "lastActivityAt": "2026-06-15T10:31:22Z",
    "lastPageViewed": 2,
    "abandonedAt": "2026-06-15T11:01:22Z"
}
```

### Panel Aggregate

```sql
-- Panel lifecycle events
-- aggregate_type = 'Panel'

-- event_type: 'PanelCreated'
{
    "panelId": "uuid",
    "organizationId": "uuid",
    "name": "Customer Advisory Panel 2026",
    "panelType": "internal",
    "createdBy": "uuid"
}

-- event_type: 'RespondentRecruited'
{
    "panelId": "uuid",
    "respondentId": "uuid",
    "email": "respondent@example.com",
    "demographics": {
        "gender": "female",
        "birthYear": 1988,
        "country": "US",
        "region": "California"
    },
    "consentGiven": true,
    "consentText": "I agree to participate in research surveys...",
    "recruitedAt": "2026-03-15T09:00:00Z"
}

-- event_type: 'RespondentOptedOut'
{
    "panelId": "uuid",
    "respondentId": "uuid",
    "reason": "too_many_surveys",
    "optedOutAt": "2026-05-20T14:00:00Z"
}

-- event_type: 'RespondentDemographicsUpdated'
{
    "panelId": "uuid",
    "respondentId": "uuid",
    "changes": {
        "city": {"from": "San Francisco", "to": "Oakland"},
        "occupation": {"from": "Engineer", "to": "Senior Engineer"}
    },
    "updatedAt": "2026-04-10T11:00:00Z"
}
```

### Organization & User Aggregates

```sql
-- Organization lifecycle events
-- aggregate_type = 'Organization'

-- event_type: 'OrganizationCreated'
{
    "organizationId": "uuid",
    "name": "Acme Research Co",
    "slug": "acme-research",
    "planTier": "professional",
    "dataResidency": "us",
    "createdBy": "uuid"
}

-- event_type: 'MemberInvited'
{
    "organizationId": "uuid",
    "userId": "uuid",
    "email": "analyst@acme.com",
    "role": "analyst",
    "invitedBy": "uuid"
}

-- event_type: 'MemberRoleChanged'
{
    "organizationId": "uuid",
    "userId": "uuid",
    "previousRole": "member",
    "newRole": "editor",
    "changedBy": "uuid"
}

-- event_type: 'PlanUpgraded'
{
    "organizationId": "uuid",
    "previousTier": "starter",
    "newTier": "professional",
    "effectiveAt": "2026-06-01T00:00:00Z"
}
```

### Analysis & Reporting Events

```sql
-- aggregate_type = 'Analysis'

-- event_type: 'TextAnalysisRequested'
{
    "analysisId": "uuid",
    "surveyId": "uuid",
    "questionId": "uuid",
    "responseCount": 1247,
    "modelVersion": "claude-sonnet-4-5-20250514",
    "requestedBy": "uuid"
}

-- event_type: 'TextThemesExtracted'
{
    "analysisId": "uuid",
    "themes": [
        {"themeId": "uuid", "label": "Onboarding confusion", "sentiment": "negative", "count": 187, "percentage": 15.0},
        {"themeId": "uuid", "label": "Product quality praise", "sentiment": "positive", "count": 423, "percentage": 33.9},
        {"themeId": "uuid", "label": "Pricing concerns", "sentiment": "negative", "count": 98, "percentage": 7.9}
    ],
    "processingTimeMs": 45000
}

-- event_type: 'StatisticalAnalysisCompleted'
{
    "analysisId": "uuid",
    "analysisType": "cross_tabulation",
    "surveyId": "uuid",
    "parameters": {
        "rowQuestion": "uuid",
        "columnQuestion": "uuid",
        "weightVariable": "uuid"
    },
    "results": {
        "chiSquare": 23.45,
        "pValue": 0.0001,
        "degreesOfFreedom": 4,
        "cellCounts": [[120, 85, 45], [30, 155, 90]]
    }
}

-- event_type: 'ReportGenerated'
{
    "reportId": "uuid",
    "surveyId": "uuid",
    "reportType": "executive_summary",
    "title": "Customer Satisfaction Q2 2026 — Executive Summary",
    "generatedBy": "ai",
    "modelUsed": "claude-sonnet-4-5-20250514"
}
```

---

## Read Model Projections

Each projection consumes events from the event store and maintains a denormalized view optimised for specific query patterns.

### Projection 1: Survey Builder View

```sql
-- Denormalized survey structure for the builder UI
-- Rebuilt from: SurveyCreated, SurveyTitleUpdated, SurveyPageAdded, QuestionAdded,
--               QuestionUpdated, QuestionRemoved, LogicRuleAdded, etc.

CREATE TABLE read_survey_builder (
    survey_id           UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    workspace_id        UUID,
    title               VARCHAR(500) NOT NULL,
    status              VARCHAR(30) NOT NULL,
    survey_type         VARCHAR(50) NOT NULL,
    default_language    VARCHAR(10) NOT NULL,
    settings            JSONB NOT NULL DEFAULT '{}',
    version             INTEGER NOT NULL,
    
    -- Denormalized full survey structure as JSON for fast loading
    pages_json          JSONB NOT NULL DEFAULT '[]',  -- [{id, title, sortOrder, questions: [{id, type, title, options, ...}]}]
    logic_rules_json    JSONB NOT NULL DEFAULT '[]',
    translations_json   JSONB NOT NULL DEFAULT '{}',  -- {fr: {title, questions: {...}}, de: {...}}
    
    question_count      INTEGER NOT NULL DEFAULT 0,
    page_count          INTEGER NOT NULL DEFAULT 0,
    
    created_by          UUID NOT NULL,
    published_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_read_survey_builder_org ON read_survey_builder (organization_id);
CREATE INDEX idx_read_survey_builder_status ON read_survey_builder (status);
```

### Projection 2: Survey Listing View

```sql
-- Lightweight list view for dashboard survey cards
CREATE TABLE read_survey_listing (
    survey_id           UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    workspace_id        UUID,
    title               VARCHAR(500) NOT NULL,
    status              VARCHAR(30) NOT NULL,
    survey_type         VARCHAR(50) NOT NULL,
    question_count      INTEGER NOT NULL DEFAULT 0,
    response_count      INTEGER NOT NULL DEFAULT 0,
    completion_rate     DECIMAL(5,2),
    average_duration    INTEGER,  -- seconds
    created_by          UUID NOT NULL,
    creator_name        VARCHAR(255),
    starts_at           TIMESTAMPTZ,
    closes_at           TIMESTAMPTZ,
    published_at        TIMESTAMPTZ,
    last_response_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_read_survey_listing_org ON read_survey_listing (organization_id, status);
CREATE INDEX idx_read_survey_listing_created ON read_survey_listing (created_at DESC);
```

### Projection 3: Real-Time Response Dashboard

```sql
-- Aggregated response statistics per survey, updated in near-real-time
CREATE TABLE read_response_dashboard (
    survey_id           UUID NOT NULL,
    time_bucket         TIMESTAMPTZ NOT NULL,  -- hourly buckets
    total_started       INTEGER NOT NULL DEFAULT 0,
    total_completed     INTEGER NOT NULL DEFAULT 0,
    total_abandoned     INTEGER NOT NULL DEFAULT 0,
    total_screened_out  INTEGER NOT NULL DEFAULT 0,
    avg_duration_seconds INTEGER,
    
    -- Breakdown by channel
    channel_counts      JSONB NOT NULL DEFAULT '{}',  -- {"web_link": 45, "email": 120, ...}
    
    -- Breakdown by geography
    geo_counts          JSONB NOT NULL DEFAULT '{}',  -- {"US": 300, "GB": 45, ...}
    
    -- Breakdown by language
    language_counts     JSONB NOT NULL DEFAULT '{}',  -- {"en": 400, "fr": 50, ...}
    
    PRIMARY KEY (survey_id, time_bucket)
);

CREATE INDEX idx_read_dashboard_survey ON read_response_dashboard (survey_id, time_bucket DESC);

-- Per-question aggregation for live results
CREATE TABLE read_question_results (
    survey_id           UUID NOT NULL,
    question_id         UUID NOT NULL,
    question_type       VARCHAR(50) NOT NULL,
    question_title      TEXT NOT NULL,
    total_answers       INTEGER NOT NULL DEFAULT 0,
    
    -- For choice questions: option distribution
    option_counts       JSONB NOT NULL DEFAULT '{}',  -- {"optionId1": 120, "optionId2": 85, ...}
    option_percentages  JSONB NOT NULL DEFAULT '{}',
    
    -- For numeric questions: statistics
    numeric_mean        DECIMAL(15,4),
    numeric_median      DECIMAL(15,4),
    numeric_std_dev     DECIMAL(15,4),
    numeric_min         DECIMAL(15,4),
    numeric_max         DECIMAL(15,4),
    numeric_histogram   JSONB,  -- [{"bucket": "1-2", "count": 30}, ...]
    
    -- For NPS
    nps_score           DECIMAL(5,2),
    nps_promoters       INTEGER DEFAULT 0,
    nps_passives        INTEGER DEFAULT 0,
    nps_detractors      INTEGER DEFAULT 0,
    
    -- For text questions: recent examples (not aggregatable by nature)
    recent_text_samples JSONB DEFAULT '[]',
    
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (survey_id, question_id)
);

CREATE INDEX idx_read_question_results_survey ON read_question_results (survey_id);
```

### Projection 4: Panel Management View

```sql
-- Denormalized panel member list for management UI and quota targeting
CREATE TABLE read_panel_members (
    respondent_id       UUID PRIMARY KEY,
    panel_id            UUID NOT NULL,
    organization_id     UUID NOT NULL,
    email               VARCHAR(255),
    full_name           VARCHAR(255),
    gender              VARCHAR(20),
    birth_year          INTEGER,
    age_bracket         VARCHAR(20),  -- computed: "25-34", "35-44", etc.
    country             VARCHAR(3),
    region              VARCHAR(100),
    language            VARCHAR(10),
    
    -- Engagement metrics
    consent_status      VARCHAR(20) NOT NULL,
    total_invitations   INTEGER NOT NULL DEFAULT 0,
    total_completions   INTEGER NOT NULL DEFAULT 0,
    response_rate       DECIMAL(5,2),
    quality_score       DECIMAL(5,2),
    last_activity_at    TIMESTAMPTZ,
    
    -- Custom demographics as flat JSONB
    custom_attributes   JSONB DEFAULT '{}',
    
    created_at          TIMESTAMPTZ NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_read_panel_org ON read_panel_members (organization_id, panel_id);
CREATE INDEX idx_read_panel_country ON read_panel_members (country);
CREATE INDEX idx_read_panel_quality ON read_panel_members (quality_score DESC);
CREATE INDEX idx_read_panel_custom ON read_panel_members USING GIN (custom_attributes);
```

### Projection 5: Compliance & Audit View

```sql
-- Compliance-focused view reconstructed entirely from events
CREATE TABLE read_audit_trail (
    id                  BIGSERIAL PRIMARY KEY,
    event_id            UUID NOT NULL,
    event_type          VARCHAR(200) NOT NULL,
    aggregate_type      VARCHAR(100) NOT NULL,
    aggregate_id        UUID NOT NULL,
    organization_id     UUID NOT NULL,
    user_id             UUID,
    user_email          VARCHAR(255),
    action_summary      TEXT NOT NULL,  -- human-readable: "User alice@acme.com added question Q5 to survey 'Customer Sat Q2'"
    ip_address          INET,
    data_categories     TEXT[],  -- ['pii', 'demographics', 'health'] for GDPR classification
    created_at          TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE read_audit_trail_2026_q1 PARTITION OF read_audit_trail
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE read_audit_trail_2026_q2 PARTITION OF read_audit_trail
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE read_audit_trail_2026_q3 PARTITION OF read_audit_trail
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE read_audit_trail_2026_q4 PARTITION OF read_audit_trail
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_read_audit_org ON read_audit_trail (organization_id, created_at DESC);
CREATE INDEX idx_read_audit_aggregate ON read_audit_trail (aggregate_type, aggregate_id);
CREATE INDEX idx_read_audit_user ON read_audit_trail (user_id);

-- GDPR consent ledger — immutable record derived from consent events
CREATE TABLE read_consent_ledger (
    id                  BIGSERIAL PRIMARY KEY,
    respondent_id       UUID NOT NULL,
    consent_type        VARCHAR(50) NOT NULL,
    consent_given       BOOLEAN NOT NULL,
    consent_text        TEXT NOT NULL,
    legal_basis         VARCHAR(50) NOT NULL,
    ip_address          INET,
    event_id            UUID NOT NULL,  -- traceability back to source event
    recorded_at         TIMESTAMPTZ NOT NULL,
    superseded_at       TIMESTAMPTZ  -- set when a newer consent event for the same type arrives
);

CREATE INDEX idx_consent_ledger_respondent ON read_consent_ledger (respondent_id, consent_type);
```

### Projection 6: Analytics / Statistical Engine View

```sql
-- Flat, denormalized response data for statistical analysis
-- Optimised for ClickHouse or DuckDB, but can be PostgreSQL for smaller deployments

CREATE TABLE read_analytics_responses (
    response_id         UUID NOT NULL,
    survey_id           UUID NOT NULL,
    respondent_id       UUID,
    channel_type        VARCHAR(30),
    status              VARCHAR(30) NOT NULL,
    language            VARCHAR(10),
    geo_country         VARCHAR(3),
    geo_region          VARCHAR(100),
    duration_seconds    INTEGER,
    weight              DECIMAL(10,6),
    
    -- Respondent demographics (denormalized from panel)
    respondent_gender   VARCHAR(20),
    respondent_age_bracket VARCHAR(20),
    respondent_country  VARCHAR(3),
    
    started_at          TIMESTAMPTZ NOT NULL,
    completed_at        TIMESTAMPTZ,

    PRIMARY KEY (survey_id, response_id)
);

-- Flat answer rows for cross-tabulation and regression
CREATE TABLE read_analytics_answers (
    response_id         UUID NOT NULL,
    survey_id           UUID NOT NULL,
    question_id         UUID NOT NULL,
    question_code       VARCHAR(50),
    question_type       VARCHAR(50) NOT NULL,
    
    -- Denormalized answer values (one populated per row)
    selected_option_id  UUID,
    selected_option_label TEXT,
    text_value          TEXT,
    numeric_value       DECIMAL(15,4),
    array_values        TEXT[],
    rank_position       INTEGER,
    
    duration_ms         INTEGER,
    answered_at         TIMESTAMPTZ,
    
    PRIMARY KEY (survey_id, response_id, question_id)
);

CREATE INDEX idx_analytics_answers_question ON read_analytics_answers (survey_id, question_id);
CREATE INDEX idx_analytics_answers_numeric ON read_analytics_answers (question_id, numeric_value)
    WHERE numeric_value IS NOT NULL;
```

---

## Command Handling Flow

```
                                    ┌─────────────────────┐
                                    │   API Gateway /      │
                                    │   GraphQL / REST     │
                                    └─────────┬───────────┘
                                              │
                              ┌───────────────┴───────────────┐
                              │                               │
                    ┌─────────▼─────────┐          ┌──────────▼──────────┐
                    │  Command Handler   │          │   Query Handler      │
                    │  (Write Side)      │          │   (Read Side)        │
                    └─────────┬─────────┘          └──────────┬──────────┘
                              │                               │
                    ┌─────────▼─────────┐          ┌──────────▼──────────┐
                    │  Aggregate Root    │          │   Read Model         │
                    │  (Domain Logic)    │          │   (PostgreSQL /      │
                    │                    │          │    Redis / Search)   │
                    └─────────┬─────────┘          └─────────────────────┘
                              │                               ▲
                    ┌─────────▼─────────┐                     │
                    │  Event Store       │─── Event Bus ───────┘
                    │  (PostgreSQL)      │    (Kafka/NATS)
                    └───────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │  Projection        │
                    │  Workers           │
                    │  (one per view)    │
                    └───────────────────┘
```

### Command Processing Pseudocode

```typescript
// Command: AddQuestionToSurvey
async function handleAddQuestion(command: AddQuestionCommand): Promise<void> {
    // 1. Load aggregate from event store (or snapshot + newer events)
    const events = await eventStore.loadEvents('Survey', command.surveyId);
    const survey = SurveyAggregate.rehydrate(events);

    // 2. Validate business rules
    if (survey.status !== 'draft') {
        throw new Error('Cannot modify a published survey');
    }
    if (!survey.pages.find(p => p.id === command.pageId)) {
        throw new Error('Page not found');
    }

    // 3. Execute command — produces new event(s)
    const event = survey.addQuestion({
        pageId: command.pageId,
        questionType: command.questionType,
        title: command.title,
        options: command.options,
        addedBy: command.userId
    });

    // 4. Persist event with optimistic concurrency check
    await eventStore.appendEvent('Survey', command.surveyId, survey.version, event);

    // 5. Publish event to event bus for projection workers
    await eventBus.publish(event);
}
```

---

## Pros and Cons

### Pros

1. **Complete audit trail by design.** Every state change is recorded as an immutable event. GDPR compliance, ISO 20252 research provenance, and institutional review board requirements are satisfied structurally — the event store IS the audit log. No separate audit table is needed; no audit events can be "forgotten."

2. **Survey versioning is automatic.** When a researcher publishes a survey, the exact design at that moment is deterministically reconstructable from the event stream. When a response is collected, the survey version that was active at response time is always recoverable. This eliminates "survey definition drift" bugs that plague mutable-state survey platforms.

3. **Temporal queries are natural.** "What did this survey look like on June 15?" is answered by replaying events up to that timestamp. "How has the NPS score trended daily?" is answered by the time-bucketed dashboard projection. "When did this respondent give consent, and when did they withdraw it?" is answered by the consent event stream.

4. **Read models are independently optimisable.** The survey builder view, the response dashboard, the analytics engine, and the compliance audit trail each have their own projection table structure, indexing strategy, and potentially their own storage technology. A dashboard query never competes with a statistical analysis query.

5. **Horizontal scalability via event partitioning.** Events for different aggregates are independent. Survey A's events never conflict with Survey B's. Response aggregates are naturally isolated per respondent session. This enables sharding by aggregate ID without cross-shard transactions.

6. **Replay and rebuild safety net.** If a read model projection has a bug, you can fix the projection logic and replay all events from the beginning to rebuild a correct read model. No data is ever lost because the event store is the source of truth.

### Cons

1. **Significantly higher implementation complexity.** Event sourcing requires building aggregate roots, command handlers, event handlers, projection workers, snapshot management, dead letter queues, and projection rebuild tooling. This is substantially more engineering work than a CRUD model, especially for an MVP.

2. **Eventual consistency in read models.** After a respondent submits an answer, the real-time dashboard projection may take milliseconds to seconds to update. This is usually acceptable for dashboards but can confuse users who expect immediate consistency in the survey builder (e.g., "I just added a question — why doesn't it appear?"). Mitigations include read-your-own-writes patterns.

3. **Event schema evolution is hard.** Once events are persisted, they are immutable. If the structure of a `QuestionAdded` event needs to change (e.g., adding a new field), you must implement event upcasters that transform old event formats into new ones during replay. This adds ongoing maintenance burden.

4. **Debugging is less intuitive.** Instead of looking at the current state of a row in a database table, developers must replay an event stream to understand why the system is in a particular state. Tooling for event stream inspection and aggregate state visualisation must be built or adopted.

5. **Storage growth.** The event store grows monotonically. A survey with 1,000 edits stores 1,000 events, whereas a CRUD model stores only the current state. For response data (which is naturally append-only), this is not a significant overhead. For survey design data with many iterative edits, it can be substantial.

6. **Cross-aggregate queries require projections.** "Show me all surveys in this organization" cannot be answered from the event store alone (which is keyed by aggregate ID). It requires a read model projection. Every new query pattern potentially requires a new projection.

---

## Migration and Scaling Considerations

### Phase 1: MVP with Simplified Event Sourcing (0-10M events)

- Single PostgreSQL instance for both event store and read model projections
- In-process event dispatching (no Kafka/NATS yet) — events are written to the store and projections are updated synchronously in the same transaction
- Snapshot every 50 events per aggregate to keep replay fast
- 3-5 projection tables covering: survey listing, survey builder, response dashboard, panel members
- Estimated infrastructure: single PostgreSQL instance + Redis for caching

### Phase 2: Asynchronous Projections with Event Bus (10M-500M events)

- Introduce NATS JetStream or Kafka for durable event streaming
- Move projection workers to separate processes/containers
- Add ClickHouse or DuckDB as the analytics projection backend
- Implement projection rebuild tooling (replay from sequence 0 with progress tracking)
- Add event store partitioning by quarter
- Introduce Elasticsearch/Meilisearch projection for full-text search

### Phase 3: Multi-Region Event Sourcing (500M+ events)

- Shard event store by organization (each org's events on a dedicated partition/database)
- Multi-region deployment with event replication (Kafka MirrorMaker or NATS leaf nodes)
- Data residency enforcement: EU organization events stored only on EU nodes
- Archive old event partitions to S3 (Parquet format) with replay-on-demand
- Introduce event store compaction for design aggregates (compact many `QuestionUpdated` events into a single `SurveyDesignCompacted` event while retaining the originals in cold storage)

### Event Schema Versioning Strategy

1. Every event type has an explicit `schemaVersion` in its metadata
2. Event upcasters transform old event formats to current format during replay
3. New fields are always optional (backward compatible) — breaking changes require a new event type
4. Event catalog (a registry of all event types with their JSON schemas) is maintained as a versioned artifact

### GDPR Data Erasure in an Event-Sourced System

Event sourcing and "right to erasure" are fundamentally at tension. The recommended approach:

1. **Crypto-shredding:** Encrypt PII fields in events using a per-respondent encryption key. To "erase" a respondent, delete their encryption key — the events remain but the PII becomes unrecoverable.
2. **Tombstone events:** Emit a `RespondentDataErased` event that projections consume to remove PII from read models.
3. **Event store rewriting (last resort):** For regulatory requirements that demand physical deletion, implement a scheduled job that rewrites event store partitions with PII fields replaced by `[REDACTED]`. This is operationally complex and should be avoided if crypto-shredding suffices.
