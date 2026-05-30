# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL JSONB)

> Project: Survey & Research Platform (Candidate #471)
> Generated: 2026-05-26

## Overview

This model exploits PostgreSQL's dual nature as both a relational database and a document store. Core entities with stable, well-defined structures — organizations, users, panels, distribution channels — are stored in fully normalized relational tables. Entities with inherently variable or schema-flexible structures — survey definitions, question configurations, answer payloads, analysis results — use JSONB columns within relational tables.

The key insight for a survey platform is that **the structure of a survey is itself data**. Different question types have radically different configuration needs: a Likert scale has labelled anchor points, a conjoint design has attributes and levels, a matrix question has row headers and column headers, a slider has min/max/step values. In a purely normalized model, these variations require either many nullable columns (wasteful and confusing) or many specialised tables (complex to query). In a purely document model, you lose referential integrity and efficient aggregation. The hybrid approach gives you the best of both: relational structure where stability matters, document flexibility where variability is the norm.

This approach follows patterns used by production survey platforms including Formbricks (which stores survey definitions as JSONB in PostgreSQL via Prisma) and SurveyJS (which represents surveys as JSON schemas rendered by a client-side engine).

## Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Primary Database | PostgreSQL 17+ | Native JSONB with GIN indexing, partial indexes, jsonpath queries |
| ORM | Prisma 6.x (with Json field support) or Drizzle ORM | Type-safe JSONB field access with Zod validation |
| Validation | Zod / AJV (JSON Schema validator) | Runtime validation of JSONB payloads against versioned schemas |
| Migration Tool | Prisma Migrate | Schema-as-code with JSONB column support |
| Connection Pooling | PgBouncer | Transaction-mode pooling |
| Search | PostgreSQL full-text search + GIN on JSONB | Search within survey definitions and response text |
| Caching | Redis 7+ | Dashboard aggregation cache, session management |
| Analytics | DuckDB (embedded) or ClickHouse | Columnar analytics for cross-tabulation on large response sets |

---

## Complete Schema Definition

### 1. Relational Core: Organizations, Users, Workspaces

These entities are stable, well-understood, and queried with standard relational patterns. No JSONB needed.

```sql
-- ============================================================
-- ORGANIZATIONS (fully relational)
-- ============================================================

CREATE TABLE organizations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    slug                VARCHAR(100) NOT NULL UNIQUE,
    plan_tier           VARCHAR(50) NOT NULL DEFAULT 'free'
                        CHECK (plan_tier IN ('free', 'starter', 'professional', 'enterprise')),
    billing_email       VARCHAR(255),
    data_residency      VARCHAR(10) NOT NULL DEFAULT 'us'
                        CHECK (data_residency IN ('us', 'eu', 'ap', 'au')),
    gdpr_dpa_signed     BOOLEAN NOT NULL DEFAULT FALSE,
    max_responses_month INTEGER NOT NULL DEFAULT 1000,
    settings            JSONB NOT NULL DEFAULT '{}'::jsonb,  -- org-level feature flags, branding
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMPTZ
);

-- ============================================================
-- USERS (fully relational)
-- ============================================================

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               VARCHAR(255) NOT NULL UNIQUE,
    email_verified      BOOLEAN NOT NULL DEFAULT FALSE,
    password_hash       VARCHAR(255),
    full_name           VARCHAR(255) NOT NULL,
    avatar_url          TEXT,
    preferences         JSONB NOT NULL DEFAULT '{}'::jsonb,  -- UI preferences, notification settings
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMPTZ
);

-- ============================================================
-- ORGANIZATION MEMBERSHIPS (fully relational)
-- ============================================================

CREATE TABLE organization_memberships (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role                VARCHAR(50) NOT NULL DEFAULT 'member'
                        CHECK (role IN ('owner', 'admin', 'editor', 'analyst', 'member', 'viewer')),
    invited_by          UUID REFERENCES users(id),
    accepted_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, user_id)
);

CREATE INDEX idx_org_memberships_org ON organization_memberships (organization_id);
CREATE INDEX idx_org_memberships_user ON organization_memberships (user_id);

-- ============================================================
-- WORKSPACES (fully relational)
-- ============================================================

CREATE TABLE workspaces (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workspace_memberships (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id        UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role                VARCHAR(30) NOT NULL DEFAULT 'member',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, user_id)
);
```

### 2. Hybrid Core: Surveys with JSONB Design Document

This is where the hybrid approach shines. The `surveys` table has relational columns for everything you filter, sort, or aggregate on, plus a JSONB `design` column that contains the complete, arbitrarily structured survey definition.

```sql
-- ============================================================
-- SURVEYS — relational metadata + JSONB design document
-- ============================================================

CREATE TABLE surveys (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    workspace_id        UUID REFERENCES workspaces(id) ON DELETE SET NULL,

    -- Relational metadata: used for filtering, sorting, dashboard queries
    title               VARCHAR(500) NOT NULL,
    internal_name       VARCHAR(255),
    status              VARCHAR(30) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft', 'scheduled', 'active', 'paused', 'closed', 'archived')),
    survey_type         VARCHAR(50) NOT NULL DEFAULT 'standard'
                        CHECK (survey_type IN ('standard', 'nps', 'csat', 'ces', 'conjoint',
                                               'maxdiff', 'experiment', 'longitudinal')),
    default_language    VARCHAR(10) NOT NULL DEFAULT 'en',
    question_count      INTEGER NOT NULL DEFAULT 0,
    page_count          INTEGER NOT NULL DEFAULT 0,
    response_limit      INTEGER,
    starts_at           TIMESTAMPTZ,
    closes_at           TIMESTAMPTZ,
    estimated_duration  INTEGER,
    published_at        TIMESTAMPTZ,
    created_by          UUID NOT NULL REFERENCES users(id),

    -- ============================================================
    -- THE JSONB DESIGN DOCUMENT
    -- This contains the complete survey structure: pages, questions,
    -- options, logic rules, translations, scoring, and advanced
    -- research designs (conjoint, MaxDiff).
    -- ============================================================
    design              JSONB NOT NULL DEFAULT '{}'::jsonb,

    -- Design schema version for migration/validation
    design_schema_version INTEGER NOT NULL DEFAULT 1,

    -- Survey settings (variable by survey type)
    settings            JSONB NOT NULL DEFAULT '{}'::jsonb,

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_surveys_org ON surveys (organization_id);
CREATE INDEX idx_surveys_status ON surveys (status);
CREATE INDEX idx_surveys_type ON surveys (survey_type);
CREATE INDEX idx_surveys_workspace ON surveys (workspace_id);
CREATE INDEX idx_surveys_created ON surveys (created_at DESC);

-- GIN index for searching within the design document
CREATE INDEX idx_surveys_design ON surveys USING GIN (design jsonb_path_ops);

-- Partial index for active surveys (most common query)
CREATE INDEX idx_surveys_active ON surveys (organization_id, starts_at, closes_at)
    WHERE status = 'active';
```

#### The Design Document Schema

The `design` JSONB column follows a versioned schema validated at the application layer. Here is the complete structure:

```jsonc
// surveys.design JSONB schema (version 1)
{
    "schemaVersion": 1,
    "pages": [
        {
            "id": "page-uuid-1",
            "title": "Demographics",
            "description": "Tell us about yourself",
            "sortOrder": 0,
            "randomize": false,
            "visibilityCondition": null,
            "questions": [
                {
                    "id": "q-uuid-1",
                    "type": "single_choice",
                    "code": "Q1",
                    "title": "What is your age range?",
                    "description": null,
                    "isRequired": true,
                    "sortOrder": 0,
                    "randomizeOptions": false,
                    "allowOther": true,
                    "otherLabel": "Prefer to self-describe",
                    "options": [
                        {"id": "opt-1", "label": "18-24", "value": "18-24", "sortOrder": 0, "score": null},
                        {"id": "opt-2", "label": "25-34", "value": "25-34", "sortOrder": 1, "score": null},
                        {"id": "opt-3", "label": "35-44", "value": "35-44", "sortOrder": 2, "score": null},
                        {"id": "opt-4", "label": "45-54", "value": "45-54", "sortOrder": 3, "score": null},
                        {"id": "opt-5", "label": "55-64", "value": "55-64", "sortOrder": 4, "score": null},
                        {"id": "opt-6", "label": "65+",   "value": "65+",   "sortOrder": 5, "score": null}
                    ],
                    "validation": null,
                    "config": {}
                },
                {
                    "id": "q-uuid-2",
                    "type": "likert",
                    "code": "Q2",
                    "title": "How satisfied are you with our product?",
                    "description": "Please rate your overall satisfaction",
                    "isRequired": true,
                    "sortOrder": 1,
                    "options": [
                        {"id": "opt-10", "label": "Very Dissatisfied", "value": "1", "sortOrder": 0, "score": 1},
                        {"id": "opt-11", "label": "Dissatisfied",      "value": "2", "sortOrder": 1, "score": 2},
                        {"id": "opt-12", "label": "Neutral",           "value": "3", "sortOrder": 2, "score": 3},
                        {"id": "opt-13", "label": "Satisfied",         "value": "4", "sortOrder": 3, "score": 4},
                        {"id": "opt-14", "label": "Very Satisfied",    "value": "5", "sortOrder": 4, "score": 5}
                    ],
                    "config": {
                        "scaleType": "agreement",
                        "showLabels": true
                    }
                },
                {
                    "id": "q-uuid-3",
                    "type": "likert_matrix",
                    "code": "Q3",
                    "title": "Rate each aspect of our service",
                    "isRequired": true,
                    "sortOrder": 2,
                    "rows": [
                        {"id": "row-1", "label": "Customer Support", "sortOrder": 0},
                        {"id": "row-2", "label": "Product Quality", "sortOrder": 1},
                        {"id": "row-3", "label": "Value for Money", "sortOrder": 2},
                        {"id": "row-4", "label": "Ease of Use", "sortOrder": 3}
                    ],
                    "options": [
                        {"id": "col-1", "label": "Poor",      "value": "1", "sortOrder": 0, "score": 1},
                        {"id": "col-2", "label": "Fair",      "value": "2", "sortOrder": 1, "score": 2},
                        {"id": "col-3", "label": "Good",      "value": "3", "sortOrder": 2, "score": 3},
                        {"id": "col-4", "label": "Very Good", "value": "4", "sortOrder": 3, "score": 4},
                        {"id": "col-5", "label": "Excellent", "value": "5", "sortOrder": 4, "score": 5}
                    ],
                    "config": {
                        "randomizeRows": false,
                        "requireAllRows": true
                    }
                },
                {
                    "id": "q-uuid-4",
                    "type": "net_promoter_score",
                    "code": "Q4",
                    "title": "How likely are you to recommend us to a friend or colleague?",
                    "isRequired": true,
                    "sortOrder": 3,
                    "config": {
                        "lowLabel": "Not at all likely",
                        "highLabel": "Extremely likely",
                        "minValue": 0,
                        "maxValue": 10
                    }
                },
                {
                    "id": "q-uuid-5",
                    "type": "text_long",
                    "code": "Q5",
                    "title": "What could we improve?",
                    "isRequired": false,
                    "sortOrder": 4,
                    "config": {
                        "maxLength": 5000,
                        "placeholder": "Share your thoughts..."
                    }
                },
                {
                    "id": "q-uuid-6",
                    "type": "ranking",
                    "code": "Q6",
                    "title": "Rank these features by importance",
                    "isRequired": true,
                    "sortOrder": 5,
                    "options": [
                        {"id": "rank-1", "label": "Performance", "sortOrder": 0},
                        {"id": "rank-2", "label": "Price", "sortOrder": 1},
                        {"id": "rank-3", "label": "Design", "sortOrder": 2},
                        {"id": "rank-4", "label": "Support", "sortOrder": 3},
                        {"id": "rank-5", "label": "Reliability", "sortOrder": 4}
                    ],
                    "config": {
                        "maxRanks": 5,
                        "dragAndDrop": true
                    }
                },
                {
                    "id": "q-uuid-7",
                    "type": "slider",
                    "code": "Q7",
                    "title": "How much would you pay monthly for this service?",
                    "isRequired": false,
                    "sortOrder": 6,
                    "config": {
                        "minValue": 0,
                        "maxValue": 100,
                        "stepValue": 5,
                        "prefix": "$",
                        "showValue": true
                    }
                }
            ]
        }
    ],
    "logicRules": [
        {
            "id": "rule-uuid-1",
            "sourceQuestionId": "q-uuid-2",
            "ruleType": "show_question",
            "targetQuestionId": "q-uuid-5",
            "conditions": [
                {
                    "operator": "less_than",
                    "value": "3",
                    "connector": "AND"
                }
            ]
        }
    ],
    "translations": {
        "fr": {
            "surveyTitle": "Enquete de satisfaction client",
            "questions": {
                "q-uuid-1": {
                    "title": "Quelle est votre tranche d'age?",
                    "options": {
                        "opt-1": "18-24",
                        "opt-2": "25-34"
                    }
                }
            }
        }
    },
    "conjointDesign": null,
    "maxdiffDesign": null,
    "scoring": {
        "enabled": false,
        "rules": []
    }
}
```

#### Conjoint Design within the Design Document

```jsonc
// When survey_type = 'conjoint', the design.conjointDesign field is populated:
{
    "conjointDesign": {
        "designType": "choice_based",
        "numTasks": 10,
        "profilesPerTask": 3,
        "includeNone": true,
        "attributes": [
            {
                "id": "attr-1",
                "name": "Price",
                "levels": [
                    {"id": "lvl-1", "label": "$9.99"},
                    {"id": "lvl-2", "label": "$14.99"},
                    {"id": "lvl-3", "label": "$19.99"}
                ]
            },
            {
                "id": "attr-2",
                "name": "Brand",
                "levels": [
                    {"id": "lvl-4", "label": "Premium Brand"},
                    {"id": "lvl-5", "label": "Value Brand"},
                    {"id": "lvl-6", "label": "Store Brand"}
                ]
            },
            {
                "id": "attr-3",
                "name": "Warranty",
                "levels": [
                    {"id": "lvl-7", "label": "1 Year"},
                    {"id": "lvl-8", "label": "2 Years"},
                    {"id": "lvl-9", "label": "Lifetime"}
                ]
            }
        ],
        "tasks": [
            {
                "id": "task-1",
                "taskNumber": 1,
                "profiles": [
                    {
                        "id": "prof-1",
                        "profileNumber": 1,
                        "levels": {"attr-1": "lvl-1", "attr-2": "lvl-5", "attr-3": "lvl-9"}
                    },
                    {
                        "id": "prof-2",
                        "profileNumber": 2,
                        "levels": {"attr-1": "lvl-3", "attr-2": "lvl-4", "attr-3": "lvl-7"}
                    },
                    {
                        "id": "prof-3",
                        "profileNumber": 3,
                        "levels": {"attr-1": "lvl-2", "attr-2": "lvl-6", "attr-3": "lvl-8"}
                    }
                ]
            }
        ]
    }
}
```

### 3. Survey Versions (Snapshot on Publish)

```sql
-- ============================================================
-- SURVEY VERSIONS — frozen snapshot of design at publish time
-- ============================================================

CREATE TABLE survey_versions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    version_number      INTEGER NOT NULL,
    design_snapshot     JSONB NOT NULL,        -- frozen copy of surveys.design at publish time
    settings_snapshot   JSONB NOT NULL,        -- frozen copy of surveys.settings
    question_count      INTEGER NOT NULL,
    published_by        UUID NOT NULL REFERENCES users(id),
    published_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (survey_id, version_number)
);

CREATE INDEX idx_survey_versions_survey ON survey_versions (survey_id, version_number DESC);
```

### 4. Distribution Channels (Relational)

```sql
-- ============================================================
-- DISTRIBUTION CHANNELS (relational + settings JSONB)
-- ============================================================

CREATE TABLE distribution_channels (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    channel_type        VARCHAR(30) NOT NULL
                        CHECK (channel_type IN ('web_link', 'email', 'sms', 'qr_code',
                                                'in_app', 'api', 'social', 'embed')),
    name                VARCHAR(255) NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    unique_link_slug    VARCHAR(100) UNIQUE,
    response_count      INTEGER NOT NULL DEFAULT 0,

    -- Channel-specific configuration as JSONB (email templates, SMS config, trigger rules)
    channel_config      JSONB NOT NULL DEFAULT '{}'::jsonb,

    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dist_channels_survey ON distribution_channels (survey_id);
CREATE INDEX idx_dist_channels_slug ON distribution_channels (unique_link_slug);

-- ============================================================
-- EMAIL INVITATIONS (relational for tracking)
-- ============================================================

CREATE TABLE email_invitations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id          UUID NOT NULL REFERENCES distribution_channels(id) ON DELETE CASCADE,
    recipient_email     VARCHAR(255) NOT NULL,
    recipient_name      VARCHAR(255),
    token               VARCHAR(100) NOT NULL UNIQUE,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'sent', 'delivered', 'opened',
                                          'clicked', 'bounced', 'unsubscribed')),
    sent_at             TIMESTAMPTZ,
    opened_at           TIMESTAMPTZ,
    clicked_at          TIMESTAMPTZ,
    reminder_count      INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_email_invitations_channel ON email_invitations (channel_id);
CREATE INDEX idx_email_invitations_token ON email_invitations (token);

-- ============================================================
-- IN-APP TRIGGER RULES (relational + condition JSONB)
-- ============================================================

CREATE TABLE in_app_triggers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id          UUID NOT NULL REFERENCES distribution_channels(id) ON DELETE CASCADE,
    trigger_conditions  JSONB NOT NULL,  -- flexible event/attribute matching rules
    delay_seconds       INTEGER NOT NULL DEFAULT 0,
    max_displays        INTEGER NOT NULL DEFAULT 1,
    cooldown_hours      INTEGER NOT NULL DEFAULT 24,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5. Responses: Relational Envelope + JSONB Answers

The critical design decision: each response has a relational record for metadata (used for filtering, aggregation, and compliance), and a JSONB column containing the complete answer payload.

```sql
-- ============================================================
-- SURVEY RESPONSES — relational metadata + JSONB answer payload
-- ============================================================

CREATE TABLE survey_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    survey_version      INTEGER NOT NULL,  -- which published version was active
    channel_id          UUID REFERENCES distribution_channels(id) ON DELETE SET NULL,
    respondent_id       UUID REFERENCES panel_respondents(id) ON DELETE SET NULL,
    external_user_id    VARCHAR(255),

    -- Relational metadata (indexed, filtered, aggregated)
    status              VARCHAR(30) NOT NULL DEFAULT 'in_progress'
                        CHECK (status IN ('in_progress', 'completed', 'abandoned',
                                          'screened_out', 'quota_full', 'disqualified')),
    language            VARCHAR(10) NOT NULL DEFAULT 'en',
    ip_address          INET,
    user_agent          TEXT,
    geo_country         VARCHAR(3),
    geo_region          VARCHAR(100),
    duration_seconds    INTEGER,
    is_test             BOOLEAN NOT NULL DEFAULT FALSE,
    is_anonymized       BOOLEAN NOT NULL DEFAULT FALSE,
    weight              DECIMAL(10,6) DEFAULT 1.0,

    -- ============================================================
    -- THE JSONB ANSWER PAYLOAD
    -- Contains all answers keyed by question ID.
    -- Structure varies by question type.
    -- ============================================================
    answers             JSONB NOT NULL DEFAULT '{}'::jsonb,

    -- Page-level timing data
    page_timings        JSONB DEFAULT '[]'::jsonb,

    started_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE survey_responses_2026_q1 PARTITION OF survey_responses
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE survey_responses_2026_q2 PARTITION OF survey_responses
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE survey_responses_2026_q3 PARTITION OF survey_responses
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE survey_responses_2026_q4 PARTITION OF survey_responses
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_responses_survey ON survey_responses (survey_id);
CREATE INDEX idx_responses_status ON survey_responses (status);
CREATE INDEX idx_responses_respondent ON survey_responses (respondent_id);
CREATE INDEX idx_responses_created ON survey_responses (created_at);
CREATE INDEX idx_responses_country ON survey_responses (geo_country);
CREATE INDEX idx_responses_completed ON survey_responses (completed_at)
    WHERE completed_at IS NOT NULL;

-- GIN index for querying within the answers JSONB
CREATE INDEX idx_responses_answers ON survey_responses USING GIN (answers jsonb_path_ops);
```

#### The Answer Payload Schema

```jsonc
// survey_responses.answers JSONB — keyed by question ID
{
    "q-uuid-1": {
        "type": "single_choice",
        "selectedOptionId": "opt-2",
        "selectedValue": "25-34",
        "durationMs": 2100
    },
    "q-uuid-2": {
        "type": "likert",
        "selectedOptionId": "opt-14",
        "numericValue": 5,
        "durationMs": 1800
    },
    "q-uuid-3": {
        "type": "likert_matrix",
        "rows": {
            "row-1": {"selectedOptionId": "col-4", "numericValue": 4},
            "row-2": {"selectedOptionId": "col-5", "numericValue": 5},
            "row-3": {"selectedOptionId": "col-3", "numericValue": 3},
            "row-4": {"selectedOptionId": "col-5", "numericValue": 5}
        },
        "durationMs": 8700
    },
    "q-uuid-4": {
        "type": "net_promoter_score",
        "numericValue": 9,
        "durationMs": 1500
    },
    "q-uuid-5": {
        "type": "text_long",
        "textValue": "The product is generally good but the onboarding process was confusing and I had to contact support twice.",
        "durationMs": 15400
    },
    "q-uuid-6": {
        "type": "ranking",
        "rankedOptions": [
            {"optionId": "rank-5", "rank": 1},
            {"optionId": "rank-1", "rank": 2},
            {"optionId": "rank-3", "rank": 3},
            {"optionId": "rank-2", "rank": 4},
            {"optionId": "rank-4", "rank": 5}
        ],
        "durationMs": 6100
    },
    "q-uuid-7": {
        "type": "slider",
        "numericValue": 25,
        "durationMs": 3200
    },
    // Conjoint responses (when applicable)
    "conjoint-task-1": {
        "type": "conjoint_choice",
        "taskId": "task-1",
        "chosenProfileId": "prof-2",
        "responseTimeMs": 4200
    },
    // MaxDiff responses (when applicable)
    "maxdiff-set-1": {
        "type": "maxdiff_choice",
        "setId": "set-1",
        "bestItemId": "item-3",
        "worstItemId": "item-1",
        "responseTimeMs": 3800
    }
}
```

#### Querying JSONB Answers Efficiently

```sql
-- Extract NPS scores for a survey
SELECT
    id,
    (answers -> 'q-uuid-4' ->> 'numericValue')::INTEGER AS nps_score,
    completed_at
FROM survey_responses
WHERE survey_id = 'survey-uuid'
  AND status = 'completed'
  AND answers ? 'q-uuid-4';

-- Cross-tabulation: age range vs satisfaction
SELECT
    answers -> 'q-uuid-1' ->> 'selectedValue' AS age_range,
    (answers -> 'q-uuid-2' ->> 'numericValue')::INTEGER AS satisfaction,
    COUNT(*) AS count
FROM survey_responses
WHERE survey_id = 'survey-uuid'
  AND status = 'completed'
GROUP BY age_range, satisfaction
ORDER BY age_range, satisfaction;

-- NPS calculation (promoters - detractors / total * 100)
SELECT
    COUNT(*) FILTER (WHERE (answers -> 'q-uuid-4' ->> 'numericValue')::INT >= 9) AS promoters,
    COUNT(*) FILTER (WHERE (answers -> 'q-uuid-4' ->> 'numericValue')::INT BETWEEN 7 AND 8) AS passives,
    COUNT(*) FILTER (WHERE (answers -> 'q-uuid-4' ->> 'numericValue')::INT <= 6) AS detractors,
    ROUND(
        (COUNT(*) FILTER (WHERE (answers -> 'q-uuid-4' ->> 'numericValue')::INT >= 9)::DECIMAL -
         COUNT(*) FILTER (WHERE (answers -> 'q-uuid-4' ->> 'numericValue')::INT <= 6)::DECIMAL)
        / COUNT(*)::DECIMAL * 100, 1
    ) AS nps_score
FROM survey_responses
WHERE survey_id = 'survey-uuid'
  AND status = 'completed'
  AND answers ? 'q-uuid-4';

-- Find responses containing specific text themes
SELECT id, answers -> 'q-uuid-5' ->> 'textValue' AS feedback
FROM survey_responses
WHERE survey_id = 'survey-uuid'
  AND status = 'completed'
  AND answers -> 'q-uuid-5' ->> 'textValue' ILIKE '%onboarding%';

-- Use jsonpath for more complex queries (PostgreSQL 12+)
SELECT id, jsonb_path_query(answers, '$.*.numericValue') AS all_numeric_values
FROM survey_responses
WHERE survey_id = 'survey-uuid'
  AND status = 'completed';
```

### 6. Panel Management (Relational + JSONB Custom Attributes)

```sql
-- ============================================================
-- PANELS (relational)
-- ============================================================

CREATE TABLE panels (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    panel_type          VARCHAR(30) NOT NULL DEFAULT 'internal'
                        CHECK (panel_type IN ('internal', 'external', 'mixed')),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    member_count        INTEGER NOT NULL DEFAULT 0,

    -- Schema definition for custom attributes (used for validation)
    custom_attribute_schema JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- Example: [{"name": "industry", "type": "single_choice", "options": ["Tech", "Finance", "Health"]}]

    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_panels_org ON panels (organization_id);

-- ============================================================
-- PANEL RESPONDENTS — relational core + JSONB custom attributes
-- ============================================================

CREATE TABLE panel_respondents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    panel_id            UUID NOT NULL REFERENCES panels(id) ON DELETE CASCADE,

    -- Standard demographics (relational for indexing and filtering)
    email               VARCHAR(255),
    phone               VARCHAR(50),
    external_id         VARCHAR(255),
    first_name          VARCHAR(255),
    last_name           VARCHAR(255),
    gender              VARCHAR(20),
    birth_year          INTEGER,
    country             VARCHAR(3),
    region              VARCHAR(100),
    language            VARCHAR(10),

    -- Consent tracking
    consent_given       BOOLEAN NOT NULL DEFAULT FALSE,
    consent_given_at    TIMESTAMPTZ,
    opt_out             BOOLEAN NOT NULL DEFAULT FALSE,
    opt_out_at          TIMESTAMPTZ,

    -- Engagement metrics
    total_surveys_sent  INTEGER NOT NULL DEFAULT 0,
    total_surveys_completed INTEGER NOT NULL DEFAULT 0,
    last_survey_at      TIMESTAMPTZ,
    quality_score       DECIMAL(5,2),

    -- ============================================================
    -- CUSTOM ATTRIBUTES (JSONB)
    -- Panel-specific demographics and segmentation data.
    -- Validated against panels.custom_attribute_schema.
    -- ============================================================
    custom_attributes   JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- Example: {"industry": "Tech", "company_size": "50-200", "years_experience": 12}

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_panel_respondents_panel ON panel_respondents (panel_id);
CREATE INDEX idx_panel_respondents_email ON panel_respondents (email);
CREATE INDEX idx_panel_respondents_country ON panel_respondents (country);
CREATE INDEX idx_panel_respondents_quality ON panel_respondents (quality_score DESC)
    WHERE quality_score IS NOT NULL;

-- GIN index for querying custom attributes
CREATE INDEX idx_panel_respondents_custom ON panel_respondents USING GIN (custom_attributes jsonb_path_ops);

-- ============================================================
-- SURVEY QUOTAS (relational + JSONB conditions)
-- ============================================================

CREATE TABLE survey_quotas (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    target_count        INTEGER NOT NULL,
    current_count       INTEGER NOT NULL DEFAULT 0,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    action_on_full      VARCHAR(30) NOT NULL DEFAULT 'screen_out',

    -- Quota conditions as JSONB (flexible matching rules)
    conditions          JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- Example: [{"questionId": "q-uuid-1", "operator": "equals", "value": "25-34"}]

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- LONGITUDINAL STUDIES (relational)
-- ============================================================

CREATE TABLE longitudinal_studies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    panel_id            UUID REFERENCES panels(id),
    study_config        JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE study_waves (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    study_id            UUID NOT NULL REFERENCES longitudinal_studies(id) ON DELETE CASCADE,
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    wave_number         INTEGER NOT NULL,
    wave_label          VARCHAR(100),
    scheduled_at        TIMESTAMPTZ,
    launched_at         TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,
    wave_config         JSONB DEFAULT '{}'::jsonb,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (study_id, wave_number)
);
```

### 7. Experiments (Relational + JSONB Config)

```sql
-- ============================================================
-- EXPERIMENTS
-- ============================================================

CREATE TABLE experiments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    hypothesis          TEXT,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft', 'running', 'completed', 'cancelled')),
    allocation_method   VARCHAR(30) NOT NULL DEFAULT 'random',

    -- Experiment design config (stratification rules, blocking factors)
    experiment_config   JSONB NOT NULL DEFAULT '{}'::jsonb,

    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE experiment_groups (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id       UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    group_name          VARCHAR(100) NOT NULL,
    group_type          VARCHAR(30) NOT NULL
                        CHECK (group_type IN ('control', 'treatment')),
    allocation_pct      DECIMAL(5,2) NOT NULL,
    participant_count   INTEGER NOT NULL DEFAULT 0,
    group_config        JSONB DEFAULT '{}'::jsonb,  -- treatment-specific variations
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE experiment_assignments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id       UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    group_id            UUID NOT NULL REFERENCES experiment_groups(id) ON DELETE CASCADE,
    response_id         UUID NOT NULL REFERENCES survey_responses(id) ON DELETE CASCADE,
    assigned_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (experiment_id, response_id)
);
```

### 8. Analysis & Reporting (JSONB Results)

```sql
-- ============================================================
-- TEXT ANALYSIS
-- ============================================================

CREATE TABLE text_analysis_jobs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    question_id         VARCHAR(100) NOT NULL,  -- references question ID within the JSONB design
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
    model_version       VARCHAR(100),
    response_count      INTEGER NOT NULL DEFAULT 0,
    results             JSONB,  -- themes, sentiments, word frequencies
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- STATISTICAL ANALYSIS RUNS
-- ============================================================

CREATE TABLE analysis_runs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    analysis_type       VARCHAR(50) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',

    -- Analysis configuration (questions, filters, weights)
    parameters          JSONB NOT NULL,

    -- Complete results as JSONB (tables, statistics, visualisation data)
    results             JSONB,

    created_by          UUID NOT NULL REFERENCES users(id),
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- AI-GENERATED REPORTS
-- ============================================================

CREATE TABLE generated_reports (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    report_type         VARCHAR(30) NOT NULL,
    title               VARCHAR(500) NOT NULL,
    content_markdown    TEXT NOT NULL,
    report_metadata     JSONB DEFAULT '{}'::jsonb,
    model_used          VARCHAR(100),
    created_by          UUID NOT NULL REFERENCES users(id),
    published_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 9. Integrations, Webhooks, and Audit Log

```sql
-- ============================================================
-- INTEGRATIONS (relational + JSONB config)
-- ============================================================

CREATE TABLE integrations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    integration_type    VARCHAR(50) NOT NULL,
    name                VARCHAR(255) NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    config_encrypted    BYTEA,  -- encrypted configuration
    integration_state   JSONB DEFAULT '{}'::jsonb,  -- sync cursor, token metadata
    last_sync_at        TIMESTAMPTZ,
    last_error          TEXT,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- WEBHOOKS (relational)
-- ============================================================

CREATE TABLE webhooks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    survey_id           UUID REFERENCES surveys(id) ON DELETE CASCADE,
    url                 TEXT NOT NULL,
    secret              VARCHAR(255) NOT NULL,
    events              TEXT[] NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    last_triggered_at   TIMESTAMPTZ,
    failure_count       INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- AUDIT LOG (relational + JSONB details)
-- ============================================================

CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    organization_id     UUID NOT NULL,
    user_id             UUID,
    action              VARCHAR(100) NOT NULL,
    entity_type         VARCHAR(50) NOT NULL,
    entity_id           UUID,
    details             JSONB NOT NULL DEFAULT '{}'::jsonb,  -- old/new values, context
    ip_address          INET,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_log_2026_q1 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE audit_log_2026_q2 PARTITION OF audit_log
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE audit_log_2026_q3 PARTITION OF audit_log
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE audit_log_2026_q4 PARTITION OF audit_log
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_audit_log_org ON audit_log (organization_id, created_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log (entity_type, entity_id);

-- ============================================================
-- CONSENT RECORDS (relational for compliance)
-- ============================================================

CREATE TABLE consent_records (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    respondent_id       UUID REFERENCES panel_respondents(id) ON DELETE SET NULL,
    response_id         UUID REFERENCES survey_responses(id) ON DELETE SET NULL,
    consent_type        VARCHAR(50) NOT NULL,
    consent_given       BOOLEAN NOT NULL,
    consent_text        TEXT NOT NULL,
    ip_address          INET,
    legal_basis         VARCHAR(50) NOT NULL DEFAULT 'consent',
    given_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    withdrawn_at        TIMESTAMPTZ
);

CREATE INDEX idx_consent_respondent ON consent_records (respondent_id);
```

---

## JSONB Validation Strategy

Since JSONB columns do not enforce schema at the database level, validation must happen at the application layer. The recommended approach:

```typescript
// Zod schema for the survey design document (version 1)
import { z } from 'zod';

const AnswerOptionSchema = z.object({
    id: z.string().uuid(),
    label: z.string().min(1),
    value: z.string(),
    sortOrder: z.number().int().min(0),
    score: z.number().nullable().optional(),
    imageUrl: z.string().url().nullable().optional(),
    isExclusive: z.boolean().optional().default(false),
    isAnchored: z.boolean().optional().default(false),
});

const QuestionConfigSchema = z.record(z.unknown());  // type-specific, validated per question type

const QuestionSchema = z.object({
    id: z.string().uuid(),
    type: z.enum(['single_choice', 'multiple_choice', 'dropdown', 'likert',
                   'likert_matrix', 'nps', 'text_short', 'text_long',
                   'ranking', 'rating', 'slider', 'date', 'file_upload',
                   'matrix_single', 'matrix_multiple', 'conjoint_profile',
                   'maxdiff_set', 'semantic_differential', 'constant_sum',
                   'image_choice', 'net_promoter_score']),
    code: z.string().max(50).optional(),
    title: z.string().min(1),
    description: z.string().nullable().optional(),
    isRequired: z.boolean().default(false),
    sortOrder: z.number().int().min(0),
    randomizeOptions: z.boolean().default(false),
    allowOther: z.boolean().default(false),
    otherLabel: z.string().optional(),
    options: z.array(AnswerOptionSchema).optional(),
    rows: z.array(z.object({
        id: z.string().uuid(),
        label: z.string(),
        sortOrder: z.number().int().min(0),
    })).optional(),
    validation: z.object({
        minSelections: z.number().int().optional(),
        maxSelections: z.number().int().optional(),
        minValue: z.number().optional(),
        maxValue: z.number().optional(),
        regex: z.string().optional(),
        message: z.string().optional(),
    }).nullable().optional(),
    config: QuestionConfigSchema.optional().default({}),
});

const SurveyDesignSchema = z.object({
    schemaVersion: z.literal(1),
    pages: z.array(z.object({
        id: z.string().uuid(),
        title: z.string().optional(),
        description: z.string().optional(),
        sortOrder: z.number().int().min(0),
        randomize: z.boolean().default(false),
        visibilityCondition: z.unknown().nullable().optional(),
        questions: z.array(QuestionSchema),
    })),
    logicRules: z.array(z.object({
        id: z.string().uuid(),
        sourceQuestionId: z.string().uuid(),
        ruleType: z.enum(['skip_to', 'show_question', 'hide_question',
                          'show_page', 'hide_page', 'end_survey']),
        targetQuestionId: z.string().uuid().optional(),
        targetPageId: z.string().uuid().optional(),
        conditions: z.array(z.object({
            operator: z.string(),
            value: z.string(),
            connector: z.enum(['AND', 'OR']).default('AND'),
        })),
    })).default([]),
    translations: z.record(z.unknown()).default({}),
    conjointDesign: z.unknown().nullable().default(null),
    maxdiffDesign: z.unknown().nullable().default(null),
    scoring: z.object({
        enabled: z.boolean().default(false),
        rules: z.array(z.unknown()).default([]),
    }).default({ enabled: false, rules: [] }),
});
```

---

## Pros and Cons

### Pros

1. **Natural fit for variable question structures.** Survey questions are inherently polymorphic — a Likert scale, a matrix question, a conjoint task, and a slider each have completely different configuration needs. JSONB handles this variability without table proliferation or nullable columns. Adding a new question type is a code change, not a database migration.

2. **Dramatically simpler schema.** Compared to the fully normalized model (Suggestion 1), this model has roughly half the tables. The entire survey design is a single JSONB document, eliminating the join-heavy queries needed to reconstruct a survey for the builder UI.

3. **Fast survey loading.** Loading a complete survey for editing or rendering is a single row fetch: `SELECT design FROM surveys WHERE id = $1`. No multi-table join needed. This is critical for the respondent-facing survey renderer, which must be fast.

4. **Schema evolution without migrations.** New question type fields, new logic rule operators, and new configuration options can be added by updating the Zod validation schema and the application code. No `ALTER TABLE` migrations on production databases.

5. **SurveyJS compatibility.** The JSONB design document format is architecturally identical to SurveyJS's JSON schema approach. If the platform decides to integrate SurveyJS for the builder UI, the design document can serve as the SurveyJS model definition with minimal transformation.

6. **Relational integrity where it matters.** Organizations, users, panels, distribution channels, and response-level metadata are all fully relational with proper foreign keys. The JSONB is used only where variability genuinely exists.

7. **Efficient analytics on response metadata.** Cross-tabulation by geography, channel, status, or respondent demographics uses fast indexed relational columns. JSONB is only queried when drilling into specific answer values.

### Cons

1. **No referential integrity within JSONB.** The database cannot enforce that a question ID referenced in a logic rule actually exists in the pages array. A question ID in an answer payload cannot be foreign-keyed to the design document. These constraints must be enforced entirely in application code.

2. **Complex JSONB queries for analytics.** While simple aggregations work well, complex statistical analysis on JSONB answer payloads (e.g., weighted cross-tabulation across multiple questions) requires verbose jsonpath expressions or extraction CTEs. Performance degrades compared to purpose-built relational answer tables.

3. **JSONB indexing limitations.** GIN indexes on JSONB support containment (`@>`) and existence (`?`) operators efficiently, but range queries on nested numeric values (e.g., "find all responses where Q4 NPS score > 8") do not benefit from GIN indexes. Expression indexes can help but must be created per-question.

4. **Harder to enforce data quality at the database level.** CHECK constraints, NOT NULL, and UNIQUE constraints on JSONB fields are not supported natively. All validation is application-layer, meaning bugs in the validation code can allow invalid data into the database.

5. **Reporting and BI tool integration is harder.** External BI tools (Tableau, Metabase, Looker) expect flat relational tables. JSONB answer payloads must be "exploded" into flat views or materialised tables for BI consumption. This adds ETL complexity.

6. **Large JSONB documents affect TOAST performance.** A survey with 200 questions and complex conjoint designs can produce design documents exceeding 100KB. PostgreSQL's TOAST mechanism handles this transparently, but very large documents increase I/O for every read.

---

## Migration and Scaling Considerations

### Phase 1: Single PostgreSQL Instance (MVP to 10M responses)

- Single PostgreSQL 17 instance with JSONB columns
- Zod validation in the application layer with comprehensive test coverage
- Redis for session cache and real-time dashboard counters
- pg_cron for automated partition creation on response tables
- Materialized views for common dashboard aggregations

### Phase 2: Analytical Sidecar (10M to 500M responses)

- **Critical step:** Create a response flattening pipeline that extracts JSONB answer payloads into a columnar analytical store (ClickHouse or DuckDB). This enables fast cross-tabulation and statistical analysis without hitting JSONB query performance limits.
- Read replicas for dashboard queries
- Consider adding expression indexes for frequently queried JSONB paths:
  ```sql
  -- Expression index for NPS question
  CREATE INDEX idx_nps_score ON survey_responses
      (((answers -> 'q-uuid-4' ->> 'numericValue')::INTEGER))
      WHERE answers ? 'q-uuid-4';
  ```
- Implement JSONB design document versioning: when the Zod schema changes, add an upcasting function that transforms old documents to the new format on read.

### Phase 3: Multi-Region and Scale-Out (500M+ responses)

- Shard by organization using Citus or application-level routing
- Multi-region PostgreSQL with data residency enforcement
- Archive old response partitions to Parquet on S3
- Full-text search index (Elasticsearch/Meilisearch) for open-text response searching
- CDC pipeline (Debezium) for streaming response data to analytical warehouse

### JSONB Schema Migration Strategy

Unlike relational columns, JSONB schema changes do not require database migrations. However, they do require a versioning strategy:

1. Every JSONB document includes a `schemaVersion` field
2. The application reads the version and applies any necessary transformations
3. Lazy migration: old documents are transformed to the latest format on read and optionally written back
4. Eager migration: a background job scans and updates all documents to the latest format
5. Zod schemas for each version are maintained in the codebase for backward compatibility
